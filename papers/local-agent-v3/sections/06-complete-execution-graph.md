---
doc_id: local-agent-v3.06.complete-execution-graph
paper: local-agent-v3
version: 2026-02-23
---

# Complete Execution Graph

This section provides the full decision tree for a text query arriving at the Telegram gateway. A scaffolding agent should treat this as the canonical reference for control flow.

TELEGRAM MESSAGE RECEIVED

│

├─ [AUTH] sender_id in ALLOWED_USER_IDS?

│     NO  → silent drop, log warning

│     YES → continue

│

├─ [CMD] Is it a /command?

│     /start   → hardware status reply

│     /status  → live metrics reply

│     /ingest  → run_in_executor(rag.ingest, path)

│     /video   → VideoEngine.generate() [see VIDEO PATH below]

│     /help    → command list

│

└─ [TEXT] Plain query → agent.run(query) via run_in_executor

│

├─ [CACHE] Hippocampus.recall(query)

│     similarity >= 0.85 AND confidence >= 0.80

│       → return cached response immediately

│     similarity >= 0.85 AND confidence < 0.80

│       → set extra_context = cached response, continue

│     no hit → continue

│

├─ [GATE] SensoryCortex.check_and_wait()

│     temp > 75°C → TDP throttle (if privileged) or sleep

│

├─ [STATE 0] Route (temp=0.1, max=128)

│     → JSON: {needs_rag, rag_query, needs_code, code}

│     → if needs_rag:  RAGStore.search(rag_query) → context

│     → if needs_code: MotorCortex.execute_python(code) → context

│

├─ [GATE] SensoryCortex.check_and_wait()

│

├─ [STATE 1] Draft (temp=0.7, max=1024×scale)

│     input: query + rag_context + soft_hit_memory

│

├─ [GATE] SensoryCortex.check_and_wait()

│

├─ [STATE 2] Critique (temp=0.4, max=768×scale)

│     input: query + draft

│

├─ [GATE] SensoryCortex.check_and_wait()

│

├─ [STATE 3] Synthesis (temp=0.3, max=2048×scale)

│     input: query + draft + critique

│     token_multiplier=2.0

│

├─ [SCORE] ConfidenceScorer.score(query, final)

│     score >= 0.55 → accept

│     score <  0.55 → re-critique → retry STATE 3 (max 2×)

│

├─ [MEMORIZE] Hippocampus.memorize(query, final, confidence=score)

│

└─ → reply to Telegram (chunked if > 4096 chars)

VIDEO PATH (/video <s> <prompt>)

│

├─ PromptRefiner.refine(prompt)  [LLM still loaded]

├─ VRAMManager.acquire()         [unload LLM]

├─ FramePackPipeline.load()      [load video weights]

├─ generate keyframe_start

├─ generate keyframe_end

├─ for each middle section:

│     generate_section() + thermal_gate()

│     checkpoint every 5 sections

├─ decode_latents() → pixel frames

├─ assemble_video() → MP4

├─ FramePackPipeline.unload()

├─ VRAMManager.release()         [LLM reloads lazily]

└─ reply_video() to Telegram
