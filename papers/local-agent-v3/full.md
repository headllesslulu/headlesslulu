---
doc_id: local-agent-v3.full
paper: local-agent-v3
version: 2026-02-23
---

# TECHNICAL SYSTEM REPORT
**Local Autonomous AI Node — v3 Architecture**
## 1.  Executive Summary
This document describes a fully local, zero-API autonomous AI node designed to run on a consumer-class NVIDIA GTX 1660 Super (6GB VRAM, Turing architecture, sm_75). The system has no runtime dependencies on external AI services — no OpenAI, no Anthropic API, no cloud compute. It operates entirely from local hardware.
The node consists of five Python modules spanning approximately 2,500 lines of production code, plus a systemd service definition and credential template. The files are:
The architecture makes one fundamental trade: it exchanges spatial complexity (large parameter counts, large VRAM footprint) for temporal complexity (multi-pass reasoning, iterative refinement, sequential pipeline stages). A quantized 7B model running four deliberate inference passes produces output quality that competes with single-pass outputs from models 3-4× its size.
## 2.  Hardware Constraints & VRAM Budget
All architectural decisions trace back to a single hard constraint: 6GB VRAM on a Turing-generation GPU. This section documents what that constraint means in practice and which features it disables.
### 2.1  VRAM Allocation — LLM Mode
The num_ctx=4096 context window cap in PrefrontalCortex.infer() is a hard limit, not a preference. Exceeding it on this card causes OOM. Do not increase it.
### 2.2  Turing Architecture Limitations
The GTX 1660 Super is sm_75 (Turing). Several commonly cited optimizations are unavailable on this microarchitecture:
Rule for scaffolding agents: never call FP8, BF16, or Flash Attention 2 on this card without probing compute capability first. The video engine contains a reference implementation of the probe pattern in resolve_attention_backend().
### 2.3  VRAM Coexistence — LLM and Video Cannot Share
LLM inference and video generation each require approximately 5.5GB. They cannot run simultaneously. The protocol for switching:
- VRAMManager.acquire() sends POST /api/generate {keep_alive: 0} to the local ollama server, which evicts the LLM from VRAM
- PyTorch GC and cache clear runs after eviction
- Video pipeline loads into the now-available VRAM
- On completion (or crash), VRAMManager.release() clears the video pipeline. Ollama reloads the LLM lazily on the next inference request
This is a cooperative protocol, not a hardware lock. If another process loads into VRAM between acquire() and pipeline load, the video engine will OOM. The recommended deployment is a single-process node where only one workload runs at a time.
## 3.  Core Reasoning Engine  (agent.py)
The reasoning engine is a five-component system orchestrated by a single LocalAgent class. Each component maps to a biological analogue from the original architecture specification.
### 3.1  Component Map
### 3.2  The 4-State Compute Graph
The central innovation is replacing single-pass autoregressive generation with a structured 4-state pipeline. Each state runs as a separate, stateless ollama inference call with a distinct system prompt and temperature setting.
Context isolation: each state receives a clean system+user message pair. No conversation history is passed between states. State 3 receives draft and critique as USER message content, not as prior assistant turns. This prevents the model from anchoring too hard on its own previous token distributions.
State 3 always receives token_multiplier=2.0. Even in hot-zone thermal scaling (50% reduction), synthesis retains at least its base token budget to fully repair the draft.
### 3.3  Thermal-Aware Token Scaling  (v3 feature)
Live GPU temperature maps to a multiplier applied to every inference call. Three zones are defined relative to thermal_limit_c (default 75°C) and token_scale_margin (default 5°C):
Floor values prevent any stage from dropping below functional minimums: route: 64, critique: 128, draft/synthesis: 192 tokens. The zone and budget change are logged on every call where scaling applies.
### 3.4  TDP Control  (v3 feature, optional)
Enabled via --tdp CLI flag. Requires root/Administrator. The agent calls nvidia-smi -pl {tdp_normal_w} once at startup as a privilege probe. If the command returns non-zero or raises FileNotFoundError, _tdp_ok is set False and a loud warning is printed immediately. No code path downstream ever calls nvidia-smi again unless _tdp_ok is True.
On thermal event: TDP throttle fires first (if available), dropping wattage from 125W to 80W. Sleep-based throttle is the fallback. On temperature recovery: TDP is restored to normal explicitly — the GPU does not stay capped after the thermal event clears.
### 3.5  Confidence-Aware Hippocampus  (v2 feature)
Every memorize() call stores the ConfidenceScorer output alongside the response in ChromaDB metadata. Every recall() returns a MemoryResult object containing the stored confidence and an is_trusted boolean.
This creates a self-healing memory: mediocre cached responses are progressively overwritten by better syntheses triggered on subsequent similar queries. The store converges to high-confidence entries through use.
### 3.6  RAGStore  (v2 feature)
A separate ChromaDB collection (rag_documents) stores ingested document chunks. Isolated from agent memory (agent_memory). Shares the SentenceTransformer encoder instance with Hippocampus — loaded once, no double VRAM cost.
- Ingests: .txt .md .rst .csv .json .py .js .ts .html .xml .yaml .toml .pdf
- Word-level chunking: 400 words/chunk, 50-word overlap
- Content-hashed IDs: re-ingesting same file is idempotent (safe to repeat)
- PDF extraction: pdfminer.six with pdftotext CLI fallback
- Retrieval: top-4 chunks by cosine similarity, returned as formatted context string
The router (STATE 0) generates a targeted rag_query string distinct from the raw user query, maximising retrieval precision. The JSON schema uses needs_rag/rag_query not needs_web/search_query.
## 4.  Telegram Gateway  (telegram_gateway.py)
Wraps LocalAgent behind a persistent Telegram bot listener. The agent is initialised once at startup and shared across all handlers, preserving the Hippocampus across conversations.
### 4.1  Security Model
ALLOWED_USER_IDS is the only authentication gate. Messages from unknown sender IDs are silently dropped — no reply, no acknowledgment. Silence gives an attacker no confirmation the bot exists. The whitelist is checked before any processing — unknown IDs never reach the agent.
Credentials live in .env only. Never hardcoded. .gitignore includes .env in the repository root.
### 4.2  Async Architecture
agent.run() is CPU/GPU-bound and can take 30-45 seconds. It runs via loop.run_in_executor(None, self.agent.run, query), pushing it to a thread pool. The event loop remains unblocked: Telegram heartbeats continue, the typing indicator stays active, and a second message during inference is queued rather than dropped.
drop_pending_updates=True in app.run_polling() discards messages received while the daemon was offline. Without this, a 1-hour outage followed by 20 queued messages would trigger 20 sequential inference calls on restart.
### 4.3  Commands
### 4.4  Daemon Deployment  (localagent.service)
The systemd service runs under the user account (not root unless TDP control is needed). Restart=on-failure with RestartSec=10s and StartLimitBurst=3 prevents crash loops while ensuring recovery from transient failures. Logs go to journald: sudo journalctl -u localagent -f.
## 5.  Video Engine  (video_engine.py)
Generates video from text or image+text using FramePack/HunyuanVideo. A 13B parameter model runs within 6GB via aggressive quantization, CPU offload, and section-based generation.
### 5.1  Architecture Overview
Generation is decomposed into five sequential phases, each class-bounded:
### 5.2  Bidirectional Seeding
Unlike purely autoregressive video generation (predict each frame from all previous frames), FramePack seeds both the start and end keyframes first, then infills the middle sections bidirectionally. This is the primary mechanism that maintains structural coherence over 60-second (1800-frame) sequences.
- Step 1: Generate latent_start (section 0, is_keyframe=True, full CFG)
- Step 2: Generate latent_end (section N-1, is_keyframe=True, full CFG)
- Step 3: Infill sections 1 through N-2 bidirectionally, alternating anchor endpoints
- Step 4: Decode all latent tensors to pixel frames (VAE slicing, one frame at a time)
- Step 5: Assemble to MP4 via imageio + ffmpeg
### 5.3  Anti-Drift Guidance Rescaling
Without correction, pixel errors accumulate over 1800 frames (CFG guidance amplifies small artifacts on each denoising step). The anti-drift mechanism rescales guidance_scale for middle sections based on their distance from the nearest keyframe:
midpoint_distance = |( i / (N-1) ) - 0.5| × 2.0
# 0.0 at sequence midpoint, 1.0 at keyframes
scale_factor = drift_strength + (1.0 - drift_strength) × midpoint_distance
guidance     = cfg_scale × scale_factor
# Default: drift_strength=0.7, cfg_scale=7.0
# Keyframes:  guidance = 7.0  (full)
# Midpoint:   guidance = 4.9  (70% of full)
### 5.4  Precision & Attention — Turing Constraints
### 5.5  Checkpointing
Latent tensors are saved to disk every checkpoint_every_n sections (default: 5). On crash, the checkpoint file survives. A future resume path can reload latents and continue from the last saved section. The checkpoint is deleted on successful completion.
### 5.6  Realistic Performance Expectations
Start with 5 seconds, not 60. A 5s clip at 512×320 24fps = 120 frames = ~5 sections. This validates the full pipeline in under an hour. Only increase duration after the 5s clip completes cleanly.
Times are estimates. Actual performance depends on inference_steps (default 25), GPU clock state, thermal throttling frequency, and whether INT8 quantization is active.
## 6.  Complete Execution Graph
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
## 7.  Configuration Reference
### 7.1  AgentConfig  (agent.py)
### 7.2  VideoConfig  (video_engine.py)
## 8.  Known Failure Modes & Guards
## 9.  Recommended Scaffolding Directions
The following extensions have been identified as high-leverage, low-risk additions. Each is ranked by implementation complexity.
## 10.  Installation Sequence
### 10.1  LLM Agent
# 1. Install Python dependencies
pip install ollama chromadb sentence-transformers pynvml rich
pip install python-telegram-bot python-dotenv
# 2. Install and start ollama
curl -fsSL https://ollama.ai/install.sh | sh
ollama serve &
# 3. Pull model (~4.1GB download)
ollama pull qwen2.5:7b-instruct-q4_K_M
# 4. Test terminal REPL
python agent.py
# 5. Configure Telegram credentials
cp .env.template .env
nano .env   # fill TELEGRAM_BOT_TOKEN and ALLOWED_USER_IDS
# 6. Test gateway in foreground
python telegram_gateway.py
# 7. Install as systemd daemon
sudo cp localagent.service /etc/systemd/system/
# Edit service file: replace 'youruser' and paths
sudo systemctl enable --now localagent
sudo journalctl -u localagent -f
### 10.2  Video Engine
# 1. Install PyTorch for CUDA 12.1
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121
# 2. Install video dependencies
pip install diffusers transformers accelerate
pip install bitsandbytes xformers
pip install imageio[ffmpeg] Pillow opencv-python pdfminer.six
# 3. Install FramePack from source
git clone https://github.com/lllyasviel/FramePack
pip install -e ./FramePack
# 4. Download weights (~6GB — follow FramePack README)
#    Place in ./weights/
# 5. Validate with a 5-second test clip
python video_engine.py --prompt 'A candle flame flickering' --duration 5
# 6. Check output
ls ./video_output/
END OF TECHNICAL REPORT
Local Autonomous AI Node v3  —  AI-to-AI Architecture Handoff

