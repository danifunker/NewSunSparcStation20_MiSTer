# SunSparcStation20_MiSTer: the Verilog rewrite

A new Sun SPARCstation 20 core for MiSTer, written from scratch in
SystemVerilog and licensed GPL-2.0-or-later.

It starts fresh, with no history from the VHDL core
(MiSTer-devel/SunSparcStation20_MiSTer). That core is built on
Grabulosaure's CPU and chipset, published with "All rights reserved" and
no license. This repository holds only what can be reused: our own code,
the GPL firmware and framework, the tests, the tools and the
documentation.

**Start with [docs/VERILOG-PLAN.md](docs/VERILOG-PLAN.md)**: the clean-room
rules, the compatibility contracts with Main and OpenBIOS, the
architecture, the phases and the requirements learned on the VHDL core.

## Status (2026-10-13)

The core runs on the board with **two SuperSPARC CPU modules** at 55 MHz
(37.5k ALMs, 89 %, every clock met). Under OpenBIOS (built from `bios/`,
which must match the core: see the handoff) it boots NetBSD 11 and
Solaris 8 from disk, NetBSD's install CD, plays CD audio, and runs the
network on both. Sun's OBP 2.25 runs its diagnostic POST on both CPUs
with no errors. The CPU suite passes on the board (82 pass, 0 fail, 5
skip). Where the work stands, and what is next:
[docs/sessions/HANDOFF-20261009.md](docs/sessions/HANDOFF-20261009.md)
("Current state" at the top); the phases: [docs/VERILOG-PLAN.md](docs/VERILOG-PLAN.md) §9.

**The core requires the MiSTer SDRAM module** (since 2026-10-19: the TCX's
frame buffer is on it).

## The OSD

The menu follows the computer cores' OSD standard (hp9000-715_MiSTer
`docs/osd-standard.md`). Disks, the CD, Load ROM and the Input page act
at once; every other setting waits for **Apply and reset** (or any other
reset), and "Changes pending" shows until then.

```
SunSparcStation
  Pri. disk (ID 3)     <- the disk the PROM boots by default
  Disk (ID 1)
  CD-ROM (ID 6)
  Load ROM             <- this session only; a normal start loads boot.rom
  Memory               464MB, 256MB, 128MB, 64MB
  Network interface    eth0, None, eth1, macvlan, tap0
  Video     (P2)       Display (Graphics / Serial console), Aspect ratio, Scale
  Input     (P3)       Keyboard layout, Swap Alt/Win keys
  Advanced  (P5)       Autoboot, NVRAM image
   Changes pending
  Apply and reset
```

`Pri.` names the default: the PROM's `boot-device` (NVRAM) can boot
another disk, the CD or the network. Load ROM is not remembered, so an
MGL can start the core with another ROM (Sun's OBP, the CPU suite) without
changing the default:

```xml
<mistergamedescription>
  <rbf>_Unstable/SunSparcStation20</rbf>
  <file delay="1" type="f" index="0" path="../../../../games/SunSparcStation/obp225.rom"/>
</mistergamedescription>
```

To make another ROM the default, copy it to
`games/SunSparcStation/boot.rom`.

## What is here

| Path | What | License |
|---|---|---|
| `bios/` | OpenBIOS (TACUS port) and fcode-utils; `scripts/build-bios.sh` builds the boot ROM | GPL-2 |
| `sys/` | Template_MiSTer framework | GPL-2+ |
| `rtl/pkg/`, `rtl/lib/` | the bus and memory packages (`iobus_pkg`, `mem_pkg`, `sbus_pkg`, `sun4m_pkg`, `ss20_ddr_pkg`), building blocks (`ram_tdp_be`, `ram_sdp_be`) | ours |
| `rtl/cpu/` | the CPU module: `iu` (pipeline) and its parts, `fpu` (`fp_unpack`, `fp_round`, `fp_divsqrt`), `srmmu`, `l1cache`, `cpu_module` (+ benches in `tb/`) | ours |
| `rtl/machine/` | the machine: `ss20_machine` (NCPU modules, default 2), `ss20_mbus`, `ss20_memctl`, `ss20_iobridge`, `ss20_loader`, `ss20_uart`, `ss20_osd` | ours |
| `rtl/obio/` | the new sun4m system devices: `slavio_timer`, `slavio_intctl`, `slavio_misc`, `m48t08`, `escc` (+ benches in `tb/`) | ours |
| `rtl/tb/` | `run.sh`: lint every module and run every block bench under Verilator; `tb_pkg.sv` | ours |
| `rtl/mister/` | `ddram_arb.sv`, `eth_hps.sv`, `nvram_sd.sv` (+ benches; the VHDL originals are in git history) | ours |
| `rtl/sun4m/` | the SBus and on-board devices: `ss20_msi` (IOMMU), `emc` (memory controller registers), `dma2`, `esp` + `scsi_targets`, `lance` (+ `wish7990_*`, MIT), `tcx`, `cs4231` (+ APC), `fdc82077`, `sun_kbd`, `sun_mouse`, `kbd_beep`; the FCode PROMs `tcx_fcode`, `cs4231_fcode`, `slote_fcode` | ours / MIT (Wish7990) / GPL-2 (TCX FCode) |
| `tests/cpu/` | bare-metal SPARC V8 / sun4m test suite that runs as the boot PROM, QEMU references | ours |
| `sim/` | Verilator harnesses: the whole machine (`build.sh`, `sim_main.cpp`: Main's hps_io, SD, DDR, UART, Ethernet models; `run-*.sh`), the CPU module (`sim/cpu/`), the IU (`sim/iu/`) | ours |
| `scripts/` | build, deploy, board regression (`hwtest.sh`, `board/`) | ours |
| `tools/` | ROM analysis (`romdis/`), FCode, NVRAM, CD-audio, UFS, NeXTSTEP and MiSTer tools; `debugarm/pcdump.c` | ours |
| `docs/rom-disassembly/` | our analysis of the SS20 OBP 2.25 and SS5 OBP 2.15 PROMs (no Sun code) | ours |
| `docs/arch/` | the bus definitions (`bus.md`) and the physical address map (`memory-map.md`) | ours |
| `docs/references/` | the index of the reference library (datasheets, Sun specs, emulator and OS sources) kept locally in `scratch/reference/` | ours |
| `docs/legacy/` | the VHDL core's plans, gap audits, design notes and session hand-offs | ours |
| `releases/` | test builds: the core's rbf and the OpenBIOS `boot.rom` it needs ([releases/README.md](releases/README.md)) | GPL |
| `scratch/` | local only, gitignored: reference manuals, Sun PROM images and their full disassembly, candidate RTL from other cores (see `scratch/README.md`) | not in git |

Not here, on purpose: Grabulosaure's RTL and tools; Keith Conger's CS4231
model; Sun PROM images and their disassembly listings. Test builds (an
rbf and the matching OpenBIOS ROM) are in [releases/](releases/README.md).

## License

GPL-2.0-or-later for our files ([LICENSE](LICENSE)); OpenBIOS and the
TCX/CG3 FCode it carries are GPL-2.0; `sys/` is GPL-2.0+. The Sun PROM
images are Sun/Oracle copyright and are never committed.
