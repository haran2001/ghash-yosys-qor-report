# 02 — A6 pipeline-granularity sweep

First complete A6 (`ITER_PER_STAGE` variation) sweep from `yosys synth_xilinx -family xcup -nodsp` on the cleaned baseline RTL, run on macOS. Same RTL, only `ITER_PER_STAGE` chparam changes.

## Raw numbers

| Variant | `ITER_PER_STAGE` | Latency (cycles) | LUTs | MUXFx | FFs | Δ LUTs vs baseline |
|---|---:|---:|---:|---:|---:|---:|
| baseline | 128 | 1 | 16,769 | 0 | 129 | +0.0% |
| **A6 ITER=32** | **32** | **4** | **11,468** | **2,136** | **1,092** | **−31.6%** |
| A6 ITER=8 | 8 | 16 | 8,915 | 1,895 | 4,944 | −46.8% |
| A6 ITER=2 | 2 | 64 | 8,634 | 0 | 20,352 | −48.5% |

DSP / BRAM / URAM are zero across all four. The combinational core is pure XOR/mux fabric.

## Why LUT count drops as `ITER_PER_STAGE` shrinks

Initial prediction was that these would be flat across `ITER_PER_STAGE`. They aren't. The mechanism:

- At ITER=128 the entire 128-iteration GHASH chain is one fully-combinational cone. Yosys ABC sees a 128-deep XOR/mux network and folds it into 16,384 LUT3 + 385 LUT2.
- At smaller ITER, each pipeline stage is a smaller, independent ABC problem. ABC retimes/merges within a stage but **cannot move logic across pipeline-register boundaries.**
- That sounds like it would *help* the unsplit case, but in practice the giant 128-deep cone overwhelms ABC's heuristics and it leaves redundant LUTs on the table. Breaking the cone into smaller pieces lets ABC actually finish its optimisation passes.

Net: **A6 reduces yosys-reported LUT count, but the savings are partly artifacts of yosys-ABC scaling, not durable Vivado wins.** Vivado's `opt_design` and `phys_opt_design` are expected to close most of this gap on the ITER=128 baseline.

## Why the yosys-composite ranking is misleading

The composite score with the customer pain-ranked weights (`w_lut=1.5, w_cong=0.9, w_latency=1.2, w_fmax=0.7`) returns:

```text
baseline ITER=128   score 3.844    ← "winner" under yosys
A6 ITER=32          score 5.250
A6 ITER=8           score 19.172
A6 ITER=2           score 76.719
```

The "winner" only wins because **the Fmax term is zero** (yosys produces no Fmax). In reality:

- ITER=128 is a fully combinational 128-iteration GHASH chain. On `xcku3p-2-e`, this is almost certainly *unable* to meet 322 MHz. Vivado WNS would be deeply negative.
- ITER=32 (customer's default) is what they're actually shipping. The 4-cycle latency is the baseline they're measuring everything against. This is the row that **must** be the starting point for any improvement claim.
- ITER=8 / ITER=2 trade latency for Fmax headroom but blow past the 5–6 cycle budget.

Honest reading:

> Yosys-only synth, with `w_fmax × 0` from missing timing data, makes the most combinational variant look best. The score is meaningful for **LUT comparisons within the same `iter_per_stage`** but **uninformative across `iter_per_stage`** until Vivado timing data is available.

## Real customer-relevant findings

1. **Per-atomic LUT count is fixed** at 128 LUT3 + 3 LUT2 regardless of how the right shift is written (A1 syntactic experiment confirms this — see result 03). Every output bit is already a 3-input function of (X-bit, V-bit, Z-bit), packed into a single LUT3.
2. **Only semantic rewrites can move per-atomic cost.** The structural candidates that can actually shift the LUT count are A2 (bit-parallel), A3 (Karatsuba), A8 (DSP48E2 offload).
3. **The A6 axis is real but yosys can't price it.** ITER=32 has meaningful structural differences from ITER=128 (more pipeline registers, smaller ABC scope, potentially different P&R outcomes) that **only Vivado P&R can evaluate**.

## How to read this for the customer

> First yosys synth pass on the cleaned RTL (open-source flow, no Vivado on this Mac) shows the per-atomic combinational logic is uniform across all 128 iterations and consists of one LUT3 per output bit — there's no syntactic-only optimisation that changes this. To meaningfully reduce per-multiplier LUT count we need a structural rewrite (A2 bit-parallel, A3 Karatsuba, or A8 DSP48E2-as-XOR offload). The pipeline granularity (`ITER_PER_STAGE`) does affect yosys-reported LUT count by up to 50% but these numbers are not reliable cross-`ITER_PER_STAGE` until we have Vivado WNS/Fmax from the customer environment. A2 and A8 are queued next, and we need ~15 minutes of Vivado time on `xcku3p` to calibrate the open-source ranking against the customer's actual numbers.

## What's NOT done yet

- **A2 / A3 bit-parallel rewrites** — see results 03 and 04.
- **A8 DSP48E2 offload** — promising on KU3P, conditional on the customer's free-DSP budget.
- **Real SV TB run** — iverilog hangs on packed-2D-array sim on macOS; needs Verilator binary mode or Vivado xsim. Python golden is the only verification today.
- **Vivado calibration** — every quotable number must be sanity-checked against at least one Vivado run on `xcku3p-ffvb676-2-e`. Until then, all LUT figures here are ~2× of what Vivado would report.
