---
title: Org policies blocking a new environment — fixing them at the folder level
project: banking-ai-platform
date: 2026-04-09
stack: [gcp, organization-policies, terraform, fast-framework, landing-zone]
topics: [multi-env, least-privilege, policy-hierarchy]
severity: high
status: draft
---

# Raw notes

## The context

We run a multi-tenant GCP landing zone built with the FAST framework. The folder hierarchy looks roughly like this:

```
Organization
|- Tenants
|  \- Macro (client folder)
|     |- dev    (folder)
|     |- qa     (folder)  <- I was creating a new environment here
|     \- prod   (folder)
```

Org-level policies enforce security defaults: no service account key creation, private Google access required on subnets, no Cloud SQL without root password set, no external WIF providers, etc. Those defaults are fine for prod. For a new QA environment some of them blocked legitimate resources.

## The 4 org policies I had to override

| Policy | Why it blocked me |
|--------|-------------------|
| `cloudsqlRequireRootPassword` | Our Cloud SQL module creates the instance with password in Secret Manager, not inline. The policy was failing the create. |
| `networkRequireSubnetPrivateGoogleAccess` | Inherited default was stricter than what the QA subnet needed (we had explicit private access setup, but the check looked at a different flag). |
| `disableServiceAccountApiKeyCreation` | Some managed services still need a legacy API key during bootstrap (Monitoring, specific APIs). |
| `workloadIdentityPoolProviders` | The default allowed only a narrow list of issuers. GitHub Actions OIDC (`token.actions.githubusercontent.com`) had to be allow-listed. |

## The trap: where to put the override

First instinct: override at the **project** level (`itmind-macro-ai-qa-0`) via the project factory. Two problems:

1. The **project factory SA does not have `roles/orgpolicy.policyAdmin`** (it should not — least privilege). So Terraform fails to apply the override as part of the project creation.
2. Even with permission, an override at project level leaves you scattered: for 20 projects you would need 20 overrides. Not maintainable.

## The right place: at the QA folder

The QA folder sits below the tenant folder. A policy override at the QA folder applies to every project in that folder. It is applied as part of `0-org-setup`, which has org-admin-level permissions (the only place that should have them).

In FAST:

```hcl
# fast/0-org-setup/folders/qa/main.tf
module "folder-qa" {
  source = "../../../../fabric/modules/folder"
  parent = var.tenant_folder_id
  name   = "qa"

  org_policies = {
    "iam.disableServiceAccountKeyCreation"       = { rules = [{ enforce = false }] }
    "compute.requireOsLogin"                     = { rules = [{ enforce = false }] }
    "sql.restrictAuthorizedNetworks"             = { rules = [{ enforce = false }] }
    "iam.workloadIdentityPoolProviders"          = {
      rules = [{ allow = { values = ["https://token.actions.githubusercontent.com"] } }]
    }
  }
}
```

## The lesson

Org policy design in landing zones is a layered contract:

- **Org level**: maximum paranoia. Defaults assume "someone is attacking us".
- **Tenant folder**: tenant-specific hardening (compliance tier, region restrictions).
- **Environment folder (dev/qa/prod)**: environment-specific exceptions that do not weaken prod.
- **Project**: only if truly per-project.

If you add an exception at the wrong layer, two things go wrong:

1. **Wrong blast radius**: a prod exception that leaked to dev is a security incident. A dev exception at org level is a governance failure.
2. **Wrong permissions**: the SA that runs that stage does not have the right scope. You fight the tool instead of fixing the policy.

## What I had to change in FAST

- Added a `folder-qa` module that mirrors `folder-dev` with QA-specific `org_policies`.
- Moved the 4 overrides from the project factory to `0-org-setup`.
- Removed the `org-policy-admin` grant attempt on the project factory SA (it had been a workaround).

## Takeaways to develop

- Read the error message carefully: "Constraint X is enforced" tells you WHICH policy. Then `gcloud resource-manager org-policies list --folder=FOLDER_ID` tells you where it is enforced.
- In FAST, every org policy override should be traceable to a comment explaining WHY the exception exists. Without the why, the next engineer either removes it (breaks things) or copies it to prod (weakens security).
- Least privilege applies to IaC runners too. The SA that creates projects should not be able to edit org policies.

## To include in the polished story

- Diagram of the folder hierarchy and which layer owns which exception.
- Before/after of the failed `terraform apply` with the explicit org-policy error message.
- Brief mention of FAST's `folder` module supporting `org_policies` inline.
