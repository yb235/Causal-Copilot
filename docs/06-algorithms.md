# 6. Algorithm Catalog

Causal‑Copilot integrates 25+ causal discovery algorithms and 10+ causal
inference estimators. This chapter inventories them, grouped by family, with
a one‑line description of each. Use it as a lookup table when reading the
LLM's algorithm choices in your reports.

> Source of truth:
> * Discovery wrappers — [`causal_discovery/wrappers/`](../causal_discovery/wrappers)
> * Inference estimators — [`causal_inference/`](../causal_inference)

---

## 6.1 Causal discovery algorithms

All wrappers expose the same interface: `Wrapper(**hyperparams).fit(data)`
returning a graph object that `causal_discovery/program.py` converts to a
numpy adjacency matrix.

### 6.1.1 Constraint‑based (conditional independence tests)

| Wrapper file | Algorithm | Notes |
|--------------|-----------|-------|
| `pc.py` | **PC** | Peter–Clark. The classical CI‑based skeleton + orientation algorithm. |
| `pc_parallel.py` | **PC (GPU)** | GPU‑accelerated PC skeleton search via `externals/pc_adjacency_search`. |
| `fci.py` | **FCI** | Like PC but tolerates *latent confounders*; produces a Partial Ancestral Graph (PAG). |
| `cdnod.py` | **CD‑NOD** | Constraint‑based discovery for **non‑stationary / heterogeneous** data; uses a domain index. |

### 6.1.2 Score‑based (search a graph space, score it)

| Wrapper file | Algorithm | Notes |
|--------------|-----------|-------|
| `ges.py` | **GES** | Greedy Equivalence Search. |
| `fges.py` | **FGES** | "Fast" GES, scalable parallel implementation. |
| `x_ges.py` | **XGES** | Extended GES (`xges` package) — accuracy and speed improvements over GES/FGES. |
| `grasp.py` | **GRaSP** | Greedy Relaxations of the Sparsest Permutation — strong empirical performance. |
| `corl.py` | **CORL** | Causal discovery with **Reinforcement Learning**. |

### 6.1.3 Continuous‑optimization (NOTEARS family)

| Wrapper file | Algorithm | Notes |
|--------------|-----------|-------|
| `notears_linear.py` | **NOTEARS (linear)** | Smooth acyclicity constraint, gradient descent over edge weights. |
| `notears_nolinear.py` | **NOTEARS (non‑linear)** | MLP variant. |
| `golem.py` | **GOLEM** | Likelihood‑based reformulation of NOTEARS, often more stable. |
| `calm.py` | **CALM** | Causal‑aware learning model (continuous optimization variant). |

### 6.1.4 Functional / non‑Gaussian (LiNGAM family)

LiNGAM exploits non‑Gaussianity to identify the *exact* causal direction.

| Wrapper file | Algorithm | Notes |
|--------------|-----------|-------|
| `direct_lingam.py` | **DirectLiNGAM** | Closed‑form, direction by direction. |
| `ica_lingam.py` | **ICA‑LiNGAM** | Original ICA‑based formulation. |
| `var_lingam.py` | **VAR‑LiNGAM** | LiNGAM extended to **time series** via VAR. |

### 6.1.5 Time‑series specific

| Wrapper file | Algorithm | Notes |
|--------------|-----------|-------|
| `pcmci.py` | **PCMCI** | Tigramite's flagship constraint‑based TS algorithm (PC + Momentary Conditional Independence). |
| `granger_causality.py` | **Granger** | Classical Granger causality (linear, predictive). |
| `ts_cdnod.py` | **TS‑CD‑NOD** | Time‑series flavour of CD‑NOD. |
| `dynotears.py` | **DyNOTEARS** | NOTEARS extended with lagged edges. |
| `nts_notears.py` | **NTS‑NOTEARS** | Non‑linear time‑series NOTEARS. |
| `var_lingam.py` | **VAR‑LiNGAM** | (Re‑listed here — also a TS method.) |

### 6.1.6 Markov‑Blanket / local feature selection

These don't return a global DAG; they return the **Markov blanket** (parents,
children, and spouses) of a target — useful when you only care about what
directly influences one variable.

| Wrapper file | Algorithm |
|--------------|-----------|
| `hiton_mb.py` | **HITON‑MB** |
| `iambnpc.py` | **IAMBnPC** |
| `inter_iamb.py` | **Inter‑IAMB** |
| `mbor.py` | **MBOR** |
| `bamb.py` | **BAMB** |

These come from the `pyCausalFS` submodule.

### 6.1.7 Hybrid

| Wrapper file | Algorithm | Notes |
|--------------|-----------|-------|
| `hybrid.py` | **HYBRID** | Combines complementary methods (e.g. constraint + score). |
| `calm.py` | **CALM** | Listed under continuous optimization but blends scoring and constraints. |

### 6.1.8 Quick chooser cheat sheet

| Your data | Try first |
|-----------|-----------|
| Linear, Gaussian, no time component | PC, GES, FGES |
| Linear, **non‑Gaussian** | DirectLiNGAM, ICA‑LiNGAM |
| Non‑linear, observational | NOTEARS‑NL, GOLEM, GRaSP |
| Latent confounders likely | FCI |
| Heterogeneous / multi‑domain | CD‑NOD |
| Time series | PCMCI, VAR‑LiNGAM, DyNOTEARS, NTS‑NOTEARS, Granger |
| Just want one variable's drivers | HITON‑MB, IAMBnPC, BAMB |
| Large graphs, GPU available | PC‑parallel, FGES, XGES |

In practice you don't pick — the LLM does. This table is for understanding
*why* it chose what it did.

---

## 6.2 Causal inference estimators

The inference half of the system is built on top of **DoWhy** (the
"identify → estimate → refute" workflow) and **EconML** (machine‑learning
treatment‑effect estimators). Wrappers and configuration live under
[`causal_inference/`](../causal_inference).

### 6.2.1 Average and conditional effects

* **DML — Double Machine Learning** (`causal_inference/DML/`)
  - `LinearDML` — linear final stage.
  - `CausalForestDML` — non‑parametric forest final stage. Good for
    heterogeneous effects.

* **DRL — Doubly Robust Learners** (`causal_inference/DRL/`)
  - Use both an outcome model and a propensity model; remain consistent
    if either is correctly specified.

* **MetaLearners** (`causal_inference/MetaLearners/`)
  - **S‑Learner** — single model treating treatment as a feature.
  - **T‑Learner** — separate models for treated and control.
  - **R‑Learner** — residualization‑based.
  - **X‑Learner** — cross‑learner; handles imbalanced treatment groups.

### 6.2.2 Instrumental variables

* **IV** (`causal_inference/IV/`)
  - 2SLS (Two‑Stage Least Squares) for unobserved confounding when a
    valid instrument exists.

### 6.2.3 Uplift / treatment‑effect forests

* **Uplift** (`causal_inference/Uplift/`)
  - **Causal Forest / Generalized Random Forest (GRF)** — non‑parametric
    estimator of CATE.

### 6.2.4 Cross‑cutting

* **Identification** — DoWhy's graph‑based identification (back‑door,
  front‑door, IV) is used to verify that the requested effect is
  estimable from the observed data.
* **Refutation** — sensitivity analyses (placebo treatment, random
  common cause, data subset) are run automatically and their results end
  up in the inference report.

### 6.2.5 Quick chooser cheat sheet

| Question | Estimator |
|----------|-----------|
| "What is the average effect of T on Y?" | LinearDML, T‑Learner |
| "How does the effect vary across people?" (CATE) | CausalForestDML, X‑Learner, GRF |
| "I have an instrument Z" | IV (2SLS) |
| "Treated and control groups are very imbalanced" | X‑Learner |
| "I want maximum robustness to model misspecification" | DRL |

---

## 6.3 Adding a new algorithm

The repo is designed to make this easy. To add a discovery algorithm:

1. **Implement** the algorithm (or pin a third‑party package). If it's a
   git project, add it as a submodule under `externals/`.
2. **Write a wrapper** in `causal_discovery/wrappers/<your_algo>.py`. Inherit
   from the base in `wrappers/base.py` and implement `__init__` and
   `fit(data)`.
3. **Register it** in `causal_discovery/wrappers/__init__.py` and in the
   algorithm context files under `causal_discovery/context/algos/`. The
   context entry tells the LLM when this algorithm is appropriate and
   what its hyperparameters mean.
4. **(Optional) add benchmarks** so the reranker has performance data.

The LLM will start considering your algorithm on the next run.

For inference, the equivalent is to add a new module under
`causal_inference/<family>/` and register it in `inference.py`.
