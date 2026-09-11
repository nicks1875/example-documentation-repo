# #platform-support — Traefik cert stuck pending

**Channel:** #platform-support
**Date:** 2026-05-14

---

**jmartinez** [09:12 AM]
hey folks, we just added a new IngressRoute for `reports.internal.example.com` and it's
been sitting on the default self-signed cert for like 20 min now. anyone seen this
before?

**jmartinez** [09:13 AM]
```
spec:
  routes:
    - match: Host(`reports.internal.example.com`)
      kind: Rule
      services:
        - name: reports-svc
          port: 8080
  tls:
    certResolver: letsencrypt-dns
```

**dpham (platform)** [09:16 AM]
yeah that resolver name looks right. can you check the traefik logs for acme errors?

```
kubectl logs -n traefik-system -l app.kubernetes.io/name=traefik --tail=200 | grep -i acme
```

**jmartinez** [09:19 AM]
got this:

```
level=error msg="Unable to obtain ACME certificate for domains \"reports.internal.example.com\": unable to generate a certificate for the domains [reports.internal.example.com]: error: one or more domains had a problem:
[reports.internal.example.com] acme: error: 403 :: urn:ietf:params:acme:error:unauthorized :: no results returned for the R53 hosted zone lookup" providerName=letsencrypt-dns.acme
```

**dpham (platform)** [09:21 AM]
ah yep, classic. that domain is under `internal.example.com` right? not the old
`.net` zone?

**jmartinez** [09:22 AM]
oh. hm. let me check DNS

**jmartinez** [09:25 AM]
😅 yeah it got created under internal.example.net by accident, someone must have
copy-pasted an old terraform module. fixing now

**dpham (platform)** [09:31 AM]
np, that .net zone really needs to just get deleted at some point, it keeps biting
people. I'll open a ticket

**jmartinez** [09:44 AM]
fixed, cert issued fine once the record was in the right zone. thanks!

**dpham (platform)** [09:45 AM]
👍 for what it's worth this exact failure mode is in the traefik admin guide under
"Common Runbook Items" now, added a note about the deprecated zone specifically

---

**smehta** [11:02 AM]
unrelated question in this thread since it's already about traefik — is there a way
to see current TLS handshake failure rate per router? trying to debug something on
our end

**dpham (platform)** [11:05 AM]
yep, Grafana dashboard "Traefik — Edge Overview" under Platform / Ingress folder, there's
a panel for TLS handshake failures broken out by router. if you don't have viewer
access to that folder let me know and I'll add you

**smehta** [11:06 AM]
found it, thanks!
