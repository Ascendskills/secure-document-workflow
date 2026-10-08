# Codex task: Complete Week 1, security-first specification (NO application implementation yet)

You are the lead software architect, security engineer, QA lead and technical writer working with a three-member student team. Work inside the currently opened Git repository. Do not claim any code or tests ran unless you actually run them and save evidence.

## Project and source of truth
Title: Blockchain-Based Secure Document Workflow for Preventing PII Leakage in Xerox and Scanning Systems.
Inspect these files if available in the repository under `reference/`:
- `Project Planning.pdf`
- `Project design.pdf`
- `B1-REVIEW-1.pptx`
- `Week1_Security_Project_Blueprint.md`
- `Week1_Endpoint_Test_Cases.csv`
- `Week1_Security_Test_Cases.csv`
If any source is missing, say which one, do not invent its contents, and proceed with accessible materials. Extract source facts and trace them to filenames/page or section wherever possible. Do not treat proposal documents as implemented features. Preserve source-stated facts; label any changes as RECOMMENDED and add them to `docs/week1/decision-log.md` for approval. Resolve conflicting requirements through an explicit decision table, not by silently choosing.

## Scope (confirmed)
Simulated scanner adapter ONLY for this implementation stage, with synthetic PDF/JPG/PNG uploads mimicking a scanning device. Architect an interface that permits future physical-scanner integration, but DO NOT implement or claim physical hardware support and DO NOT require scanner device certificates/mTLS for a browser-based mock adapter. Treat browser input as untrusted. Real-scanner integration is a deferred future phase, not Week 1 acceptance. Use one planned ingestion and validation/encryption pipeline for simulated scans and manual uploads.

Reference architecture: React, Flask, PostgreSQL, IPFS (ciphertext only), Hyperledger Fabric (minimal tamper-evident metadata and access history), private Node.js Fabric gateway/chaincode, AES-256-GCM, SHA-256, Argon2id password hashing and TOTP. Design key management with separate protected wrapping keys, no plaintext keys or PII on IPFS/Fabric or in logs. Make no absolute security claims.

## Team ownership
- Deepthi: team lead; threat model, security architecture, authentication, authorization, encryption/key design, Fabric chaincode/gateway design, document signoff.
- Harini: Flask service architecture, API/OpenAPI specs, database schema, IPFS and simulated scanner adapter interface, recovery/idempotency specifications.
- Varshith: React screen flows/wireframes, QA strategy, test fixtures, per-route tests, accessibility checks, test reports and defect tracking.
Every deliverable also needs reviewer and approver; no author self-approval. Define a RACI matrix and a responsibility-to-endpoint matrix.

## Deliverables to create under docs/week1/
1. `README.md`: scope, how to review artifacts, acceptance checklist, links.
2. `source-traceability.md`: baseline facts vs new proposals vs unresolved conflicts, with location citations in the supplied materials.
3. `SRS.md`: actors, use cases, functional and nonfunctional requirements, unique IDs, priority, measurable acceptance tests, out-of-scope, assumptions.
4. `architecture.md` and diagrams in Mermaid: components, trust boundaries, upload, simulated scan, document retrieval, authorization, access revocation and failure/retry flows.
5. `threat-model.md`: STRIDE by boundary, severity rationale, mitigations, residual risks, threat-to-test linkage. Explicitly include broken object authorization/IDOR, CSRF, XSS, SQLi, file parser attacks, upload bombs, TOTP replay, session theft, key leakage, Fabric gateway/chaincode identities, IPFS metadata exposure, tampering, and race conditions.
6. `role-permission-matrix.md`: unauthenticated, admin, owner, approved user, auditor, simulator service where applicable; role vs object-level permissions; tenant/org scope if supported; deny-by-default; who grants/revokes; consistent error behavior. Review original admin universal-read behavior and admin-only/public enrollment conflict as OPEN DECISIONS.
7. `data-model.md`: PostgreSQL entities, constraints, foreign keys, indexes, audit and permission state machine, transactional outbox, idempotency keys, encrypted key references, retention policy decisions, ER diagram.
8. `api-inventory.md` plus `openapi.yaml`: list EVERY original API endpoint from design source, explicitly verify the count, and add simulated-scanner endpoints only when justified. For every route specify operation ID, request/response schemas, status codes, authentication, authorization, CSRF, rate limits, validation, side effects, error contracts, audit event, idempotency, timeout behavior, owner/reviewer/tester, linked requirements and test IDs. Use OpenAPI 3.1. Clearly label draft routes requiring approval.
9. `simulated-scanner-contract.md`: scanner session/job state machine, fake device ID, sample synthetic payload, job status, cancel, duplicate upload and timeout handling; no unsupported hardware claims. Ensure adapter must not bypass server-side security checks.
10. `ui-screen-map.md`: pages, accessible form validation, mapping each action to endpoints and roles, frontend error behavior; descriptive text wireframes or Mermaid.
11. `test-strategy.md`: unit, API, UI, integration, security, failure injection, concurrency, SAST, dependency and secret scans, security reviews, CI and regression; stage-specific exit criteria and an evidence convention.
12. `test-cases.csv`: traceable test cases for EVERY API operation covering allowed-role success, unauthenticated, disallowed-role, cross-owner IDOR, malformed/oversized requests, boundaries, dependency failure, and state-transition/concurrency cases when relevant. Columns: test_id, requirement_id, operation_id, method, path, role, preconditions, synthetic_test_data, steps, expected_status, expected_body_or_side_effect, negative_security_assertion, owner, reviewer, tester, actual_result, status, evidence_path, defect_id. Mark `status=NOT_RUN`, leave actual_result/evidence blank.
13. `security-test-cases.csv`: STRIDE-linked attack tests with safe synthetic samples only, no real PII. Mark NOT_RUN.
14. `fixtures/README.md` plus synthetic fixtures with non-real identities and harmless test file placeholders; no valid government ID numbers, secrets, production credentials or malware. Do not copy user-provided PII into test fixtures.
15. `ownership-raci.md`, `day-by-day-plan.md`, `decision-log.md`, `risk-register.md`, `defect-register.csv`, `week1-review-checklist.md`.
16. `.github/workflows/week1-docs.yml` (or a portable local validation script) to check link integrity, duplicate requirement/test/operation IDs, CSV headers and test traceability, valid JSON/YAML/OpenAPI syntax when validation dependencies are available, and no accidental secrets. Avoid introducing network-dependent CI without documenting requirements.
17. Root `README.md`, `.gitignore`, `CONTRIBUTING.md`, `SECURITY.md`, initial folder layout; no application implementation in this task.

## Rules
- Preserve exact original role names and endpoints when quoting source, and list proposed changes separately.
- Do not assume every user can read every other user's documents. Owner relationship is object-scoped. Admin must not automatically receive decryption rights unless explicitly approved in decision log.
- Never represent a hash/blockchain alone as proof of confidentiality; privacy needs strong keys, isolation, authorization and operational controls.
- Revocation blocks future service-mediated access but cannot claw back previously exported plaintext.
- State release gates as GOALS, not achieved facts: complete route/spec coverage, full role/endpoint authorization test matrix, zero known unmitigated critical/high findings, code-coverage target for later implementation, and documented review sign-off.
- Keep sensitive errors generic and internal logs redacted.
- No placeholders such as 'TBD' in mandatory contracts without an OPEN DECISION ID and named owner; no fabricated test evidence.
- Prefer deterministic local checks; test failures must be fixed, not hidden.

## Execution approach
A. Inspect repository and reference files; summarize source inventory, conflicting statements, existing test counts, and key open questions.
B. Create requirements and ADR/decision log first; then permissions and threat model; then API/data contracts; then test matrices and role ownership; then validation tooling.
C. Execute available structural/document validation; save command/result/date in `docs/week1/validation-report.md`. Do not mark runtime app tests as PASS.
D. Review all links and requirement-to-endpoint-to-test traceability; show counts, missing links, and open approval decisions.
E. Return an evidence-based final report: files created, exact endpoint/test/requirement counts derived from files, checks actually run and their pass/fail status, open decisions requiring team approval, and recommended Week 2 kickoff tasks.

First response: show a concise execution plan and start by inspecting the files. Work through the complete Week 1 specification; don't stop at an outline.
