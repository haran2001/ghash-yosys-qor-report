# GHASH GF(2^128) — Final Recommendation

The shortest version of the report. What RTL to ship, why, and what to do next.

For the methodology + reasoning trace see [`README.md`](README.md). For the one-page condensed report see [`SUMMARY.md`](SUMMARY.md).

---

## 1. Final recommended RTL

**`ghash_mult` with `ITER_PER_STAGE = 32`** — the chained-iterative pipelined design at 4-stage granularity (variant tag `A6_ITER=32`).

```text
top:   ghash_mult #(.ITER_PER_STAGE(32))
       4 pipeline stages × 32 atomics-per-stage  =  128 GHASH iterations
       per-stage X/H/Z/V pipeline registers, i_vld / i_ack backpressure

each stage:  ghash_mult_stage #(.START_ITER(s*32), .END_ITER(s*32+31))
             32 ghash_mult_atomic instances chained combinationally

atomic:      ghash_mult_atomic #(.ITERATION_NO(n))
             conditional-XOR of Z with V, xtime(V) right-shift,
             reduction polynomial 8'hE1 on byte 0 of V
```

RTL hygiene applied to the original customer source (these are kept, they cost nothing and prevent foot-guns):

- `default_nettype none` at file scope.
- Explicit `bit_int` and `lsb` declarations (no implicit single-bit nets).
- `logic` outputs instead of `output reg` with continuous assigns.
- Named generate blocks (`gen_stage`, `gen_atomic`, `gen_first_byte`, `gen_other_bytes`, …).
- `ITER_PER_STAGE` divisibility check (`$fatal` if it doesn't divide 128 exactly).
- Consistent `logic [15:0][7:0]` byte ordering with the existing TB and golden.

**Functional contract** is unchanged: same `i_X` / `i_H` / `i_vld` / `i_ack` interface, same byte ordering, same NIST-like vectors pass.

### Measured QoR for the recommended RTL (Yosys-Tier-A, `xcku3p-ffvb676-2-e`)

| Metric | Value | Δ vs `ITER_PER_STAGE=128` baseline |
|---|---:|---:|
| yosys LCs per instance | 11,468 | −31.6% |
| MUXFx per instance | 2,136 | +2,136 |
| FFs per instance | 1,092 | +963 |
| Latency | 4 cycles | +3 cycles |
| Inferred Vivado CLB-LUT (÷ 2.05 calibration) | **~5,595** | **~−30%** vs customer-reported ~8,000 |
| Inferred aggregate (× 8 instances) | **~44,800 CLB-LUTs** | **~−19,200 LUTs** vs ~64,000 |

The Vivado-LUT numbers are projections using a single anchor calibration point. **The headline `~−30% per instance` must be confirmed by one Vivado P&R pass on baseline + `A6 ITER=32` before it goes into a customer commitment.**

---

## 2. Optimisations that led to this RTL

Variants Newton tried, in order, and what was learned from each. Only the optimisations marked **kept** are reflected in the recommended RTL above.

### 2.1 RTL hygiene rewrite — **kept**

Cleaned the customer's source: `default_nettype none`, explicit signal declarations, named generates, divisibility guard, `logic` outputs. **Zero LUT impact, but it lets every later step run without elaboration foot-guns.** This is the table-stakes step, not a QoR optimisation.

### 2.2 A1 — Syntactic XOR / shift rewrites — **rejected (no-op)**

Tried respelling the per-byte right-shift as a flat 128-bit concat and "exposing" the XOR tree. Yosys output was bit-identical: 128 LUT3 + 3 LUT2 per atomic, regardless of spelling. ABC canonicalises per output bit before tech mapping; every syntactic variant collapses to the same boolean function. **Permanently dropped.**

Lesson: per-atomic LUT count cannot be moved by syntactic rewrites. Only *semantic* restructuring (algorithmic or DSP offload) can.

### 2.3 A6 — Pipeline-granularity sweep — **partially kept (`ITER_PER_STAGE=32` selected)**

Swept `ITER_PER_STAGE` ∈ {128, 32, 8, 2}. Yosys-LC numbers:

```text
ITER=128 :  16,769 LUT  /     0 MUXFx /    129 FF  /   1 cycle
ITER=32  :  11,468 LUT  / 2,136 MUXFx /  1,092 FF  /   4 cycles  ← selected
ITER=8   :   8,915 LUT  / 1,895 MUXFx /  4,944 FF  /  16 cycles
ITER=2   :   8,634 LUT  /     0 MUXFx / 20,352 FF  /  64 cycles
```

`ITER=32` is the only setting inside the customer's 5–6 cycle latency budget that also delivers a large LUT win against the unpipelined baseline. The deeper splits (ITER=8, ITER=2) report bigger LUT savings but are out of latency budget for the customer's HFT wrapper.

**Caveat in the source-of-improvement attribution:** part of the `−31.6%` is `rtl_pipeline` (more registers, smaller ABC cones, better optimisation completion) and part is an `abc_cone_scaling_artifact` — yosys's ABC stalls on the 128-deep cone of the unpipelined baseline and leaves redundant LUTs that Vivado's `opt_design` would compress. Honest framing: expect the Vivado-side gap to be smaller than 31.6%, plausibly ~10–20% per instance, but still material at × 8 aggregate.

### 2.4 A2 — Combinational bit-parallel rewrite — **rejected for deployment (Fmax-incompatible)**

Fully-combinational bit-parallel matrix: `shifts[i] = xtime(shifts[i-1])`, then `Z = OR_i (X-bit[i] AND shifts[i])`. Yosys: 14,315 LUT / 5,842 MUXFx / 1 cycle, **−14.6% LUTs**. ABC packed ~70% into LUT6 by extracting cross-bit common-subexpression sharing across the entire combinational cone.

Rejected because the 128-deep `xtime` chain is a critical path well above 10 ns on `xcku3p-2-e` and will miss 322 MHz badly. Verified bit-exact against the Python golden over 5,000 random vectors, so the algorithm is correct — it just can't meet timing.

**Kept for the record as: the bit-parallel form is the only RTL family that gives ABC cross-bit sharing.** It's the right starting point if the customer ever wants a 1-cycle / ultra-low-Fmax variant.

### 2.5 A2b — Pipelined bit-parallel — **rejected (negative result)**

Pipelined the bit-parallel matrix at K ∈ {64, 32, 16} stage widths. Numbers:

```text
A2b K=64 : 15,035 LUT / 6,153 MUXFx /  3 cycles    ← regression vs A2 comb
A2b K=32 : 12,305 LUT / 2,905 MUXFx /  5 cycles    ← dominated by A6 ITER=32
A2b K=16 : 11,779 LUT /   284 MUXFx /  9 cycles    ← out of latency budget
```

At matched latency, A2b never beats chained A6 on both LUT and MUXFx. Mechanism: pipelining the bit-parallel matrix cuts ABC's cross-bit common-subexpression scope, leaving each stage with the same K-deep cone the chained variant already exposes for free — and chained ends up slightly more LUT-efficient because it has fewer wide-mux fanouts.

**Kept as a negative result** so the customer doesn't relitigate "what if we pipelined the bit-parallel version?" — Newton already did, with numbers.

### 2.6 Variants Newton did *not* run yet (gated on customer answers)

| Variant | Mechanism | Why it's likely a win | Customer answer needed |
|---|---|---|---|
| **A8** DSP48E2-as-XOR offload | UltraScale+ DSP48E2 `OPMODE` supports wide bitwise XOR; move LUT3 fabric into DSP slices | Highest expected aggregate-LUT win because it removes XOR fabric from CLB-LUT entirely. `xcku3p` has 576 DSPs. | How many DSP48E2 slices are free? Acceptable to consume `8 × <N>` for GHASH? |
| **A3** Karatsuba 64×64 | 3 × 64×64 GF sub-mults instead of 1 × 128×128 | Asymptotically lower LUT; only worth it if A8 doesn't close the gap | High TB-verification cost; gated on A8 result |
| **A4 / A9** H-power precompute (URAM-resident) | Precompute `H, H², H⁴, …` once; share across instances | Only applies if H is shared across the 8 multipliers | Is H shared? Is URAM available? |

---

## 3. Recommended course of action

Ordered, executable. Each step is independent and has a clear stop condition.

### Step 1 — Vivado calibration pass on the recommended RTL  *(highest priority)*

```text
Run 1: baseline           ITER_PER_STAGE=128, default strategy,            seed 1
Run 2: A6 ITER=32         ITER_PER_STAGE=32,  Flow_AreaOptimized_high,    seeds {1..4}
```

**Why first.** Every Vivado-LUT number in this report is `yosys_LC / 2.05`, where 2.05 is a one-point calibration. Until two more Vivado anchors land, the entire ranking sits on a single measurement. Two runs settle this.

**Stop condition.** A6 ITER=32 produces ≤ 6,000 CLB-LUTs per instance and meets ≥ 322 MHz Fmax with WNS ≥ 0 ns and acceptable congestion. If yes, this is shippable. If no, escalate to Step 2.

**Estimated cost.** ~30 min of Vivado time on a Linux box per run, ≤ 3 h total.

### Step 2 — Vivado-strategy sweep on locked A6 ITER=32 RTL

Lock the RTL with an md5 pin, then sweep KU3P-applicable strategies in pain-ranking order (LUT > congestion > latency > Fmax). Drop all `SSI_*` / `SLLs` / `SLRs` strategies — `xcku3p` is single-SLR.

```text
1. Flow_AreaOptimized_high
2. Area_Explore
3. Area_ExploreSequential
4. Area_ExploreWithRemap
5. Flow_AreaMultThresholdDSP        (primes A8 by forcing DSP use)
6. Congestion_SpreadLogic_high
7. Congestion_SpreadLogic_medium
8. Performance_ExplorePostRoutePhysOpt
9. Flow_AlternateRoutability
10..N. remaining Performance_ bundles
```

Each strategy × seeds {1..5}. ~22 strategies × 5 seeds = 110 runs, ~30 min each on a Linux box = ~55 h total wall-clock, or ~7 h on 8-way parallel.

**Stop condition.** Pareto frontier on (LUT, Fmax, congestion) saturates — no new dominant point after one full strategy pass.

### Step 3 — A8 DSP48E2-as-XOR offload  *(gated on Step 2 + customer answer)*

Trigger once the Step-2 frontier saturates *and* the customer confirms DSP availability. Mechanism: move the per-atomic XOR network into DSP48E2's wide-XOR `OPMODE`. Expected aggregate-LUT win is significant because `xcku3p` has 576 DSPs and each instance can offload most of its XOR fabric.

Prerequisite work before any RTL: implement A8 in Python against the existing golden, verify bit-exact over 5,000 random vectors, *then* write the SystemVerilog. (Pre-synth Python verification is mandatory; it costs ~30 s and prevents 30-min synth-then-crash debug cycles.)

### Step 4 — A3 Karatsuba 64×64  *(only if Step 3 underdelivers)*

Higher-effort algorithmic rewrite. Only justified if A8 doesn't close the LUT gap to the customer's target. Same pre-synth Python verification gate as A8.

### Step 5 — Infrastructure: TB backend swap

The SV TB has not been run on every variant because `iverilog` hangs on the packed-2D crypto RTL on macOS M-series. Move the TB backend to **Verilator binary mode** (or **Vivado xsim** once a Linux Vivado box is wired in). One-time engineering investment; unlocks running the real SV TB against every future candidate without the Python-golden detour.

---

## 4. Customer-side open questions (please answer to unblock Step 3+)

1. **Vivado version** running on your Linux box? (Assume 2023.2; confirm.)
2. **Which row of `report_utilization` did "200,000 LUTs" come from?** CLB LUTs / CLB Registers / System Logic Cells? This pins down whether the LUT-pressure framing is ~64% or softer.
3. **DSP48E2 budget free** across the 8 multipliers — how many of the 576 are already consumed elsewhere? Acceptable to consume `8 × <N>` DSPs for GHASH?
4. **URAM availability** — are any of the 28 URAM blocks free, or reserved for other features?
5. **Is H (the GHASH key) shared across the 8 multipliers**, or independent per-instance? (Gates A4 / A9 H-power precompute.)
6. **Hard Fmax target** — confirm ≥ 322 MHz, or specify the actual number.
7. **Wrapper-side latency budget** — confirm 5–6 cycles per multiplier, or specify.

Answers to (2), (3), (5), (6) are required to commit to the headline numbers in this report or to queue A8.
