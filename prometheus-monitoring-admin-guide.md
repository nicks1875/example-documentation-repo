# Prometheus Monitoring — Admin Guide

**Owner:** Platform Engineering (Observability squad)
**Cluster:** prod-east
**Last reviewed:** 2026-03-29
**Audience:** Platform admins, on-call SREs, app teams adding scrape targets

## Overview

Prometheus is deployed via the `kube-prometheus-stack` Helm chart in namespace
`observability`, alongside Alertmanager and node-exporter/kube-state-metrics. Local
TSDB retention is 15 days; long-term storage beyond that is handled by Thanos sidecar
shipping compacted blocks to the `thanos-metrics` MinIO bucket (see MinIO admin guide).

## Deployment

- Namespace: `observability`
- Helm chart: `prometheus-community/kube-prometheus-stack` pinned to `58.2.1`
- Prometheus replicas: 2 (HA pair, both scrape independently, Thanos-Query dedupes)
- Retention (local): 15d
- Retention (Thanos/MinIO): unlimited (no lifecycle policy on `thanos-metrics` bucket)
- Storage per replica: 100Gi PVC (gp3)

```bash
kubectl get statefulset -n observability
kubectl get prometheus -n observability prometheus-kube-prometheus-prometheus -o yaml
```

## Scrape Configuration

We use the Prometheus Operator, so scrape targets are defined via `ServiceMonitor` and
`PodMonitor` CRDs, not static scrape_configs. App teams can self-service add a
ServiceMonitor in their own namespace as long as it's labeled to match the Operator's
selector:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: catalog-api
  namespace: catalog
  labels:
    release: prometheus-kube-prometheus   # required for the operator to pick it up
spec:
  selector:
    matchLabels:
      app: catalog-api
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

Forgetting the `release: prometheus-kube-prometheus` label is the #1 reason a new
ServiceMonitor silently gets ignored — the Operator's `serviceMonitorSelector` is
scoped to that label, not "all ServiceMonitors in the cluster" (this was a deliberate
choice to avoid an app team's misconfigured monitor overloading scrape capacity
cluster-wide).

## Current Platform Scrape Targets

| Target                | Via         | Interval | Notes                                  |
|--------------------------|-------------|----------|-------------------------------------------|
| Traefik                  | ServiceMonitor | 15s   | `/metrics` on port 9100                   |
| Nexus                    | PodMonitor      | 30s   | JMX exporter sidecar, port 9445           |
| MinIO                    | ServiceMonitor | 30s   | `/minio/v2/metrics/cluster`               |
| kube-state-metrics       | built-in        | 30s   | cluster object state                      |
| node-exporter            | DaemonSet + SM  | 15s   | per-node host metrics                     |
| kube-apiserver           | built-in        | 30s   | control plane health                      |

## Key Alert Rules (Alertmanager routes to PagerDuty)

Defined via `PrometheusRule` CRDs in `infra/prometheus-rules/`:

- `TraefikHighErrorRate` — 5xx ratio > 2% over 5m, severity P2
- `MinIODriveOffline` — any drive reporting offline > 2m, severity P2, pages
  immediately
- `NexusDiskPressure` — PVC usage > 85%, severity P3, Slack-only (see Grafana guide,
  this overlaps with a Grafana trend alert intentionally as a belt-and-suspenders case)
- `PrometheusTSDBWALCorruption` — self-monitoring, severity P1
- `KubePodCrashLooping` — any pod restarting > 5x in 15m, severity P3
- `NodeDiskPressure` — kubelet reporting DiskPressure condition, severity P2

Alertmanager routing config lives in `infra/helm/kube-prometheus-stack/alertmanager-config.yaml`.
Route tree is roughly: P1 -> pages platform-oncall immediately + Slack
#platform-incidents; P2 -> pages platform-oncall with 10m delay (allows self-heal);
P3 -> Slack #platform-alerts only, no page.

## Common Issues

1. **New ServiceMonitor not scraping** — check the `release` label (see above). Also
   confirm the target namespace isn't excluded by `serviceMonitorNamespaceSelector` (it
   isn't, by default we watch all namespaces, but double check no one changed that).
2. **High cardinality warnings / TSDB head series growing fast** — usually an app
   exposing a metric with a high-cardinality label (user IDs, request IDs, raw URLs
   instead of route templates). Check `topk(10, count by (__name__)({__name__=~".+"}))`
   in the Prometheus UI to find the worst offenders.
3. **Alertmanager not sending to PagerDuty** — check the PagerDuty integration key
   Secret hasn't expired/rotated without updating Alertmanager's config; also check
   Alertmanager's own `/api/v2/status` for cluster peer health if we're running
   multiple replicas and one is out of sync.
4. **Thanos sidecar failing to upload blocks to MinIO** — see MinIO admin guide's
   related issue; usually an access key rotation that wasn't propagated to the sidecar
   Secret.
5. **Gaps in Grafana dashboards for data older than 15 days** — Thanos-Query datasource
   not configured correctly, or Thanos Store Gateway pod is down; check
   `kubectl get pods -n observability -l app=thanos-store`.

## Related Docs

- Grafana Dashboards Admin Guide (visualization layer, alert routing notes)
- MinIO Object Storage Admin Guide (Thanos long-term storage backend)
- Traefik Ingress Admin Guide (one of the scrape targets described here)
