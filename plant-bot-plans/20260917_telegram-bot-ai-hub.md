# Feature: Telegram Bot for ai-hub

**Date:** 2026-09-17  
**Type:** Feature  
**Repo:** ai-hub  
**Status:** PENDING EXECUTION

---

## Goal

Telegram bot that lets any Telegram user chat with ai-hub's LLM from their phone. Bot runs as a k3s pod, calls `ai-hub:8000` internally.

---

## Inspect Findings

- Endpoint: `POST ai-hub:8000/v1/chat/completions`
- Auth: `Authorization: Bearer <api_key>` — test keys exist in `app/auth.py`
  - `photo_key_test_123456789` (photographer client)
  - `market_key_test_987654321` (marketer client)
- Request body: `{"message": "<str>", "conversation_id": "<uuid>"}`
- Response: SSE stream — `data: {"delta": {"content": "token"}}` chunks
- Telegram doesn't support native streaming → collect full response, then send (or edit progressively)
- conversation_id management: one UUID per Telegram `chat_id`, stored in-memory (resets on pod restart — acceptable MVP)
- Rate limit: 10 RPM per client_id in ai-hub

---

## SDLC Stages

| Stage | Status |
|-------|--------|
| 1. Inspect | ✅ Done |
| 2. Plan | ✅ Done — this file |
| 3. Implement | ⏳ Pending |
| 4. Write tests | ⏳ Pending |
| 5. Tests pass | ⏳ Pending |
| 6. Security review | ⏳ Pending |
| 7. Update docs | ⏳ Pending |
| Push to main | ⏳ Pending |

---

## Prerequisites (user action needed before implement)

- Create Telegram bot via @BotFather → `BOT_TOKEN`
- Decide which ai-hub client role to use (photographer or marketer) — or add a new "telegram" client

---

## Steps

1. Create bot directory + app [ PENDING ]

   File: `telegram-bot/bot.py`
   - Uses `python-telegram-bot` v21 (async, polling)
   - On `/start`: create new `conversation_id` UUID, greet user
   - On any text message:
     1. Get or create `conversation_id` for this `chat_id`
     2. Send "typing..." action
     3. `POST http://ai-hub:8000/v1/chat/completions` with SSE streaming
     4. Collect all delta chunks → full response string
     5. Reply to user with full response
   - On `/reset`: clear `conversation_id` → start fresh conversation

   File: `telegram-bot/requirements.txt`
   - `python-telegram-bot==21.*`
   - `httpx` (async HTTP with streaming)

2. Create k8s Secret manifest stub [ PENDING ]

   File: `infra/k3s/telegram-bot-secret.yaml` — **NOT committed** (gitignored, created on server)
   - `BOT_TOKEN` — from @BotFather
   - `AI_HUB_API_KEY` — the chosen seed key

3. Create k3s Deployment manifest [ PENDING ]

   File: `infra/k3s/telegram-bot-deployment.yaml`
   - Deployment: `python:3.13-slim`, runs `bot.py`
   - Env from secret: `BOT_TOKEN`, `AI_HUB_API_KEY`
   - Env: `AI_HUB_URL=http://ai-hub:8000`
   - No ports needed (polling only, no incoming connections)
   - Resources: 64Mi/50m requests, 256Mi/200m limits

4. Create Dockerfile [ PENDING ]

   File: `telegram-bot/Dockerfile`
   - FROM python:3.13-slim
   - COPY requirements.txt + bot.py
   - RUN pip install -r requirements.txt
   - CMD python bot.py

5. Create k3s secret on server + deploy [ PENDING ]

   ```bash
   sudo kubectl create secret generic telegram-bot-secret \
     --from-literal=BOT_TOKEN=<token> \
     --from-literal=AI_HUB_API_KEY=photo_key_test_123456789 \
     --namespace=ai-hub
   ```

   Then build image + import into k3s (same pattern as ai-hub):
   ```bash
   docker build -t telegram-bot:latest telegram-bot/
   docker save telegram-bot:latest | sudo k3s ctr images import -
   sudo kubectl apply -f infra/k3s/telegram-bot-deployment.yaml -n ai-hub
   ```

6. Test the bot manually [ PENDING ]

   - Send `/start` → should greet
   - Send a message → should reply with ai-hub LLM response
   - Send `/reset` → should start fresh conversation

7. Write automated tests [ PENDING ]

   - Unit test: SSE parser (collect delta chunks → string)
   - Integration test: mock ai-hub, verify bot sends correct POST body

8. Security review [ PENDING ]

   - BOT_TOKEN in k8s secret, not env/code ✓
   - API key in k8s secret ✓
   - Bot only calls ai-hub internally (no external endpoints other than Telegram API)
   - Rate limit: ai-hub already enforces 10 RPM — bot inherits this

9. Update docs [ PENDING ]

   - Add Telegram bot section to `infra/README.md`

10. Push to main [ PENDING ]

---

## Expected Outcome

- Telegram user sends message → ai-hub LLM responds
- Conversation history maintained per chat_id
- `/reset` command clears context
- Runs entirely on k3s, no external infra needed beyond Tailscale + Telegram API

---

## Future

- Persist conversation_id to Redis (survives pod restarts)
- Multi-client support (different Telegram users → different client roles)
- Streaming simulation (edit message progressively with partial tokens)
