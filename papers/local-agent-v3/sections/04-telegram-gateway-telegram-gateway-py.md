---
doc_id: local-agent-v3.04.telegram-gateway-telegram-gateway-py
paper: local-agent-v3
version: 2026-02-23
---

# Telegram Gateway  (telegram_gateway.py)

Wraps LocalAgent behind a persistent Telegram bot listener. The agent is initialised once at startup and shared across all handlers, preserving the Hippocampus across conversations.

## 4.1  Security Model <a id="4-1-security-model"></a>

ALLOWED_USER_IDS is the only authentication gate. Messages from unknown sender IDs are silently dropped — no reply, no acknowledgment. Silence gives an attacker no confirmation the bot exists. The whitelist is checked before any processing — unknown IDs never reach the agent.

Credentials live in .env only. Never hardcoded. .gitignore includes .env in the repository root.

## 4.2  Async Architecture <a id="4-2-async-architecture"></a>

agent.run() is CPU/GPU-bound and can take 30-45 seconds. It runs via loop.run_in_executor(None, self.agent.run, query), pushing it to a thread pool. The event loop remains unblocked: Telegram heartbeats continue, the typing indicator stays active, and a second message during inference is queued rather than dropped.

drop_pending_updates=True in app.run_polling() discards messages received while the daemon was offline. Without this, a 1-hour outage followed by 20 queued messages would trigger 20 sequential inference calls on restart.

## 4.3  Commands <a id="4-3-commands"></a>

## 4.4  Daemon Deployment  (localagent.service) <a id="4-4-daemon-deployment-localagent-service"></a>

The systemd service runs under the user account (not root unless TDP control is needed). Restart=on-failure with RestartSec=10s and StartLimitBurst=3 prevents crash loops while ensuring recovery from transient failures. Logs go to journald: sudo journalctl -u localagent -f.
