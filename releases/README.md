# Test builds

| File | What |
|---|---|
| `SunSparcStation20_20261019.rbf` | the core, master at 2add56d (seed 3, md5 14ea4dfe): two SuperSPARC CPUs at 55 MHz |
| `boot.rom` | OpenBIOS for it, built from `bios/` at 16938f1 (md5 72d937ea) |

**This build needs the MiSTer SDRAM module**: the TCX's frame buffer is
on it. Without the module there is no picture.

The core needs this OpenBIOS: an older one lists all memory in one
`/memory` entry and Solaris 8 panics early, and before 16938f1 the second
CPU, left waiting, slowed the first one down (the CD boot took 149 s
instead of 123 s).

## Install

1. Copy `SunSparcStation20_20261019.rbf` to the SD card's `_Unstable/`
   (or `_Computer/`).
2. Copy `boot.rom` to `games/SunSparcStation/boot.rom` (Main loads it
   when the core starts).
3. Put disk images in `games/SunSparcStation/` and mount them from the
   OSD: a NetBSD or Solaris disk on Pri. disk (SCSI target 3), a CD image
   on CD-ROM (target 6). [docs/disk-images.md](../docs/disk-images.md)
   says how the test images were made.

4. Video: the screen is 1024x768 (4:3, square pixels). On a 720p output
   Main shrinks it to 960x720 and drops a pixel column and row in every
   16, which breaks Sun's one-pixel fonts. Give the core its own mode in
   `MiSTer.ini`:

   ```ini
   [SunSparcStation]
   video_mode=1        ; 1024x768@60: the picture 1:1
   ```

   (or, on a 1080p TV, `video_mode=8` with `vscale_mode=1`: integer
   scaling, 1024x768 unscaled with borders).

Sun's own PROM (OBP 2.25, from a real SS20) also runs, as `boot.rom` or
through the OSD's Load ROM; it is Sun's code and is not distributed here.

**The NVRAM** (OSD Advanced, an optional 8 KB image file): without one,
or with a new blank file, the core now starts with the Sun PROM's
defaults (`diag-switch? false`, `auto-boot? true`, `boot-device disk`),
so the Sun PROM no longer runs its full POST and a minutes-long test of
all memory at every start: it boots the disk in about 17 s. An image
keeps `setenv` changes; use one image per PROM (the main
[README](../README.md#the-nvram-image-p5)).

## New since the 20261017 build

- The Sun PROM starts at once without an NVRAM image (above).
- The TCX's frame buffer on the SDRAM module instead of the DDR3 (plan
  phase 13, step 1): the CPUs no longer share the DDR3 with the screen.
- The OSD follows the MiSTer computer cores' layout: Pri. disk, Disk,
  CD-ROM, Load ROM (this session only), Memory, Network, Video, Input,
  Advanced; settings wait for "Apply and reset".
- A smaller CPU (about 1,100 ALUTs less each) with more timing margin;
  an I-cache fix (a fill could revalidate a half-line a snoop had
  cleared) and a D-cache fix (a waiting store could update the wrong
  line if the index moved).

## Tested on the board with this build

Under the Sun PROM with no NVRAM image: the banner, then the disk boot
17 s after the core loads, Solaris 8 to the CDE desktop with both CPUs
on-line and 464 MB, a clean `init 0`. Every clock meets timing in this
build.

On the builds just before it (the same CPU and frame buffer): the CPU
suite (84 pass, 0 fail, 5 skip); Solaris 8 under OpenBIOS to the CDE
login; NetBSD 11 under OpenBIOS; the Sun PROM's banner and screen text.
Not yet re-run since the 20261017 build: the CD, the network, audio,
NeXTSTEP and the Sun PROM's POST.

Known: the time of day is not the MiSTer's (Solaris showed a date 12 days
behind; OpenBIOS warns "Time-of-day chip had incorrect date"); under
OpenBIOS the screen's white shows pale yellow, and Solaris's text console
after X exits is white on white; Solaris's mouse driver ignores the first
48 mouse packets after X starts (about a second of movement), as on a
real machine; `psrinfo` reports 40 MHz under the Sun PROM (its own
estimate from a delay loop, not the clock); NeXTSTEP's GUI needs the
console on the screen and keyboard; under OpenBIOS, NetBSD's MP kernel
does not start the second CPU.
