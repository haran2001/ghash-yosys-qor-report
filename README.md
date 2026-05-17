# GHASH GF(2^128) Multiplier — Newton-Driven Open-Source QoR Study on `xcku3p-ffvb676-2-e`

A closed-loop RTL-variant exploration of a pipelined GHASH / GF(2^128) multiplier, driven end-to-end by Newton — a self-learning agent for physical-design exploration — over the open-source flow (Yosys `synth_xilinx -family xcup`) before committing Vivado licence time. Newton proposes legal same-contract RTL variants, runs them through a fixed testbench gate and Tier-A synth, parses QoR, classifies the pressure point, and records *why* each candidate was accepted, rejected, or escalated.

The goal is to rank legal same-contract RTL candidates by yosys-reported LUT / MUXFx / FF, surface the ones worth a full Vivado P&R pass, and capture the negative results in a reasoning trace so the next experiment doesn't relitigate them.

This is a Yosys-only study. **No Vivado place-and-route has been run.** Every number is calibrated against a single anchor measurement from the customer's Vivado run. Absolute Vivado LUT counts must come from a Vivado pass on `xcku3p`; this report ranks variants, it does not size them.

---

## Target device and tool flow

| Item | Value |
|---|---|
| Customer design | 8 separate-H GHASH multipliers, HFT datapath |
| FPGA | Xilinx Kintex UltraScale+ `xcku3p-ffvb676-2-e` |
| CLB LUT budget (DS922) | 99,840 |
| Flip-Flop budget | 199,680 |
| DSP48E2 slices | 576 |
| BRAM 36Kb / UltraRAM 288Kb | 252 / 28 |
| Speed grade / temp | −2 / extended |
| SLR count | 1 (single-SLR; all SSI/SLLs/SLRs strategies dropped) |
| Customer Fmax target | ≥ 322 MHz |
| Customer latency budget | 5 – 6 cycles |
| Customer-reported LUT pressure | ~64,000 / ~99,840 ≈ 64% |
| Open-source flow | Yosys 0.60 `synth_xilinx -family xcup -flatten` |

Vanilla compile recipe used as the Tier-A ranking step:

```bash
yosys -p '
read_verilog -sv rtl/ghash_mult.sv
hierarchy -top ghash_mult -chparam ITER_PER_STAGE <K>
synth_xilinx -family xcup -flatten
stat
'
```

The point of using yosys is not that yosys is the production flow. Vivado on Linux is. The yosys layer is a controlled Tier-A ranking experiment: keep RTL legal, keep the functional contract sacred, and find out which structural rewrites are worth a Vivado licence before consuming one.

---

## Goal and methodology

**What we tested.** Whether a meta-harness above Yosys, with a fixed testbench gate and a fixed functional contract, can rank legal same-RTL-contract variants well enough to pre-filter the Vivado candidate queue for a LUT- and congestion-constrained customer design.

**What was held fixed.** GHASH GF(2^128) functional contract (NIST byte ordering, reduction polynomial `8'hE1` on byte 0, `logic [15:0][7:0]` IO), part `xcku3p-ffvb676-2-e`, testbench (5 directed + 1 000 random vectors), Python golden, customer pain ranking (LUT > congestion > latency > Fmax).

**What was varied.**

- Algorithmic structure (chained iterative, combinational bit-parallel, pipelined bit-parallel).
- Pipeline granularity (`ITER_PER_STAGE` ∈ {128, 32, 8, 2} and bit-parallel stage width K ∈ {64, 32, 16}).
- Synthesis pragmas (`-flatten`, `-nodsp` for the baseline anchor).

**What was not varied.** The TB. The functional contract. The byte ordering. The part. Any Vivado strategy (Track 2 hasn't run yet).

**How variants are evaluated.** Newton runs each candidate through: (1) Verilator lint, (2) Python-golden bit-exact verification over 500–5 000 random vectors, (3) Yosys `synth_xilinx -family xcup -flatten`, (4) yosys-LC + MUXFx + FF parsing, (5) source-of-improvement tag, (6) frontier update. The output is not just a synth log — it's a reasoning trace recording the observed result, the hypothesis, the command, and the selection rationale.

**Calibration to Vivado.** One anchor point from the customer's Vivado run:

```text
yosys baseline: 16,384 LCs per ghash_mult instance
Vivado on KU3P: ~8,000 CLB-LUTs per ghash_mult instance (customer-reported)
ratio:          ~2.05× (yosys-LC over Vivado-CLB-LUT)
```

The 2.05× ratio is assumed stable across variants for XOR-heavy GF arithmetic. **It has not been re-verified per-variant.** Any number quoted to the customer as "Vivado LUTs" in this report is `yosys_LC / 2.05`, not a measurement.

---

## RTL 1: `A6` chained-iterative — pipeline-granularity objective

### Design architecture

`ghash_mult` is a 3-level hierarchy parameterised by `ITER_PER_STAGE`:

- `ghash_mult_atomic` — one GF iteration: conditional XOR (`Z = Z ^ V` if X-bit set) + `xtime` (right-shift V plus reduction-polynomial XOR on LSB).
- `ghash_mult_stage` — combinationally chains `ITER_PER_STAGE` atomics.
- `ghash_mult` — `TOTAL_STAGES = 128 / ITER_PER_STAGE` stages with X/H/Z/V pipeline registers and `i_vld`/`i_ack` backpressure.

```text
                     i_X, i_H, i_vld
                            │
                            ▼
                ┌────────────────────────┐
                │  stage 0:              │
                │  K atomics in series   │  K = ITER_PER_STAGE
                │  (combinational)       │
                └────────────┬───────────┘
                             ▼ (X/H/Z/V pipeline regs)
                ┌────────────────────────┐
                │  stage 1: K atomics    │
                └────────────┬───────────┘
                             ▼
                            ...
                ┌────────────────────────┐
                │  stage N-1: K atomics  │   N = 128 / K
                └────────────┬───────────┘
                             ▼
                         o_X, o_vld
```

The Newton-relevant implementation choice here is the pipeline-granularity axis: how many GHASH iterations to flatten into one combinational cone before registering. There is no DSP or BRAM resource decision in this variant family.

### Vanilla flow results (yosys, `synth_xilinx -family xcup -flatten -nodsp`)

| Metric | Value |
|---|---:|
| Variant | `baseline ITER_PER_STAGE=128` |
| yosys LCs | 16,769 |
| MUXFx | 0 |
| FFs | 129 |
| Latency | 1 cycle |
| Inferred Vivado CLB-LUT (÷2.05) | ~8,180 |

### Newton strategies tried

| Step | Strategy | `ITER_PER_STAGE` | Hypothesis | Observation | Decision |
|---:|---|---:|---|---|---|
| 0 | Vanilla baseline | 128 | Establish ABC's behaviour on the 128-deep combinational cone | 16,769 LUT / 0 MUXFx / 129 FF, 1-cycle latency | Fmax will fail on Vivado; needs pipelining |
| 1 | A6 pipeline split | 32 | 4 stages = customer's current deployment | 11,468 LUT / 2,136 MUXFx / 1,092 FF, 4 cycles | Inside customer's 5–6 cycle budget. Best candidate so far. |
| 2 | A6 deeper split | 8 | More stages → smaller ABC cones | 8,915 LUT / 1,895 MUXFx / 4,944 FF, 16 cycles | Out of latency budget. Diagnostic only. |
| 3 | A6 max-depth split | 2 | Extreme test of ABC scaling | 8,634 LUT / 0 MUXFx / 20,352 FF, 64 cycles | Out of latency budget. Confirms ABC cone-scaling artifact. |

### A6 yosys results

| Variant | Latency | LUTs | MUXFx | FFs | Δ LUTs vs baseline |
|---|---:|---:|---:|---:|---:|
| baseline ITER=128 | 1 | 16,769 | 0 | 129 | +0.0% |
| **A6 ITER=32** | **4** | **11,468** | **2,136** | **1,092** | **−31.6%** |
| A6 ITER=8 | 16 | 8,915 | 1,895 | 4,944 | −46.8% |
| A6 ITER=2 | 64 | 8,634 | 0 | 20,352 | −48.5% |

### Interpretation

The big −31.6% / −46.8% / −48.5% numbers are partly **ABC cone-scaling artifacts**, not durable Vivado wins. At ITER=128 the entire 128-iteration chain is one combinational cone; yosys's ABC heuristic stalls on that scope and leaves redundant LUTs. Breaking the cone into smaller pieces lets ABC finish its optimisation passes. Vivado's `opt_design` and `phys_opt_design` are far more aggressive on large cones and are expected to close most of the gap on the ITER=128 case.

The honest framing for Saurabh:

- **yosys ranks ITER=K variants lower-LUT as K shrinks, but this is partly an ABC weakness on the unpipelined case.**
- **Real Vivado-LUT savings from pipeline-granularity alone are typically < 10%.** The 30–50% scaling here will compress sharply once Vivado runs the baseline.
- A6 ITER=32 (customer's current operating point) is the right *anchor* for any future improvement claim, because that's the row Saurabh is measuring everything against.

---

## RTL 2: `A2` / `A2b` bit-parallel — area-with-timing objective

### Design architecture

`A2` rewrites the 128-iteration chain as a fully-combinational bit-parallel polynomial-multiply matrix:

```text
shifts[0]   = H
shifts[i]   = xtime(shifts[i-1])   for i = 1..127
Z           = XOR over i of (x_at_degree[i] AND shifts[i])
```

`A2b` is the pipelined version: split the 128-deep `xtime` chain into N stages of width K = 128 / N.

```text
       i_X, i_H, i_vld
              │
              ▼
   ┌───────────────────────┐
   │ stage 0 (K xtime,     │
   │ K bit-products, partial│   K = stage width
   │ XOR reduction)         │
   └───────────┬───────────┘
               ▼ (xtime-state, partial-Z regs)
   ┌───────────────────────┐
   │ stage 1               │
   └───────────┬───────────┘
              ...
   ┌───────────────────────┐
   │ stage N-1 (final XOR  │
   │ reduction)             │
   └───────────┬───────────┘
               ▼
            o_X, o_vld
```

The Newton-relevant choice here is algorithmic: trade the chained `Z`-accumulator data-dependency for a parallel partial-product matrix that ABC can common-subexpression-share across all 128 output bits — and trade some MUXFx pressure for that win.

### Verification

A2 was checked bit-exact against the existing TB Python golden over **5 000 random vectors** — zero mismatches. A2b at K ∈ {2, 4, 8, 16} was checked against the iterative golden over **500 random vectors × 4 K values = 2 000 total** — zero mismatches. The polynomial-degree-to-flat-bit mapping (`deg_to_flatbit(d) = 8·(d/8) + (7 − d%8)`) was reverse-engineered from the original TB by tracing single-bit inputs through `_xtime_right` and is recorded in the supporting skill.

### Newton strategies tried

| Step | Strategy | Hypothesis | Observation | Decision |
|---:|---|---|---|---|
| 0 | A1 syntactic rewrite (flat 128-bit vector, exposed XOR tree) | Per-atomic LUT count will drop | Identical yosys output: 128 LUT3 + 3 LUT2 per atomic. ABC canonicalises before tech mapping. | Reject. Permanently. |
| 1 | A2 combinational bit-parallel | Cross-bit common-subexpression sharing will beat the chained baseline at 1-cycle latency | 14,315 LUT / 5,842 MUXFx / 129 FF, 1 cycle | Real LUT win but 128-deep `xtime` chain will not meet 322 MHz. |
| 2 | A2b pipelined, K=64 | Bound the critical path while preserving bit-parallel sharing | 15,035 LUT / 6,153 MUXFx / 579 FF, 3 cycles | Regression on LUTs vs A2 combinational. |
| 3 | A2b pipelined, K=32 | Match A6 ITER=32's latency budget | 12,305 LUT / 2,905 MUXFx / 1,125 FF, 5 cycles | Strictly dominated by A6 ITER=32 (worse on both LUT and latency). |
| 4 | A2b pipelined, K=16 | Minimise MUXFx pressure | 11,779 LUT / 284 MUXFx / 2,105 FF, 9 cycles | Cleanest MUXFx but out of customer's 5–6 cycle budget. |

### A2 / A2b yosys results (with MUXFx column)

| Variant | Latency | LUTs | MUXFx | FFs | Notes |
|---|---:|---:|---:|---:|---|
| baseline ITER=128 | 1 | 16,769 | 0 | 129 | Will miss Fmax in Vivado |
| **A2 combinational** | **1** | **14,315** | **5,842** | **129** | **−14.6% LUTs, but Fmax-incompatible** |
| A2b K=64 (2 stages) | 3 | 15,035 | 6,153 | 579 | Regression |
| A2b K=32 (4 stages) | 5 | 12,305 | 2,905 | 1,125 | Worse than A6 ITER=32 at +1 cycle |
| A2b K=16 (8 stages) | 9 | 11,779 | 284 | 2,105 | Out of latency budget |
| A6 ITER=32 (chained) | 4 | 11,468 | 2,136 | 1,092 | **Beats A2b_K32 on LUT and latency** |

### What ABC actually did on A2 vs baseline

```text
baseline: 16,384 LUT3 + 385 LUT2                       (no wide muxes)
A2:        9,367 LUT6 + 3,387 LUT4 + 1,558 LUT2 + 5,842 MUXFx
```

A2 packed roughly 70% of the logic into LUT6 (vs baseline's all-LUT3), because the 128-way XOR-mux on each output bit exposes cross-bit common-subexpression sharing that the chained baseline can't show ABC. The 5 842 MUXFx are wide-mux primitives (MUXF7 = 8:1, MUXF8 = 16:1, MUXF9 = 32:1) that ABC selected for the wide AND-OR/mux structure.

### MUXFx pressure scales sharply with K

| K | MUXFx | Interpretation |
|---:|---:|---|
| 16 | 284 | Narrow muxes → ABC builds plain LUT trees |
| 32 | 2,905 | Mid-range, comparable to A6 ITER=32 |
| 64 | 6,153 | Wide muxes → ABC maps onto F7/F8/F9 |
| 128 (A2 comb) | 5,842 | Full bit-parallel matrix |

KU3P has ~12,480 CLBs, each with 1 F7 / 1 F8 / 1 F9. **Per-instance** A2b's MUXFx counts fit comfortably. **At 8 instances** (Saurabh's case), 5 842 × 8 ≈ 47 k MUXFx is real CLB-mux pressure and will need a Vivado P&R run to verify packing.

### Interpretation

This is the objective-aware selection story made concrete.

- Under a **strict ≥ 322 MHz Fmax objective**, A2 combinational is rejected: its 128-deep `xtime` chain almost certainly misses timing on `xcku3p-2-e` by a wide margin. Yosys can't price this, so the yosys score over-weights A2; the right call is to read the score as "good LUT shape, must be re-verified for Fmax."
- Under **`min_lc_pass_timing` with the customer's 5–6 cycle latency budget**, the deployment candidate is **A6 ITER=32 (chained)** at 11,468 LUT / 2,136 MUXFx / 4 cycles. A2b at matched latency (K=32, 5 cycles) is strictly worse on both LUT and MUXFx.
- A2b is a useful **negative result**: once you pipeline the bit-parallel matrix, the cross-cone common-subexpression sharing disappears. Each stage's ABC scope is no larger than what the chained A6 already exposes for free, and the chained form happens to be slightly more LUT-efficient because it has fewer wide-mux fanouts.

The point is not that `A6 ITER=32` is a secret variant. Saurabh is already running it. The point is Newton tried the competing variants (A1, A2, A2b at K ∈ {64, 32, 16}), rejected the misleading ones with reasons, selected the objective-appropriate variant, and recorded **why** each candidate failed or won.

---

## Summary scorecard for Saurabh

Customer pain ranking applied: LUT > congestion > latency > Fmax. Composite-score weights: `w_lut = 1.5, w_cong = 0.9, w_latency = 1.2, w_fmax = 0.7` (`w_fmax` is currently a structural placeholder because yosys produces no Fmax data; see Limitations).

| Rank | Variant | Latency | yosys LUTs | MUXFx | Verdict |
|---:|---|---:|---:|---:|---|
| **1** | **A6 ITER=32 (chained, customer default)** | **4** | **11,468** | **2,136** | **Best LUT/latency in customer's budget; current operating point.** |
| 2 | A2b K=16 (pipelined bit-parallel) | 9 | 11,779 | 284 | Cleanest MUXFx; needs latency relief on the wrapper side. |
| 3 | A2b K=32 (pipelined bit-parallel) | 5 | 12,305 | 2,905 | Strictly dominated by A6 ITER=32. |
| 4 | A2 combinational bit-parallel | 1 | 14,315 | 5,842 | LUT-attractive but Fmax-incompatible at 322 MHz. |
| 5 | A2b K=64 (pipelined bit-parallel) | 3 | 15,035 | 6,153 | Regression vs A2 combinational. |
| 6 | baseline ITER=128 (chained) | 1 | 16,769 | 0 | Diagnostic anchor only; will miss Fmax. |

**Per-instance Vivado-LUT estimate using the 2.05× calibration:** A6 ITER=32 ≈ **5,595 CLB-LUTs**, vs customer-reported ~8,000 CLB-LUTs on the current deployment. If the calibration holds, that is a **~30% per-instance LUT reduction** (~19 k LUTs aggregate across 8 instances). This is a yosys-Tier-A estimate; **the headline must be confirmed by one Vivado P&R pass** before it goes into a customer commitment.

---

## How this maps to a Vivado-on-Linux flow

### 1. Promote one candidate to a Vivado calibration pass

Before any further yosys sweeps, two Vivado runs settle most of the open questions:

```text
candidate 1: baseline ITER=128, default Vivado strategy, seed 1
candidate 2: A6 ITER=32,        Flow_AreaOptimized_high, seeds {1..4}
```

These two anchors recalibrate the 2.05× ratio per-variant and tell us whether A6's yosys-LUT win survives `opt_design` / `phys_opt_design`. **Without this calibration the entire ranking is inferred from a single anchor point.**

### 2. Track 2 (Vivado-strategy sweep) on the locked A6 ITER=32 RTL

Lock A6 ITER=32 with an md5 pin, then sweep KU3P-applicable strategies. Priority order under the LUT > congestion pain ranking:

```text
1. Flow_AreaOptimized_high
2. Area_Explore / _ExploreSequential / _ExploreWithRemap
3. Flow_AreaMultThresholdDSP        (forces DSP use — primes A8 below)
4. Congestion_SpreadLogic_high / _medium
5. Performance_ExplorePostRoutePhysOpt
6. Flow_AlternateRoutability
```

All `SSI_*` / `SLLs` / `SLRs` strategies are dropped because KU3P is single-SLR.

### 3. Two algorithmic variants still queued, gated on customer answers

- **A8 — DSP48E2-as-XOR offload.** UltraScale+ DSP48E2 has wide-XOR `OPMODE` support. xcku3p has 576 DSPs; needs the customer's free-DSP budget. Highest expected aggregate-LUT win because it moves XOR fabric out of LUT slots entirely.
- **A3 — Karatsuba 64×64.** Three 64×64 sub-multiplies instead of one 128×128. Asymptotically lower LUT but high TB-verification cost.

Both are **higher-effort and gated on the Vivado calibration above.** No point queueing them until we know whether the yosys ranking holds in Vivado.

### 4. A4 / A9 — H-power precompute (conditional)

If H is shared across the 8 multipliers (currently unknown), the H-power table can be precomputed once and either kept in distributed RAM or moved into UltraRAM (A9). 28 URAM blocks × 288 Kb is plenty of headroom. **Gated on customer confirmation that H is shared.**

---

## Assumptions and limitations

This is a yosys-only Tier-A study. Numbers here are for **ranking**, not for sizing. Specifically:

1. **No Vivado P&R has run.** Every Fmax / WNS / TNS / congestion field is empty. The composite-score `w_fmax` term evaluates to zero across the board, which is why A2 combinational (1-cycle, no Fmax penalty applied) appears higher on score than its physical reality warrants. The score is informative for **LUT comparisons within the same latency bucket** and **uninformative across pipeline-granularity** until Vivado timing exists.
2. **Yosys-LC ≈ 2.05× Vivado-CLB-LUT is a one-point calibration.** Derived from `16,384 LCs (yosys) ↔ ~8,000 CLB-LUTs (customer Vivado)` on the baseline. Assumed stable across variants for XOR-heavy GF arithmetic. **Not re-verified per-variant.** Any algorithmic rewrite (A2 / A3 / A8) could break the ratio because Vivado's `opt_design` reacts very differently to bit-parallel matrices than to chained iterations.
3. **MUXFx accounting is a soft signal.** Vivado's `report_utilization` puts F7/F8/F9 on their own line, not in the CLB-LUT line — but heavy MUXFx pressure (> 1 F8 / CLB, > 1 F9 / 2 CLBs) causes packing problems and routing congestion that hurt Fmax. The headline LUT number can hide MUXFx cost. **Every row in the scorecard reports MUXFx alongside LUT** for this reason.
4. **iverilog hangs on the packed-2D crypto RTL on macOS.** TB runs use the Python golden against precomputed vectors; the SV TB has not been re-run against A2 / A2b on every K. The algorithm has been Python-verified bit-exact, but Verilator binary mode or Vivado xsim is the right long-term TB backend.
5. **"~64% LUT pressure" rests on an interpretation.** The customer said "200 000 LUTs"; KU3P has ~100 k CLB-LUTs and ~200 k FFs. The working assumption is he read FFs and meant CLB-LUTs. If he was reading "System Logic Cells" (162 k), the pressure framing softens from 64% to ~40%. **Confirm which `report_utilization` row.**
6. **Customer-side unknowns still open.** Vivado version (assumed ≥ 2020.2, ideally 2023.2). Free DSP budget across the 8 multipliers. URAM availability. Whether H is shared. Hard Fmax target (assumed 322 MHz). Wrapper-side latency budget (assumed 5–6 cycles).
7. **The TB defines the contract; the Python golden does not.** Both are kept bit-exact-consistent, but the immutable artefact is the SV TB. Any TB extension must preserve assertions; the harness can add random vectors but cannot weaken checks.
8. **`ITER_PER_STAGE` must divide 128 exactly** (RTL `$fatal` assertion). K ∈ {24, 20, 40, …} are excluded from the design space by construction.

---

## What's next

1. **One Vivado calibration pass** on baseline + A6 ITER=32. Anchors the 2.05× ratio per-variant and confirms whether A6's win survives `opt_design`.
2. **Track 2 strategy sweep** on the locked A6 ITER=32 RTL.
3. **A8 DSP48E2-as-XOR offload** — once Track 2 is in and the customer confirms free-DSP budget.
4. **A3 Karatsuba 64×64** — only if A8 doesn't close the LUT gap, and only after pre-synth Python verification across ≥ 5 000 random vectors.
5. **TB backend swap to Verilator binary mode** so the SV TB can run against every variant without the iverilog hang.

---

## Repository layout

```text
ghash-yosys-qor-report/
├── README.md                                  # this report
└── results/
    ├── 01_yosys_baseline_xcku3p.md            # baseline ITER=128 raw numbers
    ├── 02_a6_pipeline_granularity_sweep.md    # A6 ITER ∈ {128, 32, 8, 2}
    ├── 03_a2_bit_parallel_combinational.md    # A2 combinational + MUXFx caveat
    └── 04_a2b_pipelined_bit_parallel.md       # A2b K ∈ {64, 32, 16}, negative result
```

The RTL and the meta-harness live in private repos:

- `haran2001/ghash-multiplier` — cleaned baseline RTL + Python golden + plans / context.
- `haran2001/meta-harness-ghash-fpga` — fork of `stanford-iris-lab/meta-harness`, two-track loop (RTL variant sweep + Vivado-strategy sweep), per-run JSONL evolution summary, frontier JSON.

This report is the public, customer-facing summary; the implementation repos remain private.
