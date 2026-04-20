---
title: WIF attribute_condition for multi-branch CD
project: banking-ai-platform
date: 2026-04-16
stack: [terraform, github-actions, workload-identity-federation, gcp]
topics: [wif, multi-env, oidc, ci-cd]
severity: medium
status: draft
---

# Raw notes

## The problem

- Single repo enterprise-ai-platform, two CDs:
  - `cd.yml` triggers on merge to `main` (deploy to dev)
  - `cd-qa.yml` triggers on merge to `stage` (deploy to QA)
- Each environment has its own GCP project, own WIF pool, own provider.
- The WIF provider in Terraform (shared across environments) had hardcoded:
  ```
  assertion.ref == 'refs/heads/main'
  ```
- The QA CD failed at `google-github-actions/auth@v2` because the OIDC token from the QA CD brings `ref=refs/heads/stage`.

## The fix

A variable in Terraform:

```hcl
variable "wif_allowed_ref" {
  description = "Git ref allowed in WIF attribute_condition (dev=refs/heads/main, qa=refs/heads/stage)"
  type        = string
  default     = "refs/heads/main"
}
```

In wif.tf:
```hcl
attribute_condition = join(" && ", [
  "assertion.repository == 'data-oilers/enterprise-ai-platform'",
  "assertion.repository_owner == 'data-oilers'",
  "assertion.ref == '${var.wif_allowed_ref}'",
])
```

In tfvars per environment:
- dev.tfvars: `wif_allowed_ref = "refs/heads/main"`
- qa.tfvars: `wif_allowed_ref = "refs/heads/stage"`

## Why not solve it with `ref.startsWith(...)` or multiple `||`?

Least privilege. Each WIF provider authorizes exactly the branch its corresponding CD uses. A provider for the dev project should not accept tokens from the `stage` branch even if they come from the same repo.

## Takeaways to develop

- WIF as parameterized infrastructure, not hardcode.
- One WIF provider per environment = one allowed ref per environment.
- Error is only visible on the first CD run of the new environment. If you do not test early, you discover it after investing in the whole setup.
- Detail: the OIDC error does not say "ref rejected" — it says a generic "auth failed". You have to go to the provider logs to see the actual reason.

## Links pending

- Google Cloud docs on `attribute_condition`
- [GitHub OIDC token claims reference](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token)
