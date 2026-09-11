# #platform-alerts — KubePodCrashLooping noise from catalog namespace

**Channel:** #platform-alerts
**Date:** 2026-08-19

---

**[BOT] Alertmanager** [03:00 AM]
🔴 **KubePodCrashLooping** (P3)
namespace: catalog
pod: catalog-worker-7d9f6b-x2kqp
restarts: 6 in 15m
https://alertmanager.internal.example.com/#/alerts?...

**[BOT] Alertmanager** [03:15 AM]
🔴 **KubePodCrashLooping** (P3)
namespace: catalog
pod: catalog-worker-7d9f6b-m9plz
restarts: 6 in 15m

**[BOT] Alertmanager** [03:31 AM]
🔴 **KubePodCrashLooping** (P3)
namespace: catalog
pod: catalog-worker-7d9f6b-q7wbn
restarts: 6 in 15m

---

**nfoster (oncall)** [08:02 AM]
morning, picking up oncall — looks like catalog-worker has been crash-looping
overnight, 3 separate pods paged (P3, non-paging thankfully). anyone from catalog
around yet?

**bsingh (catalog team)** [08:41 AM]
here, looking now

**bsingh (catalog team)** [08:50 AM]
yeah this is on us, bad deploy last night around 2:45am — worker pods OOMKilling on
startup because of a memory limit that didn't get bumped after we added a new batch
job type. rolling back now

**nfoster (oncall)** [08:52 AM]
np, want me to ack the alerts in the meantime so they stop re-firing?

**bsingh (catalog team)** [08:53 AM]
yeah please, thanks

**bsingh (catalog team)** [09:15 AM]
rollback done, pods stable. will bump the memory limit properly and redeploy this
afternoon after testing

**nfoster (oncall)** [09:16 AM]
sounds good. quick note for next time — `KubePodCrashLooping` at P3 goes to
#platform-alerts only, doesn't page anyone, so overnight it just sat here for 5+
hours unactioned. might be worth catalog team having their own alert routing for
their namespace if 3am deploys are a regular thing, rather than relying on someone
noticing this channel in the morning

**bsingh (catalog team)** [09:20 AM]
fair, we don't usually deploy at 3am, this was a one-off from an EU contractor's
timezone. but noted, I'll bring up namespace-scoped alert routing with our team lead

**nfoster (oncall)** [09:21 AM]
👍 for reference the alert routing config platform owns lives in
`infra/helm/kube-prometheus-stack/alertmanager-config.yaml` if you want to see how
the P1/P2/P3 tree is structured before proposing something app-team-specific
