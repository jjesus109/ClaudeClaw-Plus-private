# Refactor: ai-hub API Keys — Hardcoded Dict → k8s Secrets

**Date:** 2026-09-17  
**Type:** Security Refactor  
**Repo:** ai-hub  
**Status:** PENDING EXECUTION

---

## Goal

Remove hardcoded API keys from `app/auth.py`. Move to k8s secret → env vars. Adds `telegram` client for connector service.

---

## Inspect Findings

Current state (`app/auth.py`):
```python
SEED_CLIENTS = {
    "photographer": {"id": "...", "api_key": "photo_key_test_123456789"},
    "marketer":     {"id": "...", "api_key": "market_key_test_987654321"},
}
```
- Keys hardcoded in source → committed to git → security debt item #N in tech-pendings
- Need to add `telegram` client for ai-hub-connector

Target state:
- Keys read from env vars: `CLIENT_PHOTOGRAPHER_KEY`, `CLIENT_MARKETER_KEY`, `CLIENT_TELEGRAM_KEY`
- k8s secret: `ai-hub-client-keys` → mounted into ai-hub deployment
- Tests: use env var values (or set env in test fixture)

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

## Steps

1. Update `app/auth.py` [ PENDING ]

   - Read keys from env: `os.environ.get("CLIENT_PHOTOGRAPHER_KEY")`, etc.
   - Add telegram client with `CLIENT_TELEGRAM_KEY`
   - Fail fast on startup if any key missing (raise on `None`)
   - Keep same UUIDs for photographer/marketer

2. Update `app/config.py` [ PENDING ]

   - Add `client_photographer_key`, `client_marketer_key`, `client_telegram_key` fields
   - Source from env vars

3. Create k8s secret manifest stub [ PENDING ]

   File: `infra/k3s/ai-hub-client-keys-secret.example.yaml` — documents the shape, NOT the values:
   ```yaml
   # Create on server:
   # sudo kubectl create secret generic ai-hub-client-keys \
   #   --from-literal=CLIENT_PHOTOGRAPHER_KEY=<key> \
   #   --from-literal=CLIENT_MARKETER_KEY=<key> \
   #   --from-literal=CLIENT_TELEGRAM_KEY=<key> \
   #   --namespace=ai-hub
   ```

4. Update `infra/k3s/ai-hub-deployment.yaml` [ PENDING ]

   Add env vars sourced from secret:
   ```yaml
   - name: CLIENT_PHOTOGRAPHER_KEY
     valueFrom:
       secretKeyRef:
         name: ai-hub-client-keys
         key: CLIENT_PHOTOGRAPHER_KEY
   ```
   (same for marketer, telegram)

5. Create k8s secret on server [ PENDING ]

   ```bash
   sudo kubectl create secret generic ai-hub-client-keys \
     --from-literal=CLIENT_PHOTOGRAPHER_KEY=photo_key_test_123456789 \
     --from-literal=CLIENT_MARKETER_KEY=market_key_test_987654321 \
     --from-literal=CLIENT_TELEGRAM_KEY=<new_strong_key> \
     --namespace=ai-hub
   ```

6. Update tests to read from env [ PENDING ]

   - `conftest.py` or test fixtures: set env vars before tests run
   - Existing e2e tests use hardcoded keys → update to read from env

7. Deploy + verify [ PENDING ]

   ```bash
   sudo kubectl apply -f infra/k3s/ai-hub-deployment.yaml -n ai-hub
   sudo kubectl rollout status deployment/ai-hub -n ai-hub
   curl -H "Authorization: Bearer photo_key_test_123456789" http://localhost:30800/healthz
   ```

8. Security review [ PENDING ]

   - Keys no longer in source code ✓
   - k8s secret base64-encoded at rest ✓
   - Add note: rotate keys every 90 days

9. Update docs + tech-pendings [ PENDING ]

   - Remove security debt item for hardcoded keys
   - Add key rotation procedure to infra/README.md

10. Push to main [ PENDING ]
