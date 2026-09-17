# Feature: Grafana Dashboard v2 — Model, RPM, CPU/Mem, Users

**Date:** 2026-09-17  
**Type:** Feature  
**Repo:** ai-hub  
**Status:** COMPLETE ✅

---

## Goal

Replace existing Grafana dashboard with one that shows: model usage, RPM per endpoint, CPU/memory for ai-hub, user usage per client_id, queue health, circuit breaker state.

---

## Inspect Findings

Available metrics in Prometheus today:
- `http_requests_total{client_id, endpoint, status}` — RPM, user usage
- `http_request_duration_seconds{client_id, endpoint}` — latency
- `queue_depth` — queue gauge
- `queue_processed_total{client_id, status}` — queue throughput
- `circuit_breaker_state` — 0=CLOSED, 1=OPEN, 2=HALF_OPEN
- `process_cpu_seconds_total` / `process_resident_memory_bytes` — ai-hub process
- `chat_requests_total{client_id, tactic_used}` — model/tactic usage (counter, 0 yet — no values until chat fires)
- `model_fallback_total{model_type, client_id}` — fallback usage (same)
- `model_errors_total{model, client_id}` — errors per model (same)

Gaps:
- Per-pod CPU/memory for ALL pods: needs kube-state-metrics + node-exporter (future)
- Model usage counters exist in code but haven't been observed yet (panels added, will populate on first chat)

---

## SDLC Stages

| Stage | Status |
|-------|--------|
| 1. Inspect | ✅ Done |
| 2. Plan | ✅ Done — this file |
| 3. Implement | ✅ Done |
| 4. Write tests | ✅ Done — test_grafana_dashboard_provisioned added |
| 5. Tests pass | ✅ Done — 4/4 |
| 6. Security review | ✅ Done — no secrets/injection vectors, datasource UID hardcoded (acceptable) |
| 7. Update docs | ✅ Done — no doc changes needed (infra/README already covers dashboard) |
| Push to main | ✅ Done |

---

## Steps

1. Update `dashboards/grafana-dashboard.json` with new panels [ PENDING ]

   Panels:
   - **Row 1 — Traffic:** RPM (rate/1m by endpoint), total requests gauge, error rate %
   - **Row 2 — Latency:** P50/P95/P99 histogram_quantile by endpoint
   - **Row 3 — Users:** requests by client_id (bar chart), rate per user
   - **Row 4 — Models:** chat requests by tactic_used, fallback rate, model errors
   - **Row 5 — Queue:** queue depth gauge, queue processed rate, queue error rate
   - **Row 6 — System (ai-hub):** CPU %, resident memory MB
   - **Row 7 — Health:** circuit breaker state, up gauge

2. Update `grafana-dashboard-json` ConfigMap on k3s [ PENDING ]

   ```bash
   sudo kubectl create configmap grafana-dashboard-json \
     --from-file=grafana-dashboard.json=dashboards/grafana-dashboard.json \
     --namespace=ai-hub --dry-run=client -o yaml | sudo kubectl apply -f -
   sudo kubectl rollout restart deployment/grafana -n ai-hub
   ```

3. Verify dashboard loads on Grafana [ PENDING ]

4. Write test: dashboard panels exist via Grafana API [ PENDING ]

5. Security review [ PENDING ]

6. Update docs [ PENDING ]

7. Push to main [ PENDING ]

---

## Future (not in scope)

- kube-state-metrics + node-exporter for per-pod k8s metrics
- Prometheus scrape for Ollama metrics
