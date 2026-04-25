# 10. Glossary

A plain‑English mini‑dictionary of the jargon you'll encounter when reading
Causal‑Copilot reports, prompts, or source code.

---

### ACF / PACF
**Autocorrelation function** / **Partial autocorrelation function**. Plots
that show how a time‑series value relates to its past values. Used by the
EDA module for time‑series data.

### Adjacency matrix
A square matrix `A` where `A[i, j] = 1` means "there is an arrow from
variable `i` to variable `j`". The internal canonical form for graphs in
Causal‑Copilot.

### ATE — Average Treatment Effect
The average effect of switching treatment from off to on across the whole
population. Estimated by causal‑inference methods like DML.

### Backdoor / Frontdoor criterion
Graph‑based rules from Pearl's framework that tell you whether a causal
effect can be estimated from observational data, and which variables you
need to condition on. Used internally by DoWhy.

### Bootstrap
Re‑sample your data with replacement, run the algorithm on each sample,
and aggregate. Causal‑Copilot does this 20× by default to estimate the
*confidence* of each edge.

### CATE — Conditional Average Treatment Effect
Like ATE, but for a specific sub‑population (e.g. "the effect of the drug
on patients over 60"). Heterogeneous‑effect estimators (DML, X‑Learner,
GRF, …) target CATE.

### CDNOD
"Causal Discovery from heterogeneous / **NO**n‑stationary **D**ata". Used
when your data comes from multiple regimes / domains.

### CI test — Conditional Independence test
Statistical test of "is X independent of Y given Z?". The fundamental
building block of constraint‑based methods (PC, FCI). Examples: Fisher‑Z,
Chi‑square, KCI.

### Confounder
A variable that influences both the treatment and the outcome, biasing
naive correlations. The whole point of causal inference is to account for
these.

### CPDAG — Completed Partially Directed Acyclic Graph
The output of PC and GES. Some edges are directed (we know the direction);
others are undirected (the data alone can't tell). Causal‑Copilot stores
whether the user accepts CPDAGs in `UserData.accept_CPDAG`.

### DAG — Directed Acyclic Graph
A graph where every edge has a direction and no cycles exist. The "ideal"
output of causal discovery.

### DML — Double Machine Learning
A technique where you fit machine‑learning models to predict the outcome
and the treatment from confounders, then estimate the effect from the
*residuals*. Robust to model misspecification in either model.

### DoWhy
Python library implementing Pearl‑style causal inference (model →
identify → estimate → refute). Used by `causal_inference/`.

### EconML
Microsoft's library of ML‑powered treatment‑effect estimators (DML, DR
learners, MetaLearners, …). Used by `causal_inference/`.

### EDA — Exploratory Data Analysis
The "look at the data before modelling it" step. Causal‑Copilot's
`preprocess/eda_generation.py` produces correlation heatmaps,
distributions, and missingness maps.

### FCI — Fast Causal Inference
Constraint‑based algorithm that *allows* for hidden confounders, producing
a PAG instead of a CPDAG.

### Granger causality
"X Granger‑causes Y" if past values of X help predict Y beyond what Y's
own past predicts. Predictive, not strictly causal — but classical for
time series.

### GRF — Generalized Random Forest
Forest‑based non‑parametric estimator of heterogeneous treatment effects.

### Ground truth
The "true" causal graph used as a reference when evaluating an algorithm
on synthetic / well‑studied data.

### HTE — Heterogeneous Treatment Effect
Same as CATE — recognising that different individuals can react
differently to a treatment.

### IV — Instrumental Variable
A variable that affects the treatment but not the outcome directly. Lets
you recover causal effects even when there are unobserved confounders.

### KCI — Kernel Conditional Independence test
A non‑parametric CI test that handles non‑linear relationships. Used by
`postprocess/judge.py` for edge pruning.

### LiNGAM — Linear Non‑Gaussian Acyclic Model
A family of algorithms that exploit *non‑Gaussianity* to identify the
exact direction of every edge. Includes DirectLiNGAM, ICA‑LiNGAM,
VAR‑LiNGAM.

### LLM — Large Language Model
The "AI assistant" component (GPT‑4, Claude, Llama, …). In
Causal‑Copilot, it makes judgment calls (algorithm choice, hyperparameter
selection, edge interpretation) but never runs the math itself.

### Markov blanket
For a target variable T, the set of T's parents, children, and the
parents of T's children. Knowing the Markov blanket is enough to predict
T optimally. Returned by HITON‑MB, IAMB family, MBOR, BAMB.

### MetaLearners
A family of treatment‑effect estimators (S, T, R, X) built on top of any
regression model. Useful when you want to plug in your favourite ML
algorithm.

### NOTEARS
"**NO** **T**ears". A score‑based method that turns DAG learning into a
*continuous* optimisation problem with a smooth acyclicity constraint.

### Outcome
The variable you want to influence (e.g. churn, revenue, blood pressure).

### PAG — Partial Ancestral Graph
A more expressive graph type than CPDAG, output by FCI. Edges can be
directed, undirected, or have circle endpoints to encode "we don't know
because of latent confounders".

### PCMCI
Time‑series causal discovery from the Tigramite library. Combines PC's
skeleton search with momentary conditional independence tests across
lags.

### Pearl, Judea
Author of *Causality* (2009). Most of the formal causal‑inference
framework Causal‑Copilot uses traces back to his work.

### Pearl ladder of causation
1. Association ("seeing"), 2. Intervention ("doing"), 3. Counterfactuals
("imagining"). Causal‑Copilot mostly operates at level 2; counterfactual
estimation is supported via DoWhy.

### Pre‑processing
Cleaning, type detection, missing‑value handling, statistical profiling.
Lives in `preprocess/`.

### Post‑processing
Bootstrap, LLM pruning, cycle breaking, plotting. Lives in `postprocess/`.

### Refutation
Sensitivity analyses you run to see if your estimated effect survives
perturbations: random common cause, placebo treatment, data subsets.
Standard part of the DoWhy workflow.

### Stationarity
A time series is stationary if its statistical properties don't change
over time. Tested by ADF (Augmented Dickey‑Fuller). Many time‑series
algorithms assume stationarity.

### Treatment
The variable you're nudging (e.g. drug yes/no, ad exposure yes/no, a
continuous price).

### Tigramite
Python library specialising in time‑series causal discovery (PCMCI and
its variants). Vendored as an `externals/` submodule.

### XGES
"Extended GES" — a faster and often more accurate score‑based
discovery algorithm.
