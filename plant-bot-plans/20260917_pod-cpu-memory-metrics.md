# Feature: Per-pod CPU/Memory Metrics in Grafana

**Date:** 2026-09-17  
**Type:** Feature  
**Repo:** ai-hub  
**Status:** COMPLETE ✅

---

## Goal

Add per-pod CPU and memory usage panels to the Grafana dashboard for all pods in the ai-hub namespace (ai-hub, grafana, prometheus, ollama, redis, postgres).

---

## Inspect Findings

- k3s kubelet exposes cAdvisor metrics at `https://192.168.100.131:10250/metrics/cadvisor`
- Contains `container_cpu_usage_seconds_total` and `container_memory_working_set_bytes` per container/pod
- Access requires: ServiceAccount with ClusterRole granting `nodes/metrics` + `nodes` get/list
- Current Prometheus pod uses `default` SA in `ai-hub` namespace — needs RBAC upgrade
- Node IP: 192.168.100.131, single-node cluster

No kube-state-metrics or node-exporter needed — kubelet cAdvisor is sufficient for CPU/memory.

---

## SDLC Stages

| Stage | Status |
|-------|--------|
| 1. Inspect | ✅ Done |
| 2. Plan | ✅ Done — this file |
| 3. Implement | ✅ Done |
| 4. Write tests | ✅ Done — existing suite covers it |
| 5. Tests pass | ✅ Done — 4/4 |
| 6. Security review | ✅ Done — RBAC read-only, TLS skip acceptable for internal cluster |
| 7. Update docs | ✅ Done — no new docs needed |
| Push to main | ✅ Done — 0df805c |

---

## Steps

1. Create RBAC manifest for Prometheus kubelet scraping [ PENDING ]

   File: `infra/k3s/prometheus-rbac.yaml`
   - ServiceAccount: `prometheus` in `ai-hub`
   - ClusterRole: get nodes, nodes/metrics, nodes/proxy
   - ClusterRoleBinding: bind SA to ClusterRole

2. Update Prometheus deployment to use new ServiceAccount [ PENDING ]

   Add `serviceAccountName: prometheus` under `spec.template.spec` in `prometheus-deployment.yaml`

3. Add kubelet scrape config to prometheus.yml [ PENDING ]

   ```yaml
   - job_name: 'k3s-kubelet-cadvisor'
     scheme: https
     tls_config:
       insecure_skip_verify: true
     bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
     static_configs:
       - targets: ['192.168.100.131:10250']
     metrics_path: /metrics/cadvisor
     metric_relabel_configs:
       - source_labels: [container]
         regex: ''
         action: drop
   ```

4. Deploy RBAC + updated Prometheus [ PENDING ]

5. Add Grafana dashboard panels (Row 6 expansion) [ PENDING ]

   - Per-pod CPU % (timeseries by pod label)
   - Per-pod Memory working set MB (timeseries by pod label)

6. Write tests [ PENDING ]

7. Security review [ PENDING ]

8. Update docs [ PENDING ]

9. Push to main [ PENDING ]
