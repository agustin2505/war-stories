# Security

Stories about pentesting our own systems before the regulator does, security controls that pass code review and never engage at runtime, fail-closed turning into a denial of service when an external dependency goes down, JWT revocation chains that look right but do nothing, AI auditors that hallucinate CVEs, multi-tenant boundary leaks via plain HTTP status codes, and the discipline cost of breaking your own scope mid-engagement.

The thread connecting these: **security is not what the code says — it is what the deployed system does under adversarial input AND under partial failure of its dependencies**. Static analysis sees syntax. Defense lives at runtime. AI audit tools find leads, not facts. Fail-closed without thinking about availability is brittleness wearing a security hat. And every rule in a scope document earns its keep precisely in the moment you are tempted to bend it.

---

## Stories

### Methodology and meta

- **[16 — Pentesting your own banking RAG before the bank does](../projects/banking-ai-platform/16-pentesting-banking-rag-meta.md)**
  Three-layer audit (static / dynamic / E2E) before an external pentest. Each layer catches a class of bug the others miss. Treats the bank's upcoming pentest as a verification pass rather than a discovery exercise.

- **[18 — I broke my own pentest scope, the cost of "agility"](../projects/banking-ai-platform/18-broke-my-own-pentest-scope.md)**
  An UPDATE I should not have run, the sponsor's six-exclamation-mark intervention, and the structural lesson: scope rules protect the trust relationship with the future auditor, not the system itself.

- **[22 — The four security controls that passed code review and never ran](../projects/banking-ai-platform/22-dead-code-security-controls.md)**
  A pre-emptive pentest surfaced four high-severity findings sharing the same shape: SAST + unit tests + code review all green, but the control did not engage end-to-end because of a one-line gap at the seam between two components. The structural fix is not "add four more unit tests" — it is "add an integration test layer that exercises the seam where security lives".

- **[24 — Your pentest suite isn't done when the tests pass](../projects/banking-ai-platform/24-pentest-suite-not-done-when-tests-pass.md)**
  The follow-up to story 22: we wrote the regression suite, it ran green in CI on the third try (after fixing a missing env var and a missing extras group), and the reflex was to stop. Wrong. A CI check that does not block the merge is informational, not protective. The work is done when branch protection requires the check — not before. The four independent states: tests written, tests pass locally, tests pass in CI, tests block merges. Two thirds of the cost is plumbing.

### Concrete vulnerabilities and patterns

- **[15 — The JTI claim that wasn't there — dead code that passed code review](../projects/banking-ai-platform/15-jwt-jti-dead-code.md)**
  Two PRs adding JWT revocation to a banking platform. SAST clean, both code reviews approved. End-to-end runtime: token survives logout. The generator never set `jti`, so the chain was inert. Pattern: write the attack as an integration test before writing the fix.

- **[19 — 403 vs 404, when better UX leaks your tenant boundary](../projects/banking-ai-platform/19-403-vs-404-enumeration-tradeoff.md)**
  A document endpoint returns specific status codes for "exists no permission" vs "not found". Cleaner UX. Also enables low-privilege users to enumerate every document ID in the system. In multi-tenant banking, distinguishability is leakage. Fail closed with a uniform 404.

- **[21 — Fail-closed turned an LLM-as-guardrail into a single point of failure](../projects/banking-ai-platform/21-fail-closed-llm-guardrail-dos.md)**
  A two-layer input guardrail (regex + LLM classifier). When the LLM provider's quota ran out, the textbook-correct `except Exception: return UNSAFE` blocked every query — including innocent ones. We discovered it by accident, while pentesting unrelated vectors. The fix was a small heuristic fallback plus an explicit `degraded` warning, accepting a slightly higher residual risk during outages in exchange for not turning the whole RAG into a denial of service.

- **[23 — The redaction that wasn't there enough](../projects/banking-ai-platform/23-redaction-that-wasnt-there-enough.md)**
  A PII redactor function called manually at each log site passed SAST, code review and unit tests. The retest against the deployed cluster found two more log sites no one had touched. The pattern was structural: any rule that requires every contributor to remember it will be forgotten. The fix moved the guarantee out of the developer's hands — a `logging.Formatter` and a `structlog` processor that redact every log message regardless of who wrote it or what fields it uses.

### AI-augmented security tooling

- **[17 — The AI auditor hallucinated a CVE that does not exist](../projects/banking-ai-platform/17-ai-auditor-hallucinated-cve.md)**
  An AI security agent reported `CVE-2025-69872` against `diskcache 5.6.3` and recommended pinning to `>=5.6.4`. Pin merged. `uv sync` later breaks because version `5.6.4` does not exist. The CVE itself is not in NVD. Verify AI-reported CVEs against PyPI / NVD / GitHub Advisory before remediation.

---

## Patterns that recur across these stories

| Pattern | Where it shows up |
|---------|-------------------|
| **Controls that pass code review and never run in production.** Components are individually correct; the seam between them is what fails. | 15, 22 |
| **Rules that require every contributor to remember them, will be forgotten.** Move the guarantee out of the developer's hands. | 23 |
| **CI green ≠ system protected.** Tests that do not block merges are informational, not protective. | 24 |
| **Fail-closed couples your security posture to the availability of your dependencies.** The dependency's outage becomes your DoS. | 21 |
| **AI-suggested fix is a hypothesis, not an action.** Verify against authoritative source. | 17 |
| **Distinguishable error responses are leak channels.** Status codes, body content, even latency. | 19 |
| **Scope rules protect the audit trail, not the system.** They earn their keep when you are tempted to bend them. | 18 |
| **Static + dynamic + E2E layered, never substituted.** Each layer sees a class of bug invisible to the others. | 16, 22, all |

---

## Stack

- JWT revocation: PyJWT, Redis 7, FastAPI dependencies
- Multi-tenant access: SQLAlchemy 2 (async), Pydantic envelope shape
- AI audit tooling: Claude-family agents in parallel orchestration, manual verification against PyPI / NVD
- Pentest workflow: scope doc, paper-trail headers, evidence directory per finding, deviation registry

---

## Regulatory backdrop

Banking RAG, BCRA Comunicación A 7724 (cyber risk management) applies. Pentest report format follows BCRA-grade expectations: CVSS 3.1 vectors, CWE / OWASP mapping, reproducible PoCs in `evidence/` directories, re-test table per finding tracking `open → mitigated → verified`.
