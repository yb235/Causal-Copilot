# 5. Module‑by‑Module Tour

This is a guided walk through every top‑level folder in the repository.
For each one we explain **what it owns, what its key files do, and how it
interacts with the rest of the system**.

> The tour is in roughly the order the pipeline uses each module. For the
> full execution flow see [`03-workflow.md`](03-workflow.md).

---

## 5.1 `main.py` (root)

Single‑file CLI entry point. It:

1. Parses CLI arguments.
2. Builds a `GlobalState`.
3. Calls into `preprocess`, `causal_discovery`, `postprocess`, and
   `report` in sequence.
4. Returns when the PDF is written.

When in doubt about the exact order of operations, **`main.py` is the
authoritative spec** for the CLI flow.

---

## 5.2 `web_demo/`

Houses the interactive Gradio app. Key files:

* `demo.py` — assembles the Gradio Blocks UI (chatbot pane, file upload,
  parameter controls). Imports the same `preprocess`, `causal_discovery`,
  `postprocess`, `report`, `causal_inference`, and `user` modules used by
  `main.py`, so the underlying logic is shared.
* `demo_config.py` — defaults + UI strings.
* Static assets (CSS / images) for the chat UI.

The web demo is the *recommended* entry point for end users because it
exposes the causal **inference** branch (HTE estimation, refutation
analyses) and the post‑report Q&A loop, neither of which are exposed by
`main.py`.

Run it with `python web_demo/demo.py`; by default it serves on
`http://localhost:7860`.

---

## 5.3 `global_setting/`

The "spine" of the project. Two files:

* `state.py` — defines the dataclasses (`UserData`, `Statistics`,
  `Algorithm`, `Results`, `Inference`, `Logging`, `GlobalState`). See
  [`04-data-flow-and-schema.md`](04-data-flow-and-schema.md).
* `Initialize_state.py` — builds the initial `GlobalState`, loads data
  (`load_data`, `load_real_world_data`, `load_local_data`), and seeds
  defaults from CLI args.

Treat this folder as the canonical schema. Every other module reads or
writes fields here.

---

## 5.4 `preprocess/`

Three big files do most of the work:

* `dataset.py` — orchestrates loading, calls
  `knowledge_info(args, global_state)` which prompts the LLM for
  domain‑knowledge documents.
* `stat_info_functions.py` (~1.2 KLOC) — the statistical battery. Major
  functions:
  - `np_nan_detect`, `numeric_str_nan_detect`
  - `missing_ratio_table`
  - `data_type_detect`
  - `linearity_test`, `gaussian_error_test`, `stationarity_test`
  - `time_series_detect`
  - `convert_stat_info_to_text` — produces the natural‑language summary
    that gets injected into LLM prompts.
* `eda_generation.py` (~1.6 KLOC) — the `EDA` class with methods such as
  `generate_eda`, `_select_visualization_features`, `plot_correlation_matrix`,
  `plot_distributions`, `plot_time_series`, `plot_missing_data`, and
  `plot_feature_relationships`.

Output: a populated `Statistics` block on `GlobalState` and a directory of
PNG plots.

---

## 5.5 `causal_discovery/`

By far the largest module. Contains:

* `filter.py` — `Filter` class. Loads `context/algos/*` plus
  `context/algo_select_prompt.txt`, asks the LLM for 1–2 candidate
  algorithms, writes them to `algorithm.algorithm_candidates`.
* `rerank.py` — `Reranker` class. Loads
  `context/algo_rerank_prompt.txt`, plus benchmark performance numbers,
  asks the LLM to pick the single best candidate.
* `hyperparameter_selector.py` — `HyperparameterSelector` class. Reads
  `context/hyperparameter_select_prompt.txt` and the algo's parameter
  documentation, asks the LLM to choose values, writes
  `algorithm.algorithm_arguments`.
* `program.py` — `Programming` class. Looks up the wrapper, runs `.fit`,
  converts output to a numpy adjacency matrix.
* `wrappers/` — one file per algorithm; each defines a class with a
  uniform `__init__(**kwargs)` + `.fit(data) → result` interface. See
  [`06-algorithms.md`](06-algorithms.md) for the full list.
* `evaluation/` — utilities for measuring graph quality (precision,
  recall, SHD, …) when ground truth exists.
* `context/` — text files providing prompts and per‑algorithm metadata
  (when to use, what hyperparameters mean, performance characteristics).

This is also where you'd plug in a new algorithm: drop a file under
`wrappers/`, register it in `context/algos/`, and the LLM will start
considering it automatically.

---

## 5.6 `causal_inference/`

The effect‑estimation half of the system. Sub‑folders by family:

* `DML/` — Double Machine Learning (LinearDML, CausalForestDML).
* `DRL/` — Doubly Robust Learners.
* `MetaLearners/` — S, T, R, X learners.
* `IV/` — Instrumental Variable estimators (e.g. 2SLS).
* `Uplift/` — Uplift modelling and Generalized Random Forests.

Plus top‑level helpers:

* `inference.py` — orchestrates: pick treatment / outcome / confounders,
  pick estimator, run, run refutations.
* `help_functions.py` — shared utilities (graph → DAG conversion,
  refutation wiring, etc.).

The graph produced by `causal_discovery/` is used as the structural input;
DoWhy and EconML do the numerical heavy lifting.

---

## 5.7 `postprocess/`

Refines and visualises the discovered graph.

* `judge.py` — the `Judge` class. Methods include `quality_judge`,
  `bootstrap`, `bootstrap_recommend`, `llm_evaluation_new`, `kci_pruning`,
  `check_cycle`. This is where the bootstrap+LLM "second opinion" lives.
* `judge_functions.py` — supporting functions for `judge.py`.
* `visualization.py` — `Visualization` class with `plot_pdag`, `get_pos`
  (force‑directed layout), `boot_heatmap_plot`, `plot_time_series_graph`.
* `draw.py` — extra publication‑quality drawing helpers.
* `pag.py` — Partial Ancestral Graph utilities for FCI‑style outputs.

---

## 5.8 `report/`

Turns the final `GlobalState` into a PDF.

* `report_generation.py` — `Report_generation` class. Calls
  `graph_effect_prompts()` to ask the LLM for a paragraph per directed
  edge, fills the LaTeX template, and runs `latexmk`.
* `inference_report_generation.py` — equivalent for the causal‑inference
  branch (effect tables, refutation summaries, sensitivity plots).
* `help_functions.py` — LaTeX escaping, figure embedding, BibTeX, PDF
  compilation helpers.
* `context/` — LaTeX templates and report prompts.

The PDF compilation goes through `plumbum.cmd.latexmk`, so a working
TeX Live or TinyTeX install is required (the Dockerfiles set this up
automatically).

---

## 5.9 `llm/`

Provider‑agnostic LLM client.

* `llm_client.py` — `LLMClient` class. Selects between OpenAI,
  OpenRouter, and Ollama based on the `LLM_PROVIDER` environment
  variable. Knows how to:
  - Send chat completions with system+user messages.
  - Request JSON‑mode output when given a Pydantic model.
  - Strip code‑block fences and recover from minor JSON errors.
* `ollama_client.py` — Ollama‑specific HTTP integration.
* `test_ollama.py` — smoke test for local Ollama.

See [`07-llm-integration.md`](07-llm-integration.md) for prompt locations
and provider details.

---

## 5.10 `user/`

Currently a small folder with `discuss.py`. It implements the
post‑report interactive Q&A: load the final report content, expose it as
context to the LLM, and stream chat responses to the web UI.

---

## 5.11 `utils/`

Cross‑cutting helpers:

* `logger.py` — `rich`‑coloured structured logging.
* `suppress_logs.py` — silence the noisier dependencies (sklearn, dowhy)
  during pipeline runs.

---

## 5.12 `data/`

Two sub‑folders:

* `simulator/` — synthetic data generator inspired by the NOTEARS paper.
  Creates a random DAG, samples linear / non‑linear / Gaussian /
  non‑Gaussian noise, and returns `(data, ground_truth)`. Used when
  `--data_mode simulated`.
* `dataset/` — sample real‑world datasets (Abalone, Sachs, CCS, …) for
  demos and tests.

---

## 5.13 `externals/`

Git submodules with the actual algorithm implementations. Wrappers in
`causal_discovery/wrappers/` adapt these to the unified interface.

| Submodule | What it provides |
|-----------|------------------|
| `causal-learn/` | PC, FCI, GES, LiNGAM family. |
| `notears/` | Linear and non‑linear NOTEARS. |
| `pc_adjacency_search/` | GPU‑accelerated PC skeleton search. |
| `pyCausalFS/` | Markov‑Blanket feature selection (HITON‑MB, IAMB family, MBOR, BAMB). |
| `acceleration/` | CUDA kernels for PC / GES. |

After cloning the repo, run `git submodule update --init --recursive`
to populate them. The Dockerfiles handle this for you.

---

## 5.14 `asset/`

Static images: project logo, screenshots, demo GIFs, sample report PDFs.
Nothing functional — only used by `README.md` and the web UI.

---

## 5.15 Top‑level scripts and config

| File | Purpose |
|------|---------|
| `Dockerfile.cpu` | CPU‑only container based on `pytorch/pytorch:*-cuda11.8`. Installs Graphviz, TeX Live / TinyTeX, and `requirements_cpu.txt`. |
| `Dockerfile.gpu` | Adds CUDA 12.1, cuDF / cuML / cuGraph, and compiles `externals/pc_adjacency_search`. |
| `setup_cpu.sh` | Bash installer for a Conda environment (Linux/macOS). |
| `setup_gpu.sh` | Same plus CUDA toolkit detection and GPU packages. |
| `install_latex.py` | Installs TinyTeX and the LaTeX packages listed in `requirements_latex.txt`. |
| `test_latex_functionality.py` | Compiles a tiny `.tex` file end‑to‑end to verify the toolchain. |
| `requirements_cpu.txt` / `requirements_gpu.txt` | Python deps. |
| `requirements_latex.txt` | LaTeX package names. |
| `.env.example` | Template environment file. |
| `.gitmodules` | Submodule URLs/paths for `externals/`. |

---

## 5.16 Where things *aren't*

A common newcomer question: where is "X"? Quick answers:

* **The web UI logic** → `web_demo/demo.py`, not `user/`.
* **CI / test suite** → there is no project‑wide test suite at the time of
  writing; the closest things are `test_latex_functionality.py` and
  `llm/test_ollama.py`.
* **Authentication / user accounts** → not part of the repo. The Gradio
  app runs unauthenticated locally.
* **A package on PyPI** → there is none; install from source.

---

For the catalog of algorithms each module provides, continue to
[`06-algorithms.md`](06-algorithms.md).
