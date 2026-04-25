# 7. LLM Integration

This document explains exactly **where, how, and why** Causal‑Copilot calls
a Large Language Model. It is the reference for anyone who wants to:

* swap providers (OpenAI ↔ Ollama ↔ OpenRouter),
* tune or replace prompts,
* understand the conversation logs in a report appendix,
* debug a "the LLM picked the wrong algorithm" issue.

The LLM is not used for math or for running algorithms. It is used for
**judgment, language, and structured decision‑making**.

---

## 7.1 Provider abstraction

The single point of entry is `LLMClient` in
[`llm/llm_client.py`](../llm/llm_client.py).

```text
LLM_PROVIDER  ∈ { openai | openrouter | ollama }
LLM_MODEL     e.g. gpt-4o, anthropic/claude-sonnet-4, llama3.2
```

| Provider | What it is | When to use |
|----------|------------|-------------|
| **OpenAI** | Hosted GPT models | Best instruction‑following; default. |
| **OpenRouter** | Aggregator API (Claude, Llama, Mistral, …) | Want a non‑OpenAI model without setting up a server. |
| **Ollama** | Local model runtime | Privacy, offline, no per‑token cost. |

Configuration is read from `.env` (see
[`08-configuration.md`](08-configuration.md)).

`LLMClient` provides:

* Chat completions with system + user messages.
* JSON‑mode output guided by a Pydantic model (auto‑generated schema
  instructions).
* Robust JSON extraction (strips `\`\`\`json` fences, retries on minor
  parse errors).
* Provider‑specific quirk handling (Ollama returns slightly different
  payloads than OpenAI).

For Ollama specifically, [`llm/ollama_client.py`](../llm/ollama_client.py)
wraps the local HTTP API, and `llm/test_ollama.py` is a smoke test you can
run after `ollama pull <model>`.

---

## 7.2 Where the LLM is called

There are seven decision points. Each has a dedicated prompt and writes its
transcript to a dedicated `Logging` field on the `GlobalState`.

| # | Caller | Decision | Prompt file(s) | Log field |
|---|--------|----------|----------------|-----------|
| 1 | `preprocess/dataset.py::knowledge_info` | Extract domain knowledge from the user's free‑text query and column names. | (built into the function) | `logging.knowledge_conversation` |
| 2 | `causal_discovery/filter.py` | Pick 1–2 candidate discovery algorithms appropriate for this data. | `causal_discovery/context/algo_select_prompt.txt`, plus per‑algo metadata in `context/algos/` | `logging.filter_conversation` |
| 3 | `causal_discovery/rerank.py` | Pick the *single* best algorithm given benchmark results. | `causal_discovery/context/algo_rerank_prompt.txt` | `logging.select_conversation` |
| 4 | `causal_discovery/hyperparameter_selector.py` | Choose hyperparameter values for the selected algorithm. | `causal_discovery/context/hyperparameter_select_prompt.txt` + per‑algo docs | `logging.argument_conversation` |
| 5 | `postprocess/judge.py::llm_evaluation_new` | Audit each suspicious edge: keep, remove, or flip? | (templated inside `judge.py`) | `logging.global_state_logging` |
| 6 | `report/report_generation.py::graph_effect_prompts` | Write a paragraph per directed edge interpreting the relationship. | `report/context/` | (embedded in PDF) |
| 7 | `user/discuss.py` | Free‑form chat about the produced report. | dynamic; the report content is the context | `logging.downstream_discuss`, `logging.final_discuss` |

For inference runs there are additional LLM calls inside
`causal_inference/inference.py` to choose treatment/outcome variables and
the appropriate estimator (logged via `inference.hte_*_json`).

---

## 7.3 Prompt structure

A typical Causal‑Copilot prompt has three blocks:

```
[ System ]   Role: "You are a causal-analysis expert."
             + algorithm catalog / parameter docs
             + JSON schema instructions

[ User   ]   Data summary (Statistics.description)
             + Domain knowledge (UserData.knowledge_docs)
             + The user's natural-language query
             + Specific question for this stage
                 ("Which two algorithms should we try?",
                  "Choose hyperparameter values…")

[ Assistant ] Strict JSON output, e.g.
              { "selected": "PC",
                "reason":   "...",
                "params":   { "alpha": 0.05, "indep_test": "fisherz" } }
```

Pydantic models in each caller validate the response. If the LLM produces
malformed output, `LLMClient` retries with a stricter prompt before raising.

---

## 7.4 Logging & auditability

Every LLM exchange is appended to the appropriate list on
`GlobalState.logging`. Because each list is per‑role
(`filter_conversation`, `select_conversation`, …), the report appendix can
show *exactly* which prompts produced which decisions. This is a key feature
for trust: you can always inspect the LLM's reasoning, not just its answer.

---

## 7.5 Cost & latency considerations

* **Filter / rerank / hyperparameter selection** each run once per analysis
  — small prompts (~few KB), low cost.
* **Knowledge extraction** runs once — small prompt.
* **Edge auditing** in `judge.py` may run multiple times (once per
  suspicious edge). On a 30‑node graph this can be ~10–30 calls.
* **Report writing** runs once per directed edge in the final graph —
  potentially the largest chunk of the LLM bill.
* **Discussion** scales with user activity.

If you're cost‑sensitive, the lowest‑hanging optimisation is choosing a
cheaper / local model for `report_generation`'s edge paragraphs while
keeping a strong model for `filter`/`rerank` decisions.

---

## 7.6 Replacing or extending prompts

Because prompts live as plain‑text files in
`causal_discovery/context/` and `report/context/`, you can A/B‑test prompt
changes without touching Python. Re‑running the pipeline picks up the new
text automatically. Just be careful to:

* Keep the **JSON schema instructions** intact, otherwise the Pydantic
  parser will fail.
* Preserve the **placeholders** (e.g. `{statistics}`, `{user_query}`)
  that the caller fills in.

To add a new LLM‑driven decision elsewhere in the pipeline, mirror the
pattern of an existing caller (e.g. `Filter`):

1. Define a Pydantic response model.
2. Author a prompt template.
3. Use `LLMClient.chat(system, user, response_model=…)`.
4. Append the messages to a new `Logging` field.

---

## 7.7 Common failure modes

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `ValidationError` from Pydantic | Local model not following JSON spec | Use a stronger model; tighten the prompt. |
| `Algorithm "XYZ" not found` | LLM hallucinated an algorithm name | Update the algorithm catalog so the LLM "sees" the canonical names. |
| Empty `algorithm_candidates` | Data summary is too sparse | Make sure preprocessing finished successfully (`statistics.description` is not empty). |
| OpenAI 401 | Missing `OPENAI_API_KEY` | See [`08-configuration.md`](08-configuration.md). |
| Ollama connection refused | `ollama serve` not running, or wrong `OLLAMA_BASE_URL` | Start Ollama; if in Docker, set `OLLAMA_BASE_URL=http://host.docker.internal:11434`. |
| Report compile fails | LaTeX not installed | Use the Docker image, or run `python install_latex.py`. |
