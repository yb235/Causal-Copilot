# 4. Data Flow & Schema

Every stage of the pipeline reads from and writes to **one shared object**:
the `GlobalState` dataclass defined in
[`global_setting/state.py`](../global_setting/state.py). This document is the
field‑by‑field reference for that object — your map of "what data exists, at
what stage, and where to find it".

---

## 4.1 The top‑level `GlobalState`

```python
@dataclass
class GlobalState:
    user_data:  UserData
    statistics: Statistics
    logging:    Logging
    algorithm:  Algorithm
    inference:  Inference
    results:    Results
```

A single `GlobalState` instance is created at startup
(see [`global_setting/Initialize_state.py`](../global_setting/Initialize_state.py))
and passed around — usually as `global_state` — until the run finishes. Each
stage **mutates** the parts of it that it owns.

The initialization priority documented in the source is:

> 1. All values start as `None`.
> 2. The user query has the highest priority.
> 3. Then values inferred from the data.
> 4. Then default values.
>
> If a value is already set, later operations skip it.

---

## 4.2 `UserData` — the dataset and user inputs

| Field | Type | Set by | Meaning |
|-------|------|--------|---------|
| `raw_data` | `pd.DataFrame` | `load_data` | The dataset as read from disk. |
| `processed_data` | `pd.DataFrame` | `preprocess` | After missing‑value imputation, type fixes, etc. |
| `ground_truth` | `np.ndarray` | simulator | Adjacency matrix when running on synthetic data. |
| `initial_query` | `str` | `--initial_query` | The free‑text user prompt. |
| `initial_query_type` | `str` | `process_user_query` | Discovery vs. inference vs. mixed. |
| `knowledge_docs` | `str` | `preprocess/dataset.py` (LLM) | Domain knowledge for downstream prompts. |
| `knowledge_docs_for_user` | `str` | same | Human‑readable variant for the report. |
| `output_report_dir` | `str` | CLI flag | Where the PDF goes. |
| `output_graph_dir` | `str` | CLI flag | Where graph images go. |
| `selected_features` | `list` | preprocessing / user | Variables actually used in analysis. |
| `important_features` | `list` | LLM | Variables flagged as key by the LLM. |
| `high_corr_feature_groups` | `list[list]` | preprocessing | Clusters of near‑duplicate columns. |
| `visual_selected_features` | `list` | EDA | Subset chosen for plots (≈ ≤30 for readability). |
| `user_drop_features` | `list` | user | Variables explicitly excluded by the user. |
| `llm_drop_features` | `list` | LLM | Variables the LLM suggested dropping. |
| `high_corr_drop_features` | `list` | preprocessing | Auto‑dropped duplicates (corr > 0.99). |
| `nan_indicator` | `str` | preprocessing | Token used in the data to mark missing values. |
| `drop_important_var` | `bool` | preprocessing | True if a flagged‑important var was dropped. |
| `meaningful_feature` | `bool` | preprocessing | Whether feature names look semantic. |
| `heterogeneity` | `str` | preprocessing | Notes on group/cluster heterogeneity. |
| `accept_CPDAG` | `bool` | user | Whether partially‑directed graphs are OK. |

---

## 4.3 `Statistics` — the data profile

| Field | Type | Default | Meaning |
|-------|------|---------|---------|
| `miss_ratio` | `list[dict]` | `[]` | Per‑column missing‑value ratios. |
| `sparsity_dict` | `dict` | `None` | Column‑level sparsity counts. |
| `linearity` | `bool` | `None` | Result of RESET test. |
| `gaussian_error` | `bool` | `None` | Result of Jarque‑Bera. |
| `missingness` | `bool` | `None` | Whether *any* missing values exist. |
| `sample_size` | `int` | `None` | Number of rows. |
| `feature_number` | `int` | `None` | Number of columns. |
| `boot_num` | `int` | **`20`** | Bootstrap iterations. |
| `alpha` | `float` | **`0.1`** | Significance threshold (CI tests). |
| `num_test` | `int` | **`100`** | Number of statistical tests. |
| `ratio` | `float` | **`0.5`** | Sub‑sample ratio for bootstrap. |
| `data_type` | `str` | `None` | `"continuous"` / `"categorical"` / `"mixed"`. |
| `data_type_column` | `str` | `None` | Per‑column types. |
| `heterogeneous` | `bool` | `None` | Whether the data has subgroup heterogeneity. |
| `domain_index` | `str` | `None` | Column to treat as a domain/group index (CDNOD). |
| `description` | `str` | `None` | Natural‑language summary used in LLM prompts. |
| `time_series` | `bool` | `False` | Is this a time‑series dataset? |
| `time_lag` | `int` | `None` | Auto‑detected lag. |
| `time_index` | `str` | `None` | Name of the time/date column. |

The most important field downstream is **`description`** — it is what the
LLM "sees" about your data when it picks an algorithm.

---

## 4.4 `Algorithm` — what was selected & how

| Field | Default | Meaning |
|-------|---------|---------|
| `handle_correlated_features` | `True` | Drop & later re‑attach near‑duplicate columns. |
| `correlation_threshold` | `0.99` | Threshold for "near‑duplicate". |
| `selected_algorithm` | `None` | Final algorithm name (`"PC"`, `"NOTEARSLinear"`, …). |
| `selected_reason` | `None` | LLM‑written rationale for the choice. |
| `algorithm_candidates` | `None` | Dict of finalists from the *filter* stage. |
| `algorithm_optimum` | `None` | Cached "best so far" during reranking. |
| `algorithm_arguments` | `None` | Hyperparameters chosen by the selector. |
| `algorithm_arguments_json` | `None` | The raw JSON the LLM returned. |
| `waiting_minutes` | `1440.0` | Hard timeout (24 h). |
| `gpu_available` | `False` | Auto‑detected; gates GPU‑only algos. |

---

## 4.5 `Results` — graphs and metrics

| Field | Meaning |
|-------|---------|
| `raw_result` | The algorithm's native output (e.g. a `causal-learn` `GeneralGraph`). |
| `raw_pos` | Node positions used for plotting the initial graph. |
| `raw_edges` | Dict view of edges in the initial graph. |
| `raw_info` | Free‑form metadata (timing, errors, …). |
| `converted_graph` | Initial graph as a uniform numpy adjacency matrix. |
| `lagged_graph` | Time‑lagged adjacency tensor (time‑series only). |
| `metrics` | Evaluation metrics vs. `ground_truth` (when available). |
| `revised_graph` | Final graph after bootstrap + LLM pruning + cycle break. |
| `revised_edges` | Dict view of `revised_graph`. |
| `revised_metrics` | Metrics on the revised graph. |
| `bootstrap_probability` | `n×n` matrix of edge appearance frequencies. |
| `bootstrap_check_dict` | Diagnostic flags per edge (suspicious, etc.). |
| `llm_errors` | Edges where the LLM and the data disagreed. |
| `bootstrap_errors` | List of failed bootstrap iterations. |
| `eda_result` | Paths and metadata of EDA plots. |
| `prior_knowledge` | User‑supplied or LLM‑derived edge constraints. |
| `refutation_analysis` | Sensitivity analysis output (causal inference). |
| `report_selected_index` | Indices of edges/figures included in the PDF. |

---

## 4.6 `Inference` — heterogeneous treatment effects

| Field | Meaning |
|-------|---------|
| `hte_algo_json` | LLM‑chosen HTE estimator + reasoning. |
| `hte_model_y_json` | Outcome model spec. |
| `hte_model_T_json` | Treatment model spec. |
| `hte_model_param` | Hyperparameters for the chosen estimator. |
| `cycle_detection_result` | Cycles found in the discovered graph. |
| `editing_history` | Steps taken to break those cycles. |
| `inference_result` | Numerical effect estimates (ATE, CATE, …). |
| `task_index`, `task_info` | Multi‑task bookkeeping. |

---

## 4.7 `Logging` — every conversation, archived

The `Logging` dataclass keeps **separate transcripts** of every LLM call so a
run is fully auditable:

* `query_conversation` — user query parsing.
* `knowledge_conversation` — domain‑knowledge extraction.
* `filter_conversation` — algorithm filtering.
* `select_conversation` — reranking.
* `argument_conversation` — hyperparameter selection.
* `errors_conversion` — adjacency conversion errors.
* `graph_conversion` — graph format conversions.
* `downstream_discuss`, `final_discuss` — user Q&A.
* `global_state_logging` — periodic dumps of the entire state.

These end up in the report appendix and are useful for debugging.

---

## 4.8 The "data" that flows on disk

In addition to the in‑memory `GlobalState`, runs produce on‑disk artefacts:

```
output/<dataset>/
├── eda/
│   ├── correlation.png
│   ├── distributions.png
│   └── …
├── initial_graph.pdf
├── boot_heatmap.pdf
├── revised_graph.pdf
├── report.tex
├── report.pdf
└── logs/
    └── conversation.json
```

The exact filenames depend on the run; the `report/` module is the
authoritative producer of these outputs.

---

## 4.9 Mutation timeline

Reading the table below, you can predict *exactly* which fields are populated
after each stage:

| Stage | Fields written |
|-------|----------------|
| 2. Initialize | All defaults, all `None`. |
| 3. Load data | `user_data.raw_data`, optionally `user_data.ground_truth`. |
| 4. Parse query | `user_data.initial_query*`, possibly `user_data.selected_features`. |
| 5. Stat profile | All of `Statistics`, plus `user_data.processed_data`. |
| 6. Knowledge | `user_data.knowledge_docs*`. |
| 7. EDA | `results.eda_result`, plus PNG files on disk. |
| 8. Filter | `algorithm.algorithm_candidates`, `logging.filter_conversation`. |
| 9. Rerank | `algorithm.selected_algorithm`, `selected_reason`, `logging.select_conversation`. |
| 10. HP select | `algorithm.algorithm_arguments*`, `logging.argument_conversation`. |
| 11. Run algo | `results.raw_*`, `results.converted_graph`. |
| 12. Postprocess | `results.bootstrap_probability`, `results.revised_*`. |
| 13. Report | files on disk + `results.report_selected_index`. |
| 14. Discuss | `logging.downstream_discuss` / `final_discuss`. |

For an inference run, stages 11+ also write to `inference.*`.

---

## 4.10 Extending the schema

If you add a new pipeline feature, the convention is:

1. Add a typed field to the right dataclass in `state.py`.
2. Default it to `None` (or an empty container).
3. Mutate it in the stage that owns it.
4. Read it (never copy it) in downstream stages.

This keeps the contract between modules in **one** file.
