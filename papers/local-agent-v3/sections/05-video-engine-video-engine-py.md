---
doc_id: local-agent-v3.05.video-engine-video-engine-py
paper: local-agent-v3
version: 2026-02-23
---

# Video Engine  (video_engine.py)

Generates video from text or image+text using FramePack/HunyuanVideo. A 13B parameter model runs within 6GB via aggressive quantization, CPU offload, and section-based generation.

## 5.1  Architecture Overview <a id="5-1-architecture-overview"></a>

Generation is decomposed into five sequential phases, each class-bounded:

## 5.2  Bidirectional Seeding <a id="5-2-bidirectional-seeding"></a>

Unlike purely autoregressive video generation (predict each frame from all previous frames), FramePack seeds both the start and end keyframes first, then infills the middle sections bidirectionally. This is the primary mechanism that maintains structural coherence over 60-second (1800-frame) sequences.

- Step 1: Generate latent_start (section 0, is_keyframe=True, full CFG)

- Step 2: Generate latent_end (section N-1, is_keyframe=True, full CFG)

- Step 3: Infill sections 1 through N-2 bidirectionally, alternating anchor endpoints

- Step 4: Decode all latent tensors to pixel frames (VAE slicing, one frame at a time)

- Step 5: Assemble to MP4 via imageio + ffmpeg

## 5.3  Anti-Drift Guidance Rescaling <a id="5-3-anti-drift-guidance-rescaling"></a>

Without correction, pixel errors accumulate over 1800 frames (CFG guidance amplifies small artifacts on each denoising step). The anti-drift mechanism rescales guidance_scale for middle sections based on their distance from the nearest keyframe:

midpoint_distance = |( i / (N-1) ) - 0.5| × 2.0

# 0.0 at sequence midpoint, 1.0 at keyframes

scale_factor = drift_strength + (1.0 - drift_strength) × midpoint_distance

guidance     = cfg_scale × scale_factor

# Default: drift_strength=0.7, cfg_scale=7.0

# Keyframes:  guidance = 7.0  (full)

# Midpoint:   guidance = 4.9  (70% of full)

## 5.4  Precision & Attention — Turing Constraints <a id="5-4-precision-attention-turing-constraints"></a>

## 5.5  Checkpointing <a id="5-5-checkpointing"></a>

Latent tensors are saved to disk every checkpoint_every_n sections (default: 5). On crash, the checkpoint file survives. A future resume path can reload latents and continue from the last saved section. The checkpoint is deleted on successful completion.

## 5.6  Realistic Performance Expectations <a id="5-6-realistic-performance-expectations"></a>

Start with 5 seconds, not 60. A 5s clip at 512×320 24fps = 120 frames = ~5 sections. This validates the full pipeline in under an hour. Only increase duration after the 5s clip completes cleanly.

Times are estimates. Actual performance depends on inference_steps (default 25), GPU clock state, thermal throttling frequency, and whether INT8 quantization is active.
