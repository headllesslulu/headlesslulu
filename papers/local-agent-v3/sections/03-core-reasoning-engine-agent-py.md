---
doc_id: local-agent-v3.03.core-reasoning-engine-agent-py
paper: local-agent-v3
version: 2026-02-23
---

# Core Reasoning Engine  (agent.py)

The reasoning engine is a five-component system orchestrated by a single LocalAgent class. Each component maps to a biological analogue from the original architecture specification.

## 3.1  Component Map <a id="3-1-component-map"></a>

## 3.2  The 4-State Compute Graph <a id="3-2-the-4-state-compute-graph"></a>

The central innovation is replacing single-pass autoregressive generation with a structured 4-state pipeline. Each state runs as a separate, stateless ollama inference call with a distinct system prompt and temperature setting.

Context isolation: each state receives a clean system+user message pair. No conversation history is passed between states. State 3 receives draft and critique as USER message content, not as prior assistant turns. This prevents the model from anchoring too hard on its own previous token distributions.

State 3 always receives token_multiplier=2.0. Even in hot-zone thermal scaling (50% reduction), synthesis retains at least its base token budget to fully repair the draft.

## 3.3  Thermal-Aware Token Scaling  (v3 feature) <a id="3-3-thermal-aware-token-scaling-v3-feature"></a>

Live GPU temperature maps to a multiplier applied to every inference call. Three zones are defined relative to thermal_limit_c (default 75°C) and token_scale_margin (default 5°C):

Floor values prevent any stage from dropping below functional minimums: route: 64, critique: 128, draft/synthesis: 192 tokens. The zone and budget change are logged on every call where scaling applies.

## 3.4  TDP Control  (v3 feature, optional) <a id="3-4-tdp-control-v3-feature-optional"></a>

Enabled via --tdp CLI flag. Requires root/Administrator. The agent calls nvidia-smi -pl {tdp_normal_w} once at startup as a privilege probe. If the command returns non-zero or raises FileNotFoundError, _tdp_ok is set False and a loud warning is printed immediately. No code path downstream ever calls nvidia-smi again unless _tdp_ok is True.

On thermal event: TDP throttle fires first (if available), dropping wattage from 125W to 80W. Sleep-based throttle is the fallback. On temperature recovery: TDP is restored to normal explicitly — the GPU does not stay capped after the thermal event clears.

## 3.5  Confidence-Aware Hippocampus  (v2 feature) <a id="3-5-confidence-aware-hippocampus-v2-feature"></a>

Every memorize() call stores the ConfidenceScorer output alongside the response in ChromaDB metadata. Every recall() returns a MemoryResult object containing the stored confidence and an is_trusted boolean.

This creates a self-healing memory: mediocre cached responses are progressively overwritten by better syntheses triggered on subsequent similar queries. The store converges to high-confidence entries through use.

## 3.6  RAGStore  (v2 feature) <a id="3-6-ragstore-v2-feature"></a>

A separate ChromaDB collection (rag_documents) stores ingested document chunks. Isolated from agent memory (agent_memory). Shares the SentenceTransformer encoder instance with Hippocampus — loaded once, no double VRAM cost.

- Ingests: .txt .md .rst .csv .json .py .js .ts .html .xml .yaml .toml .pdf

- Word-level chunking: 400 words/chunk, 50-word overlap

- Content-hashed IDs: re-ingesting same file is idempotent (safe to repeat)

- PDF extraction: pdfminer.six with pdftotext CLI fallback

- Retrieval: top-4 chunks by cosine similarity, returned as formatted context string

The router (STATE 0) generates a targeted rag_query string distinct from the raw user query, maximising retrieval precision. The JSON schema uses needs_rag/rag_query not needs_web/search_query.
