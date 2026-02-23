---
doc_id: local-agent-v3.10.installation-sequence
paper: local-agent-v3
version: 2026-02-23
---

# Installation Sequence

## 10.1  LLM Agent <a id="10-1-llm-agent"></a>

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

## 10.2  Video Engine <a id="10-2-video-engine"></a>

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
