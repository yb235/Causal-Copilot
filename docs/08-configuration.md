# 8. Configuration Reference

This is the complete reference for every knob you can turn — environment
variables, CLI flags, and important defaults baked into the code.

---

## 8.1 Environment variables (`.env`)

Copy `.env.example` to `.env` and edit it:

```bash
cp .env.example .env
```

| Variable | Required? | Default | Description |
|----------|-----------|---------|-------------|
| `LLM_PROVIDER` | ✅ | `ollama` | One of `openai`, `openrouter`, `ollama`. |
| `LLM_MODEL` | ✅ | `llama3.2` | Provider‑specific model name. |
| `OPENAI_API_KEY` | ⚠️ if `LLM_PROVIDER=openai` | — | Your OpenAI key. |
| `OPENROUTER_API_KEY` | ⚠️ if `LLM_PROVIDER=openrouter` | — | Your OpenRouter key. |
| `OLLAMA_BASE_URL` | ⚠️ if `LLM_PROVIDER=ollama` | `http://localhost:11434` | Ollama server URL. Use `http://host.docker.internal:11434` from inside Docker. |
| `OUTPUT_REPORT_DIR` | ❌ | `output_report` | Where the web demo writes report PDFs. |
| `OUTPUT_GRAPH_DIR` | ❌ | `output_graph` | Where the web demo writes graph images. |

**Security**: never commit `.env`. The repo's `.gitignore` already excludes
it.

### Provider‑specific example blocks

OpenAI:
```bash
LLM_PROVIDER=openai
LLM_MODEL=gpt-4o
OPENAI_API_KEY=sk-...
```

OpenRouter:
```bash
LLM_PROVIDER=openrouter
LLM_MODEL=anthropic/claude-sonnet-4
OPENROUTER_API_KEY=sk-or-...
```

Ollama (local):
```bash
LLM_PROVIDER=ollama
LLM_MODEL=llama3.2
OLLAMA_BASE_URL=http://localhost:11434
# Make sure: ollama pull llama3.2
```

---

## 8.2 CLI flags (`main.py`)

| Flag | Type | Default | Meaning |
|------|------|---------|---------|
| `--data-file` | path | `data/dataset/Abalone/Abalone.csv` | Input dataset. CSV or JSON. |
| `--output-report-dir` | path | `output/Abalone` | PDF output directory. |
| `--output-graph-dir` | path | `output/Abalone` | Graph image output directory. |
| `--simulation_mode` | `online`/`offline` | `offline` | If using simulated data, generate now (`online`) or use a pre‑built dataset (`offline`). |
| `--data_mode` | `real`/`simulated` | `real` | Use the supplied file vs. generate synthetic data. |
| `--debug` | flag | `False` | Substitute fake / fast statistics — useful for pipeline debugging without waiting for slow tests. |
| `--initial_query` | string | `"Do causal discovery on this dataset"` | The user's natural‑language question. |
| `--parallel` | flag | `False` | Run bootstrap iterations in parallel. |
| `--demo_mode` | flag | `False` | Mode used by the bundled demo runner. |

The exact list lives at the top of [`main.py`](../main.py); always check
there for the latest.

---

## 8.3 Built‑in defaults (`global_setting/state.py`)

These are baked into the dataclass field defaults. You can change them by
editing the file or by mutating `global_state` after initialization.

### `Statistics` defaults
| Field | Default | Effect |
|-------|---------|--------|
| `boot_num` | `20` | Bootstrap iterations during postprocessing. |
| `alpha` | `0.1` | Significance threshold for CI tests. |
| `num_test` | `100` | Number of statistical tests to run. |
| `ratio` | `0.5` | Sub‑sampling ratio used inside bootstrap. |
| `time_series` | `False` | Override if you know your data is temporal. |
| `time_lag` | `None` | Auto‑detected when `time_series=True`. |

### `Algorithm` defaults
| Field | Default | Effect |
|-------|---------|--------|
| `handle_correlated_features` | `True` | Drop near‑duplicate columns before fitting and re‑attach afterwards. |
| `correlation_threshold` | `0.99` | Pearson correlation above which columns are treated as duplicates. |
| `waiting_minutes` | `1440.0` (24 h) | Hard timeout for algorithm execution. |
| `gpu_available` | `False` | Auto‑detected. Some wrappers refuse to run if `False`. |

### `UserData` flags
| Field | Default | Effect |
|-------|---------|--------|
| `accept_CPDAG` | `True` | Whether partially‑directed graphs are an acceptable answer. Set to `False` to force full DAGs. |

---

## 8.4 Docker‑specific knobs

Both Dockerfiles set `PYTHONPATH=/app` and expose port `7860` for Gradio.
A typical run:

```bash
docker run -it --rm \
  -v $(pwd):/app \
  -p 7860:7860 \
  --env-file .env \
  causal-copilot-cpu \
  python web_demo/demo.py
```

For GPU:

```bash
docker run -it --rm --gpus all \
  -v $(pwd):/app \
  -p 7860:7860 \
  --env-file .env \
  causal-copilot-gpu \
  python web_demo/demo.py
```

If you talk to a host‑side Ollama from inside the container, add:

```bash
--add-host=host.docker.internal:host-gateway
```
on native Linux Docker Engine.

---

## 8.5 LaTeX configuration

* The list of required LaTeX packages is in
  [`requirements_latex.txt`](../requirements_latex.txt).
* [`install_latex.py`](../install_latex.py) installs TinyTeX and those
  packages on a fresh machine.
* [`test_latex_functionality.py`](../test_latex_functionality.py) is a
  smoke test that compiles a tiny `.tex` to verify the toolchain works.
* The Dockerfiles run all of the above for you, which is why the project
  recommends Docker.

---

## 8.6 Submodules

Algorithm implementations under [`externals/`](../externals) are git
submodules. After a fresh clone:

```bash
git submodule update --init --recursive
```

The Dockerfiles already do this. If you skip it, several wrappers will
import‑error.

---

## 8.7 Tuning tips

* If the LLM keeps choosing slow algorithms, lower `waiting_minutes` so
  it gives up sooner.
* If bootstrap is too slow, lower `Statistics.boot_num` (default 20).
* If your data has very correlated features that you *want* to keep, set
  `Algorithm.handle_correlated_features=False`.
* If you want shorter / faster reports, edit the LaTeX template in
  `report/context/`.
