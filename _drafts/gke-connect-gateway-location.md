---
title: GKE_REGION with Connect Gateway is not a region — it is a zone
project: banking-ai-platform
date: 2026-04-16
stack: [github-actions, gke, connect-gateway, fleet]
topics: [debugging, cryptic-errors, ci-cd]
severity: low
status: draft
---

# Raw notes

## The error

In the CD step:
```yaml
- name: Get GKE credentials (via Connect Gateway)
  uses: google-github-actions/get-gke-credentials@v2
  with:
    cluster_name: ${{ secrets.GKE_CLUSTER_QA }}
    location: ${{ secrets.GKE_REGION_QA }}
    use_connect_gateway: true
```

With `GKE_REGION_QA=us-east1` (the region where the cluster lives):
```
not found: projects/PROJECT/locations/***/clusters/***
```

I thought: "ok, the cluster is zonal, not regional". Switch to `global` (because the Fleet membership is at `locations/global`):
```
location "***" does not exist
```

Switch to `us-east1-b` (the actual zone of the cluster):
works.

## Why it is confusing

- The secret name: **GKE_REGION**. Suggests you want a region.
- With `use_connect_gateway: true`, I would expect `location` to refer to the Fleet membership, which lives in `global`. But no.
- The action uses that parameter to look up the cluster in `projects/{p}/locations/{l}/clusters/{c}`. For a zonal cluster, that is the zone. Period.

## The conclusion

`location` in `get-gke-credentials@v2` is always the GKE cluster location (zone for zonal, region for regional), regardless of whether you use Connect Gateway or not. Connect Gateway only changes how the kubeconfig is generated, not how the cluster is resolved.

Renaming the secret to `GKE_LOCATION_*` instead of `GKE_REGION_*` would be less confusing. (I did not do it because the pattern already existed in the prod CD.)

## Takeaways to develop

- Parameter names lie sometimes — read the action docs.
- With Fleet + Connect Gateway there are TWO locations: the membership one (global) and the physical cluster one (zone/region). The actions use the second.
- Error messages mask secrets with `***` — you cannot see the value you sent. To debug you need to go to the logs before the mask or log the value on purpose in an intermediate step (carefully).

## To include in the polished story

- Two different errors that confuse because both mention location but for different reasons.
- Show the 3 attempts side by side.
