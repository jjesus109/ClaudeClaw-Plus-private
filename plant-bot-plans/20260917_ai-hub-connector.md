# Feature: ai-hub-connector — Platform-Agnostic Messaging Service

**Date:** 2026-09-17  
**Type:** New Service / New Repo  
**Repo:** ai-hub-connector (new standalone repo)  
**Status:** PENDING EXECUTION

---

## Goal

New standalone service that connects messaging platforms (Telegram now, WhatsApp/Discord future) to ai-hub's LLM API. Single connector pod, one API key per platform, Redis for session persistence.

---

## Architecture

```
[Telegram user]
      │ message
      ▼
[ai-hub-connector pod]
  ├── platforms/telegram.py  (polls Telegram API)
  ├── platforms/base.py       (Platform ABC)
  ├── router.py               (on_message → ai_hub_client)
  ├── session.py              (Redis: platform:user_id → conversation_id)
  └── ai_hub_client.py        (POST /v1/chat/completions, collect SSE)
      │
      ▼
[ai-hub pod] → [Ollama]
```

**Design decisions locked:**
- Redis session storage (existing cluster Redis, persists across restarts)
- Single connector pod (all platforms in one deployment)
- One API key per platform (telegram gets `CLIENT_TELEGRAM_KEY` from ai-hub auth)
- Full SSE collect → single reply (no streaming simulation)

---

## Inspect Findings

- ai-hub endpoint: `POST http://ai-hub:8000/v1/chat/completions`
- Auth: `Authorization: Bearer <api_key>`
- Request: `{"message": "<str>", "conversation_id": "<uuid>"}`
- Response: SSE `data: {"delta": {"content": "token"}}` chunks until stream ends
- Redis already running in cluster at `redis-service:6379` (no password currently)
- Telegram Bot API: polling (no webhook needed for k3s, avoids public endpoint)
- `python-telegram-bot` v21 supports async polling natively

---

## SDLC Stages

| Stage | Status |
|-------|--------|
| 1. Inspect | ✅ Done |
| 2. Plan | ✅ Done — this file |
| 3. Implement | ✅ Done — connector built + deployed to k3s, Telegram bot polling |
| 4. Write tests | ⏳ Pending |
| 5. Tests pass | ⏳ Pending |
| 6. Security review | ✅ Done — no secrets in source, k8s secret only, no inbound ports |
| 7. Update docs | ✅ Done — README.md written |
| Push to main | ⏳ Pending — needs GitHub repo created at github.com/new (jjesus109/ai-hub-connector, private) |

---

## Prerequisites

- ai-hub auth refactor done (plan: `20260917_ai-hub-auth-refactor.md`) — needs `CLIENT_TELEGRAM_KEY`
- Telegram bot token from @BotFather (`BOT_TOKEN`)
- GitHub repo `ai-hub-connector` created under `jjesus109`

---

## Repo Structure

```
ai-hub-connector/
  connector/
    __init__.py
    ai_hub_client.py     # async HTTP client, SSE stream collector
    session.py           # Redis: get/set conversation_id per platform:user_id
    router.py            # receives (platform, user_id, text) → calls ai-hub → returns text
    platforms/
      __init__.py
      base.py            # Platform ABC: start(), stop(), on_message hook
      telegram.py        # TelegramPlatform using python-telegram-bot
  main.py                # loads enabled platforms from env, starts event loop
  requirements.txt
  Dockerfile
infra/k3s/
  connector-deployment.yaml
README.md
```

---

## Steps

1. Create GitHub repo `ai-hub-connector` [ PENDING ]

   ```bash
   gh repo create jjesus109/ai-hub-connector --private --description "Platform-agnostic messaging connector for ai-hub"
   ```

2. Implement `connector/ai_hub_client.py` [ PENDING ]

   ```python
   async def chat(message: str, conversation_id: str, api_key: str) -> str:
       # POST to AI_HUB_URL/v1/chat/completions
       # Stream SSE response, collect delta chunks
       # Return full assembled string
   ```

3. Implement `connector/session.py` [ PENDING ]

   ```python
   # key format: "connector:{platform}:{user_id}"
   async def get_or_create_conversation_id(platform: str, user_id: str) -> str
   async def reset_conversation(platform: str, user_id: str) -> None
   ```

4. Implement `connector/platforms/base.py` [ PENDING ]

   ```python
   class Platform(ABC):
       @abstractmethod
       async def start(self, on_message: Callable) -> None: ...
       @abstractmethod
       async def send(self, user_id: str, text: str) -> None: ...
       @abstractmethod
       async def stop(self) -> None: ...
   ```

5. Implement `connector/platforms/telegram.py` [ PENDING ]

   - `TelegramPlatform(token, api_key)` 
   - `/start` → reset + greet
   - `/reset` → clear conversation_id, confirm
   - Any text → `router.handle(platform="telegram", user_id=chat_id, text=message)`
   - Send "typing..." action while waiting
   - Reply with full response

6. Implement `connector/router.py` [ PENDING ]

   ```python
   async def handle(platform: str, user_id: str, text: str) -> str:
       conv_id = await session.get_or_create_conversation_id(platform, user_id)
       api_key = PLATFORM_KEYS[platform]  # from env
       response = await ai_hub_client.chat(text, conv_id, api_key)
       return response
   ```

7. Implement `main.py` [ PENDING ]

   - Read `TELEGRAM_ENABLED` env var (default true if token present)
   - Instantiate enabled platforms
   - Run async event loop with all platforms

8. Write `Dockerfile` [ PENDING ]

   ```dockerfile
   FROM python:3.13-slim
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY . .
   CMD ["python", "main.py"]
   ```

9. Write `infra/k3s/connector-deployment.yaml` [ PENDING ]

   ```yaml
   # Deployment: connector, 1 replica
   # Env:
   #   AI_HUB_URL: http://ai-hub:8000
   #   REDIS_URL: redis://redis-service:6379
   #   TELEGRAM_ENABLED: "true"
   #   TELEGRAM_BOT_TOKEN: secretKeyRef connector-secret telegram-bot-token
   #   TELEGRAM_API_KEY:   secretKeyRef connector-secret telegram-api-key
   # Resources: 64Mi/50m req, 256Mi/200m limits
   # No ports (polling only)
   ```

10. Build + deploy [ PENDING ]

    ```bash
    # On server
    git clone https://github.com/jjesus109/ai-hub-connector.git
    cd ai-hub-connector
    docker build -t ai-hub-connector:latest .
    docker save ai-hub-connector:latest | sudo k3s ctr images import -
    
    sudo kubectl create secret generic connector-secret \
      --from-literal=telegram-bot-token=<BOT_TOKEN> \
      --from-literal=telegram-api-key=<CLIENT_TELEGRAM_KEY> \
      --namespace=ai-hub
    
    sudo kubectl apply -f infra/k3s/connector-deployment.yaml -n ai-hub
    ```

11. Manual test [ PENDING ]

    - Telegram: `/start` → greeting
    - Send message → LLM reply
    - `/reset` → confirm reset
    - Send again → fresh conversation

12. Write automated tests [ PENDING ]

    - Unit: SSE parser
    - Unit: session key format / Redis round-trip (mock Redis)
    - Unit: router calls ai_hub_client with correct args
    - Integration: mock ai-hub server, full message flow

13. Security review [ PENDING ]

    - No secrets in source
    - Bot token + API key in k8s secret only
    - Connector only makes outbound calls (Telegram API + ai-hub) — no inbound ports
    - Redis connection internal only

14. Update docs [ PENDING ]

15. Push to main [ PENDING ]

---

## Env Vars Reference

| Var | Source | Description |
|-----|--------|-------------|
| `AI_HUB_URL` | Plain env | `http://ai-hub:8000` |
| `REDIS_URL` | Plain env | `redis://redis-service:6379` |
| `TELEGRAM_ENABLED` | Plain env | `"true"` / `"false"` |
| `TELEGRAM_BOT_TOKEN` | k8s secret `connector-secret` | From @BotFather |
| `TELEGRAM_API_KEY` | k8s secret `connector-secret` | `CLIENT_TELEGRAM_KEY` from ai-hub |

---

## Future Platforms

Add `WhatsAppPlatform` in `connector/platforms/whatsapp.py`, set `WHATSAPP_ENABLED=true` + token env vars. No other changes needed.
