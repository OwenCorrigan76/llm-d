# vLLM-Omni Integration: Discovery & Gap Analysis

> Scope: what llm-d has already done for vllm-omni support, what is missing for MVP,
> and what (if anything) needs to change in llm-d-router.
> MVP path = **Optimized Baseline** well-lit path, CUDA only.

---

## 1. What Already Exists in the Repo

The repo is further along than the brainstorm document implies. The install
script, deployment recipes, and router configuration are all present.

### 1.1 Docker install script

`docker/scripts/cuda/runtime/install-vllm-omni.sh`

- Activates the `/opt/vllm` virtual environment (shared with vLLM).
- Clones the vllm-omni repository at a pinned commit SHA.
- Installs it in editable mode via `uv pip install -e`.
- Strips `.git`, `docs`, and `tests` directories to keep the image lean.
- Controlled by two required env vars (`VLLM_OMNI_REPO`, `VLLM_OMNI_COMMIT_SHA`)
  and one optional gate (`VLLM_OMNI_INSTALL=true/false`).

**Status: script exists, but is not called anywhere in `Dockerfile.cuda`.**

### 1.2 Kustomization component for the vllm-omni image

`guides/recipes/modelserver/components/images/gpu-vllm-omni/kustomization.yaml`

Currently references a **third-party image** (`ghcr.io/revit13/vllm-openai:14d40dc7d`)
as a workaround. The intended `vllm/vllm-omni:v0.22.0` target is commented out.

There is also a USER env-var patch applied to both `Deployment` and `LeaderWorkerSet`
kinds (workaround for a torch 2.11 regression in vLLM ≥ v0.20.0).

### 1.3 Multimodal serving guides

`guides/multimodal-serving/` contains two complete deployment recipes:

| Guide | Path | Description |
|---|---|---|
| Aggregation | `aggregation/` | All-in-one: encode + prefill + decode on the same replica. Simpler topology, prefix-cache-aware + load-aware routing. |
| Encode-Disaggregation | `e-disaggregation/` | Dedicated encode workers (E), with combined PD or separate P/D workers. Uses NIXL for embedding transfer. |

The **Optimized Baseline MVP** maps to the **Aggregation** guide.

### 1.4 Router configuration for multimodal

`guides/multimodal-serving/aggregation/router/aggregation.values.yaml`

The EPP already handles multimodal routing. It hashes both text tokens and
visual asset bytes to build a per-request prefix key, then scores candidate
pods by estimated cache reuse and queue depth. No router changes are required
for the MVP.

---

## 2. What Is Missing (Changes Required)

All gaps are in the **llm-d** repo. The router requires no changes for the MVP.

### 2.1 `docker/common-versions` — version pins not defined

The install script needs two variables that are absent from the version
config file:

```bash
# vLLM-Omni settings (add below existing VLLM_* block)
VLLM_OMNI_REPO=https://github.com/vllm-project/vllm-omni.git
VLLM_OMNI_COMMIT_SHA=<pinned commit SHA>
VLLM_OMNI_INSTALL=true
```

Choose a commit SHA from the vllm-omni repository that is compatible with
the pinned `VLLM_COMMIT_SHA` already in `common-versions`
(`51f799c1a0c8a0476faf7e17eeb6a77983cdd778`, neuralmagic fork, ~v0.23.0).

### 2.2 `docker/Dockerfile.cuda` — install script not wired in

The runtime stage installs vLLM via `install-vllm.sh` (line 541) but never
calls `install-vllm-omni.sh`. The following block needs to be added after
the vLLM install step, still inside the `runtime` stage:

```dockerfile
# vLLM-Omni (multimodal extension) — installed on top of vLLM
ARG VLLM_OMNI_REPO
ARG VLLM_OMNI_COMMIT_SHA
ARG VLLM_OMNI_INSTALL=true

COPY docker/scripts/cuda/runtime/install-vllm-omni.sh /tmp/install-vllm-omni.sh
RUN --mount=type=cache,target=/var/cache/git \
    VLLM_OMNI_REPO=${VLLM_OMNI_REPO} \
    VLLM_OMNI_COMMIT_SHA=${VLLM_OMNI_COMMIT_SHA} \
    VLLM_OMNI_INSTALL=${VLLM_OMNI_INSTALL} \
    SUPPRESS_PYTHON_OUTPUT=${SUPPRESS_PYTHON_OUTPUT} \
    /tmp/install-vllm-omni.sh && \
    rm /tmp/install-vllm-omni.sh
```

The existing `ENTRYPOINT ["python", "-m", "vllm.entrypoints.openai.api_server"]`
should remain unchanged — vllm-omni installs alongside vLLM and uses the same
OpenAI-compatible API server. **Verify this against the vllm-omni repo before
finalising.**

### 2.3 `guides/recipes/modelserver/components/images/gpu-vllm-omni/kustomization.yaml` — swap to llm-d image

Once the image is built and published, replace the third-party workaround
image with the llm-d image:

```yaml
images:
  - name: REPLACE_MODEL_SERVER_IMAGE
    newName: <your-registry>/llm-d-cuda   # or llm-d-vllm-omni if a separate target
    newTag: <git-sha-or-semver>
```

---

## 3. llm-d-router: No MVP Changes Required

The router (EPP) already implements everything needed for multimodal optimised
baseline serving:

- **Prefix-cache-aware routing**: hashes both text prompt and visual asset
  bytes; directs requests to pods with matching cached KV tensors.
- **Load-aware routing**: scores pods by queue depth and KV-cache utilisation.
- **Helm values**: `guides/multimodal-serving/aggregation/router/aggregation.values.yaml`
  is the ready-to-use configuration.

For the **E-Disaggregation** topology (a later phase), router values are also
already provided at `guides/multimodal-serving/e-disaggregation/router/`.

---

## 4. Open Questions from the Brainstorm Document

| Question | Finding |
|---|---|
| Does vllm-omni handle prefill instance failure? | No. Handled at the Kubernetes layer (Deployment/LeaderWorkerSet pod restarts) and by the EPP health-check loop removing unhealthy endpoints from the pool. vllm-omni adds nothing new here. |
| Does vllm-omni handle decode instance failure? | Same as above. |
| Rollouts? | Standard Kubernetes rolling updates. No vllm-omni-specific logic. |
| Does vllm-omni handle scoring (multimodal content hashes)? | No. Scoring lives entirely in the EPP/router. vllm-omni is the model server only; it responds to requests, it does not influence routing decisions. |
| Does vllm-omni handle KV-offloading (Native offloadingConnector)? | The offloading connector (`llmd_fs_connector`) is a separate wheel installed via `install-offloading-connector.sh` and is independent of vllm-omni. Whether vllm-omni is compatible with the connector needs empirical verification against the pinned commit. |

---

## 5. MVP Work Order (CUDA, Optimised Baseline Only)

```
1. Identify a vllm-omni commit SHA compatible with vLLM ~v0.23.0
2. Add VLLM_OMNI_REPO / VLLM_OMNI_COMMIT_SHA to docker/common-versions
3. Add ARG declarations + COPY + RUN block to Dockerfile.cuda runtime stage
4. Verify the entrypoint is unchanged in vllm-omni
5. Build the image locally and confirm `vllm_omni` is importable
6. Push the image and update the gpu-vllm-omni kustomization to reference it
7. Deploy against the existing multimodal aggregation guide and smoke-test
```

Benchmarking and the E-Disaggregation topology are out of scope for the MVP
and can be addressed in a later phase.

---

## 6. Architectural Context

### Where vllm-omni sits

```
Client request
      │
      ▼
  Inference Gateway (Envoy L7 proxy)
      │  ext-proc
      ▼
  Endpoint Picker (EPP)
  ┌──────────────────────────────┐
  │  Filter → Score → Pick       │
  │  • prefix-cache hash (text   │
  │    + image bytes)            │
  │  • load score (queue / KV)   │
  └──────────────────────────────┘
      │  selected pod IP
      ▼
  vllm-omni pod (model server)
  ┌──────────────────────────────┐
  │  vLLM runtime                │
  │  + vllm-omni multimodal ext  │
  │  ┌──────┐ ┌────────┐         │
  │  │Encode│→│Prefill │→Decode  │
  │  └──────┘ └────────┘         │
  └──────────────────────────────┘
```

In aggregation mode (MVP), all three phases run on the same pod. The EPP's
multimodal hash routing maximises cache reuse across replicas.

### Component responsibilities

| Component | Responsibility |
|---|---|
| EPP (router) | Routing, scoring, multimodal content hash computation |
| InferencePool CRD | Defines the group of vllm-omni pods as a managed serving pool |
| vllm-omni (model server) | Model inference only; encode + prefill + decode |
| llm-d-kv-cache / offloading connector | KV-cache tiered offloading (CPU/SSD); installed separately |
| NIXL | GPU-to-GPU embedding transfer for E-Disaggregation (post-MVP) |
