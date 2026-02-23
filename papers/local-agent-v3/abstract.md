---
doc_id: local-agent-v3.abstract
paper: local-agent-v3
version: 2026-02-23
---

# Abstract

This document describes a fully local, zero-API autonomous AI node designed to run on a consumer-class NVIDIA GTX 1660 Super (6GB VRAM, Turing architecture, sm_75). The system has no runtime dependencies on external AI services — no OpenAI, no Anthropic API, no cloud compute. It operates entirely from local hardware. The node consists of five Python modules spanning approximately 2,500 lines of production code, plus a systemd service definition and credential template. The files are:
