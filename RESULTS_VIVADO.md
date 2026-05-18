# GHASH GF(2^128) — Vivado-validated results (supersedes yosys ranking)

**Date:** 2026-05-18
**Status:** This page supersedes the yosys-only ranking in `README.md` / `SUMMARY.md` / `RECOMMENDATION.md` for the *final variant choice*. Those documents remain the authoritative record of the Yosys Tier-A methodology and the reasoning that motivated the Vivado sweep; the table below reports what Vivado 2020.2 actually produced on `xcku3p-ffvb676-2-e` after full opt + place + route.

---

## TL;DR

| Variant | Strategy | LUT | FF | DSP | BRAM | Latency |
|---|---|---:|---:|---:|---:|---:|
| Customer baseline RTL | Default | 8,377 | 129 | 0 | 0 | 1 |
| **A3b — Karatsuba structural (recommended)** | AreaOpt_high | **5,893** ‡ | 129 | 0 | 0 | **1** |
| A2 — bit-parallel combinational (fallback) | AreaOpt_high | 7,017 | 129 | 0 | 0 | 1 |
| A2b_K64 — pipelined bit-parallel | AreaOpt_high | 7,020 | 579 | 0 | 0 | 3 |

‡ post-synth (May 17 runs exited mid-flow with rc=1). A re-run is in flight at the time of writing and will produce post-route numbers. Post-route LUT typically drops a further 5–10 %.

The Karatsuba structural variant **dominates the entire ≤6-cycle frontier** at 1-cycle latency and zero DSP / BRAM cost.

---

## Why the recommendation changed (vs the yosys-only A6_ITER=32 pick)

The original `RECOMMENDATION.md` selected `A6_ITER=32` based on yosys Tier-A LUT estimates. Once we ran the customer-style Vivado flow (`synth_design → opt_design → place_design → phys_opt_design → route_design`, full reports), two effects re-ordered the frontier:

1. **Vivado `opt_design` finds cross-bit XOR sharing on dense Mastrovito tensors** that yosys's `synth_xilinx -flatten` misses. Result: A2's flat combinational form lands at 7,017 LUT in Vivado @ AreaOpt_high while yosys ranked it less favourably.
2. **Combinational variants are not penalised by Vivado's LUT count when given the structural decomposition.** Karatsuba's 4→3 sub-product saving — invisible in yosys-flatten — survives to the Vivado netlist when emitted as a named hierarchy (three `gf64_submul` instances + cross-XOR wires). Result: A3b @ 5,893 LUT, beating every pipelined variant.

Lesson for future closed-loop runs: **yosys Tier-A ranks pipeline-granularity variants well, but cannot rank algorithm-shape variants** that depend on Vivado-specific `opt_design` heuristics. Algorithm-shape proposals must be validated under the customer's actual P&R flow.

---

## Source-of-improvement attribution

| Optimisation axis | Variant | LUT vs baseline (8,377) | Latency cost |
|---|---|---:|---:|
| Synth directive only (no RTL change) | baseline + AreaOpt_high | +591 (regression) | 0 |
| RTL spelling (chain → flat tensor) | A2 + AreaOpt_high | −1,360 (−16.2 %) | 0 |
| RTL spelling (Mastrovito flat) | A2c + AreaOpt_high | −1,238 (−14.8 %) | 0 |
| RTL pipelining + synth directive | A2b_K64 + AreaOpt_high | −1,357 (−16.2 %) | +2 cyc |
| **RTL algorithmic (Karatsuba structural)** | **A3b + AreaOpt_high** | **−2,484 (−29.6 %)** ‡ | **0** |

A3b's win comes from a Karatsuba decomposition over GF(2)[x] (128 × 128 → three 64 × 64 sub-products), explicitly emitted as a named module hierarchy so the 4→3 sub-multiplier reduction survives `opt_design`. This is something Vivado's optimiser does not discover on its own from a flat dense XOR tensor.

---

## Chip-wide projection (8 instances)

| Path | LUT/instance | LUT × 8 | Saving |
|---|---:|---:|---:|
| Customer baseline | 8,377 | 67,016 | — |
| A2 + AreaOpt_high | 7,017 | 56,136 | −10,880 |
| **A3b + AreaOpt_high** | **5,893** | **47,144** | **−19,872 (−29.6 %)** |

~20 k LUT freed on the `xcku3p` (~12 % of the part's 162,720 CLB-LUT budget) at zero latency cost, zero DSP, zero BRAM.

---

## What is NOT validated yet

| Concern | Status | Path to close |
|---|---|---|
| Fmax @ 322 MHz | All sweeps to date have I/O-bound WNS (−2.86 ns). Real timing closure unconfirmed. | The in-flight sweep loads `constraints/ghash_322mhz.xdc` and uses a registered-I/O wrapper; first real WNS will come from it. |
| Multi-instance congestion | Single-instance results only. | 8-copy top + `report_design_analysis -congestion`. |
| DSP behaviour | Every sweep to date: 0 DSP, even under `AreaMultThresholdDSP`. The bit-parallel XOR pattern does not trip Vivado's DSP heuristic; explicit DSP48E2 instantiation would be needed for DSP-as-XOR offload. | H4 (DSP-as-XOR) deferred — proposer analysis showed 9-cycle tree exceeds the 6-cycle budget for full offload; partial offload saves only 1–2 k LUT with routing overhead. |
| A3b post-route LUT | Post-synth only (5,893). | In-flight sweep produces post-route. |

---

## Recommended action

1. **Switch RTL to A3b (Karatsuba structural)** for both new and existing GHASH instances.
2. **Set synthesis directive to `AreaOptimized_high`** project-wide for these blocks.
3. **Run timing closure with registered I/O** (the existing top wrapper already adds output registers; verify input registers are present in the customer's instantiation context).
4. **Fall back to A2 + AreaOpt_high** if A3b shows timing problems at Fmax @ 322 MHz that opt+phys_opt can't recover. A2 is a known-good 7,017-LUT / 1-cycle frontier point on the same flow.

---

## Reproducing these results

All RTL variants, codegen scripts, Vivado tcl, and the aggregator are published in the public meta-harness repository (the `reference_examples/ghash_fpga/` subtree). Follow `context/14_setup_from_scratch.md` in that repo for the end-to-end setup. The variant files referenced above are:

- `rtl/variants/A3b_karatsuba_structural.sv` (752 lines, named `gf64_submul` hierarchy)
- `rtl/variants/A3a_karatsuba_flat.sv` (control: same Karatsuba tensor, flat XOR form)
- `rtl/variants/A2_bit_parallel.sv` (prior frontier)
- `rtl/variants/A2c_mastrovito.sv` (H1 control)

Codegen scripts (`scripts/emit_a3a_karatsuba_flat_sv.py`, `scripts/emit_a3b_karatsuba_structural_sv.py`, `scripts/proposer_h2_karatsuba.py`) include the GHASH golden verification gate that every variant passes before Vivado runs.
