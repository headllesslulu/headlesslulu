---
doc_id: local-agent-v3.02.hardware-constraints-vram-budget
paper: local-agent-v3
version: 2026-02-23
---

# Hardware Constraints & VRAM Budget

All architectural decisions trace back to a single hard constraint: 6GB VRAM on a Turing-generation GPU. This section documents what that constraint means in practice and which features it disables.

## 2.1  VRAM Allocation — LLM Mode <a id="2-1-vram-allocation-llm-mode"></a>

The num_ctx=4096 context window cap in PrefrontalCortex.infer() is a hard limit, not a preference. Exceeding it on this card causes OOM. Do not increase it.

## 2.2  Turing Architecture Limitations <a id="2-2-turing-architecture-limitations"></a>

The GTX 1660 Super is sm_75 (Turing). Several commonly cited optimizations are unavailable on this microarchitecture:

Rule for scaffolding agents: never call FP8, BF16, or Flash Attention 2 on this card without probing compute capability first. The video engine contains a reference implementation of the probe pattern in resolve_attention_backend().

## 2.3  VRAM Coexistence — LLM and Video Cannot Share <a id="2-3-vram-coexistence-llm-and-video-cannot-share"></a>

LLM inference and video generation each require approximately 5.5GB. They cannot run simultaneously. The protocol for switching:

- VRAMManager.acquire() sends POST /api/generate {keep_alive: 0} to the local ollama server, which evicts the LLM from VRAM

- PyTorch GC and cache clear runs after eviction

- Video pipeline loads into the now-available VRAM

- On completion (or crash), VRAMManager.release() clears the video pipeline. Ollama reloads the LLM lazily on the next inference request

This is a cooperative protocol, not a hardware lock. If another process loads into VRAM between acquire() and pipeline load, the video engine will OOM. The recommended deployment is a single-process node where only one workload runs at a time.
