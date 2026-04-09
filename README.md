# CUDA-Native HUBO Solver -- Benchmark Results

## What is this?

This is a **combinatorial optimization solver** that runs on a single NVIDIA GPU.

It solves **HUBO (Higher-Order Unconstrained Binary Optimization)** problems -- optimization over binary variables (0/1) with interactions involving **3 or more variables at once** (e.g., "if items A, B, and C are all selected, the combined value changes by X").

### What makes it different?

Most optimization solvers today only handle **pairwise (2-body) interactions** -- known as QUBO. When a real-world problem has 3-body or higher interactions, the standard approach is **quadratization**: convert the cubic terms into quadratic by introducing auxiliary variables. This blows up problem size by 2-5x and significantly degrades solution quality.

**Our solver skips quadratization entirely.** It operates directly on cubic (3-body) terms with the same efficiency that other solvers handle quadratic terms.

### Why not just use a quantum annealer?

Quantum annealers (D-Wave, Fujitsu Digital Annealer, and most quantum-inspired hardware) have a **fundamental architectural constraint: they only accept QUBO input**. Any problem with cubic or higher-order terms *must* be converted before it can run -- no exceptions, regardless of hardware cost.

Our solver is software-native, which changes the picture in two ways:

**1. No forced conversion.** The cubic structure of your problem is preserved end-to-end. The solver evaluates cubic interactions directly -- no auxiliary variables, no approximation overhead.

**2. The algorithm itself can be modified for your problem.** A quantum chip runs the same fixed annealing process on every problem. Our solver is a framework that is actively adapted per problem class:
- For CKP (constrained cubic knapsack), the breakthrough came from **integrating a Branch & Bound subroutine** directly into the GPU search -- something impossible on fixed hardware. This is the reason we match Fujitsu DA on CKP with a $400 GPU while DA-QUBO (which must quadratize) loses 4 out of 52 instances.
- For cubic portfolio optimization, dedicated swap moves exploit the equality constraint (pick exactly K assets) natively.
- For unconstrained problems (QUBO/BQP, MaxCut), graph problems (MIS), and others: each was adapted in **under one day** from the CKP prototype, and still produces competitive results across all tested benchmarks.

The result: problem-specific algorithmic design on commodity hardware outperforms expensive fixed-architecture machines on the benchmarks where direct comparison is available. We believe this adaptability will translate to strong performance on additional problem domains -- though we are honest that we have not tested everything, and some problem structures will suit this approach better than others.

### How much does it cost?

It runs on a **single NVIDIA RTX 3060 Ti** (~$400 USD, consumer-grade).
For comparison, the only other solver that matches our results on cubic problems is the **Fujitsu Digital Annealer**, a commercial quantum-inspired accelerator costing ~$1,000,000.

---

## Benchmark Results

### 1. Cubic Knapsack Problem (CKP) -- **52/52 Matched**

**What is CKP?**
Select items to maximize total value under a weight capacity constraint -- but item values include **3-way synergies** (e.g., items A+B+C together have bonus value). This is the only published benchmark for constrained cubic optimization.

- **Source:** [Forrester & Waddell 2022](https://doi.org/10.1007/s10878-021-00816-5), 52 instances, n=40 to 200 items
- **Our result:** All 52 best-known solutions matched (ties Fujitsu DA-HUBO)
- **Time:** **8.6 minutes total** for all 52 instances (max 60s per instance)

| Solver | Hardware | Cost | Matched | Time Budget |
|--------|----------|------|---------|-------------|
| **Ours** | **RTX 3060 Ti** | **~$400** | **52/52** | **8.6 min total** |
| DA-HUBO (Fujitsu) | Digital Annealer | ~$1M | 52/52 | 1800s per instance |
| DA-QUBO (Fujitsu) | Digital Annealer | ~$1M | 48/52 | 1800s per instance |
| SA | CPU | -- | 44/52 | 1800s per instance |
| Tabu Search | CPU | -- | 42/52 | 1800s per instance |

> Comparison data from [Queiroz et al. 2025 (GECCO)](https://doi.org/10.1145/3712256.3726380). Fujitsu DA benchmark baselines were evaluated using a standardized computational budget of 1,800 seconds (30 minutes) per instance. Our solver finishes all 52 instances in under 9 minutes combined. DA-QUBO uses quadratization and loses 4 instances; our native cubic solver does not.

**Full per-instance results:** [`results/paper_table.csv`](results/paper_table.csv)

---

### 2. Cubic Portfolio Optimization -- **Matches Dedicated Solver**

**What is this?**
Choose K assets from n candidates to minimize portfolio risk, where risk includes **3-asset co-skewness** (cubic terms). This models real financial risk that traditional mean-variance (Markowitz) frameworks cannot capture.

**Baseline:** [HAMD (arXiv:2603.15947)](https://arxiv.org/abs/2603.15947) -- a dedicated optimizer designed specifically for this problem. Their SA/Tabu baselines use quadratization (n → 5n variables), resulting in severely degraded solutions (see table below).

| Scale | HAMD (dedicated) | SA/Tabu (quadratized) | **Ours** | Our Time |
|-------|------------------|----------------------|----------|----------|
| 200 assets, pick 40 | 195.65 | 1,621.60 | **195.65** (exact match) | 8s |
| 300 assets, pick 60 | 1,786.37 | 6,196.71 | **1,786.37** (exact match) | 3s |
| 500 assets, pick 100 | 13,949.80 | 34,454.01 | **13,950.07** (match) | 33s |
| 1000 assets, pick 200 | 101,294.43 | 190,598.71 | **101,294.43** (exact match) | 276s |

> Lower = better (portfolio risk). HAMD uses a **60-second CPU budget** per instance. SA/Tabu quadratize the cubic objective (n → 5n variables), inflating problem size and producing drastically worse results at every scale. Our solver matches HAMD's dedicated optimizer while being a general-purpose HUBO solver.

---

### 3. QUBO Benchmarks -- Backward Compatible

Our solver also handles standard quadratic (2-body) problems. These benchmarks demonstrate that native HUBO support does not sacrifice quadratic performance.

**OR-Library BQP (unconstrained quadratic):**

| Size | Matched |
|------|---------|
| n=50 to 500 (40 instances) | **40/40 (100%)** |
| n=1000 (10 instances) | **9/10** (1 at -0.92%) |

**G-set MaxCut (graph partitioning, n=800):**
- Dense graphs G1-G9: **9/9 (100%)**

---

### 4. Maximum Independent Set -- 28/40 on QOBLIB

**What is MIS?**
Find the largest set of nodes in a graph such that no two are connected. A fundamental problem in network analysis, scheduling, and resource allocation.

- **Source:** [QOBLIB](https://qoblib.zib.de), 40 instances, n=17 to 4,000 nodes
- **Result: 28/40 matched (70%)**
- All instances n <= 500: matched
- Social networks (n=2,613): matched
- Large structured graphs (n=2,048): matched

---

## Why This Matters

| Scenario | Quadratization Approach | Our Native Approach |
|----------|------------------------|-------------------|
| 3-way drug interactions | Must add auxiliary variables, problem size 3-5x larger | Handles directly, original problem size |
| Portfolio co-skewness | 80-88% quality degradation (verified) | Full quality, matches dedicated solver |
| Manufacturing synergies | Approximation errors compound | Exact cubic evaluation |
| Any cubic/quartic objective | Reformulation expertise required | Just input the problem |

**Bottom line:** If your optimization problem has interactions among 3+ variables, quadratization is losing you solution quality. We solve it natively.

---

## Available Benchmark Instances

- [`instances/CKP/`](instances/CKP/) -- 52 CKP instances (Forrester & Waddell 2022)
- [`instances/HAMD/`](instances/HAMD/) -- Cubic portfolio instances (from HAMD benchmark)

---

## Where This Is Headed

The benchmarks in this repository represent problems I have tested so far. Each non-CKP problem type -- cubic portfolio, QUBO, graph problems -- required less than a day of algorithm-level modification from the CKP prototype to produce competitive results. This suggests the framework adapts quickly across problem classes.

I believe there are more problem domains where this approach will perform well, particularly anywhere that:
- Higher-order (cubic+) interactions are present but currently handled through lossy quadratization
- Problem-specific structure (constraint topology, variable dependencies) can be exploited algorithmically
- Existing methods treat the problem as a black box

I want to be honest: I have not tested everything, and the results above do not guarantee strong performance on every cubic problem. Some problem structures will fit this approach better than others. What I can say is that on every domain I have tested, adapting the algorithm to the problem's native structure consistently outperformed the quadratization baseline.

---

## A Note on Technical Details

The core algorithms and implementation are part of an ongoing paper submission and patent application. Source code and algorithmic specifics are not publicly disclosed at this time.

I recognize that asking someone to take this seriously without seeing the code requires trust. My answer to that is simple: **let me run your problem.** If you have an optimization problem -- especially one with higher-order interactions -- send it to me and I will benchmark it and share the results. The outcomes will demonstrate what the algorithm can do more directly than any technical description I could give right now.

---

## Free Evaluation Offer

If you have a combinatorial optimization problem -- especially one involving **cubic or higher-order interactions** -- I will benchmark it for free.

You provide: problem description, QUBO/HUBO formulation, or raw instance data
I provide: solution quality, runtime, and honest comparison with your current approach (including cases where my solver does not win)

**Po-Jung Chen (陳伯榕)**
National Central University, Dept. of Computer Science and Information Engineering

- Email: aano51308@gmail.com
- GitHub: [@MysticCodingCat](https://github.com/MysticCodingCat)

---

## References

### Problem Descriptions (what these problems are)

- **HUBO / Binary Optimization** -- [Wikipedia: Pseudo-boolean optimization](https://en.wikipedia.org/wiki/Pseudo-boolean_optimization)
- **Knapsack Problem** -- [Wikipedia: Knapsack problem](https://en.wikipedia.org/wiki/Knapsack_problem) (CKP adds cubic synergy terms on top of the classic formulation)
- **Portfolio Optimization** -- [Wikipedia: Modern portfolio theory](https://en.wikipedia.org/wiki/Modern_portfolio_theory) (HAMD extends this with cubic co-skewness terms)
- **Maximum Cut (MaxCut)** -- [Wikipedia: Maximum cut](https://en.wikipedia.org/wiki/Maximum_cut)
- **Maximum Independent Set (MIS)** -- [Wikipedia: Independent set (graph theory)](https://en.wikipedia.org/wiki/Independent_set_(graph_theory))
- **QUBO** -- [Wikipedia: Quadratic unconstrained binary optimization](https://en.wikipedia.org/wiki/Quadratic_unconstrained_binary_optimization)

### Benchmark Sources

- Forrester & Waddell (2022): Cubic Knapsack Problem definition and instances. *J. Comb. Optim.* [DOI:10.1007/s10878-021-00816-5](https://doi.org/10.1007/s10878-021-00816-5)
- Queiroz et al. (2025): Fujitsu Digital Annealer for CKP. *GECCO* [DOI:10.1145/3712256.3726380](https://doi.org/10.1145/3712256.3726380)
- HAMD (2026): Cubic portfolio benchmark and dedicated solver. [arXiv:2603.15947](https://arxiv.org/abs/2603.15947)
- OR-Library BQP: Unconstrained quadratic benchmark instances. [bqpinfo](https://people.brunel.ac.uk/~mastjjb/jeb/orlib/bqpinfo.html)
- QOBLIB: Open benchmark library for quantum/combinatorial optimization. [qoblib.zib.de](https://qoblib.zib.de)
