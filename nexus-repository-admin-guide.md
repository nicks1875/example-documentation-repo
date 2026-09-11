# Nexus Repository Manager — Admin Guide

**Owner:** Platform Engineering (Build Tooling squad)
**Cluster:** prod-east
**Last reviewed:** 2026-01-22
**Audience:** Platform admins, build engineers

## Overview

Nexus Repository OSS 3.x is our internal artifact registry, deployed as a single
StatefulSet in namespace `nexus` with a 500Gi EBS-backed PVC for blob storage. It hosts:

- `docker-hosted` / `docker-proxy` — internal images + Docker Hub proxy cache
- `maven-hosted` / `maven-proxy` — internal Java artifacts + Maven Central proxy
- `npm-hosted` / `npm-proxy` — internal packages + npm registry proxy
- `pypi-hosted` / `pypi-proxy` — internal Python packages + PyPI proxy
- `raw-hosted` — misc binary artifacts (Terraform providers, internal CLI releases)

Nexus sits behind Traefik at `nexus.internal.example.com`, TLS terminated at the edge.

## Deployment

- Namespace: `nexus`
- Workload: StatefulSet `nexus-repo` (single replica — Nexus OSS does not support HA
  clustering, that's a Pro feature we don't license)
- Image: `sonatype/nexus3:3.68.1`
- PVC: `nexus-data` (500Gi, gp3, retained on delete)
- JVM heap: `-Xms4g -Xmx4g` set via `INSTALL4J_ADD_VM_PARAMS` env var

Because it's single-replica with a stateful PVC, restarts cause ~2-3 minutes of
downtime while Nexus reindexes on startup. Schedule restarts outside business hours
when possible (Slack #platform-changes for a heads up).

```bash
kubectl rollout restart statefulset/nexus-repo -n nexus
kubectl rollout status statefulset/nexus-repo -n nexus --timeout=300s
```

## Repository Configuration

Repository definitions are managed through the Nexus UI (Admin -> Repository ->
Repositories) — there is no GitOps/IaC layer for this yet (tracked as a backlog item,
low priority since repo config rarely changes). If you add a new proxy repo, document
it in this guide and announce in #platform-changes.

### Proxy repos and their remote URLs

| Repo             | Type   | Remote URL                              | Cache TTL |
|-------------------|--------|-------------------------------------------|-----------|
| docker-proxy      | docker | https://registry-1.docker.io              | 24h       |
| maven-proxy       | maven2 | https://repo1.maven.org/maven2             | 24h       |
| npm-proxy         | npm    | https://registry.npmjs.org                 | 4h        |
| pypi-proxy        | pypi   | https://pypi.org                           | 24h       |
| terraform-proxy    | raw    | https://releases.hashicorp.com             | 7d        |

npm has a shorter TTL because app teams have been burned before by caching a broken
pre-release version for 24h.

## Group Repositories (what clients should actually point at)

Clients (CI runners, dev laptops, Dockerfiles) should NEVER point directly at a hosted
or proxy repo. Always use the group repo, which merges hosted + proxy:

- `docker-group` -> used as the Docker daemon mirror-registry in CI
- `maven-group` -> `~/.m2/settings.xml` mirror
- `npm-group` -> `.npmrc` registry
- `pypi-group` -> pip.conf index-url

Example `.npmrc` for CI runners:

```
registry=https://nexus.internal.example.com/repository/npm-group/
always-auth=true
//nexus.internal.example.com/repository/npm-group/:_authToken=${NEXUS_NPM_TOKEN}
```

## Authentication

Nexus uses its own internal realm plus an LDAP realm bound to the corporate directory.
Service accounts (CI, robots) are internal-realm only and use tokens, not passwords.
Human users authenticate via LDAP and get role mappings based on AD group membership:

- `nx-admin` <- AD group `Platform-Admins`
- `nx-deployer` <- AD group `Build-Engineers` (push rights to hosted repos)
- `nx-anonymous-read` <- default anonymous read access to proxy/group repos (no push)

## Storage and Cleanup

Blob store `default` backs all repos. It is NOT auto-pruned by default — Nexus keeps
every proxied artifact version forever unless a cleanup policy is attached.

Cleanup policies (Admin -> Repository -> Cleanup Policies):

- `docker-proxy-cleanup`: removes cached images not pulled in 30 days
- `npm-proxy-cleanup`: removes cached packages not pulled in 14 days
- `maven-proxy-cleanup`: removes cached artifacts not pulled in 45 days

These run on a scheduled task ("Cleanup repositories using their associated policy")
nightly at 02:00 UTC. If disk usage on the `nexus-data` PVC exceeds 80%, PagerDuty
alerts `platform-oncall` — check `kubectl exec -n nexus nexus-repo-0 -- df -h /nexus-data`.

## Common Issues

1. **502 from Traefik hitting Nexus** — usually Nexus is still starting up after a
   restart (reindexing). Check `kubectl logs -n nexus nexus-repo-0 | tail -50` for
   `Started Sonatype Nexus`.
2. **npm install fails with 401** — token expired or rotated; CI service account tokens
   rotate every 90 days via the `nexus-token-rotator` CronJob, make sure the CI secret
   was updated in the same window.
3. **Docker pull rate-limited even through proxy** — docker-proxy has its own rate
   limit config against Docker Hub's anonymous pull limits; if we're hitting them
   cluster-wide, it means the proxy cache TTL expired for a popular base image during
   a burst of deploys. Consider pre-warming with a scheduled pull.
4. **Disk pressure on nexus-data PVC** — check cleanup policies actually ran; the
   scheduled task can silently fail if blob store health check fails first.

## Related Docs

- Traefik Ingress Admin Guide (how nexus.internal.example.com is routed and TLS'd)
- MinIO Object Storage Admin Guide (note: Nexus blob storage is local PVC, not MinIO —
  a MinIO-backed blob store has been discussed but not implemented)
