# War Stories

Notes on concrete problems I had to solve while shipping production infrastructure. These are not tutorials — they are narratives: context, what failed, what I tried, why it did not work, the aha moment, and the lesson.

The focus is on the **reasoning**, not the snippet. Snippets exist to illustrate, not to copy-paste.

## Recurring stack

- **GCP** (multi-tenant landing zone with [Cloud Foundation Fabric / FAST](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric))
- **GKE** private cluster with Workload Identity, Dataplane V2 (Cilium)
- **Terraform** + **Helm** (OCI charts on Artifact Registry)
- **GitHub Actions** as CD, authentication via Workload Identity Federation
- **Cloud SQL**, **Secret Manager**, **Artifact Registry**, **VPC Service Controls**, **Shared VPC**
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
| 05 | [Org policies blocking a new environment — overriding at the right layer](./projects/banking-ai-platform/05-org-policies-by-folder.md) | gcp · org-policies · terraform · fast-framework | high |
| 06 | [Terraform state drift after a mid-apply network cut](./projects/banking-ai-platform/06-terraform-import-partial-state.md) | terraform · gke · gcp | high |
| 07 | [IAM authoritative vs additive on shared projects](./projects/banking-ai-platform/07-iam-authoritative-vs-additive-on-hub-projects.md) | gcp · iam · terraform · fast-framework | high |
| 08 | [NetworkPolicy on GKE with Cilium — evaluated post-DNAT](./projects/banking-ai-platform/08-cilium-network-policy-post-dnat.md) | gke · cilium · network-policy · airflow | medium |
| 09 | [StatefulSet volumeClaimTemplates are immutable](./projects/banking-ai-platform/09-statefulset-volumeclaimtemplates-immutable.md) | kubernetes · statefulset · helm | medium |
| 10 | [Shared VPC and cross-project NEG references](./projects/banking-ai-platform/10-cross-project-neg-shared-vpc.md) | gcp · shared-vpc · load-balancing · gke | high |
| 11 | [SSD_TOTAL_GB quota includes pd-balanced — and GKE defaults to it](./projects/banking-ai-platform/11-ssd-quota-includes-pd-balanced.md) | gcp · gke · quotas · terraform | medium |
| 12 | [Two org policies, overlapping names, one HMAC key](./projects/banking-ai-platform/12-two-org-policies-one-hmac-key.md) | gcp · org-policies · hmac · iam · fast-framework | high |
| 13 | [The Altinity operator that watched nothing](./projects/banking-ai-platform/13-altinity-operator-watch-namespaces.md) | clickhouse · altinity · kubernetes · operator | high |
| 14 | [The Helm chart that does not migrate what it does not deploy](./projects/banking-ai-platform/14-langfuse-external-clickhouse-migrations.md) | helm · langfuse · clickhouse · kubernetes | medium |
| 15 | [The JTI claim that wasn't there — dead code that passed code review](./projects/banking-ai-platform/15-jwt-jti-dead-code.md) | jwt · redis · fastapi · banking · owasp | high |
| 16 | [Pentesting your own banking RAG before the bank does](./projects/banking-ai-platform/16-pentesting-banking-rag-meta.md) | pentesting · owasp · banking · fastapi · jwt · rag · llm | high |
| 17 | [The AI auditor hallucinated a CVE that does not exist](./projects/banking-ai-platform/17-ai-auditor-hallucinated-cve.md) | ai-auditing · dependencies · supply-chain · python · uv | medium |
| 18 | [I broke my own pentest scope — the cost of "agility"](./projects/banking-ai-platform/18-broke-my-own-pentest-scope.md) | pentesting · methodology · ethics · banking | high |
| 19 | [403 vs 404 — when better UX leaks your tenant boundary](./projects/banking-ai-platform/19-403-vs-404-enumeration-tradeoff.md) | security · multi-tenant · http · fastapi · banking | medium |
| 21 | [Fail-closed turned an LLM-as-guardrail into a single point of failure](./projects/banking-ai-platform/21-fail-closed-llm-guardrail-dos.md) | llm · guardrails · gemini · vertex-ai · sre · llm10 | high |
| 22 | [The four security controls that passed code review and never ran](./projects/banking-ai-platform/22-dead-code-security-controls.md) | security · testing · runtime-vs-static · regulated | high |
| 23 | [The redaction that wasn't there enough](./projects/banking-ai-platform/23-redaction-that-wasnt-there-enough.md) | logging · pii · structlog · regulated · shift-left | high |
| 24 | [Your pentest suite isn't done when the tests pass](./projects/banking-ai-platform/24-pentest-suite-not-done-when-tests-pass.md) | github-actions · pytest · testcontainers · ci · branch-protection | medium |

---

## Stories by topic

- [helm](./by-topic/helm.md) — Helm hooks, OCI charts, atomic rollbacks, ownership
- [gke](./by-topic/gke.md) — private GKE, Connect Gateway, Cilium, node pools with taints
- [gitops](./by-topic/gitops.md) — CD with WIF, multi-branch, detect-changes
- [terraform](./by-topic/terraform.md) — multi-stage IaC, import, state drift, FAST framework
- [networking](./by-topic/networking.md) — Shared VPC, cross-project NEGs, certificates, load balancers
- [security](./by-topic/security.md) — JWT revocation, AI-audit hallucinations, multi-tenant enumeration, pentest discipline

---

## Why I write this

When you google real infrastructure problems, most results are "hello world" tutorials. The hard problems — the ones you hit in production, the ones that only show up when the pieces interact — are almost undocumented. When you solve them at 2 AM, they tend to stay in your head and in a commit message.

This is an attempt to leave a trail.

---

## How I add new stories

1. When something weird comes up during debugging, I drop raw notes into `_drafts/topic.md`. 2 minutes, no formatting.
2. At the end of the sprint I review `_drafts/`, pick the ones worth it, promote them to `projects/<project>/NN-slug.md` using `_template.md`.
3. I refresh the `by-topic/` indexes using the front-matter tags.
