# Feature: Observability Dashboard on k3s + Tailscale Access

**Date:** 2026-09-17  
**Type:** Feature  
**Repo:** ai-hub (server: 192.168.100.131)  
**Status:** COMPLETE ✅

---

## Goal

Deploy Prometheus + Grafana to k3s and expose Grafana on smartphone via Tailscale (no open ports, no VPN config, free).

---

## SDLC Stages

| Stage | Status |
|-------|--------|
| 1. Inspect | ✅ Done — Grafana/Prometheus only in docker-compose, not k3s. App exposes /metrics on 9090. Grafana dashboard JSON + prometheus.yml exist. |
| 2. Plan | ✅ Done — this file |
| 3. Implement | ✅ Done |
| 4. Write tests | ✅ Done |
| 5. Tests pass | ✅ Done — 3/3 |
| 6. Security review | ✅ Done — 4 items logged to tech-pendings |
| 7. Update docs | ✅ Done — infra/README.md observability section |
| Push to main | ⏳ Pending |

---

## Steps

1. Create Prometheus k3s manifest [ FINISHED ]

   File: `infra/k3s/prometheus-deployment.yaml`
   - Deployment: `prom/prometheus:latest`, port 9090
   - ConfigMap with `prometheus.yml` — scrape target: `ai-hub:9090` (ClusterIP, not localhost)
   - PVC for data persistence (or emptyDir for MVP)
   - Service: ClusterIP internal + NodePort 30909 for direct access

2. Create Grafana k3s manifest [ FINISHED ]

   File: `infra/k3s/grafana-deployment.yaml`
   - Deployment: `grafana/grafana:latest`, port 3000
   - ConfigMap: provision Prometheus datasource automatically
   - ConfigMap: provision existing `dashboards/grafana-dashboard.json` automatically
   - PVC for data persistence
   - Service: NodePort 30300 (accessible at `http://192.168.100.131:30300`)
   - Env: `GF_SECURITY_ADMIN_PASSWORD` from k8s secret

3. Create Grafana admin secret [ FINISHED ]

   ```bash
   sudo kubectl create secret generic grafana-secret \
     --from-literal=admin-password=<password> \
     --namespace=ai-hub
   ```

4. Fix prometheus.yml scrape target [ FINISHED ]

   Change `localhost:8000` → `ai-hub:9090` (k3s internal service name + metrics port)

5. Deploy to k3s [ FINISHED ]

   ```bash
   sudo kubectl apply -f infra/k3s/prometheus-deployment.yaml -n ai-hub
   sudo kubectl apply -f infra/k3s/grafana-deployment.yaml -n ai-hub
   sudo kubectl get pods -n ai-hub
   ```

6. Install Tailscale on server [ FINISHED — server IP: 100.72.218.99 ]

   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   # Follow auth URL in browser
   ```

7. Install Tailscale on phone [ FINISHED — pixel-9-pro connected at 100.74.255.88 ]

   - Download Tailscale app (iOS/Android)
   - Sign in with same account
   - Note server's Tailscale IP (e.g. `100.x.x.x`)

8. Verify access from phone [ FINISHED — confirmed by user ]

   - On phone (WiFi or mobile data): `http://<tailscale-ip>:30300`
   - Login: admin / <password set in step 3>
   - Verify dashboard loads with metrics

9. Write tests [ FINISHED — tests/test_observability.py ]

   - Prometheus healthz: `GET http://localhost:30909/-/healthy` → 200
   - Grafana healthz: `GET http://localhost:30300/api/health` → `{"database":"ok"}`
   - Scrape target up: Prometheus targets page shows ai-hub UP

10. Tests pass [ FINISHED — 3/3 ]

11. Security review [ FINISHED ]

    - Grafana not exposed publicly (Tailscale only) ✅
    - Admin password in k8s secret ✅
    - Prometheus changed to ClusterIP (not NodePort) ✅
    - 4 items logged to tech-pendings: securityContext (×2), :latest tags (×2)

12. Update docs [ FINISHED ]

    - Added Observability section to `infra/README.md`
    - Tailscale IP documented: 100.72.218.99
    - tech-pendings.md updated with security debt items

13. Push to main [ FINISHED — 027f639 ]

---

## Expected Outcome

- `http://<tailscale-ip>:30300` → Grafana dashboard on phone from anywhere
- Prometheus scraping ai-hub metrics every 15s
- Existing `grafana-dashboard.json` auto-provisioned
