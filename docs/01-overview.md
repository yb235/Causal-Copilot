# 1. Overview — What is Causal‑Copilot?

> **TL;DR** Causal‑Copilot is an "AI scientist" that takes a spreadsheet of
> data plus a plain‑English question and answers: *"Which variables cause
> which?"* and *"How big is the effect?"* — automatically, end‑to‑end, with a
> nicely formatted PDF report at the end.

---

## 1.1 The problem it solves

Most data analysis answers questions about **correlation** ("when X goes up,
Y goes up"). But correlation is not causation. To make decisions — change a
policy, recommend a drug, redesign a product — you need **causal**
relationships: "if I *change* X, will Y change?"

Causal inference and causal discovery are mature academic fields with dozens
of algorithms (PC, FCI, GES, NOTEARS, LiNGAM, DoWhy, EconML, …). Each has its
own assumptions, hyper‑parameters, and gotchas. A non‑expert who just wants
an answer faces a wall of jargon.

**Causal‑Copilot wraps all of that complexity behind a chat interface.** You
upload data, type a question, and a Large Language Model (LLM) drives the
entire pipeline:

1. Looks at your data and figures out what kind of data it is.
2. Picks the right algorithm.
3. Picks reasonable hyperparameters.
4. Runs the algorithm.
5. Refines the result with bootstrapping and an LLM "second opinion".
6. Writes a PDF report explaining what it found, in human language.

---

## 1.2 What you give it & what you get back

### Input

* A **tabular dataset** (CSV or JSON). Rows are observations, columns are
  variables.
* A **natural‑language query**, e.g.
  > "Find what causes customer churn in this dataset."
* (Optional) **Domain knowledge** ("Age cannot be caused by Income"), a
  ground‑truth graph for evaluation, etc.

### Output

* A **causal graph** (PNG/PDF) showing arrows between variables.
* A **bootstrap heatmap** showing how confident the system is about each edge.
* A **LaTeX‑compiled PDF report** that includes:
  * Data summary (sample size, missing values, data types).
  * Exploratory plots (correlation heatmap, distributions).
  * Which algorithm was chosen and why.
  * The discovered causal graph.
  * A plain‑English interpretation of every causal arrow.
  * (Optional) Estimated treatment effects from causal inference.
* If you used the web demo, an **interactive chat** where you can ask
  follow‑up questions about the report.

---

## 1.3 The two ways to run it

| Mode | Command | When to use |
|------|---------|-------------|
| **CLI** | `python main.py --data-file my.csv --initial_query "..."` | Batch jobs, scripting, focuses on causal *discovery*. |
| **Web demo** | `python web_demo/demo.py` | Interactive use; covers discovery + inference + chat. |

Beginners should start with the web demo — see
[`09-getting-started.md`](09-getting-started.md).

---

## 1.4 Two big subjects, one tool

The repository covers **two related but distinct tasks**:

### A. Causal **discovery** (`causal_discovery/`)
> *"I have data. Draw me the cause‑and‑effect graph."*

Output is a **directed graph** (DAG or PAG) over your variables. Algorithms
include PC, FCI, GES, FGES, XGES, NOTEARS, GOLEM, GRaSP, CORL,
DirectLiNGAM, ICALiNGAM, VARLiNGAM, PCMCI, NTS‑NOTEARS, DyNotears, Granger,
plus Markov‑Blanket variants (HITON‑MB, IAMBnPC, InterIAMB, MBOR, BAMB) and
hybrids (CALM).

### B. Causal **inference** (`causal_inference/`)
> *"Given a graph (or treatment / outcome / confounders), how big is the
> effect?"*

Output is **numerical effect estimates**, often *heterogeneous* (different
sub‑populations get different effects). Methods come from **DoWhy** and
**EconML**: Double Machine Learning (DML), Doubly Robust Learners (DRL),
Instrumental Variables (IV), Meta‑Learners (S/T/R/X), and Uplift / Causal
Forests.

The two halves talk to each other: discovery proposes a graph, inference can
then quantify any specific cause→effect relationship in that graph.

---

## 1.5 Why an LLM?

The hard parts of causal analysis aren't the algorithms — those are libraries
you can `pip install`. The hard parts are:

* **Choosing** the right algorithm given your data's quirks.
* **Tuning** its hyperparameters.
* **Sanity‑checking** the resulting graph (does it match common sense?).
* **Explaining** the result in human language.

The LLM in Causal‑Copilot is used for exactly these "soft" decisions, not for
the math itself. The math is still done by classical, deterministic libraries
(causal-learn, gcastle, DoWhy, EconML, …). See
[`07-llm-integration.md`](07-llm-integration.md) for the precise list of LLM
calls.

---

## 1.6 Versions

* **V1** (Nov 2024) — first autonomous causal analysis agent.
* **V2** (Apr 2025) — current; integrates 20+ state‑of‑the‑art techniques and
  the modular architecture documented here. Accompanied by a
  [technical report on arXiv](https://arxiv.org/pdf/2504.13263).

---

## 1.7 Where to go next

* Want the big picture? → [`02-architecture.md`](02-architecture.md)
* Want to actually run it? → [`09-getting-started.md`](09-getting-started.md)
* Confused by a word like "DAG" or "confounder"? →
  [`10-glossary.md`](10-glossary.md)
