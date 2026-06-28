---
layout: post
title: "SOSDash: Designing a Trustworthy Conversational Analytics Dashboard"
date: 2026-06-27 09:00:00 +0000
categories: [research, architecture]
tags: [agentic-ai, explainability, trustworthy-ai, architecture]
summary: "A blog-style architecture walkthrough for SOSDash, a research prototype that combines metadata discovery, evidence grounding, and conversational analytics."
---

Early in the SOSDash research project, the core design problem became clear: food-system stakeholders need dashboards that can answer natural-language questions, but the answers must not feel like plausible fiction. In practice, that meant the system had to do three things at once:

1. discover which dataset holds the answer,
2. retrieve grounded evidence from that dataset and related policy documents,
3. return a concise analytics result with a chart-ready structure and a transparent provenance trail.

This post walks through the architecture behind that design, explains the reasoning that guided it, and shows how the prototype turns a query into a verified, evidence-backed response.

## What SOSDash actually delivers

SOSDash is designed for people who need answers they can trust, not just interesting prose. It takes a question like "What happened to pesticide use in Spain between 2015 and 2020?" and delivers:

- a chart-ready analytics result,
- the exact dataset that provided the numbers,
- an explanation grounded in methodology and policy evidence,
- and a transparent path showing how the answer was assembled.

That means the system is not only faster than manual dashboard exploration; it is also safer for decision-making. When the application is food-system analytics, numbers without context can mislead. SOSDash reduces that risk by making both the data and the evidence explicit.

## The architecture in one sentence

SOSDash is a layered retrieval agent: a user query first hits a metadata catalog, then a query plan is constructed, the data source is identified, and the final answer is composed from structured dataset responses plus policy-backed explanations.

## What makes SOSDash different

A few design choices from the source repository deserve to be highlighted:

- The system is dataset-agnostic. Any dataset that conforms to the metadata contract can be registered without code changes.
- The agent never queries datasets directly. It discovers a dataset through the metadata layer first, then dispatches a structured fetch request to the named index.
- Evidence is multi-dimensional: the system retrieves not only numbers, but also methodology/context text and policy citations.
- The conversational layer is JSON-mode ReAct. The LLM emits structured tool calls and final answers, which makes the loop far more robust than free-form prompt parsing.

## The architecture stack

At the heart of SOSDash is a four-layer search stack.

- **L1: Metadata Catalog** — this is the dataset discovery layer. Each registered dataset is described by its anchors, filters, metrics, and provenance pointer. The agent uses this layer to decide which dataset to query.
- **L2: Methodology Vectors** — this layer stores chunks of methodology and data-quality notes as embeddings. When the agent needs to explain a definition or caveat, it can retrieve the relevant text.
- **L3: Policy Vectors** — this layer holds policy document excerpts, also as embeddings. It is the source of evidence when the answer needs a citation from Farm-to-Fork or similar documents.
- **Data indexes** — each dataset gets its own Elasticsearch index for raw observation records. The agent issues structured queries against one dataset index at a time.

That structure is important. In the prototype, the orchestrator in `core/orchestrator.py` first discovers the dataset in L1, then uses the findings to build a typed `QueryPlan`, and finally executes the plan against the selected data index.

## The metadata contract that makes dataset-agnostic discovery possible

The real innovation in SOSDash is the metadata-validated mapping (MVM). Every registered dataset has a machine-readable entry in the L1 catalog describing:

- the spatial, temporal, and thematic anchor columns,
- which columns are metrics,
- required filters for disambiguating units or row types,
- optional hierarchy metadata to prevent double-counting,
- a pointer to a methodology context entry.

This contract is what makes the system pluggable. If a new dataset shares anchors with existing data, the agent can align them without any hard-coded schema logic. That is also why cross-dataset queries are a feasible research goal for SOSDash.

## Query flow: from question to chart spec

A typical query in SOSDash follows this flow:

1. The user types a natural-language question in the Streamlit chat UI.
2. The orchestrator calls `CatalogSearch` against L1 to find candidate datasets and anchor values.
3. The LLM builds a `QueryPlan` using the metadata contract: filters, time range, aggregation, and grouping.
4. The orchestrator calls `DataFetcher` against the selected dataset index.
5. If the user asks for explanation, the orchestrator also retrieves methodology chunks from L2 and policy evidence from L3.
6. The final answer is returned in one of two forms: a structured analytics object with chart fields, or a textual explanation when no data answer is available.

That flow is deliberately tool-oriented. The model does not return unstructured prose and then have the system parse it. Instead, it emits JSON with explicit arguments, which eliminates a large class of hallucination bugs.

## Key implementation concepts

The prototype implements a small set of conceptual components that together enforce discovery, grounding, and reproducible answers:

- Orchestrator / planner: runs a JSON-mode ReAct loop, sequences tool calls, and builds typed query plans from user intents.
- Tool adapters: typed interfaces that translate query plans into backend queries (search, aggregation, retrieval) and return structured results.
- System prompts and policies: a constrained instruction layer that enforces a "search-before-talk" behaviour and prevents the agent from inventing dataset names or numeric values.
- UI renderer: a lightweight web front-end that accepts structured analytics objects (chart spec + data) and displays evidence citations alongside charts.

The deployed prototype uses Elasticsearch as the primary retrieval backend. L2 and L3 use vector search over embedded text; the data indexes use structured ES queries for filtering, aggregation, and chart production.

## Why the conversation is evidence-backed

The SOSDash prototype is not just a numerical answer engine; it is a dual-channel evidence system.

- Data answers come from the dataset index that the metadata catalog selected.
- Explanations can reference methodology text and policy excerpts, which are retrieved from L2 and L3.

Because the policy documents are registered explicitly and chunked into the vector index, the system can cite a specific source rather than just saying "according to policy". That is the difference between a dashboard answer and a trustworthy explanation.

## The real-world impact

For a policy analyst or food-system stakeholder, this architecture changes two things:

**Speed of decision-making.** Instead of manually searching a dashboard or wading through spreadsheets, users ask a natural-language question and get an answer with a chart in seconds. A question like *"Show pesticide usage in Spain from 2015 to 2020"* takes 20–40 seconds end-to-end in the prototype — retrieval, reasoning, visualization, and all. For exploratory analytics that previously required either manual lookups or context-switching between multiple tools, that is a material improvement.

**Trust in the numbers.** The system returns not just a chart, but a citation path. If a user asks *"Explain the drop in pesticide use in Spain,"* they receive not only the trend visualization but also excerpts from the EU Farm-to-Fork strategy or relevant methodology notes that contextualize the change. This moves the answer from a black box ("the model said so") to a transparent claim ("here is the data, and here is the policy framing"). For stakeholders making budget or strategy decisions, that traceability is not a luxury — it is a requirement.

**Cross-dataset insight without schema friction.** The metadata-driven approach means adding a new dataset (e.g., a national agricultural census) does not require retraining the agent or rewriting backend logic. Register the dataset with anchors, upload methodology notes, and the agent can immediately use it in cross-dataset queries alongside existing FAO data. For organizations that manage multiple data sources, that pluggability removes a major operational barrier to integrated analytics.

## Practical lessons from the project

### Keep the agent’s knowledge narrow

The agent only knows the metadata catalog and the registered indexes. It does not have free access to the raw dataset files. That constraint is what keeps the answer grounded.

### Use JSON-mode tool output

The prototype makes the LLM emit structured actions. The result is a cleaner control flow and far fewer parsing errors when the agent calls a tool.

### Make dataset registration explicit

The upload flow is both a product feature and a design safeguard. Users upload a CSV, generate a metadata draft, review the inferred anchors, and then register it. This is where the system preserves its dataset-agnostic guarantees.

### Guard hierarchical totals automatically

Many food-system datasets contain both totals and breakdown rows. The metadata contract includes an optional hierarchy field so the system can automatically enforce a total-row filter during aggregation, preventing double-counting.

## Reliability and benchmark design

SOSDash was built with reliability in mind. The prototype is benchmarked against a naive baseline that simply uses the same language model without metadata discovery or evidence citation.

The benchmark focuses on the kinds of queries that matter in practice:

- single-dataset trend questions,
- cross-dataset composition questions,
- explanation questions that require policy evidence.

The evaluation measures practical outcomes: answer accuracy, citation integrity, latency, token efficiency, and energy usage. In other words, it is not just testing whether the system can respond; it is testing whether the response is trustworthy, efficient, and usable.

## Where this architecture is headed

The current implementation proves the pattern: a conversational analytics dashboard can be both flexible and trustworthy if it is built around:

- a metadata-driven discovery layer,
- explicit tool-based reasoning,
- separate evidence retrieval channels,
- structured analytics output.

The next step in this line of work would be to make dataset alignment more robust across schema drift, improve policy-document reasoning with better vector re-ranking, and incorporate stronger verification of numeric claims.

## Conclusion

SOSDash is best understood as a layered retrieval agent for food-system analytics. It does not pretend to be a general-purpose chatbot. Instead, it is a specialized system that answers questions with a clear source path: metadata catalog → dataset index → evidence documents.

That is the architectural story I would tell in a blog post, and it is the version I have now aligned with the actual `se4gd-sosdash` source repository.
