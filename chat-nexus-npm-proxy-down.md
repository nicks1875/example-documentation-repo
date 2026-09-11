# #platform-incidents — npm installs failing in CI

**Channel:** #platform-incidents
**Date:** 2026-06-03

---

**alee** [02:41 PM]
CI is red across basically every JS repo, `npm ci` failing with 401s. anyone else
seeing this or is it just us?

**alee** [02:41 PM]
```
npm error code E401
npm error 401 Unauthorized - GET https://nexus.internal.example.com/repository/npm-group/lodash - Unable to authenticate, need: Basic realm="Sonatype Nexus Repository Manager"
```

**rkowalski (platform-oncall)** [02:43 PM]
looking

**rkowalski (platform-oncall)** [02:47 PM]
found it — `nexus-token-rotator` CronJob rotated the CI service account token at 2am
like normal but the sync to the CI secrets manager failed silently, so CI's been
using a stale token since then. rotator logs show a 500 from the secrets manager API
that nobody caught

**alee** [02:48 PM]
ouch. any ETA?

**rkowalski (platform-oncall)** [02:52 PM]
manually re-synced the token, CI should pick it up on next run. can someone confirm?

**alee** [02:55 PM]
confirmed, green now. thank you!

**rkowalski (platform-oncall)** [02:56 PM]
np — going to add alerting on rotator sync failures so this doesn't sit for 12+ hours
next time, filing a follow-up

**tvo** [03:10 PM]
side note while we're on this — is docker-proxy also affected by the same rotator or
is that a separate token lifecycle?

**rkowalski (platform-oncall)** [03:12 PM]
separate — docker pulls through nexus use anonymous read for docker-group (it's a
public-ish mirror internally), only push access and the npm/pypi proxies require
authenticated tokens. so no, unaffected

**tvo** [03:13 PM]
good to know, thanks

---

**jchen** [04:20 PM]
post-incident question: are the npm proxy cache TTLs long enough that we'd have kept
serving cached packages even during the 401 window, or does every request re-auth
regardless of cache state?

**rkowalski (platform-oncall)** [04:25 PM]
every request re-auths against npm-group even if the underlying npm-proxy cache entry
is warm — the auth check happens at the group repo level before cache lookup. so no,
cached packages didn't save us here. might be worth reconsidering that if we want auth
to fail open for reads during token issues, will bring it up with the build tooling
squad
