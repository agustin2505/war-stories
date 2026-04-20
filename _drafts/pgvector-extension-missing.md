---
title: Cloud SQL + pgvector — Terraform cannot install the extension
project: banking-ai-platform
date: 2026-04-16
stack: [cloud-sql, postgresql, pgvector, terraform, alembic]
topics: [managed-db, one-time-setup, extensions]
severity: low
status: draft
---

# Raw notes

## The symptom

Deploy pipeline passes every previous stage. The app Helm chart kicks off its `pre-install` DB migration hook. Alembic migration 001 (initial schema) succeeds. Migration 002 starts and fails with:

```
sqlalchemy.exc.ProgrammingError: (psycopg.errors.UndefinedObject)
type "halfvec" does not exist
LINE 1: ALTER TABLE document_chunks ADD COLUMN embedding halfvec(768)
                                                         ^
```

Atomic rollback kicks in. Deploy fails.

## Why

- The app uses `pgvector` 0.8+ with the `halfvec` type for 768-dim Gemini embeddings.
- The Terraform module for Cloud SQL created the instance, the database, the user, the flags — but not the extension.
- Cloud SQL allows `CREATE EXTENSION` only through a DB connection (SQL), not through any managed API that Terraform can call.
- The `google_sql_database_instance` resource has a `database_flags` block, but extensions are not flags.

## The fix (one-time, manual)

```bash
gcloud sql connect macro-ai-qa-db \
  --database=enterprise_ai \
  --user=postgres \
  --project=itmind-macro-ai-qa-0

# in psql:
CREATE EXTENSION IF NOT EXISTS vector;
```

That is all. The migration resumes cleanly on the next run.

## Why it is worth documenting

- Every time you create a new environment (dev, QA, prod), you will hit this again if you forget.
- Nothing in Terraform flags the problem until the first deploy fails at migration time.
- The error message does not mention pgvector or extensions — it says "type halfvec does not exist". If you do not know the chain (halfvec comes from pgvector), the google hit rate is bad.

## Options I considered

1. **A `null_resource` + `local-exec` in Terraform** that runs `gcloud sql connect` and a SQL script. Works but couples Terraform to having psql installed on the runner, and authentication is tricky.
2. **A Cloud SQL proxy sidecar in a bootstrap Job** that runs `CREATE EXTENSION` once per environment. Overkill for a one-line SQL.
3. **Alembic migration `op.execute("CREATE EXTENSION ...")`**. Works if the DB user has `rds_superuser` / `cloudsqlsuperuser` privileges. In our setup the app user does not — only `postgres` does. We could temporarily grant it, but that leaks privilege.
4. **Manual step documented in the environment bootstrap checklist**. Boring but honest. This is what we did.

## Takeaways to develop

- Managed Postgres services (Cloud SQL, RDS) draw a clear line: DDL about roles and users is infrastructure; DDL inside a database is application concern. Extensions live inside a database.
- A "manual step" that takes 2 seconds is fine — as long as it is **documented** with exact commands per environment. Undocumented manual steps are how production deploys mysteriously break 6 months later.
- The first failure mode you should expect from any new Postgres feature (pgvector, PostGIS, pg_trgm) is "extension not installed".

## To include in the polished story

- The error message does not mention pgvector. Walk through the diagnosis chain.
- Brief comparison with the 4 options above and why we picked "manual step in checklist".
