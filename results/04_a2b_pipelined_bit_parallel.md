# 04 — A2b pipelined bit-parallel (negative result)

A2b is the pipelined version of A2: split the 128-deep `xtime` chain into N stages of width K = 128 / N.

## TL;DR

A2b (pipelined bit-parallel) **does not beat the chained pipelined baseline** at matched latency. The cross-bit common-subexpression sharing that made A2 (combinational) interesting **disappears once we pipeline**, because each stage's combinational scope is the same K-deep cone that the chained baseline already exposes to ABC for free.

This is a useful negative result: it kills the "A2b → fewer LUTs than A6 at same latency" hypothesis with real data.

## Numbers

| Variant | Latency | LUTs | MUXFx | FFs | Notes |
|---|---:|---:|---:|---:|---|
| baseline ITER=128 | 1 | 16,769 | 0 | 129 | Wins yosys score only because no Fmax penalty |
| A2 combinational | 1 | 14,315 | 5,842 | 129 | Will miss timing badly; deep MUXFx pressure |
| **A2b K=64 (2 stages)** | 3 | **15,035** | 6,153 | 579 | Regressed vs A2 combinational on LUTs |
| **A2b K=32 (4 stages)** | 5 | **12,305** | 2,905 | 1,125 | Loses to A6_ITER=32 at −1 cycle |
| **A2b K=16 (8 stages)** | 9 | **11,779** | **284** | 2,105 | Low MUXFx but high latency cost |
| A6 ITER=32 (chained) | 4 | 11,468 | 2,136 | 1,092 | **Beats A2b_K32 on LUTs and latency** |
| A6 ITER=8 (chained) | 16 | 8,915 | 1,895 | 4,944 | |
| A6 ITER=2 (chained) | 64 | 8,634 | 0 | 20,352 | |

## What we learned

### 1. Bit-parallel restructuring needs combinational scope to win

A2 combinational fits all 128 `xtime` + 128-way XOR into one ABC cone, so ABC can extract cross-bit sharing across the entire matrix and pack into LUT6 + MUXFx (14,315 LUTs + 5,842 muxes).

A2b cuts that cone into K=32 (or K=16, K=64) per-stage scopes. **At K=32, ABC's per-stage scope is no larger than what the chained pipelined baseline (A6_ITER=32) gives it.** Both designs are now reduced to the same local optimisation problem, and the chained form happens to be slightly more LUT-efficient because it has fewer wide-mux fanouts.

### 2. MUXFx pressure scales sharply with K

| K | MUXFx |
|---:|---:|
| 16 | 284 |
| 32 | 2,905 |
| 64 | 6,153 |
| 128 (A2 comb) | 5,842 |

This makes sense: a K-way XOR-mux on a 128-bit output is the exact pattern ABC maps onto F7/F8/F9 wide-mux primitives. Small K → narrow muxes → ABC builds plain LUT trees instead. **K=16 is the sweet spot for clean CLB packing**, but K=16 is also 9 cycles — out of the customer's 5–6 cycle budget.

### 3. Customer-relevant comparison

At the customer's pain point (4–5 cycle latency, low LUT, must meet 322 MHz Fmax on `xcku3p`):

- **A6_ITER=32 (chained, 4 cycles)** → 11,468 LUTs / 2,136 MUXFx — **best LUT/latency tradeoff in this sweep**
- **A2b_K32 (bit-parallel, 5 cycles)** → 12,305 LUTs / 2,905 MUXFx — strictly worse on both axes

A2b does not deliver a customer-deployable improvement.

## When A2b *might* still matter

1. **Vivado Fmax data may invert the picture.** A2b's stages have a K-deep `xtime` chain (deterministic), whereas the chained variant's stages have a K-deep `Z = Z ^ x_d·V; V = xtime(V)` cone (similar depth but Z-feedback). Vivado's critical-path analysis may rank them differently than yosys LC counts suggest. Worth one Vivado run on A2b_K32 to confirm / refute.
2. **A2b's per-stage logic is simpler to formally verify** (no Z accumulator dependency between iterations within a stage). Useful if the customer wants a formal proof path.

## What's next

The two structural rewrites that remain genuinely promising:

- **A8 (DSP48E2-as-XOR offload).** Moves the 128 LUT3-per-atomic XOR network into the DSP slice's bitwise XOR mode. xcku3p has 576 DSPs, plenty likely idle in the customer's design. Expected aggregate-LUT win for the 8-instance case: significant. Untested.
- **A3 (Karatsuba 64×64).** Splits the 128×128 polynomial multiplication into three 64×64 sub-multiplications. Asymptotically lower LUT count, but high algorithmic complexity and careful TB verification needed.

Both are higher-effort. **Recommended: get Vivado calibration** (one full P&R run on baseline + A6_ITER=32) before sinking more time into algorithmic rewrites — we need to know whether yosys's ranking holds in Vivado before chasing more variants on yosys numbers alone.

## Verification

A2b algorithm verified bit-exact against the iterative Python golden over **500 random vectors × 4 K values (2, 4, 8, 16) = 2 000 total checks, 0 mismatches**, using `a2b_staged()` in the verification harness. The RTL implements the same staged structure.
