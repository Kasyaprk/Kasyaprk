# AI Engineer Growth Plan

**For:** Kasyap Rayalacheruvu  
**Role:** Trainer brief after a 2026 landscape review  
**Date:** 13 August 2026  
**Purpose:** Find *real production problems* first, then pick advanced open-source systems that actually help people — and a contribution path that makes you a more confident AI engineer.

This is not a “star 50 repos” list. Confidence in this field comes from owning a small number of hard production problems end-to-end: measuring them, shipping under latency/cost constraints, and putting the fix in code that other people run.

---

## How to read this

You already have the right base: 4+ years shipping LLM systems, Graph RAG (Neo4j + Qdrant), LangGraph / LlamaIndex agents, healthcare + enterprise context. Your own README already names the real next step — **evals and observability**. That is also what the 2026 production literature says is the difference between a demo and a system people trust.

Use this document in three passes:

1. **Problems** — the failure modes that actually show up after launch.
2. **Projects** — production OSS that exists *because* those problems exist.
3. **Plan** — 90 days of focused work: instrument your own systems, then contribute upstream.

GitHub stats below were pulled live on **13 August 2026**. They will move. The *problems* will not.

---

## 0. Trainer diagnosis (honest)

You are past “learn another agent framework.” Collecting CrewAI / AutoGen / ADK will not make you confident. The gap that shows up in senior AI engineer interviews, and in production incidents, is this:

| You already do well | The next layer that creates confidence |
|---|---|
| Build RAG and Graph RAG pipelines | Prove they did not silently regress this week |
| Orchestrate multi-agent graphs | Bound cost, loops, and error propagation |
| Ship FastAPI + Docker + a vector store | Traces, budgets, CI quality gates, hybrid retrieval |
| Domain systems (finance, research, healthcare) | Citation accuracy, abstention, auditability |

A confident AI engineer can sit in an incident review and say: *the reranker added 800ms, faithfulness dropped 6 points after the new PDF corpus, and the agent retried the same failed tool three times — here is the trace, here is the eval that now fails in CI, here is the PR.*

That is the skill we are training.

---

## 1. The problems (research first)

These are the problems worth your next 12–18 months. They are useful to real users (support bots, internal search, coding agents, clinical document Q&A). They are also where production OSS is hiring attention.

### Problem A — RAG fails silently, not loudly

After audits of 30+ production RAG systems (support bots, internal assistants, document Q&A), Tensoria’s 2026 post-mortem found the same five modes. None of them were “wrong vector database” or “wrong chunk size.” ([tensoria.fr](https://tensoria.fr/en/blog/production-rag-failure-modes))

1. **Retrieval looks fine, answers don’t.** Recall@k can be 0.91 while users are unhappy. In one support RAG, the model ignored good chunks and invented policy details in 28% of answers. Faithfulness / groundedness is the product metric; recall is a proxy.
2. **Chunking theatre.** Weeks of semantic chunking often lose to parent-document retrieval + contextual headers (document title, section, date prepended to the chunk). Anthropic’s contextual retrieval result is the cheap win; exotic chunkers usually are not.
3. **Single-shot retrieval cannot do multi-hop.** “What was the revenue impact of the Q2 pricing change?” needs two documents. Query embeddings blend both concepts and land near neither. Query decomposition beats jumping to a full agent loop for most of these.
4. **Eval is a launch artefact, not a system.** Query mix drifts. New PDFs, new pricing tiers, new regulations. A frozen golden set stays green while production goes bad. Fix: CI golden set + weekly production sampling + monthly promotion of real failures into the set.
5. **Latency, cost, and traces are treated as afterthoughts.** Teams chase faithfulness 0.78 → 0.86 on a pipeline with 6s P95 and $0.50/query. At 10k DAU that is $5k/day. Enterprise users churn past ~2s. Instrument embedding, search, rerank, and generation separately — the reranker is often the surprise (800ms–1.5s).

Redis’s production RAG metrics piece makes the same split: retrieval quality vs generation fidelity vs system reliability (latency/cost). Retrieval can be ~45% of time-to-first-token. Five excellent chunks beat twenty mediocre ones on both quality and cost. ([redis.io](https://redis.io/blog/rag-metrics/))

**Implication for you:** StockWise and Research Pod should grow an eval contract before they grow more agents.

### Problem B — Multi-agent systems fail structurally

Berkeley’s MAST study (*Why Do Multi-Agent LLM Systems Fail?*, NeurIPS 2025 / [arXiv:2503.13657](https://arxiv.org/abs/2503.13657), [sky.cs.berkeley.edu/project/mast](https://sky.cs.berkeley.edu/project/mast/)) annotated 150+ traces across popular MAS frameworks and found **14 failure modes** in three buckets:

- **Specification / system design** — vague roles, missing termination, bad tool schemas.
- **Inter-agent misalignment** — derailment, ignoring other agents, conversation reset.
- **Task verification** — premature stop, or “done” with a wrong artefact.

ChatDev hit only 33% correctness on their ProgramDev set. Prompt tweaks did not erase the taxonomy — the failures need better design (contracts, verification, topology), not more agents.

A 2026 production write-up of what actually survived is blunt: extra agents that do not bring a **new exogenous signal**, a **better interface**, or **non-redundant review** mostly rearrange the same information. Research-style orchestration can burn ~15× the tokens of a chat. In LangGraph, poisoning a hub node produced 100% system-wide failure vs 9.7% from a leaf. ([medium.com](https://medium.com/@Micheal-Lanham/multi-agent-in-production-in-2026-what-actually-survived-f86de8bb1cd1))

Openlayer’s 2026 failure-mode note adds the operational picture: malformed tool payloads propagate; retry loops eat the context window; a hallucinated intermediate becomes corrupted input for every downstream agent. ([openlayer.com](https://www.openlayer.com/blog/ai-agent-failure-modes-tool-calling-loops-propagation))

**Implication:** LangGraph recursion limits are a backstop, not a control. You want no-progress guards (same tool + args + error hashed, halt at 2–3 repeats), per-run dollar ceilings, step limits, and duration circuit breakers. Forensic write-ups of runaway loops (four agents, “schema drift in progress,” ~$47k over 11 days) all share the same missing infrastructure: no per-run budget, no loop detector, health checks that believed a truthful status string. ([clyro.dev](https://clyro.dev/blog/the-47k-loop-a-complete-forensic-analysis/), [policylayer.com](https://policylayer.com/attacks/runaway-tool-loops))

### Problem C — Knowledge is not static (this is your Graph RAG edge)

Vector RAG assumes a snapshot. Agents and enterprises do not.

- **Microsoft GraphRAG** (hierarchical Leiden communities + global summaries) is strong on corpus-level “sensemaking” and weak on *change*: indexing is an LLM-over-the-whole-corpus cost wall, and updates often mean rebuild. The repo itself warns that indexing is expensive and that the code is a methodology demo, not an official Microsoft product. (~35.5k stars, MIT) ([github.com/microsoft/graphrag](https://github.com/microsoft/graphrag), [microsoft.github.io/graphrag](https://microsoft.github.io/graphrag/))
- **LightRAG** (EMNLP 2025 lineage) exists because of that wall: dual-level retrieval + **incremental** graph updates, Neo4j / Qdrant / Postgres backends. Defaults are in-memory files — not production. (~38.8k stars, MIT, 233 open issues) ([github.com/HKUDS/LightRAG](https://github.com/HKUDS/LightRAG))
- **Graphiti (Zep)** is the 2026 answer when facts have a timeline: bi-temporal edges (`t_valid`, `t_invalid`), invalidate without delete, hybrid vector + BM25 + graph walk, Neo4j / FalkorDB / Neptune. This is agent *memory*, not batch GraphRAG. Zep Community Edition was deprecated; what you self-host is Graphiti + a graph DB. (~29.9k stars, Apache-2.0) ([github.com/getzep/graphiti](https://github.com/getzep/graphiti), [neo4j.com/blog](https://neo4j.com/blog/developer/graphiti-knowledge-graph-memory/))

**Implication:** Research Pod (arXiv Graph RAG on Neo4j + Qdrant) should not copy Microsoft GraphRAG’s batch indexer. Incremental + temporal is the production problem. That is also a cleaner OSS contribution niche than “yet another vector RAG tutorial.”

### Problem D — Garbage in: document parsing poisons every downstream metric

By 2026 the RAG quality conversation moved upstream. A scrambled table, a two-column PDF interleaved, or a dropped heading silently wrecks retrieval and faithfulness. The three names that dominate:

| Tool | What it is | When it wins |
|---|---|---|
| **Docling** (IBM → Linux Foundation AAIF) | Local layout + table models, `DoclingDocument`, Granite-Docling-258M | Privacy, air-gap, healthcare, you already run compute. ~64.7k stars, MIT |
| **Unstructured** | Broad-format ETL (25–60+ types), semantic element labels | Mixed enterprise corpora, not just PDF |
| **LlamaParse** | Hosted VLM parser, LlamaIndex-native | Zero infra, pay per page, messy scans |

The 2026 pattern is **triage**: easy digital PDFs on a cheap path, table-heavy / scanned docs on a VLM path. ([dreaming.press](https://dreaming.press/posts/2026-06-21-docling-vs-unstructured-vs-llamaparse.html), [vstorm.co](https://vstorm.co/llamaindex/top-10-document-parsing-services-for-rag-pipelines-and-llm-applications/))

### Problem E — Eval and observability are two different jobs

Mature 2026 teams run **one offline gate + one production tracer**, not one SaaS that pretends to be both. ([agentscamp.com](https://agentscamp.com/guides/evaluation/best-llm-eval-tools-2026), [genai.qa](https://genai.qa/blog/promptfoo-vs-deepeval-vs-ragas/))

| Job | OSS that actually gets used | Role |
|---|---|---|
| RAG-specific metrics | RAGAS (`vibrantlabsai/ragas`, Apache-2.0, ~15.3k) | Faithfulness, context precision/recall. Last release v0.4.3 (Jan 2026); slower cadence than DeepEval — use it, don’t bet your contribution career on it as the only home. |
| CI / pytest gates | DeepEval (~17.6k, Apache-2.0, pushed daily) | “Pytest for LLMs.” G-Eval, hallucination, agent metrics. |
| Prompt regression + red team | Promptfoo (~24.2k, MIT) | YAML matrix, jailbreak plugins, multi-provider. |
| Production traces, datasets, online judges | Langfuse (~33.1k; self-host still available after ClickHouse acquisition Jan 2026 — **verify current license**, GitHub reports `Other`) | The open default when you want to own traces. |
| OTel-native traces | Arize Phoenix (~11.0k) | Notebook → self-host; confirm license (sources disagree ELv2 vs Apache). |

The production loop that actually works: Langfuse captures traces → sample 1–5% → score with DeepEval/RAGAS → write scores back onto traces → promote failures into the golden set → Promptfoo/DeepEval gate the PR.

The March 2026 arXiv *LLM Readiness Harness* paper is the academic version of the same idea: quality without cost, latency, groundedness, and **policy as a hard CI gate** is not readiness. ([arXiv:2603.27355](https://www.arxiv.org/pdf/2603.27355))

### Problem F — Agents need memory that can be wrong later

Mem0 (~63.2k, Apache-2.0) is the drop-in memory layer (vector + optional graph). Letta / MemGPT (~24.2k, Apache-2.0, only 43 open issues) is a *runtime* where the agent pages its own memory. Graphiti is the temporal graph engine. They are not interchangeable. ([digitalapplied.com](https://www.digitalapplied.com/blog/open-source-agent-memory-mem0-letta-zep-compared))

For Research Pod and StockWise, Graphiti-shaped memory (entities change, old facts stay historically true) matches Graph RAG better than “stuff more chat into a vector store.”

### Problem G — Production needs a gateway, not 12 SDKs

LiteLLM (~56.3k stars, 4.9k open issues — noisy but real) is the open AI gateway: OpenAI-compatible API, 100–140+ providers, virtual keys, spend caps, fallbacks, Langfuse callbacks. The standard 2026 platform stack is **LiteLLM (policy + routing + budgets) + Langfuse (traces + evals)**. That is how you stop the $47k loop at $10. ([github.com/BerriAI/litellm](https://github.com/BerriAI/litellm), [oss-llmops-stack.com](https://oss-llmops-stack.com/docs))

### Problem H — Healthcare / regulated RAG is a different product

You have healthcare experience. Do not treat generic RAGAS as enough. 2026 clinical RAG eval cares whether the **citation pointer resolves to a live FDA label / CMS policy / guideline section**, whether the model **abstains** without evidence, and whether retrieval logs are audit records under HIPAA §164.312(b). Embeddings of PHI are still PHI by default (inversion / nearest-neighbour re-identification). Vector stores need the same controls as the source system. ([futureagi.com](https://futureagi.com/blog/best-healthcare-rag-evaluation-2026/), [areebi.com](https://www.areebi.com/resources/blog/hipaa-clinical-ai-protected-health-information-2026-playbook))

This is a *problem space* more than a single famous OSS repo. The leverage is: contribute **citation / faithfulness / abstention evaluators** into DeepEval, RAGAS, Langfuse, or Haystack, with clinical-shaped fixtures (public guidelines, not PHI).

### Problem I — Coding agents are the most-used agents on earth

If you want “useful to people” with a straight face: developers already live in these tools daily.

| Project | Stars (Aug 2026) | Shape |
|---|---|---|
| OpenHands (ex-OpenDevin) | ~83.9k, MIT | Sandboxed autonomous SWE agent, UI + SDK, SWE-bench native |
| Cline | ~66.1k, Apache-2.0 | VS Code / CLI, human-in-the-loop plan/act |
| Goose (Block → AAIF / Linux Foundation) | ~52.8k, Apache-2.0 | General MCP-driven agent, not only code |
| Aider | ~45k class | Git-native surgical edits |
| Continue | ~33k | **Acquired by Cursor; original repo read-only — do not start here** |

OpenHands + Cline are production-grade and contribution-hungry. They are TypeScript-heavy. Contribute if you want to learn agent sandboxes, tool policies, and evals on SWE-bench — not if you only want Python RAG.

---

## 2. How we pick projects (filters)

A repo makes this list only if it passes all five:

1. **Someone who is not an AI engineer would miss it if it disappeared** (product or infrastructure, not a tutorial).
2. **Shipped to production in 2026**, not a 2023 demo.
3. **Permissive or clearly self-hostable**, recently pushed (we dropped Flowise — **archived**).
4. **A path for an experienced engineer** — integrations, evals, drivers, docs-with-repro — not only “fix a typo.”
5. **Compounds your stack** (LangGraph, LlamaIndex, Neo4j, Qdrant, FastAPI, evals) *or* deliberately fills the gap (observability, parsing, gateway, memory).

**Explicitly not first homes:** LangChain core (too crowded), vLLM / SGLang (CUDA/systems unless you want that career), n8n (automation, not your specialty), Dify as first PR (152k stars, TS platform, slow feedback), Continue (read-only), Flowise (archived).

---

## 3. The shortlist (ranked for you)

### Home bases — pick two, not eight

These are where you should know the maintainers’ names, join Discord, and land more than one PR.

#### 1. Langfuse — `langfuse/langfuse` (~33.1k ★)

**Problem it solves:** A–E (traces, datasets, online evals, prompt versions).  
**Why you:** Your README already says this is next. It is the open production default next to LangSmith. Integrates with LangChain, LlamaIndex, LiteLLM, OpenTelemetry.  
**How an experienced engineer contributes:** Python/JS SDK gaps, LlamaIndex / LangGraph instrumentation examples, eval-from-trace recipes, dataset APIs, missing metadata on retrieval spans (chunk ids, scores, rerank). Read [CONTRIBUTING.md](https://github.com/langfuse/langfuse/blob/main/CONTRIBUTING.md) — **open an issue before any non-trivial PR**.  
**Watch-out:** License on GitHub is `Other` after the ClickHouse acquisition. Confirm the self-host terms before you build a company on it; contributing is still the right education.

#### 2. Graphiti — `getzep/graphiti` (~29.9k ★, Apache-2.0)

**Problem it solves:** C + F (temporal knowledge, agent memory, incremental graph).  
**Why you:** You already think in Neo4j + Qdrant. Graphiti is the production-shaped version of “Graph RAG that does not die when a paper is updated.” Built-in MCP server. CONTRIBUTING even has “adding a graph driver.”  
**How you contribute:** Qdrant hybrid retrieval experiments, Neo4j driver edge cases, evals for fact invalidation, better incremental ingest from arXiv/PDF (pair with Docling). This can feed Research Pod directly.  
**Watch-out:** Do not confuse Graphiti (engine) with Zep Cloud (product). Self-host = Graphiti + Neo4j/FalkorDB.

#### 3. Official Neo4j GraphRAG — `neo4j/neo4j-graphrag-python` (~1.25k ★, ~26–39 open issues)

**Problem it solves:** C, with a *small* codebase where one PR is visible.  
**Why you:** First-party library, Qdrant extra already exists, hybrid + graph-walk retrievers, Text2Cypher, KG builder pipeline. CLA required. uv + Ruff + Mypy + e2e against Neo4j/Weaviate/Qdrant.  
**How you contribute:** Qdrant retriever hardening, Docling ingest into `SimpleKGPipeline`, LangGraph example, evaluation helpers, NVIDIA NIM / Bedrock LLM wrappers. High signal-to-noise. Best “I want maintainers to know my name” Python repo on this list.

#### 4. Haystack + integrations — `deepset-ai/haystack` (~26.2k ★, Apache-2.0) and `deepset-ai/haystack-core-integrations` (~202 ★, 82 issues)

**Problem it solves:** A + B, with *typed pipelines* and production retrieval primitives (BM25, hybrid, rerank) that LangChain tutorials skip.  
**Why you:** Explicitly built for production RAG/agents; Agent lifecycle hooks (`before_llm`, `before_tool`, `on_exit`) for guardrails and cost; Agent Pack (deep research, advanced RAG). **Contributor culture is the best of the big Python frameworks** — CONTRIBUTING.md tells you which issues are *internal* vs open, and points at a contributions board. Integrations repo is the right door (Qdrant store, Neo4j, eval, Docling).  
**How you contribute:** Do **not** send drive-by PRs to issues marked handled internally. Use [good first issue](https://github.com/deepset-ai/haystack/contribute) and `haystack-core-integrations`. A Qdrant or eval integration you actually run on Research Pod will get reviewed.

### Stretch production systems (third repo, after the homes are real)

| Repo | Stars | Why it is on the list | Your likely first contribution |
|---|---|---|---|
| **Docling** `docling-project/docling` | ~64.7k, MIT | Parsing is the RAG bottleneck; LF project; LangChain/LlamaIndex/Haystack integrations | Table-heavy arXiv/10-K fixtures, LlamaIndex/Haystack connector fixes, RAG eval showing parse quality → faithfulness |
| **DeepEval** `confident-ai/deepeval` | ~17.6k, Apache-2.0 | CI quality gates; active daily | Custom faithfulness/citation metrics for Graph RAG; LangGraph callback; golden-set examples from StockWise (public data only) |
| **LlamaIndex** `run-llama/llama_index` | ~51.6k, MIT | You already use it; PropertyGraph, Qdrant, evals | Integration package (Qdrant/Neo4j/NIM), GraphRAG eval notebook, bugfix you hit in StockWise |
| **LangGraph** `langchain-ai/langgraph` | ~39.6k, MIT, 34.5M PyPI downloads/mo | Production default for stateful agents (Klarna, Replit, Elastic, Uber, LinkedIn cited in 2026 roundups) | Durable execution examples, no-progress middleware, eval harness — comment on the issue first ([docs.langchain.com contributing](https://docs.langchain.com/oss/python/contributing/code)) |
| **RAGFlow** `infiniflow/ragflow` | ~87.9k, Apache-2.0 | End-to-end production RAG *product* (parse + retrieve + agent), GitHub fastest-growing OSS by contributors in Octoverse 2025 | Deep document parse bugs, Qdrant backend, eval — read [contributing](https://ragflow.io/docs/contributing). Heavier (Go + Python + UI). |
| **Qdrant** `qdrant/qdrant` + `qdrant/qdrant-client` | ~34.0k Rust / ~1.3k Python | You already run it; hybrid sparse+dense is a 2026 production default | Client API gaps, hybrid search examples, payload-filter + Graph RAG recipes. Core is Rust — client/docs are the Python path. |
| **Instructor** `instructor-ai/instructor` | ~13.7k, MIT, **30 open issues** | Structured outputs; Tensoria: naive JSON still fails 5–15% in prod | Schema retries, graph-extraction schemas for Research Pod entities |
| **Pydantic AI** `pydantic/pydantic-ai` | ~19.3k, MIT | Type-safe Python agents without LangChain weight | Tool schema / eval / MCP extras |
| **Promptfoo** `promptfoo/promptfoo` | ~24.2k, MIT | Red-team + prompt regression in CI | RAG assertions, Graph RAG test cases (TS/YAML; still valuable) |
| **LiteLLM** `BerriAI/litellm` | ~56.3k | Gateway + spend caps | Langfuse callback bugs, budget middleware. Issue tracker is *very* noisy (4.9k) — only contribute a bug you hit. |
| **MCP Python SDK** `modelcontextprotocol/python-sdk` | ~24.0k, MIT | Tooling protocol every agent will speak | Auth, Streamable HTTP, eval of tool schemas — Graphiti already ships an MCP server |

### Useful-to-people products (if you want users, not just libraries)

| Product | Stars | Notes |
|---|---|---|
| **Open WebUI** | ~148.7k | Local ChatGPT; built-in RAG. License `Other` — read it. Huge, competitive. |
| **LibreChat** | ~42.0k, MIT | Multi-provider chat used by real teams. TS. |
| **AnythingLLM** | ~64.7k, MIT | Desktop/workspace RAG for non-engineers. |
| **Dify** | ~152.3k | Visual RAG + agents; Docker Compose production stack. Contribute *after* you have OSS reps. |
| **OpenHands** | ~83.9k, MIT | If you want coding-agent depth. TS + sandbox. |
| **Cline** | ~66.1k, Apache-2.0 | Same, IDE-shaped. |
| **Goose** | ~52.8k, Apache-2.0 | Linux Foundation general agent + MCP. |
| **CopilotKit** | ~36.7k, MIT | Agent-native UI — how real users *see* your LangGraph. |
| **Mem0** | ~63.2k, Apache-2.0 | Memory layer; easier than Graphiti, less graph-native. |
| **Letta** | ~24.2k, Apache-2.0 | Full memory runtime; small issue count (43) = maintainers can actually review you. |
| **LightRAG** | ~38.8k, MIT | Incremental GraphRAG; production backends documented (Neo4j, Qdrant). |
| **smolagents** | ~28.8k, Apache-2.0 | Hugging Face code-agents; last push July 2026 — check pulse before investing. |
| **Microsoft Agent Framework** | ~12.8k, MIT | AutoGen + Semantic Kernel successor. Enterprise .NET/Python. |
| **Google ADK** | ~21.1k, Apache-2.0 | If you want GCP-native agents. |
| **OpenAI Agents SDK** | ~28.6k, MIT, **19 open issues** | Clean, small surface; PRs may be slow if they prefer internal work. |

### Study-but-don’t-make-your-home (yet)

- **Microsoft GraphRAG** — read the paper and the cost warning; contribute only if you are improving evals or lazy/incremental variants, not “run this on my corpus” issues.
- **FalkorDB GraphRAG-SDK** (~988 ★) — interesting GraphRAG-Bench numbers; small community.
- **vLLM / SGLang** — the real inference layer. Contribute if you want a systems AI career; otherwise consume them via LiteLLM.
- **RAGAS** — still the vocabulary of RAG metrics; slower repo pulse in 2026 than DeepEval. Use it in your eval stack; prefer DeepEval/Langfuse for contribution time.
- **CrewAI** (~57k) — fine for prototypes; weaker production control story than LangGraph/Haystack.

---

## 4. The training plan (90 days)

Treat StockWise and Research Pod as your **lab**, not as the portfolio. The portfolio becomes: *I found a production failure, measured it, fixed it upstream, and brought the pattern home.*

### Days 1–14 — Instrument what you already built

Do this before any OSS PR.

1. Put **LiteLLM proxy** in front of every model call (StockWise + Research Pod) with a **hard per-run spend cap**.
2. Send traces to a local **Langfuse**. Every span: query, retrieved chunk ids + scores, rerank scores, tokens, latency, final answer, tool name/args/error.
3. Build a **golden set of 50–100 real queries** (10-K questions, arXiv questions you actually asked). Score retrieval and generation separately (context recall vs faithfulness).
4. Add **DeepEval in CI** on that set. A faithfulness drop fails the build.
5. Add a **no-progress guard** on LangGraph tool loops (hash `(tool, args, error)`, stop at 3).

**Done when:** you can debug a wrong answer in 30 seconds from a trace, and a prompt change cannot merge if it tanks faithfulness.

### Days 15–45 — First home repo

Pick **Langfuse** *or* **Graphiti** *or* **neo4j-graphrag-python** (my order if you want one default: **Graphiti** if Research Pod is the main lab; **Langfuse** if you want the career-shaped gap closed fastest; **neo4j-graphrag-python** if you want the highest chance of a merged PR).

Rules:

- Use the library in anger in *your* lab first.
- File a **high-quality issue** with a minimal repro, traces, and expected vs actual. That issue *is* a contribution.
- Comment on the issue with your intended approach before coding (LangChain/Haystack/Langfuse all ask for this; ignore it and you waste a week).
- First PR: the bug or integration you actually hit. Second PR: tests/docs so the next person does not hit it.
- Join Discord. Answer two questions a week. This is how maintainers learn your name.

### Days 46–75 — Second home + a production primitive

Add **Haystack integrations** or **Docling** or **DeepEval**, whichever hurt you in the lab.

Ship one of these primitives in public (blog or repo section, with numbers):

- Hybrid dense + sparse + rerank vs naive vector, on *your* corpus, with latency and cost.
- Graphiti incremental update vs rebuild GraphRAG, on a week of new arXiv papers.
- Parse-quality ablation: PyMuPDF vs Docling vs LlamaParse → faithfulness.
- Agent loop: with vs without spend cap + no-progress guard (cost, steps, success).

Numbers beat architecture diagrams.

### Days 76–90 — Close the loop

1. Promote 20 production-sampled failures into the golden set.
2. Land or revise OSS PRs from the issues you opened.
3. Write one technical note: “What broke in Research Pod, what I changed upstream, what the evals said.”
4. Decide the next 90 days: either go deeper on Graphiti/Haystack **or** pick a user-facing product (LibreChat / RAGFlow / OpenHands) now that you have merge history.

---

## 5. How to contribute so PRs merge

This is the experienced-engineer playbook. Junior contributors send unsolicited refactors. You will not.

1. **Use it in production-shaped work first.** Maintainers can smell a drive-by.
2. **Read CONTRIBUTING.md and the issue labels.** Haystack will reject PRs on “handled internally” issues. Langfuse wants an issue before significant changes. Neo4j wants a signed CLA and CHANGELOG.
3. **Small PRs, tests included, `make lint && make test` green locally.** Conventional commits where they ask.
4. **Integrations beat core rewrites** for your first three PRs: Qdrant, Neo4j, Docling, NIM, Langfuse callbacks, DeepEval metrics.
5. **Evals and failure fixtures are under-supplied.** A golden set of multi-hop Graph RAG questions with citations is more valuable than another README badge.
6. **Do not lead with AI-generated dumps.** Haystack explicitly asks for a disclaimer if a PR is fully AI-generated. Use the agent to draft; you own the design, the tests, and the review comments.
7. **Measure your OSS like a product:** merged PRs, issues triaged, Discord answers — not stars on your fork.

---

## 6. What “confident” looks like in 12 months

You can walk into a design review and defend:

- A **quality contract** (recall, precision, faithfulness, citation accuracy, freshness, p95, cost/query) with CI gates.
- Why Research Pod is Graphiti-shaped (incremental, temporal) rather than Microsoft GraphRAG-shaped (batch communities).
- Why StockWise agents do not do arithmetic themselves (you already have this instinct — keep it; add traces and loop guards).
- Why you added hybrid search + rerank *after* measuring, and what it cost in latency.
- Two merged PRs in repos other people deploy, plus a post-mortem that cites traces.

That is a stronger senior AI engineer signal than “I tried 12 frameworks.”

---

## 7. Recommended default path (if you want me to choose)

If we are not debating:

| Horizon | Focus |
|---|---|
| This month | Langfuse + DeepEval + LiteLLM spend caps on StockWise and Research Pod |
| This quarter | Graphiti (or neo4j-graphrag-python) as the Research Pod memory/graph layer; 2–3 upstream PRs |
| This half | Haystack-core-integrations *or* Docling, plus one public eval write-up with numbers |
| Optional parallel | Qdrant client examples; Instructor for KG extraction schemas |

Skip new agent frameworks until the lab has traces, a golden set, and a cost ceiling.

---

## Sources

Production RAG and evals

- https://tensoria.fr/en/blog/production-rag-failure-modes
- https://redis.io/blog/rag-metrics/
- https://www.braintrust.dev/articles/best-rag-evaluation-tools
- https://www.digitalapplied.com/blog/rag-system-metrics-recall-precision-faithfulness-2026
- https://www.arxiv.org/pdf/2603.27355
- https://agentscamp.com/guides/evaluation/best-llm-eval-tools-2026
- https://genai.qa/blog/promptfoo-vs-deepeval-vs-ragas/
- https://www.digitalapplied.com/blog/rag-anti-patterns-7-failure-modes-2026-engineering-guide

Multi-agent failures

- https://arxiv.org/abs/2503.13657
- https://sky.cs.berkeley.edu/project/mast/
- https://medium.com/@Micheal-Lanham/multi-agent-in-production-in-2026-what-actually-survived-f86de8bb1cd1
- https://www.openlayer.com/blog/ai-agent-failure-modes-tool-calling-loops-propagation
- https://clyro.dev/blog/the-47k-loop-a-complete-forensic-analysis/
- https://policylayer.com/attacks/runaway-tool-loops
- https://particula.tech/blog/stop-ai-agents-looping-same-tool-call-no-progress

Graph RAG and memory

- https://github.com/microsoft/graphrag
- https://microsoft.github.io/graphrag/
- https://github.com/HKUDS/LightRAG
- https://github.com/getzep/graphiti
- https://neo4j.com/blog/developer/graphiti-knowledge-graph-memory/
- https://github.com/neo4j/neo4j-graphrag-python
- https://dreaming.press/posts/2026-06-22-graphrag-vs-lightrag-vs-graphiti.html
- https://www.digitalapplied.com/blog/open-source-agent-memory-mem0-letta-zep-compared

Parsing, gateways, frameworks, products

- https://dreaming.press/posts/2026-06-21-docling-vs-unstructured-vs-llamaparse.html
- https://github.com/BerriAI/litellm
- https://oss-llmops-stack.com/docs
- https://github.com/langfuse/langfuse
- https://github.com/deepset-ai/haystack
- https://haystack.deepset.ai/
- https://github.com/infiniflow/ragflow
- https://ragflow.io/docs/contributing
- https://the-agent-report.com/2026/07/ai-agent-frameworks-comparison-2026-langgraph-crewai-autogen/
- https://langfuse.com/blog/2025-03-19-ai-agent-comparison
- https://docs.langchain.com/oss/python/contributing/code

Healthcare

- https://futureagi.com/blog/best-healthcare-rag-evaluation-2026/
- https://www.areebi.com/resources/blog/hipaa-clinical-ai-protected-health-information-2026-playbook

Coding agents

- https://dreaming.press/posts/aider-vs-cline-vs-openhands.html
- https://github.com/OpenHands/OpenHands
- https://github.com/cline/cline
- https://github.com/aaif-goose/goose
