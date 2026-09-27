# Jasmine technology register

Primary project documentation and model cards checked on 27 September 2026. This is a selection and integration register, not a tested dependency lockfile. The first build must pin exact releases, model revisions, and container digests and pass the compatibility gates below. Do not deploy floating `latest` tags.

Read this alongside the [architecture](architecture.md) and [implementation roadmap](implementation-roadmap.md).

## 1 Meaning of open source in this design

Jasmine's required application code and services use open-source licenses and can be self-hosted. There is no required commercial model endpoint, hosted vector database, managed agent service, or paid observability backend. Hardware, electricity, storage, and operations still cost money.

Four separate questions must be recorded for each component:

1. Is the software source available under an open-source license?
2. Are model weights available, and under which terms?
3. Are training code, data information, datasets, and recipes available? Public weights alone do not establish reproducibility.
4. Does the chosen deployment introduce proprietary drivers, services, or model dependencies?

The [Open Source AI Definition](https://opensource.org/ai/open-source-ai-definition) treats code, parameters, and data information separately. Full end-to-end reproducibility is a stronger practical requirement than merely downloading a checkpoint; it also depends on available data and compute. Do not describe all public datasets as uniformly licensed, or infer a model's terms from its Python package.

Prefer Olmo and Nomic for the core because their projects publish training resources as well as weights. Preserve upstream notices and record dataset provenance. The optional vision extension below has a specific inherited-model openness limitation. It is excluded from any claim of a completely reproducible model ancestry.

Jasmine's existing repository license is MIT. That does not relicense dependencies, data, or weights. Maintain a software bill of materials and a model/data manifest when implementation starts. An optional copyleft component remains open source but has its own distribution obligations.

## 2 Smallest useful stack

| Layer | Project and primary source | License scope | Why it fits and how it connects |
| --- | --- | --- | --- |
| Browser UI | [React](https://github.com/react/react), [Vite](https://github.com/vitejs/vite) | MIT for both projects | Build a static TypeScript client with chat, source previews, memory inspection, task progress, and cancellation. Serve assets through the same origin as the API. |
| API and contracts | [FastAPI](https://github.com/fastapi/fastapi), [Pydantic](https://github.com/pydantic/pydantic) | MIT for both | Typed HTTP endpoints and schema validation integrate directly with Python inference and retrieval code. Export OpenAPI for the browser client. |
| Cognitive state machine | [LangGraph OSS](https://github.com/langchain-ai/langgraph) | MIT | Use the library and PostgreSQL checkpoint integration inside Jasmine's worker. Persist transitions and interrupts. No LangSmith or hosted agent platform is required. |
| Durable memory and jobs | [PostgreSQL](https://www.postgresql.org/), [license](https://www.postgresql.org/about/licence/) | PostgreSQL License | Transactions, JSONB, full-text search, row security, and a leased job table cover the initial workload. Keep one authoritative state store. |
| Semantic retrieval | [pgvector](https://github.com/pgvector/pgvector) | PostgreSQL License | Store embeddings next to source metadata; begin with exact search and add HNSW when measured. Combine with PostgreSQL lexical ranks in application code. |
| Embedding runtime | [Sentence Transformers](https://github.com/huggingface/sentence-transformers) | Apache-2.0 | Run the selected embedding model locally in its own pinned environment; expose a narrow embed interface. |
| Embedding model | [nomic-ai/nomic-embed-text-v1.5](https://huggingface.co/nomic-ai/nomic-embed-text-v1.5) | Apache-2.0 model card | Use 768-dimensional English embeddings, documented task prefixes, and consistent normalization. Its published training resources support auditability. |
| Language model | [allenai/Olmo-3-7B-Instruct](https://huggingface.co/allenai/Olmo-3-7B-Instruct) | Apache-2.0 model card | Candidate for the first English instruction model. Ai2 links pretraining, post-training, and evaluation resources. Accept it based on Jasmine's evidence and tool-use tests, not its name or benchmark ranking. |
| Inference server | [vLLM](https://github.com/vllm-project/vllm) | Apache-2.0 | Batched inference behind a private HTTP service. Its supported-model list includes Olmo 3 through the Transformers modeling backend. CPU development can use an isolated Transformers/PyTorch service instead. |
| Document perception | [Docling](https://github.com/docling-project/docling), [Tesseract](https://github.com/tesseract-ocr/tesseract) | MIT code for Docling; Apache-2.0 for Tesseract | Parse documents locally, preserve layout/source locations, and OCR scanned pages. Audit selected model assets separately. Start with text/Markdown before enabling complex formats. |
| Local process/container management | [Podman](https://github.com/podman-container-tools/podman) | Apache-2.0 | OCI containers without a required proprietary desktop product. Use persistent volumes and rootless execution where the workload allows it. |
| HTTPS and static assets | [Caddy](https://github.com/caddyserver/caddy) | Apache-2.0 | One web origin for browser assets and API routing. Keep inference and storage ports private. |

For local single-owner development, bind the app to loopback. Add real authentication before LAN or public exposure. Start source storage as an application-owned volume with checksums, access-controlled reads, and backups. A separate object-store service is not needed on the first workstation.

Application attention, belief revision, evidence checking, memory policy, and skill promotion are Jasmine code. No library in this table supplies those behaviors automatically.

## 3 Additions with explicit triggers

| Capability or scaling trigger | Project | License | Integration and qualification |
| --- | --- | --- | --- |
| Shared or hosted access | [Keycloak](https://github.com/keycloak/keycloak) | Apache-2.0 | OIDC identity provider. The API still checks resource permissions and tenant membership. |
| Independently managed action policies | [Open Policy Agent](https://github.com/open-policy-agent/opa) | Apache-2.0 | Evaluate principal, resource, action, and context outside model prompts. Start with a tested Python policy module; extract to OPA when policy complexity justifies it. |
| Constraint solving | [Z3](https://github.com/Z3Prover/z3) | MIT | Use typed symbolic problems for satisfiability and invariant checks. It does not certify free-form generated prose. |
| Scheduling and resource allocation | [OR-Tools](https://github.com/google/or-tools) | Apache-2.0 | Register deterministic solver tools with explicit constraints, objectives, and time limits. |
| Durable multi-consumer event processing | [NATS JetStream](https://github.com/nats-io/nats-server), [documentation](https://docs.nats.io/concepts/jetstream) | Apache-2.0 server | Publish from the PostgreSQL outbox; use durable consumers, bounded retries, and application deduplication. Message acknowledgements do not make external side effects exactly once. |
| Shared source/object storage | [SeaweedFS](https://github.com/seaweedfs/seaweedfs) | Apache-2.0 community code | Use its S3 interface behind the existing storage adapter. Validate the exact API features and recovery behavior Jasmine needs; enterprise features are not dependencies. |
| Vector serving outgrows PostgreSQL | [Qdrant](https://github.com/qdrant/qdrant) | Apache-2.0 | Move vector serving behind the retrieval interface. Keep canonical permissions, claims, and source versions in PostgreSQL; recheck results against current authorization. |
| Tracing and operational metrics | [OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector), [Prometheus](https://github.com/prometheus/prometheus) | Apache-2.0 | Trace inference/retrieval/tool boundaries and scrape queue, latency, and resource metrics. Log references by default, not personal content. |
| Experiment tracking and model registry | [MLflow](https://github.com/mlflow/mlflow) | Apache-2.0 | Self-host experiment metadata and artifacts; track model, prompt, dataset, and evaluation versions. |
| Offline fine-tuning | [PEFT](https://github.com/huggingface/peft), [TRL](https://github.com/huggingface/trl) | Apache-2.0 | Add adapters and supervised/preference training after curated feedback exists. Run in an isolated training environment with evaluation before promotion. |
| Model regression evaluation | [lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness) | MIT code | Supplement application tests with selected language-model benchmarks. Individual benchmark datasets have separate terms. |
| Spoken input | [ESPnet](https://github.com/espnet/espnet) with [espnet/owsm_v3.2](https://huggingface.co/espnet/owsm_v3.2) | Apache-2.0 toolkit; CC-BY-4.0 model card | A concrete open speech candidate with public-data training lineage. Keep speech in a separate environment; measure accuracy and latency on intended speakers and conditions. CC-BY is the model's content license, not the toolkit's software license. |
| Basic spoken output | [eSpeak NG](https://github.com/espeak-ng/espeak-ng) | GPL-3.0-or-later, plus included component notices | Local speech synthesis with modest resource needs. Expect synthetic-sounding speech. Retain notices and satisfy its distribution terms; natural neural voices require a separate asset review. |
| General image/video interpretation | [allenai/Molmo2-4B](https://huggingface.co/allenai/Molmo2-4B), [code](https://github.com/allenai/molmo2) | Apache-2.0 checkpoint and code | Optional candidate for grounded image/video observations. It inherits a Qwen3 language backbone, so do not claim that its entire pretrained ancestry is reproducible from fully disclosed data. Keep OCR as the initial image feature. |

PostgreSQL, HTTP infrastructure, authentication, and numerical solvers are mature building blocks. Agent orchestration and multimodal model stacks have faster-changing APIs and need tighter integration testing. A research model's published evaluations do not make the assembled Jasmine system production-tested.

The checked [NATS release history](https://github.com/nats-io/nats-server/releases) and [SeaweedFS release history](https://github.com/seaweedfs/seaweedfs/releases) show September 2026 activity. For every adopted component, recheck releases, supported runtime versions, security policy, license, and actual required feature support when producing the lockfile. Avoid treating a star count or a newly published version as a reliability guarantee.

## 4 Model and runtime compatibility gates

| Pairing | Upstream evidence | Gate before using it |
| --- | --- | --- |
| Olmo 3 with Transformers | The model card states Transformers support from 4.57.0 | Pin a supported current release and model commit; load the official tokenizer/chat template; verify generation. The stated minimum is not a request to install an obsolete minimum version. |
| Olmo 3 with vLLM | [Supported models](https://docs.vllm.ai/en/stable/models/supported_models/) lists `Olmo3ForCausalLM` | Check the selected vLLM image, Transformers backend, chosen GPU/CPU, context cap, structured output behavior, and cancellation. |
| Nomic with Sentence Transformers | The model card documents prefixes and embedding dimensions | Ensure local inference; verify document/query preprocessing and output shape; pin any model code if the selected release requires it. Never grant arbitrary remote-code execution by default. |
| Docling with selected model assets | Docling explicitly separates code and model licenses | Inventory the chosen assets; prefetch them, pin revisions, and disable runtime auto-downloads in controlled deployments. [Docling model bundle](https://huggingface.co/docling-project/docling-models) uses CDLA-Permissive-2.0; this is not a blanket license for all integrations. |
| vLLM with AMD ROCm | [GPU installation matrix](https://docs.vllm.ai/en/stable/getting_started/installation/gpu/) documents supported combinations | Validate the exact GPU architecture, OS/kernel, ROCm, PyTorch, Python, and vLLM image. General AMD support does not certify every model/quantization combination. |
| Quantized model with serving runtime | Support depends on model format and kernel/backend | Compare output quality, tool schema reliability, memory, and latency with the reference checkpoint. Do not promise that any 4-bit conversion is a drop-in replacement. |

Use separate environments for the API, inference, embedding, perception, and training when dependency constraints conflict. This is especially useful for rapidly changing Transformers/PyTorch integrations. Prefer ordinary typed HTTP between these processes over trying to force every library into a single environment.

## 5 Hardware and environment

These are planning envelopes for a small text deployment, not throughput measurements or purchase guarantees. Use available hardware for a compatibility/latency test before committing to a configuration.

| Profile | Planning envelope | Expected use and constraint |
| --- | --- | --- |
| CPU development | Modern multicore CPU, roughly 32–64 GiB RAM; 64 GiB gives more room for FP32 loading; SSD | Validate persistence, APIs, retrieval, and single-user generation. A 7B model may be slow; smaller validated models can support early integration tests. |
| Practical accelerated prototype | Supported accelerator with about 24 GiB VRAM, 64 GiB host RAM, roughly 1 TB NVMe | Plausible starting point for 7B BF16 at bounded context and low concurrency. Runtime memory and kernel support still decide fit. |
| Larger 32B model experiment | Approximately 80–96 GiB accelerator memory for BF16 headroom, or a validated quantized configuration | Higher quality is task-dependent. Multiple accelerators require explicit sharding/interconnect support and load testing. |
| Shared deployment | Separate inference and state/storage capacity; redundant services according to availability goals | Size from workload traces. Queueing, prompt sizes, and generated tokens determine capacity more directly than account count. |

Approximate weight storage, before runtime overhead, uses parameter count times bytes per parameter:

| Nominal model size | BF16 weights | Ideal 4-bit weights |
| --- | --- | --- |
| 7 billion parameters | 14 GB, about 13.0 GiB | 3.5 GB, about 3.3 GiB |
| 32 billion parameters | 64 GB, about 59.6 GiB | 16 GB, about 14.9 GiB |

Actual counts vary by checkpoint. Quantization includes metadata and sometimes higher-precision tensors. Serving also needs activation buffers, attention/KV cache, runtime allocations, and room for concurrent requests. Training adds gradients, optimizer state, and activations; the inference figures do not size fine-tuning or pretraining.

For an ordinary full-attention decoder, an approximate KV-cache calculation is twice the sum, over layers and active sequences, of cached tokens times KV heads times head dimension times bytes per element. Sliding-window attention, cache sharing, quantization, and implementation details change the result. Measure actual allocation in the selected runtime.

A million 768-dimensional float32 embeddings alone require about 3.07 GB, or 2.86 GiB. Text, metadata, graph links, database overhead, indexes, replicas, and backups add storage. Audio and video will usually dominate source storage sooner than embeddings.

Use Linux, Git, a supported Python version compatible with each runtime, a supported Node.js LTS version compatible with Vite, a package lock for each environment, and OCI containers. CPython 3.12 is a conservative initial application compatibility target; choose and test the exact supported model-runtime Python separately. PostgreSQL 18 is a concrete initial database target, with an explicitly pinned pgvector build and supported patch release.

For a strict open-source software profile, use CPU libraries or a validated ROCm stack. Do not quietly include CUDA, Docker Desktop, a proprietary model API, or a hosted monitoring dependency in that profile. Firmware provenance remains a distinct hardware audit if the requirement extends that far.

Development prerequisites are Python async/typing, SQL transactions and migrations, browser API/authentication, container operations, retrieval evaluation, and basic model inference. Training additionally requires dataset curation, GPU profiling, and experiment design. Training a competitive foundation model from scratch is outside the workstation roadmap; use existing checkpoints and add targeted adapters only after the evidence warrants them.

## 6 Capacity and operating costs

Measure end-to-end latency as queue delay plus retrieval plus all model calls plus tool time plus verification. Report time to first useful response and time to verified completion separately. Include reasoning tokens and repeated calls in inference budgets.

For steady workloads, average in-flight requests are approximately arrival rate multiplied by average service time. This helps size admission control, but it is not a substitute for load testing at the actual prompt/output distribution. Use p95 queue age and latency to decide when to add replicas.

Keep interactive and batch budgets separate. Consolidation, embedding rebuilds, and training can consume all spare compute if unconstrained. Cache only with correct model/version and permission scope; cache hits must not expose another tenant's content. Allocate disk for model revisions, source history, indexes, and recoverable backups, not only the active checkpoint.

## 7 Release manifest requirements

Before the first executable release, record exact application commit, dependency locks, container digests, operating-system/runtime profile, model revisions and hashes, tokenizer/chat templates, embedding preprocessing, prompts, schemas, migration version, benchmark datasets, and evaluated hardware.

Review transitive package licenses and model/data terms on those exact artifacts. Run model files locally; prefer safe tensor formats where supported and avoid unaudited pickled artifacts or arbitrary model-loader code. Preserve a prior working release for rollback.

This register verifies project identities and documented capabilities. Compatibility, security, quality, and performance of the integrated deployment remain implementation tests; they have not been executed by creating this architecture.
