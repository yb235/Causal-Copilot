# Causal‑Copilot Documentation (For Dummies Edition)

Welcome! This folder is a friendly, in‑depth guide to the **Causal‑Copilot**
repository. It is written so that someone who has *never* worked with causal
analysis or with this codebase can read it top‑to‑bottom and come away
understanding **what the project does, how it is structured, how data flows
through it, and how to run / extend it**.

If you only want a short marketing‑style description, read the project's main
[`../README.md`](../README.md). If you want to *understand the system*, keep
reading these docs in order.

---

## 📚 Table of Contents

| # | Document | What you will learn |
|---|---------|---------------------|
| 1 | [`01-overview.md`](01-overview.md) | What Causal‑Copilot is, why it exists, what problem it solves, and the vocabulary you need. |
| 2 | [`02-architecture.md`](02-architecture.md) | The five‑module architecture, the role of the LLM, and a high‑level diagram. |
| 3 | [`03-workflow.md`](03-workflow.md) | The complete end‑to‑end pipeline: from a CSV file to a PDF report. |
| 4 | [`04-data-flow-and-schema.md`](04-data-flow-and-schema.md) | The `GlobalState` object, every field it carries, and how it mutates between stages. |
| 5 | [`05-modules.md`](05-modules.md) | A guided tour of every top‑level folder (`preprocess/`, `causal_discovery/`, …). |
| 6 | [`06-algorithms.md`](06-algorithms.md) | A catalog of all 25+ causal discovery and 10+ causal inference algorithms, grouped by family. |
| 7 | [`07-llm-integration.md`](07-llm-integration.md) | Where, why, and how the LLM is called; supported providers; prompt locations. |
| 8 | [`08-configuration.md`](08-configuration.md) | Every environment variable, CLI flag, and default value. |
| 9 | [`09-getting-started.md`](09-getting-started.md) | Step‑by‑step beginner install and "hello world" run. |
| 10 | [`10-glossary.md`](10-glossary.md) | Plain‑English definitions of jargon you'll encounter (DAG, PAG, confounder, HTE, …). |

---

## 🧭 How to read these docs

* **Total beginner?** Read in order: 01 → 09 → 10 → the rest.
* **Engineer joining the project?** Skim 01, then 02, 03, 04, 05 in detail.
* **Researcher who wants to plug in a new algorithm?** Read 02, 05, and 06,
  then look at any wrapper inside `causal_discovery/wrappers/` as a template.
* **Operator deploying the system?** Read 08 and 09.

---

## 🗂️ Repository at a glance

```
Causal-Copilot/
├── main.py                  ← CLI entry point
├── web_demo/                ← Gradio web UI (recommended for users)
├── preprocess/              ← Data cleaning, EDA, statistics
├── causal_discovery/        ← Algorithm selection + 25+ discovery algorithms
├── causal_inference/        ← Effect estimation (DML, DRL, IV, MetaLearners)
├── postprocess/             ← Bootstrap refinement, graph visualization
├── report/                  ← LaTeX → PDF report generation
├── llm/                     ← LLM provider abstraction (OpenAI/Ollama/OpenRouter)
├── global_setting/          ← Central state object and initialization
├── user/                    ← Post‑analysis interactive Q&A
├── utils/                   ← Logging, log suppression
├── data/                    ← Synthetic data simulator + sample datasets
├── externals/               ← Git submodules with algorithm implementations
├── asset/                   ← Images, logos, demo media
├── requirements_*.txt       ← Python dependencies (CPU vs GPU)
├── Dockerfile.*             ← CPU and GPU container definitions
└── setup_*.sh               ← Conda‑environment setup scripts
```

A folder‑by‑folder deep dive lives in [`05-modules.md`](05-modules.md).

---

## ✍️ A note on this documentation

These documents describe the repository **as it exists at the time of writing**.
Code evolves; line numbers and exact file lists can drift. Whenever a fact in
these docs disagrees with the source code, **the source code is the source of
truth**. Each section cites the relevant files so you can verify quickly.
