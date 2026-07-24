# Design Features in Practice

_How the system-design-methodology maps to the llm-d implementation_

---

## What llm-d Is

llm-d is a distributed LLM inference serving stack that sits above model servers (vLLM, SGLang) on Kubernetes. Its job is intelligent orchestration: routing requests to the right GPU, managing KV cache across a fleet, disaggregating prefill and decode, and protecting the backend from overload — all at production scale.

The sections below walk through each area of the design methodology and show exactly where it surfaces in the system.

---

## 1. Functional Requirements → What the System Does

The core user-facing actions llm-d is designed to serve:

| Action | How it's expressed |
|---|---|
| Submit a completion or chat request | OpenAI-compatible HTTP API (`/v1/completions`, `/v1/chat/completions`) via the Proxy |
| Route request to the optimal model server | EPP runs Filter → Score → Pick against the InferencePool |
| Submit a batch job | OpenAI-compatible Batch API (`/v1/batches`) via the Batch Gateway |
| Query batch job status | `/v1/batches/{id}` — Batch Gateway tracks lifecycle |
| Serve a large model across prefill and decode workers | Disaggregated serving: EPP picks a P+D pair; sidecar coordinates KV transfer |
| Autoscale serving capacity | KEDA + EPP queue metrics, or WVA for globally optimised placement |

Out of scope at the llm-d layer: executing the model, managing GPU memory within a pod, implementing the tokenizer — those are model-server concerns.

---

## 2. Non-Functional Requirements → The Constraints That Shape Everything

### Scalability

**Load balancer** — The llm-d Router (Proxy + EPP) is the horizontal entry point. The Proxy (Envoy or equivalent L7 proxy) terminates connections and distributes requests; the EPP provides the routing intelligence. Multiple model server pods form an `InferencePool`, identified by a label selector. Adding pods to the pool is automatic.

**API gateway** — The Router centralises rate limiting, auth, SLO enforcement, and tenant routing so individual model servers need none of it. The `EndpointPickerConfig` is the single place to define flow control bands, fairness policies, and saturation limits.

**Data scaling** — KV cache is stored in GPU HBM (bounded, expensive). llm-d extends effective cache capacity by offloading to CPU memory and SSD via the KV Offloader, creating a tiered storage hierarchy. Requests are then routed to whichever pod holds the matching cache tier, avoiding recomputation.

**Reduce data in transit** — Disaggregated serving uses NIXL over RDMA (InfiniBand, RoCE) to transfer KV cache directly between GPU memory on different nodes, bypassing CPU and avoiding serialisation overhead. GPUDirect RDMA means the data never touches host RAM.

### Availability

**Eliminate the single gateway as SPOF** — The `InferencePool` `failureMode` field controls what happens if the EPP is unreachable. In the default llm-d configuration (`FailOpen`), Envoy bypasses the EPP and routes directly to a model server pod — trading routing intelligence for availability. `FailClose` rejects the request rather than route unprotected.

**Multi-EPP active-active** — The KV Indexer supports a pod-discovery delivery mode where each EPP replica independently subscribes to every model server's ZMQ event stream. All replicas see the full KV event feed and maintain an identical index, enabling active-active redundancy without a shared state store.

**Graceful degradation** — If the Latency Predictor sidecar is unreachable, the `latency-scorer` falls back to a composite heuristic (KV cache utilisation + queue depth + prefix match) rather than dropping traffic. If batch processing is degraded, the Async Processor's flow-control gate closes, buffering work rather than overloading the pool.

**Replication strategy** — Multiple decode pods in the InferencePool are standard. In disaggregated deployments, prefill and decode roles are split across separately scaled replica sets, each scalable independently.

### Performance

**KV cache as the CDN** — Instead of a CDN for static content, llm-d uses prefix-cache-aware routing as its primary performance mechanism. The EPP routes each request to the pod that already holds the matching KV cache for the prompt prefix, turning cache reuse into latency reduction (measured: 2× faster TTFT and 3× higher throughput vs round-robin on Llama 70B).

**Cache at the application layer** — The KV Indexer maintains a near-real-time `block key → pod` map built from `KVEvents` emitted by model servers over ZMQ. The EPP consults this index during scoring rather than asking each pod at request time, keeping the critical path cheap.

**Read replicas** — In disaggregated serving, decode pods are memory-bandwidth-bound read-path workers. They are scaled independently of prefill pods and can use a larger tensor-parallel configuration suited to token generation rather than prompt processing.

**SLO-aware scheduling** — The `latency-scorer` computes SLO headroom — the gap between predicted TTFT/TPOT and the request's declared latency budget (via `x-llm-d-slo-ttft-ms` header). The EPP picks the pod with the most headroom rather than simply the least loaded, making latency targets first-class routing constraints.

---

## 3. API and Sequence Design → Contracts First

### Defined Call Signatures

```
# Client-facing (OpenAI-compatible)
POST /v1/completions               → { id, choices[], usage }
POST /v1/chat/completions          → { id, choices[], usage }
POST /v1/batches                   → { id, status, created_at }
GET  /v1/batches/{id}              → { id, status, request_counts }

# Internal (EPP ↔ Proxy)
ext_proc (gRPC)                    → endpoint address + mutated headers

# Internal (model server → EPP)
ZMQ PUB/SUB: KVEvent{BlockStored, BlockRemoved, AllBlocksCleared}
```

### Key Sequence: Disaggregated Request Flow

The disaggregated serving sequence diagram (from `docs/architecture/advanced/disaggregation/README.md`) traces the full P/D path — the most complex critical path in the system:

```
Client → Proxy → EPP (selects P+D) → Decode Sidecar → Prefill Worker
                                    ← KVTransferParams
                                    → Decode Worker (pull KV via NIXL RDMA)
                                    ← tokens
       ← Proxy ← response
```

This single trace surfaced the requirement for the routing sidecar on each decode pod, the `x-prefiller-host-port` header as the cross-component handoff, and the NIXL RDMA requirement as a hard infrastructure dependency.

### Key Sequence: Flow Control Dispatch

Requests with flow control enabled follow: Ingress → FlowKey assignment → per-band queue → Saturation Detector gate → late-binding Scheduler → endpoint. The "park and wait" design (requests block in the EPP rather than in the model server's local queue) is what enables dynamic re-prioritisation and late-binding cache affinity decisions.

---

## 4. Architecture Style → Microservices + Event-Driven

llm-d uses both styles, composing them by layer:

| Layer | Style | Rationale |
|---|---|---|
| Request path | Microservices | Proxy, EPP, model servers deploy and scale independently |
| Cache state tracking | Event-driven | KVEvents over ZMQ decouple producers (model servers) from consumers (EPP index) |
| Batch processing | Event-driven | Async Processor pulls from message queues (Redis, Pub/Sub), decoupling submission from dispatch |
| Latency prediction | Sidecar microservice | Training server and prediction servers colocated with EPP, sharing a volume, no external dependency |

**Component map:**

| Component | Role | Protocol to peers |
|---|---|---|
| Proxy (Envoy) | L7 data plane, TLS, connection management | HTTP to model servers; gRPC (`ext_proc`) to EPP |
| EPP | Routing intelligence, flow control, scheduling | gRPC (`ext_proc`) from Proxy; ZMQ SUB from model servers |
| KV Indexer | Block-level cache state tracking | ZMQ SUB (event consumer) |
| KV Offloader | Tiered cache capacity (CPU/SSD) | In-process with model server |
| Routing Sidecar | Disaggregated request orchestration + KV protocol translation | HTTP to model server; HTTP to prefill worker |
| Latency Predictor | Online ML for TTFT/TPOT prediction | HTTP to EPP prediction server; shared volume to training server |
| Batch Gateway | OpenAI Batch API, job lifecycle | HTTP to llm-d Router |
| Async Processor | Queue-driven dispatch with flow-control gating | Redis / Pub/Sub; HTTP to llm-d Router |
| WVA | Globally optimised autoscaling | Kubernetes controller; reads EPP + KV cache metrics |

**Data types and stores:**

| Data | Store type | Why |
|---|---|---|
| KV cache blocks (hot) | GPU HBM | Access bandwidth requirement |
| KV cache blocks (warm) | CPU RAM / SSD (KV Offloader) | Extend capacity beyond HBM limits |
| Block key → pod index | In-process EPP memory (radix tree) | Sub-millisecond lookup on the scheduling hot path |
| Flow control queues | In-process EPP memory | Low-latency; not durable by design |
| Batch job state | Batch Gateway store | Lifecycle tracking across job submission, execution, retrieval |
| Latency prediction models | Shared pod volume | Read by prediction servers, written by training server |
| Prometheus metrics | Time-series (Prometheus/KEDA) | Autoscaling and observability |

---

## 5. Optimisation Pass → Every NFR Addressed

### Eliminate Single Points of Failure

| Risk | Mitigation |
|---|---|
| EPP is the only routing brain | `FailOpen` degrades to unprotected pass-through; pod-discovery mode enables active-active multi-EPP |
| Model server pod failure | InferencePool health filter removes unhealthy pods before scoring |
| Prefill worker fails mid-transfer | vLLM sidecar falls back to decoder-only mode on prefill server error |
| Latency predictor unavailable | Scorer falls back to composite heuristic scoring automatically |
| EPP restart loses queued work | 503 with retryable signal on graceful drain; FailOpen on abrupt crash |

### Eliminate Bottlenecks

The critical bottleneck for LLM inference is GPU memory and compute. llm-d attacks it at multiple levels:

- **KV cache reuse** — routing requests to pods with matching prefix cache eliminates redundant prefill computation, the most FLOPs-intensive phase
- **Disaggregation** — separates FLOPs-bound prefill from memory-bandwidth-bound decode onto independent, separately sized worker pools; removes the "long prefill blocks decodes" interference pattern
- **Flow control** — shifts queuing from model server local queues (where it cannot be reordered) to the EPP (where priority, fairness, and SLO policies apply), preventing context thrashing on GPU memory
- **Wide Expert Parallelism** — for Mixture-of-Experts models, DP/EP deployments eliminate pipeline bubbles and use the "MaskedGEMM" format optimised for decode; validated at 50k tokens/sec on a 16×16 B200 topology

### Optimise Critical Paths

The two critical paths with SLA implications are:

**Time To First Token (TTFT):** Prefix-cache-aware routing → KV offloading to CPU/SSD → disaggregated prefill on dedicated compute → latency predictor SLO headroom scoring. Each layer reduces TTFT independently; they compose.

**Time Per Output Token (TPOT):** Flow control prevents context thrashing (the main cause of TPOT degradation under load). Flow control protects TPOT by holding requests in the EPP until backend KV cache is available, rather than letting them pile up and fragment GPU memory.

### Algorithms and Data Structures Matched to Access Patterns

| Access pattern | Data structure / algorithm |
|---|---|
| Prefix match for cache affinity (heuristic) | Radix/trie tree over token hash prefixes |
| Precise block lookup (event-driven) | `block key → pods` hash map, updated by ZMQ events |
| Endpoint scoring | Weighted sum over plugin scores (0.0–1.0), configurable per profile |
| Priority queuing | Per-band priority queues; Fairness Policy (round-robin) cycles within bands |
| Request ordering within flow | FCFS (default), EDF heap, or SLO-deadline heap |
| Latency prediction | XGBoost regression with stratified bucketing by utilisation and prefix hit rate |
| Autoscaling signal | "True Demand" queue depth metric from EPP flow control — definitive unfulfilled demand vs GPU utilisation (which is non-linear and lags reality) |

---

## Patterns in Action

### CQRS — Applied to KV Cache State

The KV cache management subsystem is a direct implementation of CQRS:

- **Write path (Commands):** Model servers (vLLM, SGLang) publish `KVEvents` — `BlockStored`, `BlockRemoved`, `AllBlocksCleared` — over ZMQ whenever their cache state changes. These are the commands that mutate the global view.
- **Read path (Queries):** The EPP's `precise-prefix-cache-producer` queries the KV Indexer's `block key → pod` map during the Filter → Score → Pick cycle. This read path is fully decoupled from the write path and optimised for the scheduling hot path.
- **Eventual consistency:** The index is near-real-time, not synchronous. A block evicted from a pod may still appear in the index for a brief window. The router handles this gracefully — a routing decision to a "stale hit" pod falls back to local prefill at the model server.

### Materialised View — The KV Index

The KV Indexer is a materialised view of the distributed KV cache state across all model servers in the pool:

- **What it pre-computes:** Which content-addressed cache blocks (`block key`) are resident on which pods, across all storage tiers (HBM, CPU, SSD)
- **How it's updated:** Continuously, event-driven — each `BlockStored`/`BlockRemoved` event from any model server immediately updates the index
- **What it saves:** Re-querying each pod individually on every incoming request (O(pods) network calls → O(1) local lookup)
- **Trade-off:** Storage and write amplification (every cache event is processed and stored in the EPP) in exchange for sub-millisecond routing decisions
- **Pairs with CQRS:** The KV index is the query-side projection; the `KVEvents` stream is the command side
