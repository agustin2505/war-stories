# War Stories

Notes on concrete problems I had to solve while shipping production infrastructure. These are not tutorials — they are narratives: context, what failed, what I tried, why it did not work, the aha moment, and the lesson.

The focus is on the **reasoning**, not the snippet. Snippets exist to illustrate, not to copy-paste.

## Recurring stack

- **GCP** (multi-tenant landing zone with [Cloud Foundation Fabric / FAST](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric))
- **GKE** private cluster with Workload Identity
- **Terraform** + **Helm** (OCI charts on Artifact Registry)
- **GitHub Actions** as CD, authentication via Workload Identity Federation
- **Cloud SQL**, **Secret Manager**, **Artifact Registry**, **VPC Service Controls**
- Apps deployed: FastAPI backend, Apache Airflow 3, Langfuse

Context: an AI / RAG platform for financial services. Compliance with ISO 27001, BCRA.

---

## Layout

```
war-stories/
├── _template.md              # Template for new stories
├── _drafts/                  # Raw notes (not polished yet)
├── projects/                 # Stories grouped by project
│   └── banking-ai-platform/
└── by-topic/                 # Indexes by topic
```

---

## Stories by project

### banking-ai-platform — AI/RAG platform on GKE for financial services

| # | Story | Stack | Severity |
|---|-------|-------|----------|
| 01 | [The catch-22 of Helm pre-install hooks](./projects/banking-ai-platform/01-helm-pre-install-hook-catch-22.md) | helm · gke · workload-identity · cloud-sql | high |
| 02 | [Workload Identity Federation for multi-branch CD pipelines](./projects/banking-ai-platform/02-wif-multi-branch-cd.md) | terraform · github-actions · wif · oidc | medium |
| 03 | [The GKE_REGION secret that is actually a zone](./projects/banking-ai-platform/03-gke-connect-gateway-location.md) | github-actions · gke · connect-gateway · fleet | low |
| 04 | [Silent Airflow rollout — the pod ran the old image for 16 days](./projects/banking-ai-platform/04-airflow-silent-rollout.md) | airflow · gke · helm · docker | medium |
| 05 | [Cloud SQL and pgvector — the extension Terraform cannot install](./projects/banking-ai-platform/05-pgvector-extension-not-managed-by-terraform.md) | cloud-sql · postgresql · pgvector · terraform | low |
| 06 | [Org policies blocking a new environment — overriding at the right layer](./projects/banking-ai-platform/06-org-policies-by-folder.md) | gcp · org-policies · terraform · fast-framework | high |
| 07 | [Terraform state drift after a mid-apply network cut](./projects/banking-ai-platform/07-terraform-import-partial-state.md) | terraform · gke · gcp | high |

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
