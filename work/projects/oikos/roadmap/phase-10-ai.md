---
type: project
status: draft
created: 2026-09-18
updated: 2026-09-18
tags: [oikos, roadmap, phase-10, ai, llm, ollama, vllm, litellm, gpu]
confidence: high
---

# Phase 10 — AI / GPU (Local LLMs + LLM Gateway)

## Objective

Deploy a **GPU-accelerated inference host** running local LLMs (Ollama for chat models, vLLM for high-throughput) plus a **LiteLLM gateway** exposing an OpenAI-compatible API for internal apps + AI agents.

## Why now

Storage (Phase 7) holds models (10–200 GB each). Kubernetes (Phase 9) serves inference. Monitoring (Phase 5) shows GPU utilization + queue depth. Identity (Phase 8) authenticates callers. Doing AI first would have been a very expensive prototype.

## Prerequisites

- Phases 0–9 complete
- Off-hours cooling / power headroom (a 3090/4090 pulls 350–450 W under load)
- Budget for the GPU box

## Hardware

| Item | Required? | Buy timing | Candidates | Rough cost |
|---|---|---|---|---|
| **GPU** — 24 GB VRAM class | **Required** | BUY NOW | NVIDIA RTX 3090 24 GB (used, ~$700) · RTX 4090 24 GB (~$1,600) · RTX A5000 24 GB (used, quiet, ~$1,200) | US$700–1,600 |
| **AI host chassis** — supports full-length dual-slot GPU, 750 W+ PSU, 16+ CPU cores, 64+ GB RAM, NVMe boot + storage NVMe | **Required** | BUY NOW | Lenovo ThinkStation P520 (used, ~$400 base) · custom AM5 Ryzen 9 build · Dell Precision 5820 | US$500–1,500 base + GPU |
| **PCIe passthrough headroom** on the host BIOS (IOMMU) | Required | — | — | — |
| Additional NVMe (2 TB) for model storage — hot | Recommended | BUY NOW | Samsung 990 Pro · WD Black SN850X · Solidigm P44 Pro | US$130–250 |
| Reasonable case fans / airflow | Required | — | — | — |
| UPS uprated for GPU draw | Conditional | BUY as needed | 1500 VA line-interactive minimum with GPU active | US$200 |

**Note**: A 3090 sipping ~350 W under sustained load is not something you plug into a residual 500 W UPS shared with everything else. Size accordingly.

## Software

- **Proxmox** on the AI host (`ai01` role) — GPU passthrough to one VM
- **vLLM** — high-throughput inference server (OpenAI-compatible API)
- **Ollama** — easy model management, good for chat / dev
- **LiteLLM** — proxy/gateway: routes OpenAI-format requests to Ollama or vLLM (or fallback to Anthropic/OpenAI if I choose); handles keys, rate limits, cost tracking, model aliasing
- **OpenWebUI** — chat frontend (optional but nice)
- **Whisper** — STT (optional)
- **Embedding models** — nomic-embed, bge — for internal RAG

## Network changes

- New host: `ai01.oikos.home.arpa` on VLAN 10 (mgmt) at `172.27.10.20/24`
- VM `ai-inference-01` on VLAN 50 (ai) at `172.27.50.10/24` — the GPU passthrough VM
- LiteLLM gateway on VLAN 30 (k8s, as a Deployment) — reachable at `llm.oikos.home.arpa`
- Firewall:
  - `k8s` → `ai`: allow HTTPS to vLLM/Ollama endpoints
  - `home` (my workstation) → LiteLLM gateway: allow HTTPS
  - `ai` → nas01 (NFS): allow (model downloads)
  - `ai` → Internet: allow HTTPS (model registry pulls); deny once models are cached

## Implementation sequence

1. **Buy + assemble** — GPU, host chassis, NVMe. Physically install.
2. **BIOS** — enable IOMMU, VT-d/AMD-Vi, resizable BAR, Above 4G Decoding.
3. **Install Proxmox** on `ai01`; join the cluster? — probably not (GPU passthrough + cluster mobility conflict). Standalone Proxmox is fine; note the tradeoff in an ADR.
4. **PCIe passthrough** — blacklist the NVIDIA driver on the host, bind GPU to vfio-pci, verify with `lspci -k`.
5. **Provision inference VM** — Ubuntu 22.04, 8 vCPU, 32 GB RAM, GPU passthrough. Install NVIDIA driver + CUDA + cuDNN inside the VM.
6. **Install Ollama** first (easier smoke test):
   - Pull `llama3.1:8b` or `qwen2.5:14b` (fits comfortably in 24 GB)
   - Test `ollama run` from the VM
7. **Install vLLM** (higher-throughput path):
   - `pip install vllm` (or docker image)
   - Serve a model: `vllm serve meta-llama/Meta-Llama-3.1-8B-Instruct --tensor-parallel-size 1`
   - Test with a curl to `/v1/chat/completions`
8. **Model storage** — models on `/mnt/models` backed by NFS from `nas01` (VLAN 40 crossings allowed per firewall rules)
9. **LiteLLM** deployment on k8s:
   - Values: define models (Ollama-local, vLLM-local, optionally external commercial models as fallback)
   - Auth: API keys per client, stored via Sealed-Secrets
   - Rate limits per key
   - Postgres backend for logging (small PVC)
   - Ingress: `llm.oikos.home.arpa`
10. **Monitoring**:
    - `nvidia_gpu_exporter` on the inference VM (GPU util, VRAM, power, temp)
    - vLLM Prometheus endpoint (request rate, TTFT, tokens/sec, queue depth)
    - LiteLLM metrics (per-key usage, latency, errors)
    - Alerts: GPU temp > 85 °C sustained, VRAM > 95 %, queue depth > threshold
11. **Test end-to-end**:
    - From your workstation: OpenAI SDK pointed at `https://llm.oikos.home.arpa/v1` with your key
    - Response returns from local vLLM
    - Metrics rise on Grafana

## CCNA topics reinforced

- Network path budgeting under sustained bursty traffic
- TLS certificate management for internal services
- Segmentation policy design (ai VLAN egress controls)

## Study track lab

Not directly. But: build an "isolated AI zone" topology in EVE-NG — router with strict egress rules on one VLAN — to prove the security model before applying to the real network.

## Physical Oikos lab exercise

- `nvidia-smi` on the inference VM under real load — watch VRAM + temp
- Chat with a local model via OpenWebUI
- From a Python script: hit LiteLLM as if it were OpenAI; get streaming tokens
- Compare local vs. commercial latency for the same prompt

## Break/fix drill

1. Fill VRAM (load a bigger model than fits). Observe OOM error handling in vLLM. Recover.
2. Take NFS down (Phase 7 drill). Observe: model load fails; running inference continues if model is memory-resident. Recover.
3. Rotate a LiteLLM API key without notifying a client — observe 401s — restore.

## Verification commands

Inference VM:

```bash
nvidia-smi
nvidia-smi dmon -s pucvmet
watch -n1 nvidia-smi

# vLLM
curl -s localhost:8000/v1/models
curl -s -X POST localhost:8000/v1/chat/completions -H 'Content-Type: application/json' -d '{"model":"...","messages":[{"role":"user","content":"hi"}]}'

# Ollama
ollama list
ollama ps
```

k8s (LiteLLM):

```bash
kubectl logs -n litellm deploy/litellm -f
kubectl port-forward -n litellm svc/litellm 4000:4000
curl -s http://localhost:4000/v1/models -H "Authorization: Bearer $KEY"
```

## Security posture at this phase

- **LiteLLM keys**: per-client, in Sealed-Secrets, rotatable
- **Model provenance**: only pull from HuggingFace or Ollama registry; verify checksums where available
- **Egress control**: after initial model download, deny VLAN 50 → Internet (models rarely need to phone home)
- **Prompt log retention**: decide policy (privacy vs. debuggability) — likely 30 days, encrypted, restricted access
- **Do not expose LiteLLM to the Internet** — cloudflared / tailscale if remote access ever needed
- Track OWASP LLM Top-10 as ai01 grows (prompt injection, insecure output handling, model DoS, sensitive info disclosure)

## Monitoring coverage

- GPU: util, VRAM, temp, power
- vLLM: queue, throughput (tokens/s), TTFT p50/p95/p99
- LiteLLM: per-key request rate, cost estimate, error rate
- Model file availability from NFS

## Kephaleos docs to create / update

- **Create**: `runbooks/proxmox-gpu-passthrough.md`
- **Create**: `runbooks/vllm-baseline.md`
- **Create**: `runbooks/ollama-baseline.md`
- **Create**: `runbooks/litellm-baseline.md`
- **Create**: `runbooks/model-download-and-verify.md`
- **Update**: `current-state.md`, `inventory/nodes.md` (`ai01`), `architecture/address-plan.md` (VLAN 50 active)
- **New ADRs**: `adr/000N-ai-serving-stack.md`, `adr/000N-ai-host-cluster-decision.md` (why ai01 is standalone Proxmox), `adr/000N-litellm-key-strategy.md`
- **Append**: `CHANGELOG.md`

## Diagram updates

- AI diagram (new): client → LiteLLM → local vLLM/Ollama → model storage on NAS
- Master diagram: add ai01 + VLAN 50
- Service dependency: LiteLLM depends on vLLM depends on GPU + NFS

## Completion checklist

- [ ] GPU host built and racked; UPS sized appropriately
- [ ] Proxmox on `ai01` with GPU passthrough verified
- [ ] `nvidia-smi` inside the VM shows the GPU
- [ ] Ollama serves at least one chat model
- [ ] vLLM serves at least one model with OpenAI-compatible API
- [ ] LiteLLM gateway on k8s routes to local + optional commercial models
- [ ] Ingress + TLS working for `llm.oikos.home.arpa`
- [ ] Monitoring covers GPU + inference + gateway
- [ ] Egress controls active on VLAN 50 post-initial-download
- [ ] Docs, ADRs, diagrams updated
