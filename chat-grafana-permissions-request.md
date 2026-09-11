# #platform-support — Grafana editor access for catalog team dashboards

**Channel:** #platform-support
**Date:** 2026-07-09**

---

**bsingh (catalog team)** [01:15 PM]
hey, trying to edit a dashboard under `app-teams/catalog/` but getting a permissions
error in the Grafana UI, "you don't have permission to edit this dashboard." I thought
our team had editor rights on our own folder?

**wgardner (platform)** [01:19 PM]
you should — what's your SSO group show as? can you paste a screenshot of your Grafana
profile page (top right -> profile)

**bsingh (catalog team)** [01:22 PM]
[image attached]
shows `Catalog-Team` under organizations but no role listed under teams

**wgardner (platform)** [01:26 PM]
ah, that's the issue — your Okta group is `Catalog-Team` but our SSO group mapping
config expects `catalog-team` lowercase, so the mapping to the Grafana team (which
grants Editor on the folder) silently doesn't match. this bit another team a few
months back too, we should really make this mapping case-insensitive

**bsingh (catalog team)** [01:27 PM]
oh interesting, is that something you can fix or does it need to go through okta admins

**wgardner (platform)** [01:35 PM]
I can patch the Grafana side (add the mapping for both cases) without touching okta,
give me a few min

**wgardner (platform)** [01:44 PM]
done — added `Catalog-Team` as an additional mapped group alongside `catalog-team`.
can you log out and back in to force a fresh SSO token? group mappings only get
re-evaluated on login

**bsingh (catalog team)** [01:48 PM]
logged back in, I've got editor rights now, dashboard saves fine. thank you!

**wgardner (platform)** [01:49 PM]
np — filed a ticket to actually fix this properly (case-insensitive group matching)
so it stops happening team by team, will update this thread when it ships

**tvo** [02:30 PM]
tagging onto this thread since it's Grafana permissions related — is there a way to
get read-only access to the `Platform / Storage` folder without going through a full
onboarding ticket? just want to glance at MinIO dashboards occasionally

**wgardner (platform)** [02:34 PM]
that one's actually open to all authenticated users already (Viewer, not Editor) —
if you're not seeing it in your folder list it's probably just not starred/favorited,
try browsing to Dashboards -> Platform -> Storage directly

**tvo** [02:36 PM]
ah there it is, my bad — thanks
