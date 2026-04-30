# Security

Stories about pentesting our own systems before the regulator does, JWT revocation chains that look right but do nothing, AI auditors that hallucinate CVEs, multi-tenant boundary leaks via plain HTTP status codes, and the discipline cost of breaking your own scope mid-engagement.

The thread connecting these: **security is not what the code says — it is what the deployed system does under adversarial input**. Static analysis sees syntax. Defense lives at runtime. AI audit tools find leads, not facts. And every rule in a scope document earns its keep precisely in the moment you are tempted to bend it.

---

## Stories

### Methodology and meta

- **[16 — Pentesting your own banking RAG before the bank does](../projects/banking-ai-platform/16-pentesting-banking-rag-meta.md)**
  Three-layer audit (static / dynamic / E2E) before an external pentest. Each layer catches a class of bug the others miss. Treats the bank's upcoming pentest as a verification pass rather than a discovery exercise.

- **[18 — I broke my own pentest scope, the cost of "agility"](../projects/banking-ai-platform/18-broke-my-own-pentest-scope.md)**
  An UPDATE I should not have run, the sponsor's six-exclamation-mark intervention, and the structural lesson: scope rules protect the trust relationship with the future auditor, not the system itself.

### Concrete vulnerabilities and patterns

- **[15 — The JTI claim that wasn't there — dead code that passed code review](../projects/banking-ai-platform/15-jwt-jti-dead-code.md)**
  Two PRs adding JWT revocation to a banking platform. SAST clean, both code reviews approved. End-to-end runtime: token survives logout. The generator never set `jti`, so the chain was inert. Pattern: write the attack as an integration test before writing the fix.

- **[19 — 403 vs 404, when better UX leaks your tenant boundary](../projects/banking-ai-platform/19-403-vs-404-enumeration-tradeoff.md)**
  A document endpoint returns specific status codes for "exists no permission" vs "not found". Cleaner UX. Also enables low-privilege users to enumerate every document ID in the system. In multi-tenant banking, distinguishability is leakage. Fail closed with a uniform 404.

### AI-augmented security tooling

- **[17 — The AI auditor hallucinated a CVE that does not exist](../projects/banking-ai-platform/17-ai-auditor-hallucinated-cve.md)**
  An AI security agent reported `CVE-2025-69872` against `diskcache 5.6.3` and recommended pinning to `>=5.6.4`. Pin merged. `uv sync` later breaks because version `5.6.4` does not exist. The CVE itself is not in NVD. Verify AI-reported CVEs against PyPI / NVD / GitHub Advisory before remediation.

---

## Patterns that recur across these stories

| Pattern | Where it shows up |
|---------|-------------------|
| **Three-component security chain — generator, store, check.** Test all three together, not each in isolation. | 15 |
| **AI-suggested fix is a hypothesis, not an action.** Verify against authoritative source. | 17 |
| **Distinguishable error responses are leak channels.** Status codes, body content, even latency. | 19 |
| **Scope rules protect the audit trail, not the system.** They earn their keep when you are tempted to bend them. | 18 |
| **Static + dynamic + E2E layered, never substituted.** Each layer sees a class of bug invisible to the others. | 16, all |

---

## Stack

- JWT revocation: PyJWT, Redis 7, FastAPI dependencies
- Multi-tenant access: SQLAlchemy 2 (async), Pydantic envelope shape
- AI audit tooling: Claude-family agents in parallel orchestration, manual verification against PyPI / NVD
- Pentest workflow: scope doc, paper-trail headers, evidence directory per finding, deviation registry

---

## Regulatory backdrop

Banking RAG, BCRA Comunicación A 7724 (cyber risk management) applies. Pentest report format follows BCRA-grade expectations: CVSS 3.1 vectors, CWE / OWASP mapping, reproducible PoCs in `evidence/` directories, re-test table per finding tracking `open → mitigated → verified`.
