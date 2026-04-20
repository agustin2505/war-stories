---
title: Terraform state drift after a mid-apply network cut
project: banking-ai-platform
date: 2026-04-09
stack: [terraform, gke, gcp]
topics: [state-management, disaster-recovery, debugging]
severity: high
status: draft
---

# Raw notes

## What happened

I was applying `5-workloads-ai` to the new QA environment. The plan had ~100 resources: GKE cluster, 4 node pools, Cloud SQL, multiple GCS buckets, IAM bindings, secrets.

About 30 minutes into the apply, my DNS dropped. `terraform apply` saw the API connection die while it was in the middle of creating node pools. The apply ended in error.

When I reconnected and ran `terraform plan` again, this is what I saw:

- The GKE cluster in the console: **created**.
- The 4 node pools in the console: **all 4 created** and attached to the cluster.
- The Terraform state: **cluster present, 0 node pools present**.

The plan wanted to create 4 node pools. But if I ran `apply`, Terraform would call the API and GCP would reject: "node pool with name X already exists on cluster Y".

## The trap

First instinct: delete the node pools in GCP and let Terraform recreate them. This works. But:

- Each node pool took ~8 minutes to create.
- Deleting and recreating 4 node pools in sequence would waste ~30 more minutes.
- There is **no data in them yet** — but there will be, and I needed to know the right procedure for future incidents when there will be.

## The right procedure: `terraform import`

`terraform import` tells Terraform "this resource exists in the real world at address X, and it maps to configuration Y in my code". It writes into the state without calling the create API.

```bash
# For each orphaned node pool:
terraform import \
  'module.gke.google_container_node_pool.pools["system"]' \
  'projects/itmind-macro-ai-qa-0/locations/us-east1-b/clusters/macro-ai-qa-gke/nodePools/system-pool'

terraform import \
  'module.gke.google_container_node_pool.pools["orchestration"]' \
  'projects/itmind-macro-ai-qa-0/locations/us-east1-b/clusters/macro-ai-qa-gke/nodePools/orchestration-pool'

# ... and the other two
```

After the imports, `terraform plan` showed:

- 4 node pools in state, matching reality.
- A handful of attributes the config computed differently from what was in GCP (e.g., upgrade settings with defaults vs explicit).
- I could either update the config to match reality, or accept those as small diffs to apply on the next run.

## The subtle detail: the resource address

The address in `terraform import` has to match **exactly** the position in your module tree. If your resource lives inside a module with a `for_each`, the address is `module.foo.resource_type.name["key"]`. One wrong bracket and import fails silently or imports into the wrong slot.

The way I confirmed the right address:

```bash
terraform state list | grep node_pool
# shows the addresses Terraform expects for the node pools
# (before the failed apply, they existed in state — now they don't)
```

And then matching against the `main.tf`:

```hcl
resource "google_container_node_pool" "pools" {
  for_each = var.node_pools
  # ...
}
```

So the address is `module.gke.google_container_node_pool.pools["<key>"]`.

## Why it matters

- In multi-tenant landing zones, partial-apply scenarios happen. Network drops, quota limits, org policies rejecting the last resource — any of them can leave you half-applied.
- Deleting the partial resources and retrying is fine **before** production traffic exists. Once it exists, you do not have that option — state has to be reconciled without touching the live resource.
- `terraform import` is the answer, but it only works if you know the exact resource address and the exact import ID format (it varies per resource type — check the provider docs).

## Takeaways to develop

- Keep the state file on a remote backend with versioning (we use a GCS bucket with object versioning enabled). If state corrupts, you can roll back to the last good version.
- Before a long apply, save a copy of `terraform state list` output locally. If something goes wrong, you have a reference of what state used to look like.
- For resources with `for_each`, the `key` in the import address is the map key, not the resource name. Wrong key = silent wrong import.

## To include in the polished story

- Table mapping GCP resource type → import ID format (node pool, cluster, SQL instance, etc.). There is no universal pattern.
- Mention that some providers have a `terraform import` block (declarative, in HCL) instead of the CLI command. Less error-prone, version-controllable.
