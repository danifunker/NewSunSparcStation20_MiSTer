# SunSparcStation20 for MiSTer: initial release (20261019)

A Sun SPARCstation 20 (sun4m), written from scratch in SystemVerilog.

| File | What |
|---|---|
| `SunSparcStation20_20261019.rbf` | the core (master 2add56d, md5 14ea4dfe) |
| `boot.rom` | OpenBIOS for it, built from `bios/` (md5 72d937ea) |

**Requires the MiSTer SDRAM module.**

> **Please test the video.** The frame buffer has just moved from the
> DDR3 to the SDRAM module. Report anything wrong on screen: garbage,
> wrong colours, a black screen, slow drawing. Say which PROM and OS
> you used (console text, CDE/X, NeXTSTEP) and which SDRAM module you
> have.

## Features

- Two SuperSPARC (TMS390Z50) CPU modules at 55 MHz, SMP: integer unit,
  IEEE single/double FPU, SRMMU, 20 KB I-cache and 16 KB D-cache with
  bus snooping.
- 64, 128, 256 or 464 MB of memory.
- Boots OpenBIOS (included) or Sun's own OBP 2.25 (not included), as
  `boot.rom` or through Load ROM.
- SCSI (ESP + DMA2): two hard disks and a CD-ROM, with CD audio.
- Ethernet (LANCE) through the MiSTer's network.
- TCX frame buffer: 1024x768, 8-bit, hardware cursor, blit and stipple
  acceleration. Or the console on the serial port.
- Sun Type 5 keyboard (layouts, Alt/Win swap) with bell and click; Sun
  mouse.
- CS4231 audio.
- Serial port A on the MiSTer's UART.
- NVRAM and clock (M48T08), saved to an image file; the IDPROM
  (Ethernet address, hostid) from the MiSTer's ID. Without an image the
  Sun PROM starts with its defaults and boots the disk, with no full POST.
- OSD in the computer cores' layout.

## Install

1. Copy `SunSparcStation20_20261019.rbf` to the SD card's `_Unstable/`
   (or `_Computer/`).
2. Copy `boot.rom` to `games/SunSparcStation/boot.rom`.
3. Put disk images in `games/SunSparcStation/` and mount them from the
   OSD: Pri. disk is SCSI target 3, CD-ROM target 6.
   [docs/disk-images.md](../docs/disk-images.md) says how to make them.
4. Give the core its own video mode in `MiSTer.ini` (a 720p output drops
   pixel columns from Sun's fonts):

   ```ini
   [SunSparcStation]
   video_mode=1        ; 1024x768@60
   ```

## Tested

- NetBSD 11: disk, install CD, network, audio; the MP kernel on both
  CPUs under the Sun PROM.
- Solaris 8: disk, CDE, both CPUs, network, audio; under both PROMs.
- NeXTSTEP 3.3: installs and runs the Workspace; under both PROMs.
- Sun OBP 2.25: the POST on both CPUs, the network, audio and CD tests.
- CD audio.
- The CPU and chipset test suite: 84 pass, 0 fail, 5 skip.
- 30-minute stress runs under Solaris, both PROMs.

## Limitations

- At most 464 MB of memory.
- CPUs at 55 MHz with no external cache (no SuperCache/MXCC).
- One frame buffer: the TCX, 8-bit only (no 24-bit TCX, cgsix or SX).
- No floppy drive, no parallel port device, no ISDN (DBRI) audio.
- Sun's OBP is not included: supply your own SS20 PROM image.
- The network and CD audio need a current Main_MiSTer (Sun support
  merged 2026-10-06/07).

## Known issues

- The clock does not take the MiSTer's time.
- Under OpenBIOS, white shows pale yellow; Solaris's text console after
  X exits is white on white.
- `psrinfo` reports 40 MHz under the Sun PROM (its own estimate).
- Under OpenBIOS, NetBSD's MP kernel does not start the second CPU.
- NeXTSTEP's GUI needs the console on the screen and keyboard.
