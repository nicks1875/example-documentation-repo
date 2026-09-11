# Traefik Ingress — Admin Guide

**Owner:** Platform Engineering
**Cluster:** prod-east (k8s 1.29)
**Last reviewed:** 2026-03-11
**Audience:** Platform admins, on-call SREs

## Overview

Traefik is deployed as the primary ingress controller for the `prod-east` and `stage-east`
clusters. It terminates TLS at the edge, routes traffic to backend Services via
IngressRoute CRDs, and integrates with cert-manager for automated certificate issuance
via Let's Encrypt (DNS-01 challenge through Route53).

Traefik runs as a DaemonSet across the `ingress` node pool (3 nodes, tainted
`role=ingress:NoSchedule`) fronted by an internal AWS NLB. Dashboard access is restricted
to the `platform-admins` group via the internal SSO proxy.

## Namespace and Helm Release

- Namespace: `traefik-system`
- Helm chart: `traefik/traefik` pinned to `27.0.2` (App version 3.1.2)
- Values file: `infra/helm/traefik/values-prod-east.yaml`
- Release name: `traefik-edge`

To check current deployed values:

```bash
helm get values traefik-edge -n traefik-system
```

To upgrade after a values change, always dry-run first:

```bash
helm upgrade traefik-edge traefik/traefik \
  -n traefik-system \
  -f infra/helm/traefik/values-prod-east.yaml \
  --dry-run --debug

helm upgrade traefik-edge traefik/traefik \
  -n traefik-system \
  -f infra/helm/traefik/values-prod-east.yaml
```

## IngressRoute Pattern

All application teams define their own `IngressRoute` objects in their own namespaces.
Platform team only owns the entrypoints, middlewares, and TLS store. Example minimal
IngressRoute:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: catalog-api
  namespace: catalog
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`catalog.internal.example.com`)
      kind: Rule
      services:
        - name: catalog-api-svc
          port: 8080
      middlewares:
        - name: default-ratelimit
          namespace: traefik-system
  tls:
    certResolver: letsencrypt-dns
```

## Middlewares Provided by Platform

| Middleware               | Namespace       | Purpose                                   |
|---------------------------|-----------------|--------------------------------------------|
| `default-ratelimit`       | traefik-system  | 100 req/s average, burst 200               |
| `strip-internal-headers`  | traefik-system  | Removes `X-Internal-*` headers on egress   |
| `redirect-to-https`       | traefik-system  | HTTP -> HTTPS 301 redirect                 |
| `basic-auth-admin`        | traefik-system  | htpasswd auth for internal admin UIs       |
| `mtls-service-mesh`       | traefik-system  | Requires client cert for east-west routes  |

Requesting a new shared middleware goes through a platform-infra ticket; app teams
should not fork these into their own namespaces unless there's a strong reason (it
creates drift and makes audits harder).

## Certificate Resolver Configuration

`letsencrypt-dns` resolver config lives in the Helm values under
`certificatesResolvers`. It uses the `route53` DNS provider plugin. Credentials are
mounted from the `traefik-route53-creds` Secret (rotated quarterly by the security
team, next rotation 2026-06-01).

Common failure mode: if a new IngressRoute's Host doesn't resolve a cert within ~90
seconds, check:

```bash
kubectl logs -n traefik-system -l app.kubernetes.io/name=traefik --tail=200 | grep -i acme
```

Look for `unable to obtain ACME certificate` — this is almost always a Route53 IAM
permission or a hosted zone mismatch (double check the subdomain is actually under
`internal.example.com` and not accidentally under the old `internal.example.net` zone
which is deprecated but not yet deleted).

## Dashboard Access

The Traefik dashboard is exposed internally only, behind `basic-auth-admin` AND SSO:

```
https://traefik-dashboard.internal.example.com/dashboard/
```

Do not expose the dashboard API (`/api`) externally under any circumstances — it leaks
full routing topology and backend service names.

## Health Checks and Monitoring

Traefik exposes Prometheus metrics on port `9100` at `/metrics`. These are scraped by
the cluster's Prometheus instance (see the Prometheus admin guide for the scrape config
and alerting rules — alert `TraefikHighErrorRate` fires on 5xx ratio > 2% over 5m).

Grafana dashboard: **"Traefik — Edge Overview"** (folder: `Platform / Ingress`), shows
request rate, p50/p95/p99 latency by router, and TLS handshake failures.

## Common Runbook Items

1. **503s cluster-wide on one route** — check the backend Service's Endpoints aren't
   empty (`kubectl get endpoints -n <ns> <svc>`); Traefik will happily route to nothing
   and return 503 if the Service has no ready pods.
2. **Cert stuck in pending** — see the ACME troubleshooting section above.
3. **Dashboard 403** — confirm the requester is in the `platform-admins` Okta group;
   group sync to the SSO proxy runs every 15 minutes.
4. **Sudden latency spike on one router** — check if a `default-ratelimit` middleware is
   throttling; look for `429` status codes in the access logs before assuming backend
   issues.

## Related Docs

- Nexus Repository Admin Guide (artifact registry Traefik fronts internally)
- Prometheus Monitoring Admin Guide (scrape config, alert definitions)
- Grafana Dashboards Admin Guide (dashboard provisioning and folder ACLs)
