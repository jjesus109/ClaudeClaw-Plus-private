# Feature: Message Debouncing — Wait for User to Finish Typing

**Date:** 2026-09-18  
**Type:** Feature Enhancement  
**Repo:** ai-hub-connector  
**Status:** PENDING EXECUTION

---

## Problem

User sends 5 messages rapidly in response to a bot question. Bot replies to each individually. Conversation fragments into 5 separate Q&A pairs instead of one coherent exchange.

---

## Solution

Debounce per user: buffer incoming messages, start a timer on each one. If another message arrives before the timer fires, cancel and restart it. When the timer fires (no new messages for N seconds), flush the buffer as a single concatenated message and call ai-hub once.

```
user: "I sell t-shirts"     → buffer: ["I sell t-shirts"], timer starts (3s)
user: "mostly online"       → buffer: ["I sell t-shirts", "mostly online"], timer resets
user: "on instagram"        → buffer: [..., "on instagram"], timer resets
--- 3s silence ---
→ flush: "I sell t-shirts\nmostly online\non instagram"
→ single router.handle() call → single reply
```

---

## Design Decisions

- **Debounce window:** 70 seconds (configurable via `DEBOUNCE_SECONDS` env var, default 70)
- **Storage:** in-memory dict per platform+user_id — no Redis needed (pod-local, ephemeral)
- **Max buffer:** 10 messages or 2000 chars — safety cap before forced flush
- **Typing indicator:** send on first message arrival (user sees bot is "receiving"), resend on flush
- **Implementation location:** `connector/platforms/telegram.py` — debounce logic stays at platform layer, router stays clean

---

## SDLC Stages

| Stage | Status |
|-------|--------|
| 1. Inspect | ✅ Done |
| 2. Plan | ✅ Done — this file |
| 3. Implement | ✅ Done — 71ffbbd |
| 4. Write tests | ✅ Done — tests/test_debounce.py (9 tests) |
| 5. Tests pass | ✅ Done — 9/9 |
| 6. Security review | ✅ Done — `get_event_loop` → `get_running_loop` fix applied |
| 7. Update docs | ✅ Done — README debounce section added |
| Push to main | ⏳ Pending |

---

## Architecture

```
connector/
  debounce.py          ← NEW: MessageDebouncer class
  platforms/
    telegram.py        ← UPDATE: use debouncer in handle_text
```

---

## Steps

1. Create `connector/debounce.py` [ PENDING ]

   ```python
   import asyncio
   import os
   from typing import Callable, Awaitable

   DEBOUNCE_SECONDS = float(os.environ.get("DEBOUNCE_SECONDS", "70"))
   MAX_BUFFER_MESSAGES = 10
   MAX_BUFFER_CHARS = 2000

   class MessageDebouncer:
       def __init__(self, on_flush: Callable[[str, str, str], Awaitable[None]]):
           # on_flush(platform, user_id, combined_text)
           self._on_flush = on_flush
           self._buffers: dict[str, list[str]] = {}   # key → messages
           self._timers: dict[str, asyncio.TimerHandle] = {}

       def _key(self, platform: str, user_id: str) -> str:
           return f"{platform}:{user_id}"

       async def receive(self, platform: str, user_id: str, text: str) -> None:
           key = self._key(platform, user_id)
           buf = self._buffers.setdefault(key, [])
           buf.append(text)

           # Cancel existing timer
           if key in self._timers:
               self._timers[key].cancel()

           # Force flush if buffer too large
           total_chars = sum(len(m) for m in buf)
           if len(buf) >= MAX_BUFFER_MESSAGES or total_chars >= MAX_BUFFER_CHARS:
               await self._flush(platform, user_id)
               return

           # Schedule flush
           loop = asyncio.get_event_loop()
           self._timers[key] = loop.call_later(
               DEBOUNCE_SECONDS,
               lambda: asyncio.ensure_future(self._flush(platform, user_id))
           )

       async def _flush(self, platform: str, user_id: str) -> None:
           key = self._key(platform, user_id)
           self._timers.pop(key, None)
           buf = self._buffers.pop(key, [])
           if not buf:
               return
           combined = "\n".join(buf)
           await self._on_flush(platform, user_id, combined)
   ```

2. Update `connector/platforms/telegram.py` [ PENDING ]

   - Import `MessageDebouncer`
   - In `start()`, instantiate debouncer with an async flush handler
   - Flush handler: send typing action → call `on_message(user_id, combined)` → reply
   - `handle_text` just calls `debouncer.receive(...)` — no direct router call
   - Send typing indicator on first message receipt (before debounce fires) so user sees bot is active

   ```python
   from ..debounce import MessageDebouncer

   async def _flush_handler(user_id: str, text: str) -> None:
       await ctx_bot.send_chat_action(chat_id=int(user_id), action=ChatAction.TYPING)
       response = await on_message(user_id, text)
       await ctx_bot.send_message(chat_id=int(user_id), text=response or "(no response)")

   debouncer = MessageDebouncer(
       on_flush=lambda platform, uid, text: _flush_handler(uid, text)
   )

   async def handle_text(update, ctx):
       user_id = str(update.effective_chat.id)
       await debouncer.receive("telegram", user_id, update.message.text)
   ```

3. Add `DEBOUNCE_SECONDS` to deployment [ PENDING ]

   Add to `connector-deployment.yaml`:
   ```yaml
   - name: DEBOUNCE_SECONDS
     value: "3"
   ```

4. Write tests [ PENDING ]

   - Unit: single message flushes after debounce window
   - Unit: two rapid messages → one flush with combined text
   - Unit: max buffer cap forces immediate flush
   - Unit: two users → independent timers (no cross-contamination)

5. Build + redeploy [ PENDING ]

   ```bash
   cd ~/Projects/ai-hub-connector
   git pull
   docker build -t ai-hub-connector:latest .
   docker save ai-hub-connector:latest | sudo k3s ctr images import -
   sudo kubectl apply -f infra/k3s/connector-deployment.yaml -n ai-hub
   sudo kubectl rollout restart deployment/ai-hub-connector -n ai-hub
   ```

6. Manual test [ PENDING ]

   - Send 3 rapid messages → bot waits → single reply to all 3
   - Send 1 message → bot replies after 3s
   - Send 10 messages fast → cap forces flush before timer

---

## Env Vars Reference

| Var | Default | Description |
|-----|---------|-------------|
| `DEBOUNCE_SECONDS` | `70` | How long to wait after last message before flushing |
