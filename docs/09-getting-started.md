# 9. Getting Started — A Beginner's Walkthrough

This is the gentlest possible "from zero to first PDF" walkthrough. If you
have *never* used Causal‑Copilot, do the steps in this document in order.

---

## 9.1 Prerequisites

You need:

* A computer with at least 8 GB RAM (16 GB recommended).
* About 5 GB of free disk space (much of that is the LaTeX install and
  Python deps).
* One of:
  * **Docker** (recommended — the easy path).
  * **Conda + Python 3.10** (the manual path, can be flaky).
* Access to **one** of:
  * An OpenAI API key, OR
  * An OpenRouter API key, OR
  * A locally‑running [Ollama](https://ollama.com/) server.

---

## 9.2 Step 1 — Clone the repo

```bash
git clone https://github.com/Lancelot39/Causal-Copilot.git
cd Causal-Copilot
git submodule update --init --recursive
```

The submodule step is important — algorithm implementations live in
`externals/` and won't exist without it.

---

## 9.3 Step 2 — Choose a setup method

### Option A — Docker (recommended)

CPU image:
```bash
docker build -f Dockerfile.cpu -t causal-copilot-cpu .
```

GPU image (requires NVIDIA + nvidia-container-toolkit):
```bash
docker build -f Dockerfile.gpu -t causal-copilot-gpu .
```

The image takes ~10 minutes to build the first time because it installs
LaTeX. Subsequent builds are cached.

### Option B — Conda

```bash
conda create -n causal-copilot python=3.10 -y
conda activate causal-copilot
bash setup_cpu.sh    # or setup_gpu.sh
```

The setup script installs system packages (Graphviz, TeX Live), Python
deps, and verifies LaTeX. If it fails, the README recommends Docker.

---

## 9.4 Step 3 — Configure your `.env`

```bash
cp .env.example .env
```

Edit `.env`. The simplest "it just works" config is OpenAI:

```bash
LLM_PROVIDER=openai
LLM_MODEL=gpt-4o
OPENAI_API_KEY=sk-...
```

For a fully local, free setup (slower, model‑dependent quality):

```bash
LLM_PROVIDER=ollama
LLM_MODEL=llama3.2
OLLAMA_BASE_URL=http://localhost:11434
```

…then in a separate terminal:

```bash
ollama pull llama3.2
ollama serve   # if not already running
```

If you're on Docker and Ollama is on the host, set
`OLLAMA_BASE_URL=http://host.docker.internal:11434`.

See [`08-configuration.md`](08-configuration.md) for every variable.

---

## 9.5 Step 4 — Run it

### The web demo (recommended for first‑timers)

```bash
# native
python web_demo/demo.py

# or in Docker
docker run -it --rm \
  -v $(pwd):/app -p 7860:7860 \
  --env-file .env \
  causal-copilot-cpu \
  python web_demo/demo.py
```

Open <http://localhost:7860>. Upload a CSV (or use the bundled examples
under `data/dataset/`), type a question like *"Find the causal structure
in this data"*, and watch the chat narrate every stage.

### The CLI (good for scripting)

```bash
python main.py \
  --data-file data/dataset/Abalone/Abalone.csv \
  --output-report-dir output/Abalone \
  --output-graph-dir output/Abalone \
  --initial_query "Discover the causal graph for this dataset"
```

When it finishes, look in `output/Abalone/` for `report.pdf` plus the
intermediate graphs.

---

## 9.6 Step 5 — Read the output

The report is structured (see [`03-workflow.md`](03-workflow.md) §3.7):

1. **Data summary** — what the system thinks of your data.
2. **EDA plots** — correlations, distributions.
3. **Methodology** — which algorithm was chosen and why (LLM rationale).
4. **Initial graph** — first cut from the algorithm.
5. **Bootstrap heatmap** — confidence per edge.
6. **Revised graph** — after LLM pruning + cycle break.
7. **Edge interpretations** — one paragraph per arrow, in plain English.
8. **Refutation / sensitivity** — how robust the answer is.

The web demo also shows the same in‑browser and lets you ask follow‑up
questions ("Why did you remove A → B?").

---

## 9.7 Troubleshooting

| Symptom | Try |
|---------|-----|
| "LaTeX command not found" | Use Docker, or run `python install_latex.py` and re‑run `test_latex_functionality.py`. |
| "ModuleNotFoundError" on a wrapper | You forgot `git submodule update --init --recursive`. |
| OpenAI 401 | `OPENAI_API_KEY` missing/invalid in `.env`. |
| Ollama "connection refused" | `ollama serve` not running, or wrong `OLLAMA_BASE_URL`. From Docker, use `host.docker.internal`. |
| "Algorithm hangs forever" | Lower `Statistics.boot_num` or `Algorithm.waiting_minutes`. Try `--debug` to skip slow stat tests. |
| Web UI shows blank chat | Check the terminal — Gradio prints the local URL and any auth errors there. |

---

## 9.8 Where to go from here

* Pick a different algorithm by name? → see [`06-algorithms.md`](06-algorithms.md).
* Understand what the report sections mean? → [`10-glossary.md`](10-glossary.md).
* Hack on the code? → [`05-modules.md`](05-modules.md) and
  [`02-architecture.md`](02-architecture.md).
* Plug in a new model? → [`07-llm-integration.md`](07-llm-integration.md).
