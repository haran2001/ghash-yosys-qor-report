# GHASH GF(2^128) — Simplified Report

A condensed version of [`README.md`](README.md). One page, three sections: problem, solution, limitations. For the executive-summary single page (final RTL + next steps only), see [`RECOMMENDATION.md`](RECOMMENDATION.md).

---

## 1. Problem statement, constraints, assumptions

### Problem

The customer instantiates **8 separate-H GHASH GF(2^128) multipliers** on `xcku3p-ffvb676-2-e` as part of an HFT datapath. The block is **LUT- and congestion-bound** and is the dominant area cost in the design. They want fewer LUTs and ideally fewer cycles, without breaking the GHASH functional contract.

### Hard constraints

| Constraint | Value |
|---|---|
| Part | Xilinx Kintex UltraScale+ `xcku3p-ffvb676-2-e`, speed grade −2, extended temp, single SLR |
| Resource budget (DS922) | 99,840 CLB LUTs · 199,680 FFs · 576 DSP48E2 · 252 BRAM · 28 URAM |
| Customer Fmax target | ≥ 322 MHz |
| Customer latency budget | 5 – 6 cycles per multiplier |
| Customer-reported LUT pressure | ~64,000 / ~99,840 ≈ 64% of CLB LUTs (8 instances) |
| Functional contract | GHASH GF(2^128) over the existing TB (5 directed + 1 000 random vectors), `8'hE1` reduction polynomial, `logic [15:0][7:0]` byte ordering — **immutable** |
| Pain ranking | LUT > congestion > latency > Fmax |

### Assumptions baked into the numbers

1. **Yosys-Tier-A only.** Every number is from Yosys 0.60 `synth_xilinx -family xcup -flatten`. **No Vivado P&R has run.**
2. **Yosys-LC ≈ 2.05× Vivado-CLB-LUT**, based on a single anchor (`16,384 LCs` yosys ↔ `~8,000 CLB-LUTs` customer Vivado on the baseline). The ratio is assumed stable across variants.
3. **`w_fmax = 0` effectively**, because yosys produces no Fmax. Composite score ranks LUT and latency well; ranks Fmax-incompatible variants too high.
4. The customer's "200 k LUTs" was interpreted as **CLB-LUTs ~100 k** (KU3P doesn't have 200 k LUTs; he likely read the FF or System-Logic-Cells row). **Unconfirmed.**
5. **`ITER_PER_STAGE` must divide 128 exactly** (RTL `$fatal` assertion), so K ∈ {24, 20, 40, …} are excluded by construction.
6. H is **assumed not shared** across the 8 instances; DSP and URAM budgets **assumed available**. All unconfirmed.

---

## 2. Solution — improvement strategies and measured QoR

An autoresearch experiment harness (Newton) proposed legal same-contract RTL variants, ran each through TB + Yosys Tier-A synth, and recorded source-of-improvement for every accepted candidate.

### Variants explored

| Family | What changes | Status |
|---|---|---|
| **baseline** | Customer's chained iterative GHASH, `ITER_PER_STAGE=128` | anchor |
| **A1** syntactic rewrite | Flat 128-bit vector / re-spelled XOR tree | **rejected (no-op)** |
| **A6** pipeline granularity | `ITER_PER_STAGE` ∈ {128, 32, 8, 2} | accepted at K=32 |
| **A2** combinational bit-parallel | One-cycle parallel partial-product matrix | LUT win but Fmax-incompatible |
| **A2b** pipelined bit-parallel | Stage width K ∈ {64, 32, 16} | **negative result** |

### Measured QoR (Yosys-Tier-A, `xcku3p`)

| Variant | Latency | yosys LUTs | MUXFx | FFs | Δ LUTs vs baseline | Verdict |
|---|---:|---:|---:|---:|---:|---|
| baseline ITER=128 | 1 | 16,769 | 0 | 129 | 0% | will miss Fmax in Vivado |
| **A6 ITER=32** ★ | **4** | **11,468** | **2,136** | **1,092** | **−31.6%** | **best LUT/latency in customer budget** |
| A6 ITER=8 | 16 | 8,915 | 1,895 | 4,944 | −46.8% | out of latency budget |
| A6 ITER=2 | 64 | 8,634 | 0 | 20,352 | −48.5% | out of latency budget |
| A2 combinational | 1 | 14,315 | 5,842 | 129 | −14.6% | will miss 322 MHz |
| A2b K=64 | 3 | 15,035 | 6,153 | 579 | −10.3% | regression vs A2 comb |
| A2b K=32 | 5 | 12,305 | 2,905 | 1,125 | −26.6% | dominated by A6 ITER=32 |
| A2b K=16 | 9 | 11,779 | 284 | 2,105 | −29.8% | out of latency budget |

★ = recommended deployment candidate.

### Headline improvement claim (under stated assumptions)

| Quantity | Customer baseline (Vivado) | Recommended (A6 ITER=32, Yosys-Tier-A ÷ 2.05) | Δ |
|---|---:|---:|---:|
| LUTs per instance | ~8,000 | **~5,595** | **−30%** |
| LUTs aggregate (× 8) | ~64,000 | **~44,800** | **−19,200 LUTs** |
| Latency | as-shipped | 4 cycles (inside 5–6 budget) | — |
| MUXFx pressure | 0 | 2,136/inst (~17 k aggregate) | needs Vivado P&R verify |

### Source-of-improvement attribution

- **A6 ITER=32 (~−30% LUTs):** mostly `rtl_pipeline` — smaller ABC cones let optimisation finish; partly an ABC cone-scaling artifact (Vivado `opt_design` will compress some of this gap on the unpipelined baseline).
- **A2 combinational (−14.6% LUTs at 1 cycle):** `rtl_algorithmic` — cross-bit common-subexpression sharing in a single combinational matrix. **Wins LUT but loses Fmax.**
- **A1 syntactic rewrites (0%):** `no_improvement` — ABC canonicalises per output bit before tech-mapping; every spelling collapses to the same 128 LUT3 + 3 LUT2 atomic.
- **A2b pipelined bit-parallel (negative):** once the matrix is pipelined, cross-cone CSE sharing disappears; chained A6 is at least as good and usually better at matched latency.

### Variants queued, not yet run

| Variant | Mechanism | Gating |
|---|---|---|
| A8 DSP48E2-as-XOR offload | Move LUT3 XOR fabric into DSP `OPMODE` wide-XOR | customer DSP-budget confirmation |
| A3 Karatsuba 64×64 | 3 × 64×64 sub-multiplies instead of 1 × 128×128 | higher TB-verification effort; gated on Vivado calibration |
| A4 / A9 H-power precompute (URAM-resident) | Precomputed key-power table | customer confirms H is shared |

---

## 3. Limitations

1. **No Vivado P&R has been run.** Every Fmax / WNS / TNS / congestion field is empty. The 30% per-instance LUT claim rests on a one-point Yosys-LC ↔ Vivado-CLB-LUT calibration ratio. **One Vivado pass on baseline + A6 ITER=32 is required before this number goes into a customer commitment.**
2. **Calibration ratio (2.05×) is not re-verified per-variant.** Any algorithmic rewrite (A2/A3/A8) may break the ratio because Vivado's `opt_design` reacts very differently to bit-parallel matrices than to chained iterations.
3. **MUXFx is reported but not P&R-stressed.** Vivado's `report_utilization` puts F7/F8/F9 on their own line, not in the CLB-LUT line — but heavy MUXFx pressure causes packing problems and routing congestion that hurt Fmax. The headline LUT number can hide MUXFx cost. **MUXFx at × 8 instances (~17 k MUXFx total for A6 ITER=32, ~47 k for A2 combinational) needs Vivado P&R confirmation.**
4. **SV testbench has not been run on every variant.** iverilog hangs on the packed-2D-array crypto RTL on macOS; verification is via a Python golden against precomputed vectors. Algorithm is bit-exact-verified, but Verilator binary mode or Vivado xsim is the right long-term TB backend.
5. **"~64% LUT pressure" rests on an interpretation** of which `report_utilization` row the customer was reading. If he was reading System Logic Cells (162 k), the pressure framing softens from 64% to ~40%. **Confirm.**
6. **Customer-side unknowns still open:** Vivado version, free DSP budget across the 8 multipliers, URAM availability, whether H is shared, exact Fmax target, exact wrapper-side latency budget.
7. **`w_fmax` term in the composite score evaluates to zero**, which is why A2 combinational appears higher on score than its physical reality warrants. The score is informative for **LUT comparisons within the same latency bucket** and **uninformative across pipeline-granularity** until Vivado timing exists.
8. **Design-space restriction:** `ITER_PER_STAGE` must divide 128 exactly. Off-grid K values are excluded by construction.

---

## TL;DR

> Under Yosys Tier-A ranking with a single anchor calibration to the customer's Vivado run, A6 ITER=32 (chained, 4-cycle, customer's current operating point) is the best LUT/latency variant in their constraint envelope, projecting to ~−30% CLB-LUTs per instance and ~19 k aggregate LUT savings across 8 instances. The bit-parallel rewrites (A2 / A2b) either miss Fmax or are dominated by A6 once pipelined. The next concrete step is **one Vivado P&R pass** on baseline + A6 ITER=32 to confirm the calibration ratio survives before any algorithmic rewrites (A8 DSP-XOR offload, A3 Karatsuba) are queued.
