# SPARCstation 20 in Verilog: the rewrite plan

A new SPARCstation 20 (sun4m) core for MiSTer, written from scratch in
SystemVerilog under GPL-2.0-or-later. It replaces the VHDL core
(MiSTer-devel/SunSparcStation20_MiSTer), whose CPU, bus and most of the
chipset are Grabulosaure's (TEMLIB), published "All rights reserved"
with no license, so they cannot be relicensed or redistributed. This
repository keeps only code we own or that carries a compatible license.
Everything else is rebuilt from public specifications, GPL references and
our own test suite.

Plan written 2026-10-09. The VHDL core's own plans and records are in
[legacy/](legacy/). They are useful as hardware notes, but they are not
this plan.

## 1. Ground rules

### 1.1 Clean room

1. **Never open the VHDL core's source while writing RTL here.** That
   means the VHDL core's `rtl/cpu`, `rtl/plomb`, `rtl/peri`, Grabulosaure's
   files in `rtl/sun4m` and `rtl/mister`, the backup tarball and
   `danifunker/ss`. Do not translate it by hand or by tool
   (`sim/gen_verilog.sh`'s GHDL output is still his code).
2. **Allowed sources:** chip datasheets and Sun manuals (§8); QEMU
   (GPL-2: `hw/sparc/sun4m*.c`, `hw/scsi/esp.c`, `hw/net/lance.c`,
   `hw/display/tcx.c`, `hw/char/escc.c`, `hw/dma/sparc32_dma.c`,
   `hw/timer/slavio_timer.c`, `hw/intc/slavio_intctl.c`,
   `hw/misc/slavio_misc.c`, `hw/sparc/sun4m_iommu.c`,
   `target/sparc/`); OpenBIOS (in `bios/`); the Linux and NetBSD
   drivers; our own docs, tests and tools in this repository; and GPL- or
   MIT-compatible RTL from other cores (§5).
3. **The old core may be used as a black box only**: its released
   `.rbf` on a board, to compare observable behaviour (console output,
   boot times, register values read by a test ROM). Never its source.
4. The legacy docs name the old core's files and signals. Treat those
   names as history and take only the hardware facts from them.
5. Sun PROM images and anything that reproduces their code (listings,
   decompiled Forth, detokenized FCode) never go into git. They live in
   the gitignored `scratch/private/` (see `scratch/README.md`).

### 1.2 Compatibility contracts (keep these unchanged)

- **Main (`support/sun`, upstream in Main_MiSTer #1344, plus the CD-audio
  branch #1345).** The new core speaks the same `hps_io` protocol, so
  Main needs no change:
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

  The details are in [legacy/design/main-bridges.md](legacy/design/main-bridges.md),
  [legacy/design/scsi-hps.md](legacy/design/scsi-hps.md) and
  [legacy/design/ethernet-hps.md](legacy/design/ethernet-hps.md). The
  reusable RTL that already implements them (`scsi_targets.vhd`,
  `eth_hps.vhd`, `nvram_sd.vhd`) is ours, and is ported in §4.
- **OpenBIOS (`bios/`, `CONFIG_TACUS`).** Keep the same physical address
  map, device set and IDs, so the existing OpenBIOS build boots
  unchanged, and it becomes the bring-up firmware from phase 5 on.
- **The OSD**: the same options and status bits where they still apply
  (README "OSD"), so users' `.CFG` files keep working.
- **Sun OBP 2.25 (525-1377-08)** must boot as well as it does on the VHDL
  core. Its requirements are in
  [rom-disassembly/ss20-obp-2.25/](rom-disassembly/ss20-obp-2.25/).

### 1.3 Targets carried over from the VHDL core (release 20261006)

| Measure | VHDL core | Goal |
|---|---|---|
| CPUs × clock | 2 × 55 MHz (3 failed timing) | ≥ 2 × 55 MHz; 3-4 CPUs if they fit |
| Device use | 74-90 % ALMs with 2 CPUs | ≤ 85 % with 2 CPUs |
| CPU suite (`tests/cpu`, SS20) | 76 PASS / 0 FAIL / 2 SKIP | 78/0/0 |
| Solaris 8 from disk to login (OpenBIOS) | 117 s | ≤ 117 s |
| NetBSD install CD to installer | 98 s (OpenBIOS), 90 s (OBP) | ≤ same |
| OSes | NetBSD 11, Solaris 8, NeXTSTEP 3.3 (both PROMs) | same |
| Sun POST | passes to the EMC/SMC test (ECC out of scope) | same |

## 2. Architecture

```
emu (SunSparcStation.sv, Template_MiSTer)
 └─ ss20_machine
     ├─ cpu[0..N-1]  SuperSPARC-compatible module: IU + FPU + SRMMU + I$/D$
     ├─ mbus         interconnect: arbiter, snoop broadcast, MID, bus errors
     ├─ memctl       → ddram_arb (port) → DDR3; boot-PROM window; RAM size
     ├─ msi          MBus-to-SBus: IOMMU, SBus slot decode and time-outs
     │   ├─ dma2     espdma + ledma (+ bpp stub)
     │   ├─ esp      NCR 53C9x → scsi_targets (port) → hps_io SD slots
     │   ├─ lance    Am7990 → eth_hps (port) → Main mailbox
     │   ├─ tcx      8-bit framebuffer, DAC, cursor → MiSTer video
     │   ├─ slot 3   CS4231 + APC (new) + FCode PROM (ours)
     │   └─ slot e   "dbri-absent" FCode (ours)
     └─ obio         interrupt controller, counter/timers, system regs,
                     ESCC ×2 (kbd/mouse, ttya/b), M48T08 NVRAM/RTC
                     (+ nvram_sd port), AUXIO, FDC stub (port), ts_beep (port)
```

Decisions (change them here if needed):

- **The CPU is a single-issue, in-order pipeline that looks like a
  SuperSPARC (TMS390Z50) to software.** The aim is the PSR, MMU control
  and ASI behaviour that OBP 2.25, Solaris, NetBSD and NeXTSTEP expect,
  not SuperSPARC's three-way superscalar core. The modules run in MBus
  mode with no MXCC or E-cache (as on the VHDL core; this keeps the
  PROM and the OSes off their MXCC paths).
- **Caches:** physically tagged, write-through D-cache with snoop
  invalidation on MBus writes and DMA; main memory cached even where a
  PTE says C = 0 (this was the VHDL core's E3 fix). Write-back is out.
- **TLB:** large enough that OpenBIOS's Forth does not thrash it (the old
  core's 4-entry DTLB cost ~2× at `ok`, see
  [legacy/PLAN.md](legacy/PLAN.md) E4). The first version copies the
  Viking's 64 fully associative entries (Viking doc §4.11: one CAM over
  the four mapping sizes, limited-history LRU, lock bits, the ASI 6
  image) in flops: ~9,200 ALMs, half the CPU module (2026-10-12 fit).
  **Decided (2026-10-12): keep all 64 entries, with the CAM built from
  RAM** — the 36 compared bits (VPN 20 + context 16) in nine 4-bit
  slices, each a 16-row RAM of 64-bit match vectors; a lookup ANDs the
  nine vectors; a fill writes 16 rows (mapping sizes above 4 KB set
  their ignored slices to "match all"); used/lock bits and the ASI 6
  image stay exact. Target: the MMU at a few thousand ALMs. Done before
  phase 9 (two CPUs must fit: ~11k ALMs each at most), not before.
- **The interconnect is our own simple protocol** (request/grant, 36-bit
  PA, 32-byte line bursts, a snoop broadcast on writes). It does not
  model MBus signal by signal. Document it in `docs/arch/bus.md` before
  phase 1 ends; every device uses it.
- **One clock domain** for the machine, as now. Device timing that
  software depends on (§6) is modelled in that clock.
- **Language:** SystemVerilog that Quartus 17.0 and Verilator 5 both
  accept. Each block gets a unit testbench under `rtl/**/tb/`.

## 3. Phases

Each phase ends with an exit test. Sizes are relative: S (days),
M (1-2 weeks), L (several weeks), XL (a month or more).

### Phase 0: skeleton and tooling (S)
- `emu` top from the current Template_MiSTer, with CONF_STR/OSD rewritten
  to the same options; `files.qip`, `.qsf`/`.sdc`; `scripts/build.sh`
  pointed at the new tree.
- `sim/`: a new `sim_top.sv` around `ss20_machine`, keeping
  `sim_main.cpp`'s hps_io, SD, DDR, UART and Ethernet models; drop the
  GHDL step (`gen_verilog.sh`).
- Write the bus spec (`docs/arch/bus.md`) and the address map
  (`docs/arch/memory-map.md`) from
  [rom-disassembly/ss20-obp-2.25/hardware-access.md](rom-disassembly/ss20-obp-2.25/hardware-access.md),
  QEMU's `sun4m.c` (`ss20_hwdef`) and OpenBIOS's `CONFIG_TACUS` map.
- **Exit:** an empty machine builds in Quartus and Verilator; the board
  shows the OSD.

### Phase 1: integer unit (L)
- SPARC V8 IU: 8 register windows (NWINDOWS = 8), traps and RETT,
  delayed control transfers with annul, MUL/DIV (`umul`, `sdiv` and so
  on), `ldstub`/`swap`, alternate-space loads and stores, the `%y`,
  `%asr`, `%psr`, `%wim` and `%tbr` registers, error mode.
- Run from a behavioural memory in simulation.
- Known hazards to test from the start: a JMPL decoded behind a stalled
  RETT (A1, `t_dcti`); error mode → watchdog reset (IU-1, `wdtest`).
- **Exit:** `tests/cpu` `t_alu`, `t_branch`, `t_dcti`, `t_ldst`, `t_psr`,
  `t_traps`, `t_window`, `t_iu2` and `brktest` pass in Verilator, with
  results equal to QEMU's (`tests/cpu/expected`).

### Phase 2: FPU (L)
- The V8 FPop set in single and double precision (quad traps as
  unimplemented, as SuperSPARC does), FSR/FQ, deferred traps, IEEE
  underflow and inexact exactly as the POST and `t_fpu` check (including
  "a product shifted entirely out of the mantissa is an underflow").
- Start with an iterative divide and square root. Write the FPU for
  SPARC directly: an FPU built for another architecture (for example a
  68040's 80-bit extended datapath and exception model) does not carry
  over. QEMU's `target/sparc/fop_helper.c` is the behavioural reference.
- **Exit:** `t_fpu` passes; the SS20 POST's FPU tests pass in simulation
  (run the real PROM in sim from `scratch/private/roms`).

### Phase 3: SRMMU and caches (XL)
- SPARC reference MMU: context table, three-level table walk, R/M bit
  updates, the fault registers (SFSR/SFAR, AFSR/AFAR), the MMU control
  register with SuperSPARC's IDs and bits, and the TLB flush and probe
  ASIs.
- I-cache and D-cache with SuperSPARC's diagnostic ASIs (0x0c-0x0f) and
  flash clears (0x36/0x37); the diagnostic TLB (ASI 6); the L2 TLB/PTP
  cache (MCNTL bit 6; an ASI 5/6/7 write drops the TLBs, which NeXTSTEP
  needs).
- Bus errors: a read the bus refuses traps with SFSR FT 5 and TO/BE; a
  table walk's FT 4 + L (A3, `t_buserr`).
- **Exit:** `t_mmu`, `t_mmu_diag`, `t_mmu_l2`, `t_mmu_swift` (expected
  skips), `t_cache*`, `t_buserr`, `memstress`, `tlbbench` and `bmbench`
  pass; the whole suite runs in simulation from the real boot address.

### Phase 4: memory, PROM and the reused HPS blocks (M)
- `memctl` on DDR3 through `ddram_arb.sv` (ours, unchanged, with its
  testbenches); the boot-PROM window at `0xF_F000_0000`, loaded by ioctl
  index 0; RAM-size option; RAM cleared on power-up, OSD reset or a new
  ROM.
- Port to SystemVerilog: `nvram_sd.vhd` (NVRAM on the SD card, IDPROM
  stamping), `eth_hps.vhd` (the mailbox, 10 Mb/s RX pacing, LANCE
  loopback), and `scsi_targets.vhd` (the target engine: disks, CD,
  SCSI-2 audio commands, the 44.1 kHz player, two-byte messages, SDTR).
  Give each the same interface contract and port its testbench
  (`sim/run-nvram.sh`, `run-eth.sh`, `run-scsi.sh`).
- **Exit:** the CPU suite runs on the board from the OSD's Boot PROM;
  `rtl/mister/tb/run.sh` passes; the ported blocks pass their benches.

### Phase 5: sun4m system devices → OpenBIOS `ok` on ttya (L)
- Interrupt controller (per-CPU and system registers, mask readbacks
  with bits 26:23 reading 0: INT-2), counter/timers (user-timer mode,
  TMR-4, TMR-8: RUN resets to 1), system control/status (RS/WD for
  OpenBIOS's warm-reset memory clear), MID, AUXIO, the EMC/MSI
  registers the PROMs read, and SBus slot time-outs on empty slots (B1).
- ESCC (Z85C30) with ttya/ttyb, BREAK detection (RR0 bit 7, ext/status
  interrupt), and the debug-link escape if `pcdump` is kept. Options:
  write it new, or start from minimigmac's `scc.v` (GPL-2), which needs
  async interrupts and the ESCC additions.
- M48T08 NVRAM/RTC, seeded from the HPS clock, BCD leap years.
- **Exit:** `t_chipset`, `t_msi`, `kbdtest` pass; OpenBIOS from `bios/`
  reaches `0 >` on ttya; `mknvram.py`'s image is read and written back.

### Phase 6: keyboard, mouse and TCX → the screen console (M)
- Sun Type 4/5 keyboard protocol on ESCC channel A of the kbd/mouse
  ESCC: reset reply with held keys, layout byte, bell and click commands
  (→ `ts_beep` port), Stop-A/L-keys/AltGraph chords as designed in
  [legacy/PLAN.md](legacy/PLAN.md) B2. PS/2 → Sun scan-code table
  written new. The Sun-2 core's `sun2_mister_kbd_mouse.sv` is GPL-3;
  using it would make the whole core GPL-3 (§5).
- Sun mouse (Mouse Systems 5-byte protocol).
- TCX: 8-bit VRAM in DDR3, the DAC/palette, hardware cursor, the
  blitter/stippler registers OpenBIOS's fast console uses; the FCode
  PROM from `ts_fcode_pack.vhd` (OpenBIOS/QEMU, GPL-2, regenerate it as
  SV with `tools/fcode2vhd.py`). CG3 is optional.
- **Exit:** OpenBIOS on the screen console; `brktest`, `tb_kbd`'s cases;
  the OSD's keyboard layouts type their AltGraph characters.

### Phase 7: IOMMU, DMA2 and SCSI → disks boot (L)
- IOMMU (page table walk, CAM/TLB, flush; the SS20's fixed rev 0x26),
  DMA2 espdma with its partial-last-word rule (DMA-5).
- ESP (NCR 53C9x as Sun's `esp` driver sees it) on the ported target
  engine. Start from our Quadra core's `ncr53c96.sv` (GPL-2, ours) and
  check it against the 53C90A/ESP100A differences. Behaviour that
  software depends on: a select's interrupt held ~1 ms (ESP-7); MESSAGE
  ACCEPTED gives Bus Service while the target wants more (ESP-8);
  OpenBIOS's exact command sequence (`scsitest` mode 2).
- **Exit:** `scsitest` (all modes) in sim and on the board; NetBSD 11
  and Solaris 8 boot from disk under OpenBIOS; the CD boots NetBSD's
  installer; CD audio plays (`hwtest.sh cdaudio`).

### Phase 8: Ethernet (M)
- Am7990 LANCE with ledma. Vendor **Wish7990** (MIT,
  MelkhiorVintageComputing, already vendored in the Sun-3 core at
  `8610ef5`) and replace its MII side with the ported `eth_hps`, or
  write a LANCE on the same backend. Frames that span several TX
  descriptors at any byte; oversize frames dropped; MODE LOOP.
- **Exit:** `ethtest` (9 cases) in sim; `hwtest.sh net-netbsd`,
  `net-solaris`, `obp-testnet`.

### Phase 9: SMP (L)
- First, the CPU's size: the RAM-built TLB CAM (§2 "TLB") and the FPU's
  align/normalise shifters on DSP blocks, so that a CPU module is at
  most ~11k ALMs (it was ~19k on 2026-10-12, the old core's ~7.3k) and
  two fit with the system at 55 MHz (`syn/cpu_timing/run.sh` measures).
- 2 CPUs (then 3-4 if timing allows): MID per module, the per-CPU
  interrupt and timer registers, IPIs (soft interrupts), `ldstub`/`swap`
  atomic across CPUs, snoop invalidation, the arbiter-enable register
  (the POST's "CPU_#2 NOT installed" race), and the OpenBIOS secondary
  start (MCNTL 0x041).
- **Exit:** `t_smp`, `t_msi` (MP); Solaris and NetBSD with 2 CPUs;
  `hwtest.sh stress` for 30-40 minutes.

### Phase 10: sound and the rest (M)
- CS4231 + APC playback DMA, written new from the CS4231A datasheet
  and the `audiocs` drivers (Keith Conger's model is not used), at SBus
  slot 3 with our FCode (`tools/fcode_cs4231.py`); the R2 INT and
  8-bit stereo; mixed with CD audio and `ts_beep` into `AUDIO_L/R`.
- FDC stub (port of `ts_fdc.vhd`), the slot-e `dbri-absent` FCode.
- **Exit:** `hwtest.sh audio-*`, NetBSD/Solaris `audioplay`.

### Phase 11: Sun OBP 2.25, POST, NeXTSTEP (M)
- Boot OBP 2.25 with 2 CPUs; the POST passes up to EMC/SMC; `test net`.
- NeXTSTEP 3.3 under both PROMs: TMR-8, DEC-4 (named slot-e node),
  ESP-7/8, and its use of the L2 TLB.
- **Exit:** `hwtest.sh 20 all` with `--obp`, and `nextstep-install`.

### Phase 12: performance, timing, release (M)
- Timing closure at ≥ 55 MHz with the target CPU count; the boot-time
  table in §1.3; `hwtest.sh speed`, `boottime`.
- README, RELEASE-NOTES, a `releases/` rbf, and the OpenBIOS ROM built
  from `bios/`.

### Phase 13: video memory on SDRAM, a 24-bit frame buffer (M)
- **The core requires the SDRAM module** (user, 2026-10-18: almost every
  MiSTer has one). Today the SDRAM pins are idle and the TCX's 1 MB VRAM
  sits in DDR3, where its scan-out (~47 MB/s at 1024x768x8 bits, 60 Hz)
  shares the port with the CPUs. Moving the VRAM and the scan-out to SDRAM
  frees that DDR3 bandwidth for the CPUs (measure it) and makes room for a
  deeper frame buffer.
- Then a 24-bit mode, TCX's S24 (QEMU `tcx` depth 24; Solaris's driver;
  check NetBSD's, Linux's and OpenBIOS's): 1024x768 at 60 Hz is ~190 MB/s
  at 32 bits a pixel against ~200-260 MB/s for one 16-bit SDRAM, before
  the CPUs' writes and the S24's per-pixel control plane. Design the
  layout (3 bytes a pixel, the control bits kept small or on chip) before
  the RTL.
- Needs the area from the phase 12 shrink first (an SDRAM controller is a
  few hundred ALMs).

Order: 0 → 1 → 2 and 3 (in parallel once the IU runs) → 4 → 5 → 6 →
7 → 8 → 9 → 10 → 11 → 12. Phase 9 can start after phase 5 if the bus
was designed for SMP from the start (it should be).

## 4. What we carry over (in this repository)

| Item | Where | Use |
|---|---|---|
| SCSI target engine, CD audio | `rtl/sun4m/scsi_targets.vhd` | port to SV (phase 4) |
| Ethernet mailbox to Main | `rtl/mister/eth_hps.vhd` | port (phase 4) |
| NVRAM on SD, IDPROM | `rtl/mister/nvram_sd.vhd` | port (phase 4) |
| DDR3 arbiter + benches | `rtl/mister/ddram_arb.sv`, `tb/` | as is |
| FDC stub, bell/click | `rtl/sun4m/ts_fdc.vhd`, `ts_beep.vhd` | port |
| FCode images | `ts_fcode_pack.vhd` (OpenBIOS, GPL-2), `ts_fcode_cs4231_pack.vhd` (ours) | regenerate as SV |
| Keyboard bench | `rtl/sun4m/tb/tb_kbd.vhd` | port the cases |
| Firmware | `bios/` (OpenBIOS, GPL-2) | as is |
| CPU/chipset test suite | `tests/cpu/` (78 tests, QEMU references) | as is |
| Simulation harness | `sim/sim_main.cpp`, run scripts | new `sim_top.sv` |
| Board regression | `scripts/hwtest.sh`, `scripts/board/` | as is |
| ROM tools | `tools/romdis/`, `fcode2vhd.py`, `fcode_cs4231.py`, `mknvram.py`, `mkcdda.py`, `sparc_link.py`, `ufsread.py`, `ufscmp.py`, `nextstep/`, `mister/sun-idprom.sh` | as is |
| `pcdump` | `tools/debugarm/pcdump.c` | needs a new debug link and its own `lib.h` (the rest of `debugarm` was Grabulosaure's) |
| Hardware knowledge | `docs/rom-disassembly/`, `docs/legacy/` | requirements |

The VHDL files above use small helper types from the old core's packages
(`base_pack`, `ts_pack`, `plomb_pack`), which are not here, so
they will not compile here as they are. Port their logic, not those
types.

## 5. Outside RTL we may use

Copies of the GPL-2 and MIT candidates below, with their licenses, are in
the gitignored `scratch/third-party-rtl/`.

| What | Source | License | Note |
|---|---|---|---|
| Am7990 LANCE | Wish7990 (`Sun-3_MiSTer/rtl/vendor/wish7990`) | MIT | compatible with GPL-2 |
| NCR 53C96 ESP | `MacQuadra800_MiSTer/rtl/ncr53c96.sv` | GPL-2 (ours) | same chip family |
| NCR 53C90 ESP | `NeXT_MiSTer/rtl/next/next_scsi.sv` | GPL-2 (mostly Adam Polkosnik) | second reference |
| Z8530 SCC | `MacQuadra800_MiSTer/rtl/scc.v` (minimigmac lineage) | GPL-2 | simplified; needs the ESCC parts |
| Z8530 SCC | z8530_scc (vz50938) | **GPL-3** | would make the core GPL-3 |
| Sun keyboard/mouse | `Sun-2_MiSTer/rtl/sun2_mister_kbd_mouse.sv` | **GPL-3** | same |
| SPARC V8 + SRMMU reference | GRLIB LEON3 (Frontgrade Gaisler) | GPL edition | read as reference; check its version |

**License decision (user, 2026-10-18): GPL-3 parts are allowed.** Our
files stay GPL-2.0-or-later and `sys/` is GPL-2+, so a build that
includes a GPL-3 part (e.g. Sorgelig's `sdram.sv`, the GPL-3 entries in
the table above) is distributed as GPL-3 as a whole; keep each outside
file's license header. OpenBIOS is a separate firmware image, so it does
not affect the core's license either way.

## 6. Behaviour requirements learned on the VHDL core

These are facts about the hardware and the software, recorded in the
legacy docs and commits. Each one must be true of the new core. Write a
test for each in `tests/cpu` or a unit bench.

| ID | Requirement | Source |
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
| DEC-3 | Registers no device answers read 0 | legacy/impl-gaps/chipset.md |
| DEC-4 | Slot e answers with a named FCode node | NeXTSTEP |
| DMA-5 | ESP DMA: the partial last word | `scsitest` t_scsi_sense1 |
| ESP-7 | A select's interrupt is held ~1 ms | NeXTSTEP |
| ESP-8 | MESSAGE ACCEPTED → Bus Service while the target wants more; after each MESSAGE IN byte (TI or ICCS) the phase reads MESSAGE IN until MESSAGE ACCEPTED (ACK held: Solaris under OBP) | `scsitest` t_scsi_sdtr1, `tb_espdma` |
| ESP-9 | A non-DMA TI in an OUT phase waits for its first FIFO byte (written after the command) | NeXTSTEP, `tb_espdma` |
| SCSI-TAG | Two-byte (queue tag) messages accepted | `scsitest` t_scsi_tag1 |
| CD-512 | The CD powers up at 512-byte blocks, follows MODE SELECT | Solaris |
| LAN-2 | Oversize received frames dropped | `ethtest` t_eth_oversize |
| C3 | Received frames paced at 10 Mb/s | `net.sh` large pings |
| KBD | Reset reply lists held keys; it answers within the real keyboard's delay (NetBSD's `kbd0: reset failed`) | `tb_kbd` |
| BRK | BREAK on ttya reaches the ESCC (RR0 bit 7, ext/status interrupt) | `brktest` |
| RST | RS/WD status after a software or watchdog reset (OpenBIOS clears RAM) | NeXTSTEP fsck reboot |
| ID | IDPROM from the host (ioctl 64), stamped into a blank NVRAM | `hwtest.sh idprom-obp` |

The full lists are in [legacy/IMPLEMENTATION_GAPS.md](legacy/IMPLEMENTATION_GAPS.md) §6,
[legacy/impl-gaps/](legacy/impl-gaps/) and
[legacy/HARDWARE_GAPS.md](legacy/HARDWARE_GAPS.md). Go through them at
the start of each phase.

## 7. Test strategy

- **Unit benches** per block (Verilator or iverilog), in `rtl/**/tb/`.
- **The CPU suite** (`tests/cpu`) in Verilator against QEMU's expected
  logs from phase 1 on, and on the board from phase 4. QEMU has known
  V8 deviations ([tests/cpu/README.md](../tests/cpu/README.md)).
- **Whole-machine simulation** (`sim/`): OpenBIOS to `ok`, `scsitest`,
  `ethtest`, and the real Sun PROM's POST (`build.sh --diag`).
- **Board regression** (`scripts/hwtest.sh 20 all`, `--obp`), unchanged.
- **Black-box comparison** with the VHDL core's released rbf (§1.1, 3)
  when a behaviour is unclear, along with QEMU.

## 8. Reference documents

In `scratch/reference/` (PDFs, with text extractions): the SPARC V8
architecture manual (get it from SPARC International if it is not
there), SuperSPARC (`SuperSparc.pdf`, `SuperSPARC2.pdf`,
`supersparcwhitepaper.pdf`, STP1021UG), the *Sun-4M System Architecture*
(`Sun4M_SystemArchitecture_edited2.pdf`), the SPARCstation 20 field
service manual, microSPARC-II and TurboSPARC manuals (for the SS5 and
the SRMMU), HyperSPARC, LEON2. To add: the NCR 53C90A/ESP100A and
53C9x datasheets, Am7990, Z85C30/85230, M48T08, CS4231A, MBus
specification, and the TCX section of the Sun framebuffer docs or QEMU.

## 9. Status

Updated 2026-10-13 (phases 4-8 integrated; see the table). Summary of 2026-10-12: the new CPU (phases 1-3, pipelined: it
meets 55 MHz alone with +2.5 ns, and in the hybrid build at 92 %), inside
the old core as the "CPU hybrid" at the full 55 MHz, boots NetBSD 11 and
Solaris 8 under OpenBIOS and the Sun PROM, and runs X and CDE on the
board; phase 0 of the new core is done (an empty machine builds and shows
a pattern). The running detail is in
[sessions/HANDOFF-20261009.md](sessions/HANDOFF-20261009.md) ("Current
state").

| Phase | State |
|---|---|
| 0 skeleton and tooling | `docs/arch/bus.md` (the I/O bus; the interconnect still to define), `docs/arch/memory-map.md`, `docs/arch/cpu.md`, `docs/arch/iu-notes.md` (the V8/QEMU/test-suite digest), `rtl/tb/run.sh` (Verilator lint + benches), `scripts/qcheck.sh` (a Quartus 17 syntax check in half a minute). **The skeleton (2026-10-12, branch `phase0`):** `SunSparcStation.sv` (the `emu` top on the repository's `sys/`: CONF_STR with the VHDL core's status bits, `FC0` Boot PROM, `SC0`-`SC3` disks/CD/NVRAM, `hps_io` WIDE with 4 slots), `rtl/pll.v` (clk_sys 55 MHz, clk_vid 65 MHz), `SunSparcStation.qpf`/`SunSparcStation20.qsf`/`.sdc`, `files.qip`; `rtl/machine/ss20_osd.sv` (the status-word decoder, shared with the simulation), `rtl/machine/ss20_machine.sv` (the empty machine: the full port list, every port idle, a 1024x768 test pattern, LED_USER blinking), `rtl/video/video_timing.sv` (+ `tb_video_timing`); `sim/sim_top.sv` around `ss20_machine`, `sim/build.sh` taking its RTL from `files.qip` (the GHDL step is gone), `sim_main.cpp` setting the OSD status word; `scripts/build.sh` for the new tree (one flow per project directory, not per machine). Quartus 17.0.2: 7,403 ALMs (18 %), all clocks met (worst setup +0.467 ns on the framework's HDMI PLL, clk_vid +4.900, clk_sys +9.518; worst hold +0.249), 3.5 minutes. Verilator: 4 frames in 5M cycles, 33 MHz. **Exit:** builds in both; the board's OSD check is pending. |
| 1 integer unit | **done to the suite.** `rtl/cpu/`: `iu` (the 5-stage pipeline, docs/arch/cpu.md §1), `iu_decode`, `iu_regfile`, `iu_alu`, `iu_muldiv`, each with a bench; `sim/iu/run.sh` builds the phase-1/2 subset of the suite (`tests/cpu/src/main_iu.S`: t_alu, t_ldst, t_branch, t_traps, t_window, t_psr, t_fpu, t_iu2, t_dcti) and runs it on the IU + FPU with a behavioural memory and the new `escc` on ttya, against a QEMU 11 reference with its V8 deviations corrected: **39 PASS, 0 SKIP** (2026-10-09). Not yet: `brktest` (needs the interrupt path), the watchdog reset (IU-1), a timing estimate. |
| 2 FPU | **done to the suite.** `rtl/cpu/fpu.sv` with `fp_unpack`, `fp_round`, `fp_divsqrt` (docs/arch/cpu.md §2): V8 single/double FPops, FSR, FQ, deferred traps; quad → unimplemented. Checked against an exact IEEE 754 model with the SPARC conventions (`tests/fpu/`, 617k vectors: `rtl/tb/run.sh tb_fpu` PASS 602,375 checks) and `t_fpu` + the `t_iu2` FPU checks pass in `sim/iu`. Iterative (one FP instruction in flight); divide 57 and square root 58 cycles. Quartus syntax check pending. |
| 3 SRMMU and caches | **done to the suite.** `rtl/cpu/srmmu.sv` (ASI 4 registers, 64-entry TLB = the ASI 6 image, walker with R/M write-back, V8 fault classes/OW, probe/flush), `rtl/cpu/l1cache.sv` (the SuperSPARC geometries: I 20 KB 5-way 64-byte lines with half-line valid bits, D 16 KB 4-way 32-byte; write-through, snoop invalidation, diagnostic ASIs 0xC-0xF, flash clears), `rtl/cpu/cpu_module.sv` (IU + FPU + SRMMU + caches, the ASI decode, the memory-port arbiter, ASI 0x38/0x4C registers, the watchdog), `rtl/pkg/mem_pkg.sv` (the interconnect, docs/arch/bus.md). Design: docs/arch/mmu.md; digest: docs/arch/mmu-notes.md. `sim/cpu/run.sh` runs the whole suite minus chipset/MSI/SMP on the module against QEMU: **65 PASS, 0 FAIL, 0 SKIP** (2026-10-10), passing 15 tests QEMU fails or skips (probes, SFSR L bits, ASI 6 image, bus errors, cache force, …). Quartus 17 analysis passes. Not yet: the L2 TLB option (bit 6 stored), `wdtest`, `tlbbench`/`bmbench`, the fetch path's speed. Two tests adapted to a TLB that is the 64-entry image (t_mmu_l2_asi6/_enable). **On the board (2026-10-10): `cpu_module` runs inside the old VHDL core in place of its IU + MMU (the CPU hybrid, old repo branch `hybrid`, at half the system clock through the `plomb_master` shim): CPU suite 71 pass, 2 fail (a read-after-write ordering of the shim's posted writes, fenced since), 5 skip (the multi-CPU tests on a one-CPU machine). Its one-cycle paths miss 55 MHz (FPU datapath −21 ns, IU→TLB→I-cache −15.6 ns): the pipeline pass is next.** |
| 4 memory, PROM, HPS blocks | **done (2026-10-13).** `rtl/machine/`: `ss20_memctl` (main memory and the PROM in DDR3, `rtl/pkg/ss20_ddr_pkg.sv` the layout: bits 28:20 inverted, clear of the scaler's buffer; SIMM slot 7 a 16 MB alias), `ss20_loader` (ioctl index 0 and the memory clear), `ss20_iobridge`, `ss20_uart` (ttya on the HPS UART), `ss20_mbus` (the CPU and DVMA onto the memory controller); `nvram_sd`, `eth_hps`, `scsi_targets` ported with their benches. **Exit met:** the CPU suite on the board from the OSD's Boot PROM, `CPUTEST DONE pass=70 fail=0 skip=3`; `rtl/mister/tb/run.sh` and the ported benches pass. |
| 5 system devices | **done in simulation (2026-10-13).** The rtl/obio blocks plus the TACUS configuration words in the Aux 0 page (`slavio_misc`: HWCONF 0x12, CPU clock, RAM size, serial/CG3/autoboot), `rtl/sun4m/ss20_msi.sv` (IOMMU with a 16-entry TLB, M-to-S AFSR/AFAR, arbiter enable, slot configuration, MID; IMPL/VER 0x13), `fdc82077` (no drive), the MACIO ID. In Verilator: OpenBIOS reaches `0 >` on ttya; the full suite (`cputest.rom`, incl. t_chipset and t_msi) 79/0/8, the skips multi-CPU or the harness's DMA/IRQ sources. **Board (2026-10-13): done** — the suite 82/0/5 with two CPUs (= baseline), `brk` and `kbd` as their baselines. The SMC registers (`emc`, phase 11) made it to here too. |
| 6 keyboard, mouse, TCX | **done in simulation (2026-10-13).** `sun_kbd` (Type 5 from hps_io's PS/2 events, 1200 baud, reset/layout/LED/bell/click, the B2 chords), `sun_mouse` (Mouse Systems 5-byte), `tcx` (+ `tcx_scan`, `tcx_dcram`, `tcx_fcode`: 8-bit VRAM in DDR3 at 0x1D10_0000, Bt458 DAC, THC cursor, blit/stipple, scan-out on ddram_arb's priority port, QEMU's FCode). OpenBIOS's screen console renders (sim frame). **Board: done** — the screen console and typing work; `kbd` PASS (baseline re-recorded 2026-10-13 with the Type 5 idle 0x7F; `kbdtest` times its waits on the system counter). |
| 7 IOMMU, DMA2, SCSI | **done in simulation (2026-10-13).** `dma2` (espdma with DMA-5, ledma with E_BASE_ADDR, bpp stub), `esp` (53C90A/ESP100A from the Quadra `ncr53c96.sv` on `scsi_targets`; ESP-7, ESP-8). `scsitest` in sim: all three configurations pass (one disk 10/0/2, two disks 11/0/1, disk + CD with CD audio 11/0/1). **Board: NetBSD 11 and Solaris 8 boot from disk under OpenBIOS** (`hwtest netbsd|solaris`), `scsi` as its baseline; the slot-3 CS4231 + APC (OpenBIOS declares it; NetBSD faulted without it) came forward from phase 10. **CD on the board (2026-10-13): `cd` (NetBSD 11 install CD, OpenBIOS) PASS in 149 s (target 98 s; 123 s before the boot-mode PROM pace), `cdaudio` PASS.** Solaris 8 needs the OpenBIOS from `bios/` with `/memory reg` per 64 MB SIMM slot (e3efb1c): with the SMC registers present, Solaris counts only listed slots as RAM. |
| 8 Ethernet | **done in simulation (2026-10-13).** `lance` (Wish7990, MIT, vendored as `wish7990_*`, on `eth_hps`; CSR3 BSWP honoured for frame data). `ethtest` 9/9 with `--eth-loop`. **Board: `net-netbsd` and `net-solaris` PASS** (Main 20261010, again with two CPUs on 2026-10-13); `obp-testnet` and `net-solaris-obp` PASS (aaaf9c2). |
| 9 SMP | **two CPUs on the board (2026-10-13).** First the CPU's size: the TLB CAM in RAM (19.1k → 11.7k ALMs a CPU), then the shrink (FPU registers, snoop queues, micro-TLB PPNs in MLAB; one FP unpacker; subnormals normalised a bit a cycle; one subtractor for divide and square root; the FCode ROMs in M10K; the SRMMU CAM on `ram_sdp_be`): a CPU 8.5k ALMs, FPU 1.9k. `ss20_machine` NCPU (default 2; `SIM_NCPU=1` for one), `ss20_mbus` N masters round robin + DVMA, line writes split, the MSI's arbiter enable parks CPUs 9+; per-CPU interrupts and timers; MCNTL bits 3:2 = the CPU index (what TACUS OpenBIOS and the suite park by); MCNTL.SE gates snooping of other CPUs' writes only (DVMA always snooped). Fit 37,476 ALMs (89 %), all clocks met (clk_sys +1.50 ns, HDMI +0.15). Sim: suite 82/0/5 (2 CPUs), 79/0/8 (1); OpenBIOS `CPUs: 2`. Board: suite 82/0/5, NetBSD (GENERIC is uniprocessor: `cpu1` "not configured" — a GENERIC.MP kernel is needed to test it), Solaris 8 boots (`CPUs: 2` from OpenBIOS; `psrinfo` not yet seen), the OBP POST on both CPUs. Board (aaaf9c2): `stress` 30 min PASS under OBP and under OpenBIOS (Solaris, all CPUs); Solaris under OBP brings both CPUs on-line. Not yet: NetBSD with an MP kernel. |
| 10 sound and the rest | **mostly done.** `cs4231` + APC (slot 3, our FCode), `kbd_beep` (bell and click, replaces `ts_beep`), `fdc82077`, the slot-E `dbri-absent` FCode (`slote_fcode`). Board: `audio-obp` and `audio-solaris-obp` PASS (aaaf9c2; 2 s played in 2.91 s). `audio-solaris` PASS (2.94 s). `audio-netbsd` PASS (2.44-2.66 s) since the test writes through a write-only open (see the handoff: NetBSD's cs4231_sbus counts blocks twice while capture runs); `audiotest` (the APC at NetBSD's 48 kHz, timed) PASS in sim and on the board. Not yet: `audioplay`. |
| 11 Sun OBP 2.25 | **POST passes on the board (2026-10-13).** In sim: OBP to `ok`, `probe-scsi`, `boot disk` to NetBSD login; the diagnostic POST clean on both CPUs. New: `emc` (SMC registers, ECC injection/correction, MEM_ERR on level 15), SW reset of `slavio_misc` (SW_RST_STAT), ESP MESSAGE IN after ICCS, intctl VME mask bits, posted SBus write errors to the MSI's AFSR + level 15, DMA2 size errors, the parallel port at slot F 0x480_0000, `tools/obp_nvram.py`, `sim/run-obp.sh`, `--ram`; boot-mode PROM reads at the EPROM's pace (`ss20_memctl` PROM_WAIT = CLK_HZ/1.5M, gated by the master's MCNTL.BM; the POST's counter/timer test counts instructions). Board: `hwtest post` PASS (no ERROR lines; OBP 2.25 prints no PASSED banner). **`solaris-obp` PASS on the board (aaaf9c2)**: Solaris's esp driver negotiates SDTR under OBP and needs the phase to read MESSAGE IN until MESSAGE ACCEPTED after each reply byte (ESP-8). Board (aaaf9c2): `net-solaris-obp`, `obp-testnet`, `idprom-obp`, `audio-obp`, `audio-solaris-obp`, `cd-obp` (111 s) PASS; Solaris under OBP brings both CPUs on-line. Not yet: NeXTSTEP (no `ns33.img` on the board), `hwtest all --obp`. |
| 12 timing, release, speed | started. **Speed: UnixBench 2.6 (one CPU, OpenBIOS) / 4.0 (two, OBP); target ~2x; profile and plan in docs/benchmarks/profile-20261014.md (store buffer, load hits without holding M, fetch, FP overlap, then an L2).** The CD boot is back to 123 s (16938f1: OpenBIOS's waiting secondary CPU runs from its I-cache instead of taking half the memory). Open: HDMI PLL slack (−0.15..+0.35 between builds; the failing path is inside `ascal`: a seed sweep), release notes. A test build is in `releases/` (20261014). The X mouse works (Solaris's `ms` discards its first 48 packets). |

**Fit (2026-10-13, two CPUs and everything above):** 37,476 ALMs (89 %); every clock met (clk_sys +1.499 ns, HDMI +0.152, h2f_user0 +1.326; hold ≥ +0.17).

**The serial port, at integration (phase 5 exit):** the ESCC runs ttya at
the baud rate the OS programs (WR12/13 from PCLK), so the wire to the HPS
UART needs either Main's `uart_speed` to match (declare `UART…` in
CONF_STR, HARDWARE_GAPS #3) or a byte-level rate bridge between the ESCC's
TxD/RxD and the HPS UART; drive `UART_DTR/RTS` from WR5 and feed
`UART_DSR/CTS` into DCD/CTS (kms S2); connect port B's RxD high (kms S3).

Reference material: [references/INDEX.md](references/INDEX.md) lists the
datasheets, Sun specs and emulator/driver sources in `scratch/reference/`
with their URLs; each RTL header names the sections it was written from.

## 10. Work queue (from 2026-10-19)

In this order (the user's, 2026-10-19). Each item: unit benches, the
`sim/iu` regression and `perfbench` ticks for anything in the CPU, the
2-CPU CPU suite in the simulation, a full build under
`flock /tmp/quartus.lock`, then the board; commit before every build.

**The board:** the MiSTerPi at 192.168.99.143 (in the gitignored
`scripts/local.env` as `MISTER_HOST`; .92 is the HP cores' board). Take
`/tmp/board.lock` on it (noclobber) while testing and remove it after.
Screenshots and keys through `misterkvms-cli` (`$MISTERKVMS` in
local.env), never MiSTer's own screenshot command. Its
`games/SunSparcStation/boot0.rom` is the Sun PROM: Main sends boot0.rom
then boot.rom as index 0, so a boot.rom (the test scripts' ROM) wins.

1. **Board check of the CPU area pass** (146a1de, rbf 08a60f05: 36,250
   ALMs, clk_sys +0.692 ns): cpu, netbsd, solaris, then the Sun PROM
   tests and the CD. docs/sessions/notes-area-20261019.md.
   *Done 2026-10-19:* cpu 84/0/5, NetBSD under OpenBIOS, Solaris 8 on
   both CPUs under the Sun OBP (loaded through the OSD). Not yet: the CD,
   net, audio, NeXTSTEP, the POST.
2. *Step 1 done 2026-10-19* (1124d48, 4be08f5; the board: Solaris's CDE
   login and the Sun PROM's banner drawn right, cpu 84/0/5; partial
   writes are read-merge-write: DQM is not reliable on the board).
   Left: the build's HDMI seed (-0.083 ns on seed 3: sweep), the full
   board matrix, and the VRAM's speed (a closed-page controller, a
   read-merge-write per partial write: measure the console and X against
   the DDR3 build; open pages / write bursts if it is slower).
   **Phase 13 step 1: the TCX's VRAM and scan-out on SDRAM** (§3 phase
   13; the core requires the SDRAM module). The controller: the user's
   hp9000-715 `rtl/mister/sdram64.sv` (GPL-2+, board-proven) or
   Sorgelig's `sdram.sv` (GPL-3, allowed since 2026-10-18). A pin-level
   SDRAM model in the simulation. Measure what the CPUs gain on DDR3.
3. *Done 2026-10-19 (e5b0e09, tb_l1cache).* **D-cache: the issues the 2026-10-18 review found.** (a) S_WRITE_WAIT
   re-checks for a hit every cycle against tags read at the current
   `idx_i`: if the index moves while a store waits (after a watchdog
   reset of the IU, for one) a store could update the wrong line; check
   the hit once, in the accept cycle's tags, and prove it with a bench.
   (b) The store-to-load forward (`fw_*`) can never fire, since the D-cache
   takes no request outside S_IDLE: an assertion first, then remove it.
   (l1cache has no unit bench yet: write one, tb_l1cache, for these.)
4. **Instruction fetch** (profile-20261014.md: fetch was 12-28 % of
   cycles under NetBSD): the I-cache miss latency, forward-branch
   mispredicts, the bubble after a redirect. perfbench and the NetBSD
   profile before and after. Also the Sun PROM's CPU clock estimate
   (user, 2026-10-19: psrinfo says 40 MHz under OBP 2.25, 27 MHz before
   the speed work; OpenBIOS takes 55 from the TACUS word): the PROM times
   a delay loop against the counter/timer and assumes a SuperSPARC's
   cycles per iteration. Find the loop in the simulation (+pcsample under
   OBP), count our cycles per iteration and remove what costs more than
   the chip's (a taken backward branch, the fetch bubble): the goal is
   ~55 MHz from the real speed, not from a changed timer.
5. **An L2 cache** between ss20_mbus and ss20_memctl (~128 KB of the free
   M10K, ~2k ALMs): fork/exec, file and shell tests.
6. **The Sun PROM's start-up memory test** (user, 2026-10-19: it runs at
   every start and takes minutes). Seen on the board: with no NVRAM
   image mounted (slot 3, P5) the PROM finds a blank NVRAM ("Incorrect
   configuration checksum"), sets its defaults and diag-switch? true, so
   it tests all 464 MB (~1 MB/s) and then boots the diagnostic device
   (the network), not the disk. (a) Give the core a valid default NVRAM
   (diag-switch? false, auto-boot? true, boot-device disk; the IDPROM from
   Main's seed as now) when no image is mounted, and make the image's
   use clear in the README. (b) Measure the test's speed in the
   simulation: the boot-mode PROM pace (ss20_memctl PROM_WAIT, kept for
   the POST's counter/timer test), the caches being off in boot mode; make
   it faster where the POST still passes. Compare with a real SS20's time.
   *(a) done 2026-10-19 (0a72b3c, then sbus-probe-list):* the cause was OBP
   itself: a bad configuration checksum makes it set its defaults AND
   diag-switch? true (the word after "Incorrect configuration checksum").
   nvram_sd now writes the OBP defaults (rtl/pkg/obp_nvram_pkg.sv from
   `tools/obp_nvram.py --sv`; diag-switch? false, boot-device disk) into an
   empty NVRAM: at power-up and after loading a blank image. sbus-probe-list
   must be "fe0123" (OBP computes it from the IOMMU; empty = no SBus probe,
   "screen not found." on ttya and a black screen). Board, the image of the
   defaults mounted: the banner 5 s after reset, Initializing Memory (all
   464 MB, every start, even non-diag) done and `ok` at 16 s; printenv
   matches OBP's own defaults. Build 2add56d (seed 3: clk_sys +1.020 ns,
   HDMI +0.303, 36,279 ALMs, rbf 14ea4dfe), loaded through the OSD with NO
   NVRAM image: banner by 9 s, Boot device sd@3,0 by 17 s, Solaris 8 to
   CDE; psrinfo both CPUs on-line, 464 MB, `init 0` clean, printenv
   diag-switch? false. OpenBIOS on 0a72b3c (the same Sun-format defaults)
   booted Solaris to its login. (b) matters only in diag mode now.
   Seen, not yet looked at: the TOD is not the MiSTer's time (Solaris
   showed Oct 7 on Oct 19; OpenBIOS warns "Time-of-day chip had incorrect
   date"); under OpenBIOS the screen's white is pale yellow, and Solaris's
   text console after X is white on white (palette).
7. As room is needed: the area list left in notes-area-20261019.md (the
   FPU's aligner/normaliser shifter shared, the cache valid bits in
   their own byte lanes, IU f_npc and the delay-slot search, ...).
