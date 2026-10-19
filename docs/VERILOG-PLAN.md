# SPARCstation 20 core: the plan

A SPARCstation 20 (sun4m) core for MiSTer, written from scratch in
SystemVerilog under GPL-2.0-or-later. Everything in it is built from
public specifications, GPL references and our own test suite.

This file holds the ground rules, the compatibility contracts, the
architecture, the phases, the behaviour requirements and the work queue.
The section numbers are cited from the RTL; keep them.

## 1. Ground rules

### 1.1 Clean room

1. **Write RTL only from allowed sources.** Never open, quote or
   translate the source of another SPARCstation core that does not carry
   a GPL-compatible license (by hand or by tool).
2. **Allowed sources:** chip datasheets and Sun manuals (§8); QEMU
   (GPL-2: `hw/sparc/sun4m*.c`, `hw/scsi/esp.c`, `hw/net/lance.c`,
   `hw/display/tcx.c`, `hw/char/escc.c`, `hw/dma/sparc32_dma.c`,
   `hw/timer/slavio_timer.c`, `hw/intc/slavio_intctl.c`,
   `hw/misc/slavio_misc.c`, `hw/sparc/sun4m_iommu.c`,
   `target/sparc/`); OpenBIOS (in `bios/`); the Linux and NetBSD
   drivers; our own docs, tests and tools in this repository; and GPL- or
   MIT-compatible RTL from other cores (§5).
3. Sun PROM images and anything that reproduces their code (listings,
   decompiled Forth, detokenized FCode) never go into git. They live in
   the gitignored `scratch/private/`.

### 1.2 Compatibility contracts (keep these unchanged)

- **Main (`support/sun`, upstream in Main_MiSTer #1344, plus the CD-audio
  branch #1345).** The core speaks this `hps_io` protocol:
  - ioctl index 0: the boot PROM (`boot0.rom`, `boot.rom`, or the OSD's
    `F0` Load ROM for one session);
  - ioctl index 64: the ID PROM seed;
  - SD slots: disk 0, disk 1, CD, and the NVRAM image (8 KB);
    `sd_blk_cnt` chunks of up to 32 blocks;
  - CD: 512-byte blocks at power-up, MODE SELECT block size; the
    CD-audio windows (TOC at block `0xC0000000`, frame f at
    `0x80000000 + 5f`);
  - Ethernet: the `SSETH002` mailbox in DDR3 at ARM physical
    `0x1FF00000`, with an 8-slot TX ring and a 16-slot RX ring.

  The RTL headers of `rtl/sun4m/scsi_targets.sv`, `rtl/mister/eth_hps.sv`
  and `rtl/mister/nvram_sd.sv` give the details.
- **OpenBIOS (`bios/`, `CONFIG_TACUS`).** Keep the physical address map,
  device set and IDs, and the configuration words in the Aux 0 page
  (`rtl/obio/slavio_misc.sv`), so the OpenBIOS build boots unchanged.
- **The OSD**: the status bits stay fixed (`rtl/machine/ss20_osd.sv`), so
  users' `.CFG` files keep working.
- **Sun OBP 2.25 (525-1377-08)** must boot, its POST included. Its
  requirements are in
  [rom-disassembly/ss20-obp-2.25/](rom-disassembly/ss20-obp-2.25/).

### 1.3 Targets

| Measure | Goal |
|---|---|
| CPUs × clock | ≥ 2 × 55 MHz; 3-4 CPUs if they fit |
| Device use | ≤ 85-90 % ALMs with 2 CPUs |
| CPU suite (`tests/cpu`, SS20) | no FAIL; SKIP only for what the machine lacks |
| Solaris 8 from disk to login (OpenBIOS) | ≤ 117 s |
| NetBSD install CD to installer | ≤ 98 s (OpenBIOS), 90 s (OBP) |
| OSes | NetBSD 11, Solaris 8, NeXTSTEP 3.3 (both PROMs) |
| Sun POST | passes to the EMC/SMC test (ECC out of scope) |

## 2. Architecture

```
emu (SunSparcStation.sv, Template_MiSTer)
 └─ ss20_machine
     ├─ cpu[0..N-1]  SuperSPARC-compatible module: IU + FPU + SRMMU + I$/D$
     ├─ mbus         interconnect: arbiter, snoop broadcast, MID, bus errors
     ├─ memctl       → ddram_arb → DDR3; boot-PROM window; RAM size
     ├─ vram_sdram   → sdram64 → SDRAM: the TCX's frame buffer
     ├─ msi          MBus-to-SBus: IOMMU, SBus slot decode and time-outs
     │   ├─ dma2     espdma + ledma (+ bpp stub)
     │   ├─ esp      NCR 53C9x → scsi_targets → hps_io SD slots
     │   ├─ lance    Am7990 (Wish7990) → eth_hps → Main mailbox
     │   ├─ tcx      8-bit framebuffer, DAC, cursor → MiSTer video
     │   ├─ slot 3   CS4231 + APC + FCode PROM
     │   └─ slot e   "dbri-absent" FCode
     └─ obio         interrupt controller, counter/timers, system regs,
                     ESCC ×2 (kbd/mouse, ttya/b), M48T08 NVRAM/RTC
                     (+ nvram_sd), AUXIO, FDC stub, kbd_beep
```

Decisions (change them here if needed):

- **The CPU is a single-issue, in-order pipeline that looks like a
  SuperSPARC (TMS390Z50) to software.** The aim is the PSR, MMU control
  and ASI behaviour that OBP 2.25, Solaris, NetBSD and NeXTSTEP expect,
  not SuperSPARC's three-way superscalar core. The modules run in MBus
  mode with no MXCC or E-cache (this keeps the PROM and the OSes off
  their MXCC paths).
- **Caches:** physically tagged, write-through D-cache with snoop
  invalidation on MBus writes and DMA; main memory cached even where a
  PTE says C = 0 (E3). Write-back is out.
- **TLB:** the Viking's 64 fully associative entries (Viking doc §4.11:
  one CAM over the four mapping sizes, limited-history LRU, lock bits,
  the ASI 6 image), with the CAM built from RAM: the 36 compared bits
  (VPN 20 + context 16) in nine 4-bit slices, each a 16-row RAM of
  64-bit match vectors; a lookup ANDs the nine vectors; a fill writes 16
  rows (mapping sizes above 4 KB set their ignored slices to "match
  all"); used/lock bits and the ASI 6 image stay exact.
- **The interconnect is our own simple protocol** (request/grant, 36-bit
  PA, 32-byte line bursts, a snoop broadcast on writes), defined in
  `docs/arch/bus.md`. It does not model MBus signal by signal.
- **One clock domain** for the machine. Device timing that software
  depends on (§6) is modelled in that clock.
- **Language:** SystemVerilog that Quartus 17.0 and Verilator 5 both
  accept. Each block gets a unit testbench under `rtl/**/tb/`.

## 3. Phases

Each phase ended with an exit test; §9 has where each stands.

- **Phase 0, skeleton and tooling:** the `emu` top, `files.qip`,
  `.qsf`/`.sdc`, `scripts/build.sh`, `sim/sim_top.sv`; the bus spec
  (`docs/arch/bus.md`) and the address map (`docs/arch/memory-map.md`).
- **Phase 1, integer unit:** SPARC V8 IU, 8 windows, traps and RETT,
  delayed control transfers with annul, MUL/DIV, `ldstub`/`swap`,
  alternate-space accesses, error mode (A1 `t_dcti`, IU-1 `wdtest`).
- **Phase 2, FPU:** the V8 FPops in single and double precision (quad
  traps as unimplemented), FSR/FQ, deferred traps, IEEE underflow and
  inexact exactly as the POST and `t_fpu` check. QEMU's
  `target/sparc/fop_helper.c` is the behavioural reference.
- **Phase 3, SRMMU and caches:** the reference MMU, the fault registers,
  MCNTL with SuperSPARC's IDs, TLB flush/probe; the I/D caches with the
  diagnostic ASIs (0x0c-0x0f) and flash clears (0x36/0x37); the
  diagnostic TLB (ASI 6); the L2 TLB/PTP cache (MCNTL bit 6; an ASI
  5/6/7 write drops the TLBs, which NeXTSTEP needs); bus errors (A3).
- **Phase 4, memory, PROM and the HPS blocks:** `memctl` on DDR3
  through `ddram_arb`; the boot-PROM window at `0xF_F000_0000`, loaded by
  ioctl index 0; RAM size; RAM cleared on power-up, OSD reset or a new
  ROM; `nvram_sd`, `eth_hps`, `scsi_targets`.
- **Phase 5, sun4m system devices:** interrupt controller, counter/timers,
  system control/status, MID, AUXIO, MSI/EMC registers, SBus slot
  time-outs; ESCC with BREAK; M48T08. Exit: OpenBIOS `0 >` on ttya.
- **Phase 6, keyboard, mouse, TCX:** Sun Type 5 keyboard and the Mouse
  Systems mouse from hps_io's PS/2; TCX with DAC, cursor and the
  blitter/stippler; its FCode PROM (QEMU's, GPL-2). Exit: the screen
  console.
- **Phase 7, IOMMU, DMA2, SCSI:** IOMMU, espdma (DMA-5), ESP (ESP-7,
  ESP-8, ESP-9). Exit: NetBSD and Solaris boot from disk; the CD; CD
  audio.
- **Phase 8, Ethernet:** Am7990 (Wish7990, MIT) on `eth_hps`. Exit:
  `ethtest`, the board's network tests.
- **Phase 9, SMP:** the CPU's size first, then 2 CPUs: MID per module,
  per-CPU interrupts and timers, IPIs, atomics across CPUs, snoop
  invalidation, the arbiter enable, the OpenBIOS secondary start.
- **Phase 10, sound and the rest:** CS4231 + APC playback DMA written
  from the CS4231A datasheet and the `audiocs` drivers, at SBus slot 3
  with our FCode (`tools/fcode_cs4231.py`); FDC stub; slot-e FCode.
- **Phase 11, Sun OBP 2.25, POST, NeXTSTEP.**
- **Phase 12, performance, timing, release.**
- **Phase 13, video memory on SDRAM, a 24-bit frame buffer:** the core
  requires the SDRAM module (user, 2026-10-18). Step 1, the TCX's VRAM
  and scan-out on SDRAM, is done; step 2 is TCX's S24 mode (QEMU `tcx`
  depth 24; Solaris's driver): 1024x768 at 60 Hz is ~190 MB/s at 32 bits
  a pixel against ~200-260 MB/s for one 16-bit SDRAM, so design the
  layout (3 bytes a pixel, the control bits kept small or on chip) first.

## 4. Repository layout

See the README ("What is here").

## 5. Outside RTL we may use

Copies of the candidates below, with their licenses, are in the
gitignored `scratch/third-party-rtl/`.

| What | Source | License | Note |
|---|---|---|---|
| Am7990 LANCE | Wish7990 (MelkhiorVintageComputing) | MIT | vendored as `rtl/sun4m/wish7990_*` |
| NCR 53C96 ESP | `MacQuadra800_MiSTer/rtl/ncr53c96.sv` | GPL-2 (ours) | the base of `esp.sv` |
| SDRAM controller | `hp9000-715_MiSTer/rtl/mister/sdram64.sv` | GPL-2+ (ours) | `rtl/mister/sdram64.sv` |
| NCR 53C90 ESP | `NeXT_MiSTer/rtl/next/next_scsi.sv` | GPL-2 | second reference |
| Z8530 SCC | `MacQuadra800_MiSTer/rtl/scc.v` (minimigmac lineage) | GPL-2 | reference |
| SPARC V8 + SRMMU reference | GRLIB LEON3 (Frontgrade Gaisler) | GPL edition | read as reference |

**License decision (user, 2026-10-18): GPL-3 parts are allowed.** Our
files stay GPL-2.0-or-later and `sys/` is GPL-2+, so a build that
includes a GPL-3 part (e.g. Sorgelig's `sdram.sv`) is distributed as
GPL-3 as a whole; keep each outside file's license header. OpenBIOS is a
separate firmware image, so it does not affect the core's license.

## 6. Behaviour requirements

Facts about the hardware and the software that the core must keep. Each
has a test in `tests/cpu` or a unit bench.
The digests in `docs/arch/` use further IDs of the same kind (IU-n,
FPU-n, MMU-n, C-n, SMP-n, CFG-n) as labels for facts they record.

| ID | Requirement | Test |
|---|---|---|
| A1 | A JMPL/CALL decoded behind a stalled RETT links the right pc | `t_dcti` |
| IU-1 | Error mode → watchdog reset, SFSR EM | `wdtest` |
| MMU-2 | 64-entry diagnostic TLB image (ASI 6) on every CPU | `t_mmu_diag` |
| A4 | Cache diagnostic ASIs 0x0c-0x0f, flash clear 0x36/0x37 | `t_cache_diag` |
| E3 | Main memory cached where a PTE says C = 0 (the walker's own writes coherent) | `t_cache_force` |
| L2 | L2 TLB safe to switch at run time; ASI 5/6/7 writes drop the TLBs | `t_mmu_l2` |
| A3/B1 | Bus errors: empty SBus slots time out; SFSR FT 5, TO/BE | `t_buserr` |
| INT-2 | System interrupt mask bits 26:23 read 0 | POST |
| TMR-4 | User-timer config readback | POST |
| TMR-8 | User-timer mode: RUN resets to 1 | NeXTSTEP |
| DEC-3 | Registers no device answers read 0 | POST |
| DEC-4 | Slot e answers with a named FCode node | NeXTSTEP |
| DMA-5 | ESP DMA: the partial last word | `scsitest` t_scsi_sense1 |
| ESP-7 | A select's interrupt is held ~1 ms | NeXTSTEP |
| ESP-8 | MESSAGE ACCEPTED → Bus Service while the target wants more; after each MESSAGE IN byte (TI or ICCS) the phase reads MESSAGE IN until MESSAGE ACCEPTED (ACK held: Solaris under OBP) | `scsitest` t_scsi_sdtr1, `tb_espdma` |
| ESP-9 | A non-DMA TI in an OUT phase waits for its first FIFO byte (written after the command) | NeXTSTEP, `tb_espdma` |
| SCSI-TAG | Two-byte (queue tag) messages accepted | `scsitest` t_scsi_tag1 |
| CD-512 | The CD powers up at 512-byte blocks, follows MODE SELECT | Solaris |
| LAN-2 | Oversize received frames dropped | `ethtest` t_eth_oversize |
| C3 | Received frames paced at 10 Mb/s | `net.sh` large pings |
| KBD | Reset reply lists held keys; it answers within the real keyboard's delay (NetBSD's `kbd0: reset failed`) | `tb_sun_kbd` |
| BRK | BREAK on ttya reaches the ESCC (RR0 bit 7, ext/status interrupt) | `brktest` |
| RST | RS/WD status after a software or watchdog reset (OpenBIOS clears RAM) | NeXTSTEP fsck reboot |
| ID | IDPROM from the host (ioctl 64), stamped into a blank NVRAM | `hwtest.sh idprom-obp` |

## 7. Test strategy

- **Unit benches** per block (Verilator or iverilog), in `rtl/**/tb/`;
  `rtl/tb/run.sh` lints and runs them all.
- **The CPU suite** (`tests/cpu`) in Verilator (`sim/iu`, `sim/cpu`)
  against QEMU's expected logs, and on the board. QEMU has known V8
  deviations ([tests/cpu/README.md](../tests/cpu/README.md)).
- **Whole-machine simulation** (`sim/`): OpenBIOS to `ok`, `scsitest`,
  `ethtest`, and the real Sun PROM's POST (`sim/run-obp.sh`).
- **Board regression** (`scripts/hwtest.sh 20 all`, `--obp`).

## 8. Reference documents

In `scratch/reference/` (PDFs, with text extractions), indexed with
their URLs in [references/INDEX.md](references/INDEX.md): the SPARC V8
architecture manual, SuperSPARC (`SuperSparc.pdf`, `SuperSPARC2.pdf`,
`supersparcwhitepaper.pdf`, STP1021UG), the *Sun-4M System
Architecture*, the SPARCstation 20 field service manual, microSPARC-II
and TurboSPARC manuals (for the SS5 and the SRMMU), HyperSPARC, LEON2;
the NCR 53C90A/ESP100A and 53C9x, Am7990, Z85C30/85230, M48T08 and
CS4231A datasheets; the MBus specification; the TCX in the Sun
framebuffer docs and QEMU. Each RTL header names the sections it was
written from.

## 9. Status (2026-10-19)

The core runs on the board with two SuperSPARC-compatible CPUs at
55 MHz (~36.3k ALMs, 86 %, every clock met on the release seed). Phases
0-11 are done; 12 is under way; 13's first step is done.

| Phase | State |
|---|---|
| 0-3 CPU | IU, FPU (checked against an exact IEEE 754 model, `tests/fpu/`), SRMMU (64-entry RAM-CAM TLB), I/D caches, `cpu_module`. The suite in `sim/cpu` passes everything QEMU passes, and 15 tests QEMU fails or skips. |
| 4 memory, PROM, HPS | `ss20_memctl` (`rtl/pkg/ss20_ddr_pkg.sv`: bits 28:20 inverted, clear of the scaler's buffer; SIMM slot 7 a 16 MB alias), `ss20_loader`, `ss20_iobridge`, `ss20_uart`, `ss20_mbus`; `nvram_sd`, `eth_hps`, `scsi_targets`. |
| 5 system devices | `rtl/obio/*`, `ss20_msi` (IOMMU, 16-entry TLB, IMPL/VER 0x13), `fdc82077`, the MACIO ID, the OpenBIOS configuration words in Aux 0. |
| 6 keyboard, mouse, TCX | `sun_kbd` (Type 5, the Right Alt chords), `sun_mouse`, `tcx` (+ `tcx_scan`, `tcx_dcram`, `tcx_fcode`). |
| 7 IOMMU, DMA2, SCSI | `dma2`, `esp`; NetBSD 11 and Solaris 8 from disk under OpenBIOS; the NetBSD install CD; CD audio. Solaris 8 needs the OpenBIOS from `bios/` with `/memory reg` per 64 MB SIMM slot. |
| 8 Ethernet | `lance` on `eth_hps`; `ethtest` 9/9; network under NetBSD, Solaris, and OBP's `test net`. |
| 9 SMP | Two CPUs; MCNTL bits 3:2 = the CPU index (what the OpenBIOS and the suite park by); MCNTL.SE gates snooping of other CPUs' writes (DVMA always snooped). `stress` 30 min under both PROMs; Solaris under OBP brings both CPUs on-line. Not yet: NetBSD with an MP kernel. |
| 10 sound | `cs4231` + APC, `kbd_beep`, the slot-E `dbri-absent` FCode; audio tests pass under NetBSD, Solaris and OBP. |
| 11 Sun OBP 2.25 | The POST passes on both CPUs; Solaris boots under OBP; `boot-mode` PROM reads at the EPROM's pace (`ss20_memctl` PROM_WAIT). Not yet: NeXTSTEP on the board, `hwtest all --obp`. |
| 12 timing, release, speed | UnixBench ([benchmarks/unixbench.md](benchmarks/unixbench.md)); the profile and plan in [benchmarks/profile-20261014.md](benchmarks/profile-20261014.md). Test build in `releases/` (20261019). Open: HDMI PLL slack between seeds (the path is inside `ascal`), release notes. |
| 13 VRAM on SDRAM | Step 1 done (`vram_sdram` on `sdram64`; partial writes are read-merge-write, DQM is not reliable on the board). Step 2 (S24) not started. |

**The serial port:** the ESCC runs ttya at the baud rate the OS programs
(WR12/13 from PCLK); the bridge to the HPS UART is `ss20_uart`.

## 10. Work queue (from 2026-10-19)

In this order (the user's). Each item: unit benches, the `sim/iu`
regression and `perfbench` ticks for anything in the CPU, the 2-CPU CPU
suite in the simulation, a full build, then the board; commit before
every build.

1. **The board matrix on the current build:** cpu, NetBSD, Solaris
   under both PROMs (done 2026-10-19); still to run: the CD, net, audio,
   NeXTSTEP, the POST.
2. **Phase 13 step 1 follow-up:** the HDMI seed (sweep), and the VRAM's
   speed (a closed-page controller, a read-merge-write per partial
   write): measure the console and X; open pages / write bursts if it is
   slower than DDR3 was.
3. **Instruction fetch** (fetch was 12-28 % of cycles under NetBSD): the
   I-cache miss latency, forward-branch mispredicts, the bubble after a
   redirect; perfbench and the NetBSD profile before and after. Also the
   Sun PROM's CPU clock estimate (psrinfo says 40 MHz under OBP 2.25): the
   PROM times a delay loop against the counter/timer assuming a
   SuperSPARC's cycles per iteration. Find the loop in the simulation
   (+pcsample under OBP), count our cycles per iteration and remove what
   costs more than the chip's: the goal is ~55 MHz from the real speed,
   not from a changed timer.
4. **An L2 cache** between ss20_mbus and ss20_memctl (~128 KB of the free
   M10K, ~2k ALMs): fork/exec, file and shell tests.
5. **The Sun PROM's start-up memory test.** Done: an empty NVRAM gets the
   Sun OBP's defaults (`rtl/pkg/obp_nvram_pkg.sv` from
   `tools/obp_nvram.py --sv`: `diag-switch? false`, `boot-device disk`,
   `sbus-probe-list fe0123`), so the PROM boots the disk at once. Left:
   the test's speed in diag mode (the boot-mode PROM pace, the caches off
   in boot mode), as long as the POST still passes.
   Seen, not yet looked at: the TOD is not the MiSTer's time (OpenBIOS
   warns "Time-of-day chip had incorrect date"); under OpenBIOS the
   screen's white is pale yellow, and Solaris's text console after X is
   white on white (palette).
6. As room is needed: area (the FPU's aligner/normaliser shifter shared,
   the cache valid bits in their own byte lanes, IU f_npc and the
   delay-slot search, ...).
