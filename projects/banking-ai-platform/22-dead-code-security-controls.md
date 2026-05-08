---
title: The four security controls that passed code review and never ran
project: banking-ai-platform
date: 2026-05-07
stack: [security, testing, fastapi, redis, postgres, pydantic]
topics: [security, testing-strategy, runtime-vs-static, regulated]
severity: high
---

# The four security controls that passed code review and never ran

**TL;DR** — During a pre-emptive pentest of a banking RAG, four high-severity findings turned out to share the exact same shape: a security control was correctly designed, written, code-reviewed, unit-tested, and merged — but it never engaged at runtime because of a one-line gap between two adjacent components. None of these gaps were visible to SAST, code review, or unit tests. They only show up in end-to-end integration. The lesson stopped being "fix these four bugs" and started being "we are not testing the *seam* where security controls live."

---

## Context

Banking RAG platform — FastAPI on GKE, JWT auth in HTTPOnly cookies, Redis for session state and rate limiting, PostgreSQL for app data with a `hash_chain` column on the audit log table. Multiple security controls implemented over time: token revocation, brute-force lockout, audit log integrity, document upload validation. Each one was its own ticket, its own PR, its own unit tests, its own code review. CI is reasonably strict — ruff, mypy, bandit, pip-audit, detect-secrets. Test coverage above the line set by the team.

Then a pre-emptive pentest was run before the regulated bank's external audit (see story 16). The pentest produced 17 findings. Four of them — let us call them PT-A, PT-B, PT-C, PT-D — were classified as high or notable independently. Only when we wrote them up did the pattern jump out.

---

## The four cases

### PT-A — JWT revocation chain (story 15)

**Designed**: token revocation list in Redis, populated on `POST /auth/logout`, checked on every authenticated request. Three components.

**What actually happened**: the writer and the reader were correct. The token *generator* never put a `jti` claim into the token. So the writer was effectively `redis.set(f"token:revoked:{None}", ...)`, and the reader was `if payload.get("jti") in blacklist:` — which is always `False` for a `None` key. Logout returned 200, the cookie was cleared in the browser, and the same token kept working server-side until natural expiry.

**Visible to SAST**: no. Each function is locally correct.
**Visible to unit tests**: no. Tests for the writer mocked the JTI. Tests for the reader mocked the blacklist. Nobody tested *issue token → use → logout → reuse*.
**Visible to code review**: only to a reviewer holding all three files in their head simultaneously.

### PT-B — Brute-force lockout

**Designed**: after 5 failed logins on the same email, Redis sets `login:failed:{email}` and the next attempt — even with the correct password — returns 401 for 15 minutes.

**What actually happened**: six failed logins followed by the correct password returned 200 with fresh tokens. The counter was never being incremented because the failure path raised `AuthenticationError` *before* reaching the `redis.incr(...)` call. The lockout *code* was reachable (linters confirmed), but the *control flow* did not let it run.

**Visible to SAST**: no. The lockout code is reachable in principle.
**Visible to unit tests**: no. Tests that mocked Redis verified the lockout *given* the counter was at 5. No test forced 5 real failed logins and checked the counter actually moved.

### PT-C — Audit log integrity verification

**Designed**: every `audit_logs` row carries a `hash_chain` column. The hash of row N depends on the hash of row N-1 plus the row's own data. There is an admin endpoint `POST /admin/audit-logs/verify` that walks the chain and returns `is_valid: true/false`. Standard tamper-evidence pattern.

**What actually happened**: the chain was being written correctly — 1600+ rows, no nulls. The verify endpoint had been deployed for weeks. The first time anyone called it during the pentest, it returned **HTTP 500**. The Pydantic model `AuditLogEntry` declared `ip_address: str`, but the PostgreSQL `inet` column is returned by psycopg as an `IPv4Address` object. Pydantic refused. The endpoint crashed before reading the second row.

In other words: nobody had ever verified the audit chain in production. The whole tamper-evidence pattern was decorative.

**Visible to SAST**: no. Type annotations look fine in isolation.
**Visible to unit tests**: yes, in principle — but the unit tests for `verify_chain` constructed `AuditLogEntry` instances *manually*, with `ip_address=str("...")`. They never went through psycopg.

### PT-D — Upload endpoint stub

**Designed**: `POST /documents` accepts a multipart upload, validates magic bytes against MIME, computes SHA-256, returns `201 Created` with a `document_id` and `status: "pending"`. Downstream Airflow DAG ingests it.

**What actually happened**: the endpoint validated the file, generated a random UUID, logged the upload, returned 201 — and discarded the bytes. No GCS write. No DB row. No DAG trigger. A comment in the code said `# Generate a mock document_id (since actual persistence is T2-S5-02)`. The real ingestion path lives elsewhere (a separate DAG sources files from a content-management system). Nobody noticed because the endpoint was never load-tested end-to-end with "upload then list then retrieve".

**Visible to SAST**: no. There is nothing wrong with the code; it just does not do what its 201 promises.
**Visible to unit tests**: no. Tests mocked the validator and asserted on the return value, which was correct.

---

## The pattern

Four cases. One shape:

```
[ Component A ]  →  [ seam ]  →  [ Component B ]
   well-tested      untested      well-tested
```

In each finding the components were individually fine. The bug was in the *contract* between them — a missing field, a missing increment, a type mismatch, a missing persistence step — and that contract was never exercised by any automated test.

The reasons SAST and unit tests cannot see these:

1. **SAST analyzes one file at a time, conservatively.** It does not trace data flow across modules + ORM + JSON serialization + HTTP boundary at runtime fidelity.
2. **Unit tests with mocks pass exactly the data the test author imagined.** The bug is precisely that the production data is *not* what the author imagined (a `None`, an `IPv4Address`, a too-early raise).
3. **Code review by humans is single-file biased.** A reviewer reading `auth.py` sees the logout code is correct. They do not also pull up `jwt.py` and verify a claim is included. Trust composes upward; reviewers expect adjacent components to behave as advertised.

This is not a process failure of "we should have caught it." It is a structural property: there is a class of bugs that is *only* visible end-to-end. If the test pyramid does not include that layer, those bugs ship.

---

## The aha moment

For each finding, the fix was tiny. Add `"jti": secrets.token_urlsafe(16)` to the payload. Move the `redis.incr` before the raise. Coerce `IPv4Address` to `str` in a Pydantic validator. Either complete the upload spec or return 501. Lines of code: 1, 2, 5, varies.

The wrong reaction is to feel bad about four small bugs. The right reaction is to look at the test suite and ask: *what is the smallest test that, had it existed, would have caught this?* For each finding, the answer is the same shape:

> Stand up the system end-to-end. Make a real HTTP request. Assert on the *external* observable behavior, not on the return value of an internal function.

That kind of test was missing from all four. Not because anyone decided not to write it — it just is not what unit-test culture rewards. Unit tests are fast, isolated, and many. Integration tests like the one above are slow, stateful, and few. The ratio favors unit tests. For most application code, that is fine. For *security controls*, where the bug surface is precisely the seam, the ratio is wrong.

---

## The solution

Two parts.

### Part 1 — convert each pentest PoC into a regression test

Every pentest finding came with a reproducible PoC — a shell script with curl commands, or a Python harness. We took those PoCs and ported them as integration tests under `tests/security/`. Same input, same expected output. They run against a real FastAPI app with a real Redis and a real Postgres (TestContainers).

```python
# tests/security/test_jwt_revocation_e2e.py
async def test_token_revoked_after_logout_e2e(api_client):
    """PT-A regression: a token that was used to logout cannot be reused."""
    token = await login_get_token(api_client, "user@bank.local", "...")
    # Pre-logout: token works
    assert (await api_client.get("/conversations", cookies={"access_token": token})).status_code == 200
    # Logout
    assert (await api_client.post("/auth/logout", cookies={"access_token": token})).status_code == 200
    # Post-logout: same token must NOT work
    assert (await api_client.get("/conversations", cookies={"access_token": token})).status_code == 401
```

That single test would have flagged PT-A on day one. If anyone removes the `jti` from the generator in the future, the test goes red.

### Part 2 — make `tests/security/` a first-class CI job

The unit-test job already runs on every PR. We added `tests/security/` as a separate job with its own coverage report and its own merge-block. The cost is a few minutes of extra CI time per PR. The benefit is that the four bugs above can never silently ship again, and the same pattern catches similar future bugs at the seam.

---

## Takeaways

1. **There is a class of bugs that only end-to-end tests can see.** No amount of SAST, unit tests, or code review will surface them. For security controls, this class is not edge — it is the main attack surface.
2. **Mocked unit tests pass the data the author expected.** The bug is precisely that production data is not what the author expected. Mocks make this invisible by design.
3. **Code review composes trust upward.** A reviewer of file X assumes adjacent files Y and Z behave as advertised. The bug at the seam between X and Y is exactly what no single reviewer is positioned to see.
4. **For each pentest finding, ask: what is the smallest test that would have caught it?** That test is your regression. Adding it costs minutes. Not adding it costs a re-test cycle with the bank.
5. **Track your security regression suite separately.** Mixing it with the general unit-test pile dilutes both signal and ownership. A dedicated `tests/security/` directory + a dedicated CI job makes the contract explicit.

---

## Stack involved

- FastAPI 0.115+, Pydantic v2 (the `IPv4Address` mismatch is a v2-specific behavior — v1 was more permissive).
- psycopg 3 with PostgreSQL `inet` columns.
- Redis 7 for blacklist + counter primitives.
- TestContainers for spinning up real Redis and Postgres in the security test job.
- pytest-asyncio.

---

## Links / references

- [Pydantic v2 strict-by-default behavior](https://docs.pydantic.dev/latest/concepts/strict_mode/)
- [psycopg adaptation for inet types](https://www.psycopg.org/psycopg3/docs/basic/adapt.html)
- Story 15 — *The JTI claim that wasn't there* (the original write-up of PT-A)
- Story 16 — *Pentesting your own banking RAG before the bank does* (the methodology that surfaced all four)
