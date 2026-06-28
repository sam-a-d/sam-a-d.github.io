---
layout: post
title: "SOSDash: an AI analytics agent that proves every number it gives you"
date: 2026-06-27 09:00:00 +0000
categories: [projects, ai-engineering]
tags: [agentic-ai, rag, llm, elasticsearch, data-engineering, evaluation, trustworthy-ai]
pin: true
image:
  path: /assets/img/sosdash/architecture.png
  alt: "SOSDash architecture — a ReAct LLM agent over a metadata catalog and Elasticsearch retrieval, returning grounded answers with cited evidence."
summary: "A conversational analytics agent for large statistical datasets: ask in plain English, get a chart-ready answer with a verifiable citation for every number. The LLM never sees raw data; it works from a metadata contract and retrieves grounded values and evidence from Elasticsearch, so the system stays scalable, data-sovereign, and auditable."
---

> **TL;DR** — For my master's thesis I built **SOSDash**, an LLM agent that turns a plain-English question into a chart-ready answer *with a verifiable citation for every number*. The trick is that the model never touches your data: it reasons over a compact **metadata contract**, then retrieves grounded values and policy evidence from **Elasticsearch**. That one decision is what keeps the system **scalable, data-sovereign, and auditable** in ways a context-stuffing chatbot can't be. Validated against an independent oracle, it answered **every benchmark question correctly**, with a **verifiable, page-referenced source** behind every number.
>
> 🚀 [**Live demo**](https://sustaindatachat.app.citius.gal/) &nbsp;·&nbsp; 💻 [**Source code**](https://github.com/sam-a-d/se4gd-sosdash) *(private for now — available on request)* &nbsp;·&nbsp; 📄 [**Sample output (PDF)**](/assets/files/sosdash-sample-output.pdf)

![Built with: Python, Elasticsearch, LangChain, Streamlit, Docker and Ollama — plus RAG, LLM agents, ETL pipelines and evaluation](/assets/img/sosdash/built-with.png){: width="820" }

## The problem it solves

Organisations sit on rich statistical data and still can't *use* it to answer a plain question. Three failures stack up:

- **Dashboards show numbers, not meaning.** You watch a line go down, but the tool can't tell you *why*. The explanation lives in policy PDFs and methodology notes nobody reads.
- **The data is fragmented.** A real question (*"does barley yield track pesticide use in this country?"*) spans several datasets that don't share a schema, so answering it means manual joins in a spreadsheet.
- **LLM chatbots hallucinate.** Bolt a chatbot on and it explains fluently, but it will invent a plausible statistic or quote a policy document that doesn't exist.

For a policy analyst deciding where to put a budget, a confident wrong number, or a fabricated citation, is worse than no answer at all.

**SOSDash closes the gap.** Ask a question in plain English and it:

- finds the right dataset and returns the numbers as a **chart-ready answer**,
- **joins across datasets automatically** when the question spans more than one,
- and pairs every quantitative claim with **verifiable evidence**: a real passage from a real document, with its page.

The domain here was EU food-system statistics (FAO data + EU policy documents), but the architecture is domain-agnostic. Point it at any registered dataset and it works unchanged.

## How it works

SOSDash is a layered retrieval agent. A question never goes straight to the data. The agent first discovers *which* dataset can answer it, then dispatches a structured query, then grounds any explanation in retrieved policy text.

![SOSDash architecture: query → ReAct orchestrator → tool layer → Elasticsearch (L1 metadata, data indexes, L2 methodology, L3 policy) → verified answer](/assets/img/sosdash/architecture.png){: width="860" }

### The metadata contract

The agent's entire knowledge of a dataset is one machine-readable entry: the spatial / temporal / thematic *anchor* columns, the metrics, required filters, a hierarchy guard, and a pointer to methodology. It reasons over that contract and issues queries the system runs on its behalf. It never reads the rows themselves. (Why that single decision matters so much is the [next section](#scalable-sovereign-verifiable).)

### The agent loop

The orchestrator runs a **JSON-mode ReAct loop** on a self-hosted **`qwq:32b`** model (32 billion parameters, served with Ollama). Each turn, the model emits one JSON object: either a tool call (`thought` / `tool` / `args`) or a final answer. There is no brittle prose-parsing. When it needs numbers, it compiles a typed **QueryPlan** (filters, range, aggregation, group-by), and the tool layer translates that into an Elasticsearch **Query DSL** request. Sums, joins, and aggregations run *in the database*, so the model spends zero tokens on arithmetic. Two safety rails are automatic and dataset-agnostic. A **required-filters** default picks the right measurement variant, and a **hierarchy guard** injects the total-row filter when aggregating, so parent and child rows never double-count. The LLM adapter is provider-agnostic, but every result reported here is from `qwq:32b`.

### Retrieval-Augmented Generation (RAG)

Explanations use textbook **RAG** with one strict rule: a citation can only be a passage that was actually retrieved. Policy PDFs are chunked, embedded as 768-dimensional dense vectors (`nomic-embed-text`) and indexed in Elasticsearch. At answer time the agent runs a **k-NN similarity search** for the most relevant excerpts and renders them with their source document and page. Because the cited text is pulled from the index rather than generated, a fabricated quote simply isn't an available move. The citation is, by construction, a real passage from a real document.

## Data architecture

Under the hood, SOSDash runs on a single **Elasticsearch** cluster that plays four roles at once: structured analytics, lexical discovery, and two kinds of semantic search, each fed by its own small, repeatable ingestion path. This is the write side of the system: how raw sources become queryable layers.

![Data architecture — CSV/metadata/PDF sources flow through ingestion scripts into four Elasticsearch layers: data indexes, L1 catalog, L2 and L3 vectors](/assets/img/sosdash/data-architecture.png){: width="860" }

| Layer | What it holds | Populated from | Retrieval | Role |
|---|---|---|---|---|
| **Data indexes** | observation records, one index per dataset | FAO CSVs → `data-ingestion.py` | structured — Query DSL + aggregation | the numbers |
| **L1 · Metadata catalog** | dataset contracts: anchors, filters, methodology pointer | metadata → `ingest-metadata-l1.py` | lexical | dataset discovery |
| **L2 · Methodology vectors** | definitions, data-quality notes (~300 chunks) | notes → `ingest-metadata-l2.py` | dense-vector / kNN | data-literacy |
| **L3 · Policy vectors** | policy excerpts, chunked 500/50, 768-dim (~169 chunks) | PDFs → chunk → embed (`nomic-embed-text`) | dense-vector / kNN | grounded citations |

Two things make this a data-engineering story rather than a schema diagram:

- **One store, three retrieval modes.** Exact aggregation (Query DSL), keyword discovery, and dense-vector kNN all run against the same cluster, so a single question can pull a precise number *and* the semantic evidence that explains it, with no second system to operate.
- **Schema-agnostic ingestion.** Each source type has its own ingestion path, and a new dataset is registered by writing one metadata (MVM) entry to L1: no migration, no re-prompt, no application-code change. The metadata contract is the only coupling between the data and the agent.

## Scalable, sovereign, verifiable

A plain LLM chatbot tries to answer by *reading the data*: you paste rows into the prompt and hope they fit. That breaks the moment the data is larger than the context window, and it means the model sees everything. SOSDash inverts the relationship: the model reads *about* the data (the metadata contract), and the system fetches and computes the answer. Four properties fall out of that one decision:

- **Scalable.** Because the model never ingests the data, dataset size is decoupled from the LLM entirely. The data can grow without bound, while the agent still only reads a compact catalog and pulls the few rows it needs, aggregating them in Elasticsearch. Capacity is bounded by your database, not the model's context window. A context-stuffing chatbot is left behind the moment the data outgrows the prompt.
- **Data-sovereign.** The LLM never sees a raw row. It operates on the metadata contract and dispatches queries the system runs; the data stays in your store. No PII or proprietary records enter the prompt: the difference between "we can run this on regulated data" and "legal says no."
- **Verifiable.** Every number is the result of a deterministic database query, and every citation is a passage that was actually retrieved (with document and page). The derivation trace shows exactly how each figure was produced, so answers are auditable end-to-end.
- **Pluggable.** Onboarding a dataset is a metadata entry plus an ingestion run, not a code change or a re-prompt. Datasets that share anchor values become joinable automatically, which is what makes cross-dataset questions possible at all.

## One query, traced end to end

Abstract layers are easier to trust when you can watch a real question move through them. Here is the exact path *"What is the pesticide usage in Spain from 2015 onwards? Explain the trend."* takes, hop by hop, with the technology doing the work at each step:

![End-to-end trace of one query through SOSDash's layers and technologies — LLM agent, Elasticsearch lexical catalog, ES Query DSL, vector/kNN policy retrieval, Streamlit render](/assets/img/sosdash/query-trace.png){: width="780" }

The agent never guesses. It discovers the right dataset by **lexical search** over the metadata catalog (L1), then compiles a typed query plan that **Elasticsearch** executes (the maths runs in the database, not the model). Finally it retrieves real policy passages by **vector (k-NN) similarity** (L3) to explain *why* the trend moved, and only then composes a single structured answer for the front-end to render.

For this question, that resolves to a verbatim, fully-grounded answer:

> Pesticide usage in Spain decreased from 77,217 tonnes in 2015 to 52,907 tonnes in 2023, with a notable drop starting in 2022. This aligns with the EU's 2030 target to reduce pesticide use by 50%, as outlined in the EU Biodiversity Strategy and Farm-to-Fork initiatives.

Every figure in that sentence came from an Elasticsearch aggregation, and the 50%-by-2030 claim is backed by three retrieved excerpts (EU Biodiversity Strategy p.7 & p.13, Farm-to-Fork p.7), not by the model's memory.

## What the user sees

The trace above is the under-the-hood view. In the product, all of it lands on a single screen. That same question produces five stacked panels, each doing a distinct job:

**1. The natural-language answer.** At the top sits the grounded summary from the trace above, leading with the numbers and the *why*. Notice what's *not* there: no hedging, no invented precision. The two figures (77,217 and 52,907 tonnes) are lifted verbatim from the database, and the policy framing comes from retrieved documents. The model is summarising, not estimating.

**2. The visualization.** Directly below sits a labelled line chart, *"Pesticide Usage in Spain (2015–2023)"*, with year on the x-axis and tonnes on the y-axis. It shows the shape the prose only hints at: a plateau around 75k tonnes in 2015–2016, a dip near 72k in 2017, a slow recovery to ~75k by 2021, then the sharp fall the summary calls out. This isn't a screenshot of a BI tool. The agent emitted a structured chart spec (chart type, axes, and the data array), and the front-end rendered it.

![SOSDash grounded answer and trend chart for the Spain pesticide query](/assets/img/sosdash/ui-answer-chart.png){: width="820" .shadow }
_Panels 1–2 in the app: the grounded natural-language answer, and the chart-ready trend the agent emitted as a spec._

**3. "How this answer was derived" — the methodology.** An expandable panel states, in plain language, exactly which dataset answered the question and how it was filtered: the `pesticide_usage` dataset, restricted to *area = Spain*, *element = Agricultural Use*, *item = Pesticides (total)*, over *2015 onwards*, aggregated as the **sum of value grouped by year**, covering **9 records**. It also surfaces the dataset's own methodology notes (Layer 2). For instance, it notes that gaps were filled by linear interpolation, and that the figures come from the FAOSTAT Pesticides Use domain (global coverage, 1990–2023). This is the data-literacy layer: the user sees the *definition* behind the number, not just the number.

**4. "Show exact steps" — the calculation trail.** Toggling this reveals the literal sequence of tool calls the agent made:

1. Searched the dataset catalog for *"pesticide usage Spain"* → found 1 candidate dataset.
2. Queried `pesticide_usage`, filtered area = Spain, 2015 onwards, grouped by year.
3. Queried `pesticide_usage`, filtered element = Agricultural Use, area = Spain, item = Pesticides (total), 2015 onwards, **summed by year → 9 records**.
4. Searched policy documents for *"Spain pesticide reduction policies 2022"* → 3 excerpts.

Every number in the answer is reconstructible from these steps. There is no hidden reasoning, and nothing the user has to take on faith.

![How this answer was derived — dataset, filters, methodology notes, and the exact tool-call steps](/assets/img/sosdash/ui-derivation.png){: width="820" .shadow }
_Panels 3–4: the methodology (dataset, filters, sum-by-year over 9 records, FAOSTAT notes) and the exact tool-call trail._

**5. Policy evidence — the citations.** Finally, the three retrieved excerpts are shown in full, each with its source document and page:

- **EU Biodiversity Strategy, p. 7** — "…reduce by 50% the overall use of — and risk from — chemical pesticides by 2030…", the passage that backs the headline "50% by 2030" claim.
- **Farm-to-Fork (`farm.pdf`), p. 7** — the Harmonised Risk Indicator showing a 20% decrease in pesticide risk over five years, and the commitment to cut overall use and risk by 50%.
- **EU Biodiversity Strategy, p. 13** — the Integrated Nutrient Management Action Plan and the Farm-to-Fork reduction targets.

These are the *actual passages* the explanation was built from: quoted, not paraphrased. A sceptical reader can check the claim against the source.

![Policy evidence — three retrieved excerpts, each with source document and page](/assets/img/sosdash/ui-policy-evidence.png){: width="820" .shadow }
_Panel 5: the three retrieved excerpts, shown verbatim with their source document and page._

Put together, that is the whole product in one view: a number, a picture, the method behind the number, the steps that produced it, and the evidence behind the explanation. Every panel above is a real screen from the live system, and the complete output is also available as a [single-page PDF](/assets/files/sosdash-sample-output.pdf).

The hardest query class is **cross-dataset composition** (for example, *"does barley yield in Lithuania correlate with that country's total pesticide use?"*), which requires aligning two differently-shaped datasets on shared keys. The metadata anchors turn that into a planned join rather than a guess, and it's where the layered architecture earns its keep.

## Validation

The point wasn't just to build a chatbot. It was to show the answers can be trusted. Every quantitative claim is checked against an independent **Elasticsearch oracle**: ground truth computed straight from the indexes with no LLM in the loop, so the system never grades itself. Across a frozen, pre-registered benchmark (run on the self-hosted `qwq:32b` model for full reproducibility), SOSDash:

- answered **every question correctly** against the oracle,
- produced a **verifiable, page-referenced citation for every policy answer** (zero fabricated),
- made **zero unsupported claims**,
- and did it **efficiently**, emitting only ~580 output tokens per query (≈3.9 mWh), because it offloads the maths to Elasticsearch instead of computing in generated text. Output (decode) tokens dominate both energy and latency, so generating fewer of them is what keeps the cost down.

The benchmark was deliberately small and deep: 6 questions × 3 repeats = 36 runs, with energy attributed from per-token coefficients calibrated on the GPU host (R² = 0.988). The harness logs accuracy, citation integrity, tokens, latency, and energy per run against the oracle and a frozen question set, and it scales to more questions without code changes. Latency is set by the self-hosted 32B model, not the architecture: the agent's own overhead (catalog lookup, query planning, the Elasticsearch calls) is small, so a faster model would bring it down. For reference, a naive same-model baseline given the identical data kept up on simple lookups but fell short on cross-dataset composition, and it produced no verifiable citations. The gains come from the architecture, not the model.

## Limitations & what's next

I'd rather name the edges than hide them. SOSDash is a research prototype. It's single-user with no authentication or access control. The benchmark is intentionally small (6 questions, 3 repeats): illustrative and deep rather than a broad leaderboard. End-to-end latency on the self-hosted 32B model is tens of seconds (the architecture is cheap; the local model is the bottleneck). The five FAO datasets are a development corpus, not a scale test.

Natural next steps: faster/hosted inference to bring latency to interactive speeds, a larger and broader benchmark, vector re-ranking for sharper policy retrieval, and stronger verification of numeric claims beyond the oracle check.

## Tech stack

| Area | What I used |
|---|---|
| **Languages & core** | Python, pandas, NumPy |
| **LLM / agents** | JSON-mode **ReAct** orchestrator (provider-agnostic adapter), prompt engineering, structured JSON output, LangChain-compatible adapters; self-hosted **`qwq:32b`** (32B params) via **Ollama** on an RTX 5090 |
| **Retrieval & RAG** | **Elasticsearch** — Query DSL for the numbers, **dense-vector / k-NN search** for grounding; **embeddings** (`nomic-embed-text`, 768-dim); metadata-driven semantic modeling |
| **Data engineering** | **Ingestion / ETL pipelines** — FAOSTAT CSVs → per-dataset indexes, policy PDFs → chunked vector index; a **schema-agnostic metadata contract** for plug-in datasets; **multi-source integration & cross-dataset joins** on shared keys |
| **App & infra** | **Streamlit** (chat + chart + evidence + dataset/document upload), **Docker** / docker-compose, remote-GPU inference over an SSH tunnel |
| **Evaluation / MLOps** | Custom **benchmark harness** with an independent oracle, accuracy + citation-integrity scoring, and **codecarbon-calibrated energy accounting** (Green-AI cost analysis) |

---

This was my **solo SE4GD master's thesis**: the design, implementation, and evaluation are all my own work, framed here for an engineering audience rather than an academic one. For the deeper design rationale and the full evaluation methodology, the [source repository](https://github.com/sam-a-d/se4gd-sosdash) *(available on request)* and the [live demo](https://sustaindatachat.app.citius.gal/) are the best places to dig in.
