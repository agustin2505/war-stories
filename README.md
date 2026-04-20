# War Stories

Notes on concrete problems I had to solve while shipping production infrastructure. These are not tutorials — they are narratives: context, what failed, what I tried, why it did not work, the aha moment, and the lesson.

The focus is on the **reasoning**, not the snippet. Snippets exist to illustrate, not to copy-paste.

## Recurring stack

- **GCP** (multi-tenant landing zone with [Cloud Foundation Fabric / FAST](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric))
- **GKE** private cluster with Workload Identity
- **Terraform** + **Helm** (OCI charts on Artifact Registry)
- **GitHub Actions** as CD, authentication via Workload Identity Federation
- **Cloud SQL**, **Secret Manager**, **Artifact Registry**, **VPC Service Controls**
- Apps deployed: FastAPI backend, Next.js frontend, Apache Airflow 3, Langfuse

Context: an AI / RAG platform for financial services. Compliance with ISO 27001, BCRA.

---

## Layout

```
war-stories/
|- _template.md              # Template for new stories
|- _drafts/                  # Raw notes (not polished yet)
|- projects/                 # Stories grouped by project
|  \- banking-ai-platform/
|- by-topic/                 # Indexes by topic (generated)
```

---

## Stories by project

### banking-ai-platform — AI/RAG platform on GKE for financial services

| # | Story | Stack | Severity |
|---|-------|-------|----------|
| 01 | [The catch-22 of Helm pre-install hooks](./projects/banking-ai-platform/01-helm-pre-install-hook-catch-22.md) | helm, gke, workload-identity | high |

---

## Stories by topic

- [helm](./by-topic/helm.md) — Helm hooks, OCI charts, atomic rollbacks, ownership
- [gke](./by-topic/gke.md) — private GKE, Connect Gateway, node pools with taints
- [gitops](./by-topic/gitops.md) — CD with WIF, multi-branch, detect-changes

---

## Why I write this

When you google real infrastructure problems, most results are "hello world" tutorials. The hard problems — the ones you hit in production, the ones that only show up when the pieces interact — are almost undocumented. When you solve them at 2 AM, they tend to stay in your head and in a commit message.

This is an attempt to leave a trail.

---

## How I add new stories

1. When something weird comes up during debugging, I drop raw notes into `_drafts/topic.md`. 2 minutes, no formatting.
2. At the end of the sprint I review `_drafts/`, pick the ones worth it, promote them to `projects/<project>/NN-slug.md` using `_template.md`.
3. I refresh the `by-topic/` indexes using the front-matter tags.
