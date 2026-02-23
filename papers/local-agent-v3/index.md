# TECHNICAL SYSTEM REPORT

**Local Autonomous AI Node — v3 Architecture**

## How agents should use this paper
- Prefer `claims.ndjson` for *allowed* claims and cite the supporting `chunk_id`s.
- If you need full context, fetch only the relevant `sections/*.md` pages (avoid pulling `full.md` unless necessary).
- Summaries should include `paper version` and at least one citation per paragraph.

## Entry points
- Abstract: /papers/local-agent-v3/abstract.md
- Executive summary: /papers/local-agent-v3/executive-summary.md
- Full text: /papers/local-agent-v3/full.md

## Sections
- /papers/local-agent-v3/sections/01-executive-summary.md — Executive Summary
- /papers/local-agent-v3/sections/02-hardware-constraints-vram-budget.md — Hardware Constraints & VRAM Budget
- /papers/local-agent-v3/sections/03-core-reasoning-engine-agent-py.md — Core Reasoning Engine  (agent.py)
- /papers/local-agent-v3/sections/04-telegram-gateway-telegram-gateway-py.md — Telegram Gateway  (telegram_gateway.py)
- /papers/local-agent-v3/sections/05-video-engine-video-engine-py.md — Video Engine  (video_engine.py)
- /papers/local-agent-v3/sections/06-complete-execution-graph.md — Complete Execution Graph
- /papers/local-agent-v3/sections/07-configuration-reference.md — Configuration Reference
- /papers/local-agent-v3/sections/08-known-failure-modes-guards.md — Known Failure Modes & Guards
- /papers/local-agent-v3/sections/09-recommended-scaffolding-directions.md — Recommended Scaffolding Directions
- /papers/local-agent-v3/sections/10-installation-sequence.md — Installation Sequence

## Exports
- /exports/manifest.json
- /exports/chunks.ndjson
