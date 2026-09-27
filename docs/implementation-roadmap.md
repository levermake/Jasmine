# Jasmine implementation roadmap

This roadmap turns the [architecture](architecture.md) into independently reviewable build increments. Components and hardware profiles are specified in the [technology register](technology-register.md). The repository currently contains design documentation; milestones below are planned, not completed application features.

## 1 Definition of the first useful system

Jasmine's minimal viable cognitive system should let its owner upload a document, ask a question, receive a source-backed answer, retain a visible memory across sessions, correct that memory, and resume an interrupted task. It should know when the available evidence does not answer a question. A browser is the client; memory, inference, authorization, and task execution live on the backend.

The first release uses text, Markdown, and a deliberately limited set of document formats, one instruction model, one embedding model, PostgreSQL, and a bounded controller. Start with read-only retrieval and simple deterministic computation. Source viewing, retention controls, cancellation, and observable failure states are part of the first useful product.

No broad web crawling, arbitrary code execution, multi-agent debate, online weight updates, or automatic production self-modification is required for this milestone. Each introduces a separate evaluation and operational burden and should have a demonstrated task benefit before adoption.

## 2 Build stages and completion gates

Effort ranges are planning estimates for one experienced engineer working consistently with compatible hardware. They exclude research breakthroughs, procurement, and a broad public launch. Stages 0–3 together are approximately 5–10 engineering weeks; actual integration and corpus quality can move that substantially.

| Stage | Deliverable | Exit gate | Indicative effort |
| --- | --- | --- | --- |
| 0 Foundations | Reproducible development environments, schema migrations, model compatibility probe, evaluation fixtures | A clean setup loads the selected models; restart and migrations preserve a source and run; all artifact revisions are recorded | 0.5–1 week |
| 1 Evidence-grounded retrieval | Source ingestion, hybrid retrieval, local generation, evidence links, minimal browser | Annotated questions retrieve the right sources, citations resolve, unanswerable questions are handled explicitly | 1.5–3 weeks |
| 2 Durable memory | Working state, episodes, qualified claims, corrections, retention and deletion | Memory survives restart; correction and deletion propagate to derived retrieval records; interrupted work resumes consistently | 1–2 weeks |
| 3 Bounded tool reasoning | Typed plans, read-only tool registry, deterministic calculation/solver tools, action ledger | Plans stay within budgets; failures and duplicate delivery do not corrupt state; unsafe or unauthorized arguments are rejected | 2–4 weeks |
| 4 More modalities and shared operation | OCR/speech as required, authenticated users, independent workers, operational dashboards | Per-modality accuracy and end-to-end provenance pass; isolation, backup restoration, and load behavior are measured | 2–6 additional weeks, scope-dependent |
| 5 Evaluated learning | Skill promotion, offline adapter experiments, model/prompt registry, rollback | Measurable held-out gain without unacceptable regression, leakage, or resource growth | Several additional weeks per learning cycle |
| 6 Bounded research autonomy | Curriculum proposals, simulation feedback, transfer experiments, protected evaluator | Improvements reproduce on unseen tasks under fixed budgets and comparison baselines | Open-ended research; no capability guarantee |

Scaling work can run alongside stages 4–5 when real workloads justify it. Stage numbers do not imply that all later features must be implemented.

## 3 Concrete engineering backlog

### Increment A Persist sources and runs

Implement PostgreSQL migrations for tenants, principals, source metadata/versions, observations, runs, jobs, and outbox records. Add indexes for ownership, run state, readiness, and lease selection. Apply tenant constraints and deletion markers from the start.

Implement a storage interface with local-volume support, durable write completion, digest checking, staged uploads, and authorized reads. Model loaders run in a separate process. Add `/health/live` and `/health/ready` with distinct semantics: a live API is not necessarily ready to serve model-backed requests.

Done means a submitted run remains inspectable after process restart, source staging failure leaves no falsely ready source, and two concurrent workers cannot both commit the same state transition. This is the first vertical slice; there is no need to build every memory table before it works.

### Increment B Ask one document a question

Implement text/Markdown normalization with source-location mapping, chunking, local embeddings, lexical search, and vector search. Store the full embedding-generation identity. Add a prompt builder with a hard tokenizer-checked budget.

Implement a model adapter returning text, token usage where available, model identity, and structured error states. Keep provider-specific response parsing inside this adapter. Make evidence IDs part of the answer contract; resolve them server-side to actual source locations.

Build a browser with upload status, a question box, answer display, clickable source excerpts, task status, and cancellation. Verify document selection is enforced server-side. Do not create a decorative cognitive dashboard before these interactions work.

### Increment C Measure retrieval and answer support

Create a labeled corpus with answerable, unanswerable, contradictory, time-sensitive, and exact-identifier queries. Record which passages support each answer. Compare lexical-only, dense-only, and fused retrieval before tuning chunk size or introducing a reranker.

Implement evaluation reports with retrieval recall, citation support, answer correctness, abstention, latency, and token usage. Use executable checks for IDs and source existence, and human-reviewed labels for semantic support. A model grader may assist triage but must not be the sole ground truth.

### Increment D Add durable cognitive memory

Integrate LangGraph checkpoints with runs. Keep episode evidence separate from derived summaries. Add provisional claim extraction and explicit correction endpoints. Create a memory viewer showing source, date, status, and how to correct or delete an entry.

Add consolidation as a low-priority job, with lineage tracking and source-authorization inheritance. Recovery tests should interrupt both before and after a checkpoint. Implement deletion across source storage, indexes, claims, summaries, checkpoints, and caches, with tombstones guarding against delayed workers.

### Increment E Add bounded tools

Start with `search_sources`, `read_source_span`, and a deterministic calculation tool accepting a constrained structured expression. Add a narrow OR-Tools or Z3 capability when a concrete task needs it. Do not pass arbitrary strings to a host shell or language evaluator for basic arithmetic.

Implement tool schemas, permissions, argument validation, deadlines, operation IDs, and durable outcome records. Test one-step execution before adding multi-step planning. Store verification summaries and results so an interrupted run can resume without pretending to know an external action's outcome.

Only add write-capable tools with a concrete authorization flow. The request must show the exact effect and destination; approval is invalidated by a changed argument digest. Include a denial path and a reconciliation path for timeouts.

### Increment F Prepare shared operation

Deploy Keycloak and verify OIDC/session behavior. Test isolation across the full path, including source previews, model/retrieval caches, checkpoints, and logs. Add metrics, structured traces, capacity limits, backups, and restore procedures.

Move parsing and embedding to independent pools if they affect interactive latency. Introduce a distributed broker, object store, or dedicated vector engine only after profiling identifies the bottleneck and the migration preserves the same contracts.

### Increment G Learn from outcomes

Build a dataset from vetted corrections and independently verified outcomes. Separate consented training examples from ordinary memory. First test improved prompts, retrieval settings, or reusable tool workflows; these are cheaper to inspect and reverse than weight updates.

If parameter changes are justified, train a PEFT adapter offline. Record base checkpoint, adapter, data split, training settings, evaluation, and hardware. Compare against the unchanged baseline on old and new tasks. Promote gradually and keep rollback available. Do not let the training process alter the evaluator or its private holdout.

## 4 Suggested code ownership and repository layout

These paths describe the implementation structure to create in stages 0–3; they are not claims that application files already exist.

| Path | Responsibility |
| --- | --- |
| `apps/web/` | Browser UI, generated API client, accessible status and evidence views |
| `services/api/` | HTTP endpoints, sessions, authorization, request validation |
| `services/worker/` | Job leasing, LangGraph execution, cancellation, recovery |
| `services/inference/` | Model-runtime configuration and compatibility probes |
| `packages/contracts/` | Shared schemas for observations, plans, evidence, tools, and events |
| `packages/memory/` | Source/episode/claim repositories, correction, consolidation, deletion lineage |
| `packages/retrieval/` | Index generations, lexical/vector fusion, evidence packing |
| `packages/tools/` | Tool registry, deterministic tools, authorization and execution adapters |
| `packages/evaluation/` | Fixtures, rubrics, regression runners, result reports |
| `migrations/` | Database schema changes and rollback/recovery notes |
| `deploy/` | Pinned container configuration, service health checks, backup/restore operations |
| `docs/` | Architecture, decisions, component register, operational guidance |

Start with a single Python workspace for application packages and separate model environments where necessary. Internal typed interfaces should be usable in process before becoming network APIs. Do not create empty service implementations merely to fill the directory structure.

## 5 Acceptance suite

The initial numbers below are proposed engineering targets, not achieved results or universal standards. Freeze the evaluation dataset before tuning. Track each task category separately so an aggregate score cannot hide a major failure.

| Test | Proposed gate |
| --- | --- |
| Retrieval | At least 90% recall@10 on a reviewed set of at least 100 answerable corpus questions, with annotated relevant passages |
| Citation validity | Every emitted citation resolves to an existing, permitted source version and span |
| Citation support | At least 95% of sampled cited factual claims are supported under a human-reviewed rubric |
| Answer correctness | At least 85% correct on the initial answerable set; report partially correct and unsupported answers separately |
| Useful abstention | At least 90% of at least 50 deliberately unanswerable cases are handled without an unsupported factual answer; report answerable-case over-abstention too |
| Contradictions and corrections | All scripted temporal/correction fixtures return the expected current and historical states |
| Isolation | Zero cross-tenant disclosures in the defined adversarial test suite; this is a release gate, not proof that no vulnerability exists |
| Restart/replay | Forced crashes at selected boundaries preserve state and avoid duplicate committed database effects |
| External action uncertainty | Timeout-after-possible-effect cases enter reconciliation or `outcome_unknown`; no blind duplicate action |
| Cancellation and budgets | Runs stop scheduling new work after cancellation or budget exhaustion; completed effects remain visible |
| Deletion | New retrieval cannot expose a deleted source; descendants and caches are removed; delayed jobs do not resurrect it |
| Injection handling | Untrusted documents cannot grant tool permissions or change resource scope in the fixed attack fixtures |
| Latency | Establish a measured hardware-specific target; an initial accelerated text target is p95 under 15 seconds for a short verified answer at one active generation |
| Restore | Restore a chosen backup on a clean instance and verify source checksums, permissions, versions, and deletion state |

Report counts and uncertainty intervals where useful. A small fixture set cannot support claims about universal reliability. Keep some source/task families entirely outside prompt development. Repeat stochastic task evaluations and record sampling settings.

Use ordinary unit tests for policy, data transformations, and state transitions; integration tests for PostgreSQL, queues, model adapters, and storage; and browser tests for evidence access and cancellation. Model-quality tests belong in a separate, resource-bounded suite. Documentation-only changes do not require downloading or running large models.

## 6 Browser behavior and observability

Expose five initial views or panels: conversation, source library, memory, active task details, and settings. Task details show the current operation, remaining budget, requested action authorization, outcome, and evidence. Settings include retention choices and enabled modalities.

A source displays ingestion status and parse failures before it becomes searchable. A memory entry shows whether it is an observation, a provisional interpretation, or a confirmed preference. A response provides direct supporting source spans and clearly identifies gaps.

Support keyboard navigation, readable contrast, and screen-reader status updates. Stream progress through SSE; add bidirectional audio transport only when spoken interaction is a tested requirement. Preserve model/provider details in diagnostic views where they help debugging, not as obstacles in ordinary user flows.

## 7 From useful complexity to research

Useful complexity appears when independently tested components compose: a corrected memory changes retrieval, retrieval informs a plan, a tool produces new evidence, and verification produces a reusable skill. The relevant question is whether that composition improves task outcomes under a controlled resource budget.

Use ablations: model alone; model with retrieval; retrieval plus durable memory; memory plus deterministic tools; and the same system with a promoted skill. Measure task success, transfer, correction retention, latency, and tokens per success. Failure on a harder holdout should reduce the claim, not trigger an unbounded agent loop.

Bounded self-directed learning becomes a concrete engineering feature once Jasmine can identify a failure type, propose permitted practice, get independent feedback, and submit a candidate improvement to a protected gate. General autonomous discovery and reliable open-ended self-improvement remain research questions.

## 8 Decisions needed before deployment

The architecture can proceed now. Before provisioning or exposing an instance, select the actual hardware, local-only versus hosted access, first corpus, intended languages, maximum concurrent active generations, retention policy, and whether the open-source requirement includes hardware firmware and complete model ancestry.

Those choices determine exact runtime versions and capacity; they do not require redesigning the memory and control boundaries. The next executable change should implement Increment A and a verified local model probe, followed by the one-document question flow in Increment B.
