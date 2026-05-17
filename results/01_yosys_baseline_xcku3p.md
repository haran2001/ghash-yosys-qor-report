# 01 — Yosys baseline on `xcku3p`, `ITER_PER_STAGE=128`

Pure-data note: the actual numbers from the first end-to-end yosys synth of the cleaned baseline RTL.

## Command

```bash
yosys -p '
read_verilog -sv rtl/ghash_mult.sv
hierarchy -top ghash_mult -chparam ITER_PER_STAGE 128
synth_xilinx -family xcup -nodsp
stat
'
```

Yosys version: 0.60 (`5bafeb77d`).

## Wall-clock

~10 minutes. CPU work itself was ~30 s; the rest was opt / opt_dff / opt_clean / abc / abc9 walking 128 distinct `ghash_mult_atomic` paramod instances individually.

## Per-atomic iteration

Every one of the 128 `ghash_mult_atomic` paramod instances compiled identically:

```text
=== $paramod\ghash_mult_atomic\ITERATION_NO=... ===

        12 wires (1282 bits)
         8 ports (1024 bits)
       131 cells:
         3   LUT2
       128   LUT3

   Estimated number of LCs: 128
```

Interpretation: 128 LUT3 = one GHASH bit-row per output bit, each taking (X-bit, V-bit, Z-bit) as inputs. The 3 LUT2 cells are the iteration's three small control muxes (`bit_int`, `lsb`, the polynomial-XOR enable). Confirms the algorithm is uniform across all 128 iterations.

## Full `ghash_mult` (ITER_PER_STAGE=128, 1 pipeline stage)

```text
=== ghash_mult ===

        22 wires (1293 bits)
         9 ports (390 bits)
       392 cells local:
         1   BUFG
       260   IBUF
       130   OBUF
         1   LUT2
       129   FDRE                     # 128 pipeline + 1 valid
       130   submodules
         1   ghash_mult_stage         # the 128-atomic chain

   Estimated number of LCs:        16,384
```

Total hierarchy (all submodules) = 17,160 cells.

## Resource breakdown

| Resource | Count | Notes |
|---|---:|---|
| LC estimate | 16,384 | yosys-LC; ~2× Vivado CLB-LUT (calibrated) |
| LUT3 (per atomic) | 128 | × 128 atomics = 16,384 LUT3 in the combinational core |
| LUT2 (per atomic) | 3 | × 128 = 384 LUT2 |
| LUT2 (top-level) | 1 | I/O wrapper logic |
| FDRE | 129 | 128-bit pipeline reg + 1 valid bit |
| BUFG | 1 | clock buffer |
| IBUF / OBUF | 260 / 130 | I/O cells (constant across variants) |
| CARRY8 / CARRY4 | 0 | pure XOR/mux, no arithmetic |
| DSP48E2 | 0 | `-nodsp` flag; A8 DSP-offload would change this |
| RAMB18E2 / RAMB36E2 | 0 | no memories |
| URAM288_BASE | 0 | no memories |
| MUXF7 / MUXF8 / MUXF9 | 0 | small LUT3 functions don't need mux fabric |

## Calibration vs the customer's Vivado run

| Quantity | Yosys (xcup) | Customer Vivado (KU3P) | Ratio |
|---|---:|---:|---:|
| Per `ghash_mult` instance | 16,384 LCs | ~8,000 CLB-LUTs | ~2.05× |
| Aggregate × 8 instances | ~131,000 LCs | ~64,000 CLB-LUTs | ~2.05× |

The ratio is stable on this single anchor. Yosys is **valid for relative ranking** of variants. Quote absolute numbers only from Vivado.

## What changes with `ITER_PER_STAGE`

- The **combinational core stays the same** (still 128 atomics in sequence overall).
- The **pipeline register count scales**: `129 × (128 / ITER_PER_STAGE)` FFs total.
- At ITER_PER_STAGE=32 (customer's default): 4 stages × 129 = 516 FDREs.
- At ITER_PER_STAGE=1: 128 stages × 129 ≈ 16 k FFs.
- Synthesis wall-clock should be lower at smaller ITER_PER_STAGE because each stage is a much smaller combinational block, but module-paramod count stays 128.
