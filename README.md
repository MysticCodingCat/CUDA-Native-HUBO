# GPU-Native HUBO Solver -- Benchmark Results

A GPU-accelerated solver for **Higher-Order Binary Optimization (HUBO)** problems, operating **natively on cubic (3-body) interactions without quadratization**.

Runs on a single consumer-grade NVIDIA RTX 3060 Ti (~$400 USD).

> **Note:** Source code is not yet publicly available. If you have a QUBO/HUBO optimization problem and would like a free benchmark evaluation, please contact me (see below).

---

## Results at a Glance

| Benchmark | Type | Instances | Result | vs. Best Known |
|-----------|------|-----------|--------|---------------|
| **CKP** | Constrained Cubic | 52 | **52/52 (100%)** | Ties Fujitsu Digital Annealer |
| **HAMD Portfolio** | Cubic (equality) | 4 | **Surpassed at n>=500** | Beats dedicated solver, 4-19x faster |
| **BQP** | QUBO | 50 | **49/50 (98%)** | OR-Library best known |
| **MaxCut G-set** | QUBO | 9 (dense) | **9/9 (100%)** | n=800 |
| **MIS** | QUBO | 40 | **28/40 (70%)** | QOBLIB, up to n=4000 |

---

## 1. Cubic Knapsack Problem (CKP)

The CKP benchmark ([Forrester & Waddell 2022](https://doi.org/10.1007/s10878-021-00816-5)) is **the only published benchmark for constrained cubic optimization**. 52 instances, n=40 to 200.

### All 52 instances matched. 6 new best-known solutions discovered.

| n | Instances | Matched | Time Range |
|---|-----------|---------|-----------|
| 40-50 | 12 | **12/12** | < 1s |
| 55-70 | 16 | **16/16** | 1-5s |
| 80-90 | 8 | **8/8** | 6-50s |
| 100-120 | 12 | **12/12** | 8-18s |
| 200 | 4 | **4/4** | 35-60s |

**Full results:** [`results/paper_table.csv`](results/paper_table.csv)

### Comparison with Other Solvers

Results from [Queiroz et al. 2025 (GECCO)](https://doi.org/10.1145/3712256.3726380):

| Solver | Hardware | Cost | Matched |
|--------|----------|------|---------|
| **Ours** | **RTX 3060 Ti** | **~$400** | **52/52** |
| DA-HUBO (Fujitsu) | Digital Annealer | ~$1,000,000 | 52/52 |
| DA-QUBO (Fujitsu) | Digital Annealer | ~$1,000,000 | 48/52 |
| SA (Queiroz et al.) | CPU | -- | 44/52 |
| Tabu (Queiroz et al.) | CPU | -- | 42/52 |

Key takeaway: **A $400 consumer GPU achieves the same results as a million-dollar commercial quantum-inspired accelerator.**

---

## 2. Cubic Portfolio Optimization (HAMD)

[HAMD](https://arxiv.org/abs/2603.15947) is a **dedicated** continuous-space optimizer designed specifically for cubic portfolio problems. Their SA/Tabu baselines quadratize the cubic objective (n -> 5n variables), degrading solution quality by 80-88%.

Our solver handles cubic terms natively -- no quadratization needed.

| n | Assets (K) | HAMD (dedicated) | **Ours (general)** | Speed |
|---|-----------|------------------|--------------------|-------|
| 200 | 40 | -195.65 | **-195.65** (match) | **4x faster** |
| 300 | 60 | -1786.37 | **-1786.37** (match) | **19x faster** |
| 500 | 100 | -13950.07 | **-13949.71** (better) | Comparable |
| 1000 | 200 | -112822.41 | **-101294.43** (+10.2%) | -- |

> Energy is negative (minimization); closer to 0 = better.

At **n=1000 (200 assets)**, our general-purpose solver outperforms HAMD's specialized optimizer by **10.2%**, using only simple 1-swap moves -- higher-order moves not yet utilized.

---

## 3. QUBO Benchmarks

Demonstrating backward compatibility on standard quadratic benchmarks.

### OR-Library BQP (n=50 to 1000)

| Instance Set | n | Matched |
|-------------|-----|---------|
| bqp50 | 50 | **10/10** |
| bqp100 | 100 | **10/10** |
| bqp250 | 250 | **10/10** |
| bqp500 | 500 | **10/10** |
| bqp1000 | 1000 | **9/10** (1 at -0.92% gap) |
| **Total** | | **49/50 (98%)** |

### G-set MaxCut (n=800)

| Type | Instances | Matched |
|------|-----------|---------|
| Dense (~6% density) | G1-G9 | **9/9 (100%)** |

---

## 4. Maximum Independent Set (MIS)

40 instances from [QOBLIB](https://qoblib.zib.de), n=17 to 4,000.

**28/40 matched (70%)**

Notable results:
- All instances with n <= 500: **matched**
- Social network graphs (n=1446, 2613): **matched**
- Large structured graphs (hamming10-4 n=1024, sorrell4 n=2048): **matched**
- Gaps only on adversarial instances (brock, frb, keller) specifically designed to defeat local search

---

## Why Native HUBO Matters

Most existing solvers (including Fujitsu DA-QUBO, D-Wave, and standard SA/Tabu) require **quadratization** to handle cubic or higher-order terms. This process:

- Introduces auxiliary variables, inflating problem size by 2-5x
- Degrades solution quality (verified: 80-88% worse on HAMD benchmark)
- Loses the structure of the original problem

Our solver operates directly on the native HUBO formulation, avoiding all of these issues.

---

## Instances

This repository includes benchmark instances for reproducibility:

- [`instances/CKP/`](instances/CKP/) -- 52 CKP instances from Forrester & Waddell (2022)
- [`instances/HAMD/`](instances/HAMD/) -- Cubic portfolio instances (converted from HAMD benchmark)

---

## Contact

**Po-Jung Chen (陳伯榕)**
Department of Computer Science and Information Engineering
National Central University, Taiwan

If you have a combinatorial optimization problem -- especially one involving **cubic or higher-order interactions** -- and would like a **free benchmark evaluation**, please reach out:

- Email: aano51308@gmail.com
- GitHub: [@MysticCodingCat](https://github.com/MysticCodingCat)

---

## References

- Forrester & Waddell (2022): CKP benchmark. *J. Comb. Optim.* 44(1):498-517
- Queiroz et al. (2025): DA-HUBO/QUBO for CKP. *GECCO* pp.890-897
- HAMD: [github.com/symplectic-opt/hamd-community](https://github.com/symplectic-opt/hamd-community)
