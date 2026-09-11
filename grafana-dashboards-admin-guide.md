# Grafana — Admin Guide

**Owner:** Platform Engineering (Observability squad)
**Cluster:** prod-east
**Last reviewed:** 2026-02-18
**Audience:** Platform admins, dashboard authors

## Overview

Grafana is the visualization layer over Prometheus (metrics), Loki (logs), and Thanos
(long-term metrics via MinIO-backed object storage). Deployed as a Deployment
(2 replicas, no local state — dashboards are provisioned via ConfigMaps + the Grafana
API, not stored on disk) in namespace `observability`.

Public URL: `grafana.internal.example.com` (Traefik, SSO required, no anonymous access).

## Deployment

- Namespace: `observability`
- Helm chart: `grafana/grafana` pinned to `8.4.2`
- Replicas: 2 (stateless, session store is Redis-backed for HA login sessions)
- Database: uses the shared `observability-postgres` instance for dashboard/user/org
  metadata (NOT sqlite — sqlite would break with 2 replicas)

## Datasources (provisioned, not click-ops)

Datasources are defined in `infra/helm/grafana/datasources.yaml` and provisioned on
pod startup. Do not add datasources through the UI — they won't survive a redeploy and
will drift from what's in git.

| Datasource       | Type       | URL                                             |
|--------------------|------------|---------------------------------------------------|
| Prometheus         | prometheus | http://prometheus-server.observability:9090        |
| Thanos-Query        | prometheus | http://thanos-query.observability:10902             |
| Loki                | loki       | http://loki-gateway.observability:3100              |

Default datasource for new dashboards is `Thanos-Query` (gives you both recent and
long-term data without switching).

## Dashboard Provisioning and Folders

Dashboards live as JSON in `infra/grafana-dashboards/` in the platform-infra repo,
organized by folder:

```
infra/grafana-dashboards/
├── platform-ingress/       # Traefik dashboards
├── platform-storage/       # MinIO, PVC usage
├── platform-build/         # Nexus, CI pipeline metrics
├── platform-cluster/       # node/pod resource usage, k8s API server health
└── app-teams/              # per-team dashboards, self-service via PR
```

A `grafana-dashboard-sync` CronJob runs every 5 minutes, diffs this directory against
what's loaded via the Grafana API, and applies changes. App teams can add their own
dashboards under `app-teams/<team-name>/` via PR — no platform ticket needed for that
path specifically.

Folder permissions:

- `Platform / *` folders — Editor access restricted to `platform-admins`, Viewer open
  to all authenticated users
- `app-teams/*` folders — each team owns Editor rights on their own subfolder via SSO
  group mapping

## Key Dashboards

- **Traefik — Edge Overview** (`platform-ingress`) — request rate, latency percentiles,
  TLS failures, 5xx ratio per router
- **MinIO — Cluster Health** (`platform-storage`) — drive status, erasure set health,
  per-bucket request latency
- **Nexus — Build Artifact Traffic** (`platform-build`) — proxy cache hit ratio per
  repo, pull volume, disk usage trend
- **Cluster Overview** (`platform-cluster`) — node CPU/mem, pod restarts, PVC usage
  cluster-wide
- **Prometheus — Self Monitoring** (`platform-cluster`) — scrape duration, TSDB
  head series count, WAL size (useful when Prometheus itself is under pressure)

## Alerting (Grafana-managed alerts vs Prometheus Alertmanager)

We use Prometheus + Alertmanager as the source of truth for paging alerts (see the
Prometheus admin guide). Grafana's own unified alerting is used ONLY for a small set
of dashboard-adjacent, non-paging notifications (posted to Slack, not PagerDuty) —
mainly slow-burn trend alerts like "Nexus disk usage trending to full in 7 days."
Don't duplicate a Prometheus alert into Grafana; pick one system per alert.

## Common Issues

1. **New dashboard PR merged but not showing up** — check the sync CronJob actually
   ran: `kubectl logs -n observability -l job-name=grafana-dashboard-sync --tail=50`.
   Also verify the JSON validates; a malformed dashboard JSON gets silently skipped
   by the sync job (known gap, logged as a backlog item to make it fail loudly).
2. **"Datasource not found" errors on a dashboard** — dashboard JSON is referencing a
   datasource by hardcoded UID instead of by name/template variable; re-export the
   dashboard using the datasource variable pattern documented in the repo README.
3. **Login redirect loop through SSO** — usually a clock skew issue between Grafana
   pods and the SSO provider, or a stale session in Redis; check
   `kubectl exec -n observability deploy/grafana-redis -- redis-cli INFO server` for
   uptime vs. pod restart time.
4. **Panels showing "No data" only for the last few minutes** — check Thanos-Query is
   healthy; recent data sometimes hasn't been shipped from Prometheus's local TSDB to
   the MinIO-backed long-term bucket yet, this is expected and self-resolves within a
   couple minutes as the sidecar uploads blocks.

## Related Docs

- Prometheus Monitoring Admin Guide (data source + alerting rules)
- MinIO Object Storage Admin Guide (Thanos long-term storage backend)
- Traefik Ingress Admin Guide (grafana.internal.example.com routing)
