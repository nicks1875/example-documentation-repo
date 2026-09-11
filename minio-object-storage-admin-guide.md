# MinIO Object Storage — Admin Guide

**Owner:** Platform Engineering (Storage squad)
**Cluster:** prod-east
**Last reviewed:** 2026-04-02
**Audience:** Platform admins, app teams needing S3-compatible storage

## Overview

MinIO provides S3-compatible object storage for workloads that need it in-cluster
rather than going to actual AWS S3 — mainly for cost reasons on high-churn, low-value
data (build caches, Prometheus long-term-storage blocks via Thanos, Loki chunks) and
for local dev/stage parity with S3 APIs.

Deployed as a 4-node distributed MinIO cluster (erasure coding, EC:2, tolerates 2 node
loss) in namespace `minio-system`, backed by local NVMe on dedicated storage nodes
(`role=storage:NoSchedule` taint).

## Deployment Topology

- Namespace: `minio-system`
- StatefulSet: `minio` (4 replicas)
- Per-pod storage: 4x 2TB NVMe volumes (16 drives total across the cluster)
- Service: `minio` (S3 API, port 9000), `minio-console` (web console, port 9001)
- Operator: MinIO Operator `v5.0.14` manages the Tenant CRD

```bash
kubectl get tenant -n minio-system
kubectl describe tenant minio -n minio-system
```

## Buckets in Use

| Bucket                 | Owning team        | Purpose                                  | Lifecycle policy      |
|--------------------------|---------------------|--------------------------------------------|-------------------------|
| `thanos-metrics`         | Platform / Observability | Prometheus long-term storage blocks   | none (retained)         |
| `loki-chunks`            | Platform / Observability | Log chunk storage for Loki            | 90-day expiry           |
| `ci-build-cache`         | Build Tooling       | Bazel/Gradle remote cache                 | 14-day expiry           |
| `catalog-image-uploads`  | Catalog team        | User-uploaded product images (staging only) | none                 |
| `ml-feature-store-parquet` | Data Science team | Feature store parquet exports            | 30-day expiry           |

Bucket creation requests go through a platform-infra ticket. We do not allow
self-service bucket creation via mc/console currently — every bucket needs an owning
team, a lifecycle policy decision, and a bucket policy review.

## Access and IAM

MinIO's IAM is separate from AWS IAM — it's MinIO's own policy engine. Access keys are
issued per-application, not shared. Policies follow least-privilege:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::ci-build-cache",
        "arn:aws:s3:::ci-build-cache/*"
      ]
    }
  ]
}
```

Apply with the `mc` client against the MinIO alias:

```bash
mc alias set prod-minio https://minio.internal.example.com $MINIO_ROOT_USER $MINIO_ROOT_PASSWORD
mc admin policy create prod-minio ci-build-cache-rw ./policies/ci-build-cache-rw.json
mc admin user add prod-minio ci-build-svc <generated-secret>
mc admin policy attach prod-minio ci-build-cache-rw --user=ci-build-svc
```

Root credentials (`MINIO_ROOT_USER`/`MINIO_ROOT_PASSWORD`) live in the `minio-root-creds`
Secret and are only ever used by platform admins for bootstrap/break-glass — application
workloads always get scoped access keys.

## TLS and Ingress

MinIO S3 API and console are both routed through Traefik at:

- `minio.internal.example.com` (S3 API, port 9000 backend)
- `minio-console.internal.example.com` (web console, port 9001 backend, SSO + basic-auth
  middleware applied since the console has full admin capability if root creds leak)

## Monitoring

MinIO exposes Prometheus metrics at `/minio/v2/metrics/cluster` — scraped by the
cluster Prometheus. Grafana dashboard **"MinIO — Cluster Health"** (folder:
`Platform / Storage`) shows per-drive usage, erasure set health, and API request
latency. Alert `MinIODriveOffline` pages platform-oncall immediately (P2) since we're
only tolerant of 2 drive losses per erasure set before write availability degrades.

## Common Issues

1. **`mc` commands hang or time out** — check the `minio` Service has all 4 pod
   endpoints ready; a single pod being NotReady is usually fine (erasure coding
   tolerates it) but check `kubectl get pods -n minio-system` for CrashLoopBackOff
   first.
2. **"Server not initialized" errors after a node replacement** — new node's drives
   need to be formatted/joined into the existing erasure set; this should happen
   automatically on Tenant reconcile but sometimes needs a manual
   `kubectl delete pod` to kick the operator.
3. **Bucket policy denies access unexpectedly** — remember MinIO evaluates bucket
   policy AND user policy; a bucket-level `Deny` will override an otherwise-permissive
   user policy. Check both with `mc admin policy info` and `mc anonymous get`.
4. **Thanos can't write blocks to `thanos-metrics`** — usually an expired/rotated
   access key that wasn't updated in the Thanos sidecar's Secret; check
   `kubectl logs -n observability -l app=thanos-sidecar | grep -i "access denied"`.

## Related Docs

- Prometheus Monitoring Admin Guide (Thanos long-term storage uses `thanos-metrics`
  bucket described here)
- Traefik Ingress Admin Guide (routing/TLS for both MinIO endpoints)
