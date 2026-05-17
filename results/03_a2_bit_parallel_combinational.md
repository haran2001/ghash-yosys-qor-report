# 03 — A2 combinational bit-parallel + MUXFx accounting

A2 implements GHASH as a fully-combinational bit-parallel matrix:

```text
shifts[0]   = H
shifts[i]   = xtime(shifts[i-1])   for i = 1..127
Z           = XOR over i of (x_at_degree[i] AND shifts[i])
```

## Result table (with MUXFx column added)

| Variant | `iter_per_stage` | Latency | LUTs | MUXFx | FFs | Δ LUTs vs baseline | Score |
|---|---:|---:|---:|---:|---:|---:|---:|
| `baseline` | 128 | 1 | 16,769 | **0** | 129 | +0.0% | 3.844 |
| `A6_iter_per_stage_32` | 32 | 4 | 11,468 | **2,136** | 1,092 | −31.6% | 5.250 |
| `A6_iter_per_stage_8` | 8 | 16 | 8,915 | **1,895** | 4,944 | −46.8% | 19.172 |
| `A6_iter_per_stage_2` | 2 | 64 | 8,634 | **0** | 20,352 | −48.5% | 76.719 |
| **`A2_bit_parallel`** | 1 | **1** | **14,315** | **5,842** | 129 | **−14.6%** | **3.384** |

## Functional correctness

Verified bit-exact against the existing TB's Python golden over **5 000 random vectors** — zero mismatches. The polynomial-degree-to-flat-bit mapping (`deg_to_flatbit(d) = 8·(d/8) + (7 − d%8)`) was reverse-engineered from the existing TB by tracing single-bit inputs through `_xtime_right`.

## What ABC actually did

A2 yosys output composition is very different from baseline:

```text
baseline: 16,384 LUT3 + 385 LUT2                       (no wide muxes)
A2:        9,367 LUT6 + 3,387 LUT4 + 1,558 LUT2 + 5,842 MUXFx
```

A2 packed roughly 70% of the logic into LUT6 (vs baseline's all-LUT3), because the 128-way XOR-mux on each output bit exposes **cross-bit common-subexpression sharing** that the chained baseline can't show ABC. The 5 842 MUXFx are wide-mux primitives (MUXF7 = 8:1, MUXF8 = 16:1, MUXF9 = 32:1) that ABC selected for the wide AND-OR/mux structure.

## MUXFx caveat (and why earlier comparison was incomplete)

The first read of A2 vs baseline tempted a "secretly LUT-positive" conclusion because A2 has 5 842 MUXFx and baseline has 0. Re-examination with a MUXFx column shows **A6_ITER=32 (customer's current deployment) also uses 2 136 MUXFx, and A6_ITER=8 uses 1 895**. ABC emits wide muxes whenever there's a multi-way choice — this is not unique to A2.

What matters in Vivado:

- **LUT count** = LUT2/LUT3/LUT4/LUT5/LUT6 totalled (yosys reports each separately).
- **MUXFx** use the **dedicated mux fabric inside each CLB**, not LUT slots.
- Heavy MUXFx usage adds pressure on the dedicated mux network and can cause **CLB packing inefficiency** if a region runs out of F7/F8/F9 resources, but **does not add LUTs**.
- The customer's "8 000 LUTs per multiplier" is the Vivado CLB-LUT line, which excludes MUXFx. So yosys-LUT-count is the apples-to-apples metric, with the caveat that very high MUXFx counts (~5 k+) raise a flag for P&R review.

Corrected story for A2 vs baseline:

- A2 reduces yosys LUT count by **−14.6%** (16,769 → 14,315).
- A2 adds **5 842 MUXFx**, which baseline has zero of. This is a real risk factor for CLB packing on `xcku3p`, but not necessarily a deal-breaker per-instance.
- A2 keeps **1-cycle latency**, matching baseline.
- A2's Pareto position is the best among non-pipeline-extreme variants under the current score weights — but the score weights have `w_fmax × 0`, which is hiding A2's biggest risk.

## A1 syntactic rewrite (no-op result — recorded so it's not retried)

Rewriting the per-byte right-shift as one flat 128-bit concat produced **identical** yosys output: 128 LUT3 + 3 LUT2 per atomic. ABC canonicalises per output bit before tech mapping. **Any syntactic-only rewrite is a no-op on this design.** Permanently dropped from the queue.

## What we don't yet know

- Whether A2's MUXFx-heavy structure survives Vivado's `opt_design` / `phys_opt_design`. Vivado often **re-decomposes MUXFx into LUT6** if it improves routing or Fmax, which would change the LUT/MUXFx balance.
- Whether A2's 128-deep `xtime` chain meets the customer's Fmax target. The critical path is `H → xtime → xtime → ... × 128 → XOR-mux → output_reg`. On `xcku3p-2-e` that's almost certainly **> 10 ns**, missing 322 MHz by a wide margin.
- Whether the 5 842 MUXFx fit in `xcku3p`'s CLB grid without congestion. KU3P has ~12,480 CLBs (1 F7 / 1 F8 / 1 F9 each). 5 842 per-instance is well under budget; **× 8 instances = ~47 k**, which is real pressure.

## Recommended next variants from A2

The honest path forward was to **pipeline A2** to bound the critical path. See result 04 for A2b — and for the negative result that came back.
