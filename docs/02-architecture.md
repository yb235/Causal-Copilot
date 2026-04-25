# 2. Architecture

Causal‑Copilot is built as **five cooperating modules with an LLM at the
center**. This section describes each module's responsibility, the
interfaces between them, and the design principles that hold the system
together.

---

## 2.1 The five‑module view

```
                    ┌────────────────────────────┐
                    │            LLM             │
                    │   (OpenAI / OpenRouter /   │
                    │           Ollama)          │
                    └──┬───────┬──────┬───────┬──┘
                       │       │      │       │
        ┌──────────────┘       │      │       └────────────┐
        ▼                      ▼      ▼                    ▼
┌───────────────┐    ┌───────────────────┐    ┌──────────────────┐
│  USER         │    │  PRE‑PROCESSING   │    │  POST‑PROCESSING │
│  INTERACTION  │    │  • cleaning       │    │  • bootstrap     │
│  • CLI        │    │  • stat profiling │    │  • LLM pruning   │
│  • Gradio Web │    │  • EDA            │    │  • cycle break   │
│  • Q&A        │    └─────────┬─────────┘    └────────┬─────────┘
└──────┬────────┘              │                       │
       │                       ▼                       │
       │             ┌───────────────────┐             │
       │             │  ALGORITHM        │             │
       │             │  SELECTION        │             │
       │             │  filter→rerank→   │             │
       │             │  hyperparams      │             │
       │             └─────────┬─────────┘             │
       │                       ▼                       │
       │             ┌───────────────────┐             │
       │             │  EXECUTION        │             │
       │             │  (wrappers/*.py)  │─────────────┘
       │             └─────────┬─────────┘
       │                       ▼
       │             ┌───────────────────┐
       └────────────►│  REPORT           │──► PDF
                     │  (LaTeX → PDF)    │
                     └───────────────────┘
```

The same picture, in words:

| Layer | Folder(s) | Job |
|-------|-----------|-----|
| **Simulation / Data** | `data/`, `preprocess/` | Load real or synthetic data, clean, profile, visualize. |
| **User Interaction** | `main.py`, `web_demo/`, `user/` | Accept a query, drive the loop, and chat about the result. |
| **Pre‑processing** | `preprocess/` | Detect data type, missingness, linearity, stationarity, time‑series structure. |
| **Algorithm Selection** | `causal_discovery/filter.py`, `rerank.py`, `hyperparameter_selector.py` | LLM decides *which* algorithm and *which* hyperparameters. |
| **Algorithm Execution** | `causal_discovery/wrappers/`, `causal_discovery/program.py`, `causal_inference/` | Run the chosen algorithm; produce a graph or effect estimate. |
| **Post‑processing** | `postprocess/` | Bootstrap, LLM pruning, cycle breaking, plotting. |
| **Reporting** | `report/` | Compile a LaTeX PDF with embedded figures and narratives. |
| **State / Plumbing** | `global_setting/`, `llm/`, `utils/`, `externals/` | Carry state across modules, talk to LLM, log, host third‑party libs. |

---

## 2.2 The LLM as orchestrator

The LLM is **not** the brain that runs algorithms — classical libraries do
that. The LLM makes **judgment calls** that a human expert would otherwise
make:

| Decision point | Module | What the LLM is asked |
|----------------|--------|-----------------------|
| Domain understanding | `preprocess/dataset.py` | "Given this column list and user description, what variables matter?" |
| Algorithm filtering | `causal_discovery/filter.py` | "Given these data characteristics, which 1–2 algorithms are appropriate?" |
| Algorithm reranking | `causal_discovery/rerank.py` | "Given benchmark results, pick the single best one." |
| Hyperparameter tuning | `causal_discovery/hyperparameter_selector.py` | "Choose values for these knobs." |
| Result auditing | `postprocess/judge.py` | "Look at this edge — does it make sense given domain knowledge?" |
| Report writing | `report/report_generation.py` | "Write a paragraph explaining why X likely causes Y." |
| User Q&A | `user/discuss.py` | Free‑form chat grounded in the produced report. |

The LLM provider abstraction lives in [`llm/llm_client.py`](../llm/llm_client.py)
and supports OpenAI, OpenRouter, and Ollama. See
[`07-llm-integration.md`](07-llm-integration.md) for details.

---

## 2.3 Design principles

1. **Separation of concerns.** Each folder corresponds to one stage of the
   pipeline. Stages communicate only through the `GlobalState` object
   (see [`04-data-flow-and-schema.md`](04-data-flow-and-schema.md)).

2. **Wrappers, not forks.** External algorithms are pulled in as git
   submodules under `externals/`. Each algorithm gets a thin Python wrapper
   in `causal_discovery/wrappers/<algo>.py` that adapts it to the unified
   `.fit(data)` interface. Adding a new algorithm = adding a new wrapper +
   updating the algorithm context files.

3. **Two‑step algorithm selection.** A *filter* narrows from N candidates
   to a small shortlist; a *reranker* picks the winner. This mimics the way
   experts decide.

4. **Statistical + LLM post‑processing.** Bootstrap gives statistical
   uncertainty on each edge. The LLM gives a "domain sanity" pass. Combining
   the two is more robust than either alone.

5. **One central state object.** Instead of passing many arguments between
   functions, the `GlobalState` dataclass is mutated through the pipeline.
   This keeps function signatures short and makes it easy to add new fields.

6. **Pluggable LLM.** Three providers are supported and selected via the
   `LLM_PROVIDER` environment variable. No code change needed to swap.

7. **PDF as a first‑class output.** The system targets non‑experts, so a
   self‑contained LaTeX report is treated as the "real" deliverable, not an
   afterthought.

---

## 2.4 What lives at the very top of the repo

| File | Role |
|------|------|
| `main.py` | CLI entry point. Wires preprocess → discovery → postprocess → report. |
| `Dockerfile.cpu` / `Dockerfile.gpu` | Container recipes (recommended install path). |
| `setup_cpu.sh` / `setup_gpu.sh` | Conda‑environment setup scripts. |
| `requirements_cpu.txt` / `requirements_gpu.txt` | Python dependency pins. |
| `requirements_latex.txt` | LaTeX packages required for report PDFs. |
| `install_latex.py` | Helper that installs TinyTeX and the LaTeX package list. |
| `test_latex_functionality.py` | Smoke test that LaTeX → PDF works. |
| `.env.example` | Template for environment variables (LLM provider, API keys). |
| `.gitmodules` | Declares the submodules under `externals/`. |
| `README.md` | User‑facing project description. |

---

## 2.5 Where to go next

* Trace a single run from start to finish → [`03-workflow.md`](03-workflow.md)
* See the data structures that flow between modules →
  [`04-data-flow-and-schema.md`](04-data-flow-and-schema.md)
* Explore each folder in detail → [`05-modules.md`](05-modules.md)
