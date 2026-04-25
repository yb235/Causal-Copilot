# 3. The End‑to‑End Workflow

This document walks through **one complete run** of Causal‑Copilot, from the
moment a user types a command to the moment a PDF report appears. The CLI
flow is documented step‑by‑step; the web demo follows the same logic but
exposes most steps through a Gradio chat interface instead of a console.

The driver is [`main.py`](../main.py). Read alongside that file for line‑level
detail.

---

## 3.1 The 13 stages of a run

```
[1]  parse_args
[2]  global_state_initialization
[3]  load_data
[4]  process_user_query
[5]  stat_info_collection            ◄─ Pre‑processing
[6]  knowledge_info (LLM)
[7]  EDA generation
[8]  Filter (LLM)                    ◄─ Algorithm selection
[9]  Reranker (LLM)
[10] HyperparameterSelector (LLM)
[11] Programming.forward             ◄─ Algorithm execution
[12] Judge.forward                   ◄─ Post‑processing
[13] Report_generation.generation    ◄─ Report
[14] Discussion.forward (web only)   ◄─ User Q&A
```

The number "13" is a guideline; the exact list of method calls evolves with
the codebase. Below, each stage is explained in plain English with pointers
to the implementing files.

---

## 3.2 Stage 1–4 · Boot‑up & ingestion

### Stage 1 — `parse_args`
`main.py` defines the CLI flags. The most important ones:

| Flag | Default | Meaning |
|------|---------|---------|
| `--data-file` | `data/dataset/Abalone/Abalone.csv` | Input dataset (CSV/JSON). |
| `--output-report-dir` | `output/Abalone` | Where the PDF lands. |
| `--output-graph-dir` | `output/Abalone` | Where graph PNG/PDFs land. |
| `--initial_query` | `"Do causal discovery on this dataset"` | Free‑text user question. |
| `--data_mode` | `real` | `real` = use the file you gave; `simulated` = generate synthetic data. |
| `--simulation_mode` | `offline` | `offline` = pre‑built; `online` = generate now. |
| `--debug` | `False` | Skip slow stat tests with mocked values. |
| `--parallel` | `False` | Run bootstrap iterations in parallel. |

Full list lives at the top of `main.py` and is mirrored in
[`08-configuration.md`](08-configuration.md).

### Stage 2 — `global_state_initialization`
[`global_setting/Initialize_state.py`](../global_setting/Initialize_state.py)
constructs an empty `GlobalState` dataclass (defined in
[`global_setting/state.py`](../global_setting/state.py)) and stamps it with
defaults: bootstrap iterations = 20, significance α = 0.1, correlation
threshold = 0.99, etc. See
[`04-data-flow-and-schema.md`](04-data-flow-and-schema.md) for every field.

### Stage 3 — `load_data`
Reads the CSV/JSON into a pandas `DataFrame`, attaches it to
`global_state.user_data.raw_data`. If `--data_mode simulated`, it instead
calls into [`data/simulator/`](../data/simulator) to generate synthetic data
plus a known ground‑truth graph (used for evaluation).

### Stage 4 — `process_user_query`
Parses the `--initial_query` string. The query can be free text *or* a
semi‑structured form (e.g. `"selected variables: A, B, C; algorithm: PC"`).
This stage extracts any explicit user preferences (specific variables,
explicit algorithm names, etc.) and stores them on the `GlobalState`.

---

## 3.3 Stage 5–7 · Pre‑processing & EDA

### Stage 5 — Statistical profiling
[`preprocess/stat_info_functions.py`](../preprocess/stat_info_functions.py)
runs a battery of tests on the data and writes the results onto
`global_state.statistics`:

* **Missingness** — `np_nan_detect`, `numeric_str_nan_detect`, missing‑ratio
  table.
* **Imputation** — `SimpleImputer` (mean/median) for low missingness;
  `IterativeImputer` for higher missingness.
* **Data type** — continuous, categorical, mixed (`data_type_detect`).
* **Linearity** — RESET‑style test (`linearity_test`).
* **Gaussian errors** — Jarque‑Bera test (`gaussian_error_test`).
* **Stationarity** — Augmented Dickey‑Fuller (`stationarity_test`).
* **Time‑series shape** — `time_series_detect` infers whether the data is
  temporal and estimates the lag.

The result is a human‑readable description string
(`statistics.description`) which is later embedded into LLM prompts.

### Stage 6 — Domain knowledge extraction
[`preprocess/dataset.py::knowledge_info`](../preprocess/dataset.py) prompts
the LLM with the column names + user description and asks it to surface
relevant background knowledge ("Age cannot be caused by Income", "Variable
X is binary treatment"). The text is stored in
`global_state.user_data.knowledge_docs`.

### Stage 7 — EDA generation
[`preprocess/eda_generation.py`](../preprocess/eda_generation.py) creates the
exploratory plots that will be embedded in the report:

* Correlation matrix heatmap (Seaborn).
* Per‑variable distributions (histograms / KDE).
* Time‑series ACF/PACF and trend decomposition (when relevant).
* Missing‑data patterns.
* Pairwise scatter or violin plots for important relationships.

PNGs are saved under `--output-graph-dir`.

---

## 3.4 Stage 8–10 · Algorithm selection (LLM‑driven)

This three‑step funnel converts "data + query" into "specific algorithm with
specific hyperparameters".

### Stage 8 — Filter
[`causal_discovery/filter.py`](../causal_discovery/filter.py) loads the
algorithm catalog from
[`causal_discovery/context/algos/`](../causal_discovery/context) and prompts
the LLM:

> "Given these data characteristics (linear / non‑linear, time‑series y/n,
> sample size, missingness…) which 1–2 algorithms from this catalog are
> appropriate?"

Output: `algorithm.algorithm_candidates` (a small dict).

### Stage 9 — Rerank
[`causal_discovery/rerank.py`](../causal_discovery/rerank.py) feeds the
short‑list, plus benchmark performance numbers, back to the LLM:

> "Among these candidates, which is most likely to perform best on this
> dataset?"

Output: `algorithm.selected_algorithm` (a single name, e.g. `"PC"`).

### Stage 10 — Hyperparameter selection
[`causal_discovery/hyperparameter_selector.py`](../causal_discovery/hyperparameter_selector.py)
asks the LLM, for the chosen algorithm, to fill in each tunable knob (α
threshold, kernel choice, max degree, …). The LLM is given the parameter
documentation as context.

Output: `algorithm.algorithm_arguments` (a dict of kwargs).

---

## 3.5 Stage 11 · Algorithm execution

[`causal_discovery/program.py`](../causal_discovery/program.py)'s
`Programming` class:

1. Optionally drops near‑duplicate columns (correlation > 0.99) so brittle
   algorithms don't choke. It re‑inserts them afterwards as redundant copies
   in the final graph.
2. Looks up the wrapper class:
   `getattr(causal_discovery.wrappers, selected_algorithm)`.
3. Constructs it with `algorithm_arguments`, calls `.fit(processed_data)`.
4. Converts the algorithm's native output (whatever it is — adjacency
   matrix, CPDAG, PAG, …) into a uniform numpy adjacency matrix.

Outputs land in `global_state.results.converted_graph` and
`global_state.results.raw_result`.

The graph is then visualized with
[`postprocess/visualization.py`](../postprocess/visualization.py) (force‑
directed layout via Graphviz / NetworkX) and saved as `initial_graph.pdf`.

---

## 3.6 Stage 12 · Post‑processing

[`postprocess/judge.py`](../postprocess/judge.py)'s `Judge` class refines the
raw graph:

1. **Bootstrap.** Resample the data 20 times, re‑run the algorithm on each
   sample, count how often each edge appears. The result is a probability
   matrix in `results.bootstrap_probability`.
2. **Bootstrap recommendations.** Identify edges that are *frequently* in
   bootstrap samples but *missing* from the initial graph (and vice versa) —
   these are the suspect edges.
3. **LLM evaluation.** Feed the suspect edges + domain knowledge to the LLM:
   "Should we add/remove/flip this edge?"
4. **KCI pruning** (optional). Run a Kernel Conditional Independence test
   for additional evidence.
5. **Cycle breaking.** Detect any cycles in the LLM‑pruned graph and orient
   the offending edges to keep the result a valid DAG.

Output: `results.revised_graph`, `results.revised_edges`.

A bootstrap heatmap (`boot_heatmap.pdf`) and the revised graph PDF are
written next to the initial graph.

---

## 3.7 Stage 13 · Report generation

[`report/report_generation.py`](../report/report_generation.py) drives the
LaTeX build:

1. Gathers everything: dataset description, EDA images, chosen algorithm,
   hyperparameters, initial graph, bootstrap heatmap, revised graph,
   metrics.
2. Calls the LLM with `graph_effect_prompts()` to produce a paragraph for
   each directed edge ("X likely causes Y because…").
3. Fills a LaTeX template (the project uses an arXiv‑style preprint
   template) and writes it to disk.
4. Calls `latexmk` (via `plumbum`) to render the PDF, running multiple
   passes for cross‑references and BibTeX.

Output: `report.pdf` in `--output-report-dir`. If causal *inference* was
run, an additional report is produced by
[`report/inference_report_generation.py`](../report/inference_report_generation.py).

---

## 3.8 Stage 14 (web only) · Interactive discussion

[`user/discuss.py`](../user/discuss.py) keeps a chat alive after the report
is built. The LLM is grounded in the just‑produced LaTeX content, so it can
answer questions like "Why didn't you keep the X→Y edge?" by referring to
the bootstrap probability and pruning rationale.

---

## 3.9 Causal inference branch

If the user query (or the web demo controls) requests causal *inference*
rather than just discovery, the pipeline forks after stage 12:

1. The discovered graph is handed to
   [`causal_inference/`](../causal_inference).
2. The user (or the LLM, by inferring from the graph) picks treatment,
   outcome, and confounders.
3. An estimator is selected from the DML / DRL / IV / MetaLearners /
   Uplift suites.
4. DoWhy or EconML produces an effect estimate, often heterogeneous (CATE).
5. Sensitivity analyses (refutation tests) are run.
6. An inference‑specific PDF is produced.

See [`06-algorithms.md`](06-algorithms.md) for the inventory of inference
estimators.

---

## 3.10 Visual cheat‑sheet

```
CSV   ─►  load_data ─►  stat_info ─►  EDA ─►  filter ─►  rerank ─►  hp_select
                                                                       │
                                                                       ▼
                                  bootstrap+LLM judge  ◄──  run algorithm
                                          │
                                          ▼
                                   LaTeX → PDF report ─►  Q&A (web)
```

The next document, [`04-data-flow-and-schema.md`](04-data-flow-and-schema.md),
zooms in on the *data* that travels along these arrows.
