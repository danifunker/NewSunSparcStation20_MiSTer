# SunSparcStation20_MiSTer

A Sun **SPARCstation 20** (sun4m) for the MiSTer FPGA, written from
scratch in SystemVerilog: SuperSPARC-compatible CPUs, the sun4m chipset,
SCSI disks and CD-ROM, Ethernet, the TCX frame buffer, Sun keyboard and
mouse, and CS4231 audio. Licensed GPL-2.0-or-later.

## Status (2026-10-19)

The core runs with **two SuperSPARC CPU modules at 55 MHz** (about 86 %
of the FPGA, every clock met). It boots under two PROMs:

- **OpenBIOS** (built from `bios/`; a matching `boot.rom` is in
  [releases/](releases/README.md)): NetBSD 11 and Solaris 8 from disk,
  NetBSD's install CD, CD audio, and the network.
- **Sun's OBP 2.25** (from a real SS20; not distributed here): its
  diagnostic POST on both CPUs with no errors, and Solaris 8 with both
  CPUs on-line.

The bare-metal CPU and chipset test suite (`tests/cpu`) passes on the
board (84 pass, 0 fail, 5 skip). What is next:
[docs/VERILOG-PLAN.md](docs/VERILOG-PLAN.md) §9 and §10.

**The core requires the MiSTer SDRAM module**: the TCX's frame buffer is
on it.

## Getting started

1. Copy the rbf from [releases/](releases/README.md) to `_Unstable/` (or
   `_Computer/`) on the SD card.
2. Copy `releases/boot.rom` (OpenBIOS) to `games/SunSparcStation/boot.rom`,
   or use Sun's own PROM there instead.
3. Put disk images in `games/SunSparcStation/` and mount them from the
   OSD ([docs/disk-images.md](docs/disk-images.md) says how the test
   images were made).

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

### The NVRAM image (P5)

The NVRAM holds the PROM's settings (`printenv`/`setenv`), the IDPROM
(the Ethernet address and hostid) and the clock. Without an image in the
NVRAM slot it lives only until the core is loaded again; with an 8 KB
image file (any name, `.nvr`) the core loads it at start-up and writes
every change back, so `setenv` sticks.

- **No image, or a new blank file**: the core gives it an IDPROM (from
  the MiSTer's ID, or 08:00:20:12:34:56) and the Sun OBP 2.25's defaults
  (`auto-boot? true`, `boot-device disk`, `diag-switch? false`, the
  console on the keyboard and screen), so the Sun PROM starts at once:
  it tests 1 MB of memory (`selftest-#megs`) and boots the disk. A blank
  file is filled in and saved at its first load. Make one with
  `dd if=/dev/zero of=sun.nvr bs=8192 count=1`, or with the options
  already set: `tools/obp_nvram.py --new sun.nvr output-device=ttya`.
- **The two PROMs keep different formats.** OpenBIOS reads the Sun
  format as unformatted ("nvram error detected"), uses its own defaults
  and overwrites the image only at its first `setenv`. The Sun OBP reads
  an OpenBIOS image (`tools/mknvram.py`) as a bad checksum: it then sets
  its defaults with `diag-switch? true`, which runs the full POST and a
  test of all of memory (minutes at 464 MB) and boots the network, at
  every start until `setenv diag-switch? false`. Keep one image per PROM.

## What is here

| Path | What | License |
|---|---|---|
| `rtl/cpu/` | the CPU module: `iu` (pipeline) and its parts, `fpu` (`fp_unpack`, `fp_round`, `fp_divsqrt`), `srmmu`, `l1cache`, `cpu_module` (+ benches in `tb/`) | ours |
| `rtl/machine/` | the machine: `ss20_machine` (NCPU modules, default 2), `ss20_mbus`, `ss20_memctl`, `ss20_iobridge`, `ss20_loader`, `ss20_uart`, `ss20_osd`, `vram_sdram` | ours |
| `rtl/obio/` | the sun4m system devices: `slavio_timer`, `slavio_intctl`, `slavio_misc`, `m48t08`, `escc` | ours |
| `rtl/sun4m/` | the SBus and on-board devices: `ss20_msi` (IOMMU), `emc`, `dma2`, `esp` + `scsi_targets`, `lance`, `tcx`, `cs4231` (+ APC), `fdc82077`, `sun_kbd`, `sun_mouse`, `kbd_beep`; the FCode PROMs `tcx_fcode`, `cs4231_fcode`, `slote_fcode` | ours, except below |
| `rtl/mister/` | the MiSTer side: `ddram_arb`, `sdram64`, `eth_hps`, `nvram_sd` | ours |
| `rtl/pkg/`, `rtl/lib/`, `rtl/video/` | packages, RAM templates, the hang reporter, video timing | ours |
| `rtl/tb/` | `run.sh`: lint every module and run every block bench under Verilator | ours |
| `sim/` | Verilator harnesses: the whole machine (`build.sh`, `sim_main.cpp`, `run-*.sh`), the CPU module (`sim/cpu/`), the IU (`sim/iu/`) | ours |
| `tests/cpu/`, `tests/fpu/` | the bare-metal SPARC V8 / sun4m suite that runs as the boot PROM, with QEMU references; FPU vectors | ours |
| `syn/` | a Quartus fit of the CPU module alone, for timing work | ours |
| `scripts/` | build, deploy, board regression (`hwtest.sh`, `board/`) | ours |
| `tools/` | ROM analysis (`romdis/`), FCode, NVRAM, CD-audio, UFS, NeXTSTEP, UnixBench and MiSTer tools | ours |
| `docs/` | the plan ([VERILOG-PLAN.md](docs/VERILOG-PLAN.md)), the architecture (`arch/`), our analysis of the SS20 and SS5 PROMs (`rom-disassembly/`, no Sun code), benchmarks, the reference index | ours |
| `bios/` | OpenBIOS and fcode-utils; `scripts/build-bios.sh` builds the boot ROM | GPL-2 |
| `sys/` | the Template_MiSTer framework, unchanged | GPL-2+ |
| `releases/` | test builds: the rbf and the OpenBIOS `boot.rom` it needs | GPL |

Not here, on purpose: Sun PROM images and anything reproduced from them.

## Building

- The core: Quartus 17.0.2 Lite, `scripts/build.sh 20` (output in
  `output_files/`).
- OpenBIOS: `scripts/build-bios.sh` (a SPARC cross GCC and `xsltproc`).
- Benches: `rtl/tb/run.sh` (Verilator 5); the CPU suite in simulation:
  `sim/cpu/run.sh`; the whole machine: `sim/build.sh` and
  [sim/README.md](sim/README.md).

## Thank you

A big thank you to **Grabulosaure**, whose SPARCstation work on FPGA
inspired us to write this core.

## Third-party components

These keep their own licenses and authors:

- **OpenBIOS** (`bios/openbios/`), GPL-2.0: the OpenBIOS project, with
  the sparc32 port for FPGA SPARCstations (`CONFIG_TACUS`) by
  Grabulosaure, which we extend for this machine.
- **fcode-utils** (`bios/fcode-utils/`), GPL-2.0: the OpenBIOS project.
- **The TCX FCode PROM** (`rtl/sun4m/tcx_fcode.sv`), GPL-2.0: the image
  QEMU ships (`pc-bios/QEMU,tcx.bin`), built from OpenBIOS's
  `drivers/tcx.fs`.
- **Template_MiSTer** (`sys/`), GPL-2.0+: the MiSTer project
  (Sorgelig and contributors).
- **Wish7990** (`rtl/sun4m/wish7990_*.sv`, `rtl/pkg/wish7990_pkg.sv`),
  MIT: MelkhiorVintageComputing, the Am7990 LANCE core.

## License

GPL-2.0-or-later for our files ([LICENSE](LICENSE)); the components above
under their own licenses. The Sun PROM images are Sun/Oracle copyright
and are never committed.
