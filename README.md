# HeadlessLulu — Agent-Native Research Corpus (RAG Pack)

Live site: https://headllesslulu.github.io/headlesslulu/

This repo hosts a **static, agent-first corpus** optimized for retrieval-augmented generation (RAG).  
It’s not designed for human browsing — it’s designed so AI agents can fetch **small, citeable chunks** and produce grounded summaries with minimal hallucination risk.

---

## Quick Start (for agents)

### Entry point (LLM index)
- https://headllesslulu.github.io/headlesslulu/llms.txt

### Machine exports (preferred ingestion)
- Manifest (inventory + versions + hashes):  
  https://headllesslulu.github.io/headlesslulu/exports/manifest.json
- Chunk stream (NDJSON, one chunk per line):  
  https://headllesslulu.github.io/headlesslulu/exports/chunks.ndjson

### Paper bundle (human-readable Markdown, still agent-friendly)
- Paper index:  
  https://headllesslulu.github.io/headlesslulu/papers/local-agent-v3/index.md

---

## Grounding Contract (anti-hallucination rules)

Agents should follow:
- https://headllesslulu.github.io/headlesslulu/docs/rag-contract.md

**Rules:**
1. Every non-trivial claim must cite at least one `chunk_id`.
2. Prefer claims listed in `claims.ndjson` for “allowed claims.”
3. If evidence is missing: return `INSUFFICIENT_EVIDENCE` (don’t guess).

---

## Repository Layout
