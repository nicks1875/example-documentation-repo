# #data-science-help — need a MinIO bucket for feature store exports

**Channel:** #data-science-help
**Date:** 2026-04-28

---

**ppatel (data science)** [10:03 AM]
hi platform team, we want to start dumping our feature store parquet exports
somewhere other than local disk on the training nodes — heard MinIO is the move for
this? what's the process to get a bucket

**wgardner (platform)** [10:07 AM]
yep! open a platform-infra ticket with: bucket name, owning team, rough size/growth
estimate, and whether you want a lifecycle/expiry policy on it. we don't do
self-service bucket creation right now, every bucket needs an owner on record

**ppatel (data science)** [10:09 AM]
makes sense. size is maybe 200GB now growing ~20GB/week, and honestly a 30-day expiry
would be perfect since we re-export from source on a rolling basis anyway

**wgardner (platform)** [10:15 AM]
ticket looks good, I'll spin this up today. name it `ml-feature-store-parquet`?

**ppatel (data science)** [10:16 AM]
works for me

**wgardner (platform)** [11:02 AM]
bucket's up, 30-day lifecycle policy attached. here's a scoped access key for your
training pipeline's service account (DM'd the secret, don't post it here obviously)

```
endpoint: https://minio.internal.example.com
bucket:   ml-feature-store-parquet
policy:   read+write, scoped to this bucket only
```

**ppatel (data science)** [11:10 AM]
perfect, testing now

**ppatel (data science)** [11:24 AM]
getting AccessDenied on PutObject even with the key you gave me, here's what I'm
running:

```python
import boto3
s3 = boto3.client('s3',
    endpoint_url='https://minio.internal.example.com',
    aws_access_key_id='ml-feature-store-svc',
    aws_secret_access_key='***')
s3.put_object(Bucket='ml-feature-store-parquet', Key='test.parquet', Body=b'x')
```

**wgardner (platform)** [11:29 AM]
hm, let me check the policy attachment

**wgardner (platform)** [11:34 AM]
found it — I attached the policy to the user but forgot the actual `mc admin policy
attach` step, the user existed with no policy bound so it was defaulting to deny-all.
fixed now, try again

**ppatel (data science)** [11:36 AM]
that did it, writes working now. thanks for the quick turnaround!

**wgardner (platform)** [11:37 AM]
👍 and heads up, our Grafana "MinIO — Cluster Health" dashboard has per-bucket
request metrics if you ever want to keep an eye on your write volume/errors without
pinging us
