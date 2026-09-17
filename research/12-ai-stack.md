---
tags: [research, ai, ollama, open-webui, llm, gpu, nvidia]
created: 2026-09-17 21:38:22 +05
modified: 2026-09-17 21:42:45 +05
---

# AI / LLM Stack Deep-Dive (RTX 5060 8GB)

Research date: 2026-09-17. Complements research/02-gpu-passthrough (hardware path) - this is the application layer. HW target: RTX 5060 Max-Q 8GB (GB206, ~7.5GB usable, 448GB/s), 62GB RAM.

## Where to run Ollama

- Host owns the NVIDIA kernel driver and /dev/nvidia*; the LXC/VM only needs matching userspace libs.
- Best for a single-user homelab: native Ollama in one Debian LXC (simplest, smallest overhead, no cgroup pain).
- Docker-in-LXC also works: nesting=1,keyctl=1 + `no-cgroups = true` in /etc/nvidia-container-runtime/config.toml + `nvidia-ctk runtime configure --runtime=docker`.
- Full VM with PCI passthrough = Proxmox-recommended Docker path but wasteful here.
- In unprivileged LXC create dev entries for /dev/nvidia0, nvidiactl, nvidia-uvm, nvidia-uvm-tools, nvidia-modeset (lxc.mount.entry) + lxc.cgroup2.devices.allow. Ensure /dev/nvidia-uvm exists (run nvidia-persistenced) or CUDA fails.

Basics: `ollama pull llama3.1:8b`, `ollama run`, `ollama list`, `ollama ps`, `ollama rm/cp`. API: POST /api/generate, /api/chat; OpenAI-compatible at /v1/chat/completions. Expose with `OLLAMA_HOST=0.0.0.0:11434`.

## Model fit on 8GB VRAM

Rule of thumb: Q4_K_M ~= 0.6GB per B + KV cache. 7-9B Q4 fits fully; 14B+ spills to CPU and collapses.

| Model | Quant | Size | Fits 8GB? | Usable ctx | Est. speed (RTX 5060) |
|-------|-------|------|-----------|-----------|------------------------|
| phi4-mini (3.8B) | Q8_0 | ~2.5GB | yes | 8-16K | ~130-150 tok/s |
| llama3.1/3.3:8b | Q4_K_M | ~4.9GB | yes | 4-8K | ~45-51 tok/s |
| qwen2.5:7b / qwen3:8b | Q4_K_M | ~4.4-5.2GB | yes | 4-8K | ~50 tok/s |
| gemma2/gemma3:9b | Q4_K_M | ~6.1-6.6GB | tight | <=4K | ~38-40 tok/s |
| qwen2.5:14b | Q4_K_M | ~8.7-9GB | offload | 4K | ~5-8 tok/s |
| deepseek-r1:32b | Q4_K_M | ~18.8GB | mostly RAM | - | <2 tok/s (skip) |

- 32B Q4 with 62GB RAM loads but is unusably slow; 7-9B Q4 is the sweet spot.
- MoE models (e.g. gemma3n:e4b) work well - only ~3-4B active.

## Open WebUI

- Chat UI with native Ollama protocol (11434): pull models through the Admin panel, RAG (9 vector DBs, hybrid search), web search, Python tools, multi-user auth.
- Docker: `ghcr.io/open-webui/open-webui:main`, `-p 3000:8080`, `-v open-webui:/app/backend/data`, `-e OLLAMA_BASE_URL=http://<lxc-ip>:11434`. Or `pip install open-webui && open-webui serve`.
- Ollama defaults to 2048-token context (cripples RAG) - raise num_ctx.
- Alternatives: AnythingLLM (doc-Q&A focus), LibreChat; LocalAI/vLLM overkill for 8GB.

## Small-card tuning

- Quantization: stick to Q4_K_M (k-quant); Q5/Q8 only for <=4B.
- OLLAMA_KV_CACHE_TYPE=q4_0 + OLLAMA_FLASH_ATTENTION=1 (saves ~0.5-1GB at longer ctx).
- OLLAMA_NUM_PARALLEL=1, OLLAMA_MAX_LOADED_MODELS=1, keep OLLAMA_KEEP_ALIVE short.
- OLLAMA_CONTEXT_LENGTH=4096 default, or num_ctx 8192 per chat; KV cache grows -> OOM first.
- Offload: num_gpu/OLLAMA_NUM_GPU (=-1 auto-fill), num_thread for CPU portion; check 100% GPU via `ollama ps`. One model at a time.

## Recommended config for this box

Single unprivileged Debian 12 LXC (8 cores, 32GB RAM); host driver 5xx + nvidia-persistenced; native Ollama + Open WebUI in same LXC. Expose 11434 + 3000/8080.

Model shortlist: llama3.1:8b (general), qwen2.5-coder:7b (code), phi4-mini (fast/Q8), gemma2:9b (quality ceiling), optional deepseek-r1:7b. Expect ~40-50 tok/s generation, near-instant prompt processing.

## Sources

- https://github.com/Bishop-trevorstuart/Nvidia-Proxmox-LXC-Docker
- https://www.virtualizationhowto.com/2025/05/run-ollama-with-nvidia-gpu-in-proxmox-vms-and-lxc-containers
- https://localaimaster.com/vram/best-ollama-models-8gb-vram
- https://www.fitmyllm.com/gpu/geforce-rtx-5060
- https://docs.ollama.com/faq