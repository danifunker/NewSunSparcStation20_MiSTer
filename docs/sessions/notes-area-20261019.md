# CPU area pass, 2026-10-18/19

Goal: room for the speed features (fetch, an L2) and for builds, by
writing the CPU's logic smaller with its behaviour cycle-exact. Work on
master on m900 (a box shared with other cores' sessions: every Quartus
flow under `flock /tmp/quartus.lock`; Verilator 5.044 in
`~/opt/verilator-5.044`, m900's own 5.028/5.032 reject this RTL).

## Method

- Four read-only reviews (IU, FPU, MMU + cpu_module, L1 caches) proposed
  ranked candidates; each was checked against the RTL before it was used.
- Per change: the block's unit bench (tb_fpu 602,375 checks, tb_srmmu
  137,480, tb_iu_alu, tb_iu_muldiv), lint, Verilator 5.044.
- `sim/iu/run.sh` as a 4-minute bit-exact regression: its whole run log
  (117,745 lines) must equal master b85de2c's. (Its QEMU reference is
  stale: the diff against `ss20-iu-qemu.log` is 270 lines on master too,
  the suite having grown since; compare runs with each other instead.)
- Batches: perfbench ticks equal the baseline and the CPU suite equals
  `tests/cpu/expected/ss20-core-hw.log` (84/0/5) on the 2-CPU model. The
  2-CPU suite takes ~90 minutes in the simulation.
- Area: Quartus synthesis of `cpu_module` alone
  (`scripts/qcheck.sh --synth cpu_module rtl/pkg/*.sv rtl/lib/*.sv
  rtl/cpu/*.sv`, ~2 minutes). Its per-entity numbers move by +-70 ALUTs
  between runs without a change to that entity (packing across the
  boundaries): trust totals and changes above ~50.

perfbench baseline on master (the ROM compiled on m900 with LLVM 18;
the ticks differ from docs/benchmarks/profile-20261014.md's, a different
compile): DHRY 67281, COPY 76443, ZERO 57326, CHASE 133006, SORT 128562,
FP 69898.

## Changes (cpu_module synthesis, one CPU: ALUTs / registers)

Baseline 12,072 / 5,671.

1. cpu_module: `bk_m` (ASI 0x38/0x49-0x4C) through one write port. The
   2026-10-17 pass meant it as MLAB, but three write addresses stopped
   the inference (warning 10999): it was 288 flops. -> 11,921 / 5,383.
   Committed b85de2c (checked alone: perfbench = baseline, suite 84/0/5).
2. l1cache: `fill_data` dropped (the demand beat goes to `rdata_q`, read
   only with done); `db_wdata` defaults to the diagnostic data (it
   matters only with `db_we`); the tag write words built from `r_pa` /
   `q_cur` above the tag bits only (a hit or a snoop match implies the
   tag); the I-cache's D-cache-only terms guarded with `!ICACHE`.
   D-cache: one read-data way mux for both hits, selected by the state
   and `done_q` (early) instead of `hit_now` / `fast_ld` (late): it
   targets the chip's tightest path (D-cache hit data into iu m_rdata);
   +~35 ALUTs, kept to see its timing in the fit. (Shared in the I-cache
   too it cost 136 ALUTs: not used there.)
3. l1cache bug fix (I-cache): a fill took the other half-line's valid bit
   from a snapshot made in S_LOOKUP and wrote it back at fill start and
   end, so a snoop, an invalidation or the flash that cleared that half
   during the fill was undone: stale code could execute (two CPUs, a
   run-time linker patching PLT code; or a flash right after writing
   code). `fill_other_gone` now records such an event and the fill's tag
   writes leave the other half invalid.
4. fpu: S_DONE is entered only for UNIMPL (its other branches and
   `fin_flags` go); the multiplier's low 48 bits registered as their OR
   (the sticky); the rounder's operand chosen by the op alone (a direct
   result never uses it); the direct results as a small datapath (an
   operand with optional NaN conversion/quieting and a sign change, a
   {sign, E ones/zero, F ones/zero} constant, or FTOI's integer) instead
   of a ~30-input priority chain (-136); the aligner's sticky by shift
   stage; a compare's equality from the adder (sum == 0 with a subtract
   forced for equal signs) instead of a 67-bit compare.
   fp_round: no renormalising shift after a carry out (the kept bits are
   zero either way; the implicit bit read as carry | kept_r[52]).
   fp_divsqrt: the remainder kept doubled (no first-step special case),
   the divisor in rad (unused by a divide), the exponent's -1/-2 at the
   load, the datapath out of reset (-79 ALUTs, -54 registers).
5. iu: the D-cache read-ahead index from `e_addr` (equal to `e_result`
   for memory ops; unused otherwise), off the result mux; `e_pc8` gone
   (the branch adder adds 8 for a predicted DCTI: its target is already
   being fetched). iu_alu: a tag trap returns the plain result (it writes
   and forwards nothing: the memory request is gated while W traps); one
   right shifter for srl/sra; one carry chain. iu_muldiv: the numerator
   and quotient in one shift register, one subtractor (its borrow is the
   compare).
6. srmmu: the lookup answer's pa/acc/m/pte_c formed whatever the flags
   (cpu_module reads them with hit only; tb_srmmu compares them only on
   a hit); the CAM's rows in `ram_sdp_bit` (new: a mixed-width M10K,
   1-bit write, 32-bit read; Quartus 17 infers it as Simple Dual Port
   512x1 / 16x32), so a rewrite writes the entry's bit without reading
   the row: the byte merge and the CAM read-address mux go (-261).

After 1-6: 10,979 / 4,844 (-1,093 ALUTs, -827 registers a CPU).

Tried and not kept: one adder in srmmu's level_addr (no change: Quartus
had shared them); the diagnostic read's way clamp in l1cache (+14).

## The fit (146a1de, seed 3)

36,250 ALMs (86 %), down from 37,197 (89 %, 81985c6 seed 3): -947 ALMs.
Every clock met: clk_sys (the CPUs) +0.692 ns (was +0.257), HDMI +0.320
(was +0.401), h2f_user0 +1.161, worst hold +0.201. 37,800 registers, 340
RAM blocks, 41 DSPs. rbf md5 08a60f05. The CPU clock's gain fits the
D-cache read mux change (its tightest path was the hit data into iu
m_rdata).

Board (2026-10-19, the MiSTerPi at 192.168.99.143, rbf 08a60f05):
- the CPU suite: CPUTEST DONE pass=84 fail=0 skip=5, the output equal to
  ss20-core-hw.log (hwtest cpu; the core was loaded through
  /dev/MiSTer_cmd then, which the user has since ruled out: loads go
  through the OSD from here on);
- NetBSD 11 GENERIC under OpenBIOS (boot.rom 72d937ea): root shell,
  uname, a clean `shutdown -h`;
- the core loaded through the OSD (F12, System, Core), HD0 mounted and
  Load ROM = SunOBP.rom through the OSD: the official Sun OBP 2.25 with
  both CPUs ("SPARCstation 20 MP (2 X 390Z50)", 464 MB); with no NVRAM
  image it set its defaults and diag-switch? true, tested all 464 MB and
  tried the network; Stop-A, `boot disk`: Solaris 8 to the CDE login,
  then a command-line login: psrinfo shows both CPUs on-line; `init 0`
  back to ok, clean.

## Left on the list

From the reviews, not yet done: IU f_npc as "pc + 4" plus a flag,
the delay slot found by position, decode fields from the instruction
word, pc[1:0] dropped; FPU one shifter for the aligner and the
normaliser, FTOI's negate through the adder, res_bits_q/fin_bits merged;
MMU the CAM rewrite's key from the image RAM, the PA's low 12 bits from
the VA, w_result into reg_rdata_o; l1cache the valid bits in their own
byte lanes, the store-to-load forward that cannot happen (with an
assertion first). Also seen: a possible false store hit in S_WRITE_WAIT
if idx_i moves (low confidence, not examined yet).

## After: VRAM on SDRAM and the D-cache fixes (2026-10-19)

- e5b0e09 (D-cache: a waiting store checks its hit once; the dead store
  forward goes; tb_l1cache reproduces the bug on the old cache).
- 1124d48 (the TCX's VRAM and scan-out on SDRAM; two DDR3 arbiters gone):
  fit seed 3: 35,824 ALMs (85 %), clk_sys +0.902 ns, HDMI +0.509, hold
  +0.194 (rbf bfb12994). Board: Solaris's CDE login with every glyph
  smeared, fills right: DQM byte masks ignored on the board.
- 4be08f5 (partial VRAM writes read, merge and write whole; the chip
  model's IGNORE_DQM shows the old adapter failing): fit seed 3: 36,107
  ALMs (86 %), clk_sys +0.715 ns, **HDMI -0.083 ns** (the scaler's
  seed-sensitive path: sweep), hold +0.190 (rbf a6343955). Board,
  loaded through the OSD: OpenBIOS booting Solaris 8 to the CDE login,
  drawn right; psrinfo both CPUs; `hwtest cpu` 84/0/5 (now loading
  through scripts/board/osd_load.sh); the Sun OBP 2.25's banner and
  scrolling text right. rd_mode 0 (clk edge 3) reads correctly at 55 MHz.
- perfbench in the simulation (one CPU, the scan-out running): CHASE
  133,006 -> 123,409 ticks (-7.2 %) with the scan-out off DDR3; the rest
  within a few ticks.

