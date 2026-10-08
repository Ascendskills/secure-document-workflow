# Week 1 — Security-First Project Specification (Baseline v1.0)
**Project:** Blockchain-Based Secure Document Workflow for Preventing PII Leakage in Xerox and Scanning Systems  
**Status:** DRAFT / NOT YET IMPLEMENTED — proposed requirements and expected tests only  
**Date:** 2026-10-08  |  **Scope:** simulated file uploads **and** physical-scanner integration

## 0. Source traceability and decisions
**From supplied files:** React, Flask, PostgreSQL, IPFS, Hyperledger Fabric, Node.js gateway/chaincode, AES-256-GCM, SHA-256, password+TOTP, four roles, an eight-week plan, three named team assignments, and 21 baseline HTTP routes. Project Planning explicitly describes uploads as a stand-in for scanning; physical scanner integration is a new scope requirement. The existing design gives administrators broad read capability and differs on registration (admin-only permissions versus public registration UI).
**Proposed hardened decisions requiring team approval:** admin-invited account creation; admins can inspect metadata but cannot automatically decrypt any document; device identity required for real scanning; scanner integration via approved local adapter rather than publicly exposing a printer; privacy-preserving logs; role and document-level checks; private/IP-restricted IPFS; non-public Fabric gateway; defined retention and threat acceptance.
**Architecture assumption:** physical scanners are integrated through a controlled adapter on the scanning workstation or via an approved scan-to-folder protocol. Actual vendor/model capabilities, TWAIN/WIA/SANE/eSCL/IPP support, and spool deletion controls are **unverified** until hardware is selected. Avoid promising any real-scanner feature before compatibility tests.

## 1. Scope, actors and security boundaries
**In scope:** synthetic PDF/JPEG/PNG uploads, external scanner acquisition through authorized adapter, MFA, least-privilege workflow, encryption, private IPFS, committed Fabric reference, access decision lifecycle, integrity check, audit, monitoring and automated testing. **Out of scope for v1 unless approved:** vendor-specific printer firmware modification, universal scanner compatibility, offline user decryption, guaranteed prevention of screenshots/photos or copies already downloaded, automatic compliance certification.
**Trust boundaries:** (B1) browser → HTTPS Flask; (B2) authenticated scanner → restricted adapter → private ingestion API; (B3) Flask → PostgreSQL; (B4) Flask → key management; (B5) Flask → private IPFS; (B6) Flask → internal Fabric gateway → endorsed chaincode; (B7) developer CI and secret store → deployment. Every crossing needs explicit identity, authentication, authorization, validation, logging and failure policy.
**End-to-end manual flow:** browser upload → auth + CSRF + quota → byte/type validation + parser isolation → document ID → envelope encryption → SHA-256(ciphertext blob) → private IPFS CID → DB pending + outbox → Fabric register commit → mark ACTIVE → audit. **Scanner flow:** admin enrolls device → owner submits job → adapter with mTLS acquires scan → private `SCN-07` ingestion → **same** validation/encryption/persistence flow. **Retrieval:** MFA session + object grant + approval state + expiry + recent step-up as needed → committed ledger hash lookup → retrieve encrypted blob → SHA-256 compare → AES-GCM authenticate/decrypt → restricted response + audit.

## 2. Functional and non-functional requirements
| ID | Priority | Acceptance requirement | Mapped routes |
|---|---|---|---|
| FR-01 | MUST | Provision accounts via administrator invitation; prohibit self-assignment of roles | `AUTH-01` |
| FR-02 | MUST | Password hashing with Argon2id and TOTP MFA; replay-resistant TOTP challenges | `AUTH-02..04` |
| FR-03 | MUST | Enforce server-side sessions, CSRF protection, rotation and logout invalidation | `AUTH-05..06` |
| FR-04 | MUST | Accept PDF/JPG/PNG simulated uploads; validate magic bytes, size, parsing budget | `DOC-01` |
| FR-05 | MUST | Encrypt each document with unique random AES-256-GCM key and nonce and protected key envelope | `DOC-01, DOC-04` |
| FR-06 | MUST | Store ciphertext only in private IPFS and SHA-256(ciphertext) + minimum metadata in Fabric | `DOC-01, FAB-01` |
| FR-07 | MUST | Owner-driven access request, approval, rejection, expiry and revocation workflow | `ACC-01..05` |
| FR-08 | MUST | Authorize every access by role, tenant, object ownership, permission state and step-up | `DOC-02..06` |
| FR-09 | MUST | Verify committed hash and AEAD authentication tag before releasing plaintext | `DOC-04..05` |
| FR-10 | MUST | Audit denied and permitted sensitive actions without raw PII, tokens or document content | `AUD-01` |
| FR-11 | MUST | Enroll approved scanner adapters with cryptographic device identities | `SCN-01..03` |
| FR-12 | MUST | Submit, monitor and cancel physical scan jobs with bounded device profiles | `SCN-04..06` |
| FR-13 | MUST | Ingest scanned bytes solely through private, mutually authenticated adapter channel | `SCN-07` |
| FR-14 | MUST | Use one document validation/encryption pipeline for manual and scanner ingestion | `DOC-01, SCN-07` |
| FR-15 | MUST | Recover correctly from partial IPFS, database and Fabric failures using outbox and idempotency | `DOC-01, SCN-07` |
| FR-16 | SHOULD | Add offline/timeout-safe scan job expiry and jam/cancel recovery | `SCN-04..06` |
| FR-17 | SHOULD | Provide redacted integrity and transaction history views | `DOC-05, FAB-01..03` |
| NFR-01 | MUST | TLS, secure cookies, HSTS in deployment, minimized CORS and service isolation | `All` |
| NFR-02 | MUST | No plaintext or private key material in logs, IPFS, Fabric, git or scanner spool after cleanup | `All` |
| NFR-03 | MUST | Deny-by-default policies; unauthorized cross-owner requests do not disclose metadata | `All` |
| NFR-04 | MUST | Secret and key rotation, versioned envelope format and restricted key access | `DOC-01, DOC-04` |
| NFR-05 | MUST | CI code, dependency, secret, container and configuration scans; blocking gate for critical/high | `All` |
| NFR-06 | MUST | Complete endpoint positive/negative/boundary/role tests and traceable evidence | `All` |
| NFR-07 | MUST | Synthetic PII only in development/test; scrub test artifacts and backups | `All` |
| NFR-08 | SHOULD | Define benchmark thresholds after measuring reference hardware and device speed | `All` |

**Required additional decisions in Week 1:** physical scanner vendor/model and connection path; public versus invite-only enrollment (proposed invite-only); allow downloaded plaintext versus view-only (recommended restricted viewer + optional approved export); retention duration; maximum file size/page count; tenancy/org model; legal permission to collect/test real PII; whether admin break-glass is needed. Pending answers are recorded as open issues, not silently treated as agreed scope.

## 3. Threat model — STRIDE and risk register
Risk is qualitative and provisional (no measured exploit likelihood yet). Critical items require architectural mitigation and tests before deployment.
| ID | STRIDE | Attack scenario | Boundary | Risk | Mitigation | Verification |
|---|---|---|---|---|---|---|
| T-01 | Spoofing | Stolen session / TOTP replay | Browser → Flask | High | Secure session cookies, rotation, MFA replay prevention, step-up | AUTH-02/04/05; SEC-04/05 |
| T-02 | Tampering | Modified encrypted IPFS bytes | IPFS → Flask | High | Committed SHA-256, AES-GCM authentication; fail closed | DOC-04/05; SEC-09 |
| T-03 | Repudiation | Owner denies access decision | Flask → DB/Fabric | Medium | Append-only audit chain, Fabric transactions, synchronized UTC time | ACC-03/05; SEC-13 |
| T-04 | Information disclosure | PII in logs, CID metadata, chain, printer spool | API/ledger/scanner | Critical | Pseudonymous IDs, redact, private IPFS, device spool cleanup | SEC-10/16/18 |
| T-05 | Denial of service | Upload bombs / oversized PDFs / queue flooding | Browser/scanner → ingest | High | Byte/page/CPU/time limits, rate quotas, bounded queues | DOC-01, SCN-04/07; SEC-11 |
| T-06 | Elevation of privilege | User edits role or requests others’ document IDs | Browser → API | Critical | No mass assignment, object-level authorization and negative matrix tests | AUTH-01, DOC-03/04; SEC-01/02 |
| T-07 | Spoofing | Rogue scanner sends scanned files | Scanner → adapter → ingest | Critical | mTLS device identity, certificate revocation, bound job nonce | SCN-01/07; SEC-14 |
| T-08 | Tampering | Scanner delivers bytes for wrong job or sends replay | Adapter → Flask | High | Job binding, digest, idempotency, freshness, state-machine checks | SCN-07; SEC-15 |
| T-09 | Information disclosure | SSRF / hostile network scan target | Flask → scanner | Critical | Allowlisted device IDs and fixed scan profiles; no arbitrary URL | SCN-04; SEC-17 |
| T-10 | Elevation of privilege | Compromised Fabric gateway service | Flask → gateway → Fabric | Critical | Private gateway, mTLS, least privilege identities, chaincode caller checks | FAB-01; SEC-19 |
| T-11 | Tampering | Permission revoked during retrieval | DB/Fabric → decrypt | High | Serialized grant-check/stream decision; short leases; abort on revoke when feasible | DOC-04, ACC-05; SEC-12 |
| T-12 | Information disclosure | Plaintext in temporary disk, backups or CDN | Flask → storage | Critical | Memory/ephemeral encrypted buffering, retention deletion, encrypted backups, no-store | DOC-01/04; SEC-16 |
| T-13 | Denial of service | IPFS/Fabric unavailable midway through save | Flask → external | High | Outbox/retry/idempotency, terminal states, never release before commit | DOC-01; SEC-20 |
| T-14 | Spoofing | Credential stuffing / enumeration | Public → auth | High | IP/account rate-limit, generic messages, alerts | AUTH-01/02; SEC-03 |
| T-15 | Tampering | Malicious document parser exploit | Uploaded PDF/image → parser | Critical | File sniffing, sandboxed conversions, no macros, patched parsers | DOC-01/SCN-07; SEC-11 |
| T-16 | Information disclosure | Authorized recipient redistributes decrypted document | Recipient device | Medium | Clear residual risk; view-only policy where possible, watermark, least privilege | DOC-04; policy review |

**Residual risks:** an authorized user can photograph or redistribute a displayed document; a compromised endpoint can expose plaintext; some scanner firmwares cache output beyond adapter control; Fabric and hash records are tamper-evident, not a substitute for secure key handling. Threat acceptance requires explicit owner signature.

## 4. Role-permission matrix (deny by default)
Roles are contextual, not a simple hierarchy: a document owner is determined per document, an approved user per permission record, and admin scope is administrative rather than universal decryption. For every resource, enforce organization scope, account status, document status, ownership/permission, MFA, and least privilege; use consistent 404 where resource existence must not be revealed.
| Operation | Public | Admin | Owner | Authorized user | Auditor | Scanner adapter |
|---|---|---|---|---|---|---|
| Register / manage users | — | Yes (admin provisioning) | — | — | — | — |
| Complete own MFA / profile | Login only | Own | Own | Own | Own | — |
| Submit manual upload | — | Policy-scoped | Yes | — | — | — |
| Start scan job | — | Policy-scoped | Yes, approved device | — | — | — |
| Ingest scan bytes | — | — | — | — | — | mTLS + bound job only |
| Read metadata | — | Admin metadata | Own | Allowed metadata | Audit metadata | — |
| Request document access | — | Policy-scoped | Yes, other owners | Yes, discoverable | — | — |
| Approve / reject / revoke | — | Only delegated/scoped | Own document | — | — | — |
| Retrieve plaintext | — | Only explicit grant | Own active doc | Active grant | — | — |
| Read audit history | — | Scoped | Own activity | — | Redacted assigned scope | — |
| Manage scanners | — | Yes | — | — | Read redacted | — |
| Read ledger metadata | — | Scoped | Own | Only if authorized | Scoped | — |
**Other hard rules:** admin cannot self-grant decryption; owner cannot approve their own request as a bypass; auditor never receives keys/plaintext; adapter cannot list documents or choose an owner arbitrarily; user cannot set `role`, `owner_id`, `status`, `hash`, `cid` or `key_reference` via JSON. Cross-organization access denied. Permission expiry is checked at retrieval time. Audit records are redacted.

## 5. HTTP endpoint contract inventory — 21 existing + 7 proposed scanner routes = 28
**Conventions:** `/api/v1` version prefix recommended at implementation (existing routes below shown in original unversioned form); HTTPS; `application/json` except multipart upload or document stream; RFC 7807-style safe error body `{type,title,status,code,correlation_id}`; strict JSON schema and size limits; UTC timestamps; `Idempotency-Key` for side-effecting operations; opaque UUIDs; stable 400/401/403/404/409/413/415/429/503 handling. Auth cookie must be HttpOnly, Secure, SameSite; state-changing cookie-authenticated requests require CSRF token and Origin check. Internal scanner endpoint uses mTLS and workload/job identity, not browser session.
**Response code note:** values below are proposed contracts; implement OpenAPI and approval test before treating them as final. Role names denote contextual checks, not unrestricted access.
### AUTH-01 — `POST /api/auth/register`
- **Purpose:** Create invited account; no public privileged registration. **Authorized actor:** Admin. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** email, username, display_name, role invitation. **Response/status contract:** 201 user_id; 400/409/403/429.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `AUTH-01-P01`, `AUTH-01-N01`, `AUTH-01-A01`, `AUTH-01-B01`, `AUTH-01-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### AUTH-02 — `POST /api/auth/login`
- **Purpose:** Validate password, start restricted MFA challenge. **Authorized actor:** Public. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** username, password. **Response/status contract:** 202 challenge_id; 401/429.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `AUTH-02-P01`, `AUTH-02-N01`, `AUTH-02-A01`, `AUTH-02-B01`, `AUTH-02-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### AUTH-03 — `POST /api/auth/totp/setup`
- **Purpose:** Bind authenticator for enrollment only. **Authorized actor:** Authenticated in MFA enrollment. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** challenge_id or step-up session. **Response/status contract:** 200 provisioning URI once; 401/403/409.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `AUTH-03-P01`, `AUTH-03-N01`, `AUTH-03-A01`, `AUTH-03-B01`, `AUTH-03-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### AUTH-04 — `POST /api/auth/totp/verify`
- **Purpose:** Verify code and rotate into full session. **Authorized actor:** MFA challenge. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** challenge_id, totp_code. **Response/status contract:** 200 session; 401/429.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `AUTH-04-P01`, `AUTH-04-N01`, `AUTH-04-A01`, `AUTH-04-B01`, `AUTH-04-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### AUTH-05 — `POST /api/auth/logout`
- **Purpose:** Invalidate server-side session. **Authorized actor:** Authenticated. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** CSRF token. **Response/status contract:** 204; 401/403.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `AUTH-05-P01`, `AUTH-05-N01`, `AUTH-05-A01`, `AUTH-05-B01`, `AUTH-05-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### AUTH-06 — `GET /api/auth/profile`
- **Purpose:** Return minimal profile and roles. **Authorized actor:** Authenticated. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** none. **Response/status contract:** 200 safe profile; 401.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `AUTH-06-P01`, `AUTH-06-N01`, `AUTH-06-A01`, `AUTH-06-B01`, `AUTH-06-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### DOC-01 — `POST /api/documents`
- **Purpose:** Submit validated encrypted upload via unified ingestion. **Authorized actor:** Owner/Admin-scoped. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** multipart file, title, category, idempotency_key. **Response/status contract:** 202 processing document_id; 400/413/415/429/503.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `DOC-01-P01`, `DOC-01-N01`, `DOC-01-A01`, `DOC-01-B01`, `DOC-01-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### DOC-02 — `GET /api/documents`
- **Purpose:** List only documents user may discover. **Authorized actor:** Authenticated. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** cursor, page_size, filters. **Response/status contract:** 200 paginated list; 400/401.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `DOC-02-P01`, `DOC-02-N01`, `DOC-02-A01`, `DOC-02-B01`, `DOC-02-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### DOC-03 — `GET /api/documents/{id}`
- **Purpose:** Return scoped metadata only. **Authorized actor:** Owner/approved requester/Admin-metadata. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** document_id. **Response/status contract:** 200 metadata; 404/403.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `DOC-03-P01`, `DOC-03-N01`, `DOC-03-A01`, `DOC-03-B01`, `DOC-03-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### DOC-04 — `GET /api/documents/{id}/retrieve`
- **Purpose:** Authorize, verify hash and GCM tag, stream decrypted output. **Authorized actor:** Owner/active grantee. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** document_id, fresh step-up if required. **Response/status contract:** 200 secure stream; 401/403/404/409/503.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `DOC-04-P01`, `DOC-04-N01`, `DOC-04-A01`, `DOC-04-B01`, `DOC-04-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### DOC-05 — `GET /api/documents/{id}/integrity`
- **Purpose:** Verify ciphertext hash against committed Fabric record. **Authorized actor:** Owner/Auditor-summary/Admin-metadata. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** document_id. **Response/status contract:** 200 integrity summary; 403/404/503.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `DOC-05-P01`, `DOC-05-N01`, `DOC-05-A01`, `DOC-05-B01`, `DOC-05-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### DOC-06 — `DELETE /api/documents/{id}`
- **Purpose:** Disable future access; lifecycle with retention. **Authorized actor:** Owner/Admin-with-policy. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** document_id, reason, idempotency_key. **Response/status contract:** 202 revoked; 403/404/409.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `DOC-06-P01`, `DOC-06-N01`, `DOC-06-A01`, `DOC-06-B01`, `DOC-06-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### ACC-01 — `POST /api/access/request`
- **Purpose:** Request access to discoverable document. **Authorized actor:** Authenticated eligible requester. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** document_id, reason, idempotency_key. **Response/status contract:** 201 request_id; 403/404/409.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `ACC-01-P01`, `ACC-01-N01`, `ACC-01-A01`, `ACC-01-B01`, `ACC-01-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### ACC-02 — `GET /api/access/pending`
- **Purpose:** List only actionable access requests. **Authorized actor:** Owner of target/Admin-scoped. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** cursor. **Response/status contract:** 200 scoped list; 403.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `ACC-02-P01`, `ACC-02-N01`, `ACC-02-A01`, `ACC-02-B01`, `ACC-02-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### ACC-03 — `POST /api/access/{id}/approve`
- **Purpose:** Atomic approval and bounded permission expiry. **Authorized actor:** Owner/Admin-delegated. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** request_id, expires_at, reason, idempotency_key. **Response/status contract:** 200 approval; 403/404/409.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `ACC-03-P01`, `ACC-03-N01`, `ACC-03-A01`, `ACC-03-B01`, `ACC-03-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### ACC-04 — `POST /api/access/{id}/reject`
- **Purpose:** Reject pending request once. **Authorized actor:** Owner/Admin-delegated. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** request_id, reason. **Response/status contract:** 200 rejected; 403/404/409.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `ACC-04-P01`, `ACC-04-N01`, `ACC-04-A01`, `ACC-04-B01`, `ACC-04-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### ACC-05 — `POST /api/access/{id}/revoke`
- **Purpose:** Revoke existing grant and invalidate future retrieval. **Authorized actor:** Owner/Admin-delegated. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** permission_id or request_id, reason. **Response/status contract:** 200 revoked; 403/404/409.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `ACC-05-P01`, `ACC-05-N01`, `ACC-05-A01`, `ACC-05-B01`, `ACC-05-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### AUD-01 — `GET /api/audit`
- **Purpose:** Return redacted tamper-evident audit records. **Authorized actor:** Auditor/Admin-scoped/Owner-own. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** filters, cursor. **Response/status contract:** 200 audit rows; 403/429.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `AUD-01-P01`, `AUD-01-N01`, `AUD-01-A01`, `AUD-01-B01`, `AUD-01-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### FAB-01 — `GET /api/blockchain/document/{id}`
- **Purpose:** Return non-sensitive Fabric document record. **Authorized actor:** Owner/Auditor/Admin-metadata. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** document_id. **Response/status contract:** 200 record; 403/404/503.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `FAB-01-P01`, `FAB-01-N01`, `FAB-01-A01`, `FAB-01-B01`, `FAB-01-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### FAB-02 — `GET /api/blockchain/history/{id}`
- **Purpose:** Return sanitized ledger events. **Authorized actor:** Owner/Auditor/Admin-metadata. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** document_id, cursor. **Response/status contract:** 200 history; 403/404/503.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `FAB-02-P01`, `FAB-02-N01`, `FAB-02-A01`, `FAB-02-B01`, `FAB-02-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### FAB-03 — `GET /api/blockchain/transaction/{id}`
- **Purpose:** Return transaction status with no sensitive payload. **Authorized actor:** Scoped participant/Auditor. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** transaction_id. **Response/status contract:** 200 status; 403/404/503.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `FAB-03-P01`, `FAB-03-N01`, `FAB-03-A01`, `FAB-03-B01`, `FAB-03-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### SCN-01 — `POST /api/scanners/enroll`
- **Purpose:** Enroll mutually authenticated adapter/device. **Authorized actor:** Admin + device approval. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** device_public_key, attestation, label, location_code. **Response/status contract:** 201 scanner_id; 400/403/409.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `SCN-01-P01`, `SCN-01-N01`, `SCN-01-A01`, `SCN-01-B01`, `SCN-01-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### SCN-02 — `GET /api/scanners`
- **Purpose:** List registered and revoked scanners. **Authorized actor:** Admin/Auditor-redacted. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** cursor. **Response/status contract:** 200 devices; 403.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `SCN-02-P01`, `SCN-02-N01`, `SCN-02-A01`, `SCN-02-B01`, `SCN-02-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### SCN-03 — `POST /api/scanners/{id}/disable`
- **Purpose:** Disable compromised scanner certificate/identity. **Authorized actor:** Admin. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** scanner_id, reason. **Response/status contract:** 200 disabled; 403/404/409.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `SCN-03-P01`, `SCN-03-N01`, `SCN-03-A01`, `SCN-03-B01`, `SCN-03-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### SCN-04 — `POST /api/scans/jobs`
- **Purpose:** Queue authorized scan job; prohibit arbitrary remote targets. **Authorized actor:** Owner/Admin-scoped + approved scanner. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** scanner_id, profile_id, pages_limit, document_owner_id. **Response/status contract:** 202 job_id; 403/404/409/429.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `SCN-04-P01`, `SCN-04-N01`, `SCN-04-A01`, `SCN-04-B01`, `SCN-04-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### SCN-05 — `GET /api/scans/jobs/{id}`
- **Purpose:** Report non-sensitive scan state. **Authorized actor:** Job owner/Admin-scoped. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** job_id. **Response/status contract:** 200 state; 403/404.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `SCN-05-P01`, `SCN-05-N01`, `SCN-05-A01`, `SCN-05-B01`, `SCN-05-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### SCN-06 — `POST /api/scans/jobs/{id}/cancel`
- **Purpose:** Cancel queued/running scan safely. **Authorized actor:** Job owner/Admin-scoped. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** job_id. **Response/status contract:** 202 cancellation; 403/404/409.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `SCN-06-P01`, `SCN-06-N01`, `SCN-06-A01`, `SCN-06-B01`, `SCN-06-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).
### SCN-07 — `POST /internal/scans/ingest`
- **Purpose:** Accept scan bytes over mTLS private network into common ingestion. **Authorized actor:** Authenticated scanner adapter ONLY. **API owner:** Harini. **Security reviewer:** Deepthi. **UI/test owner:** Varshith.
- **Input:** job_id, document bytes, digest, idempotency_key. **Response/status contract:** 202 accepted; 401/403/409/413/415.
- **Required checks:** schema, authentication mechanism, correct object/organization scope, safe audit data and rate/size limits as applicable. Reject unknown mutable security fields. Test success, invalid input, anonymous request, disallowed role, other-owner ID, stale/replayed request and dependent-service fault; exceptions for public auth and device-only routes are enforced per contract.
- **Traceable tests:** `SCN-07-P01`, `SCN-07-N01`, `SCN-07-A01`, `SCN-07-B01`, `SCN-07-F01` (defined in companion CSV; replace inapplicable authorization preconditions with explicit not-applicable records).

### Example JSON contracts (synthetic data only)
```json
POST /api/access/request
{"document_id":"11111111-1111-4111-8111-111111111111","reason":"Project review"}
201 {"request_id":"22222222-2222-4222-8222-222222222222","status":"PENDING"}

POST /api/scans/jobs
{"scanner_id":"33333333-3333-4333-8333-333333333333","profile_id":"A4_300DPI_GRAY","pages_limit":3}
202 {"job_id":"44444444-4444-4444-8444-444444444444","status":"QUEUED"}

GET /api/documents/11111111-1111-4111-8111-111111111111/integrity
200 {"status":"VERIFIED","verified_at":"2026-10-08T00:00:00Z"}
```
**Specific design caveats:** never return TOTP seed after enrollment, raw master key, password hash, verbose exception, private scan path, raw PII, or document plaintext to audit endpoints. Do not expose `/internal/*` through public ingress. Retrieval should set `Cache-Control: no-store`, restrictive `Content-Disposition`, `X-Content-Type-Options: nosniff` and avoid proxy caching. POST scan ingest should use bounded streaming ingestion and reject unsupported content safely.

## 6. Team work ownership, reviews, and RACI
| Work package | Accountable/implementer | Collaborator | Independent reviewer / test executor | Evidence |
|---|---|---|---|---|
| Project scope, approvals, threat model | Deepthi | All | Harini + Varshith | SRS, threat register, signed scope |
| Encryption, TOTP, RBAC, keys | Deepthi | Harini | Varshith + Harini | Unit/security tests and design review |
| Fabric chaincode, gateway permissions | Deepthi | Harini | Varshith | Fabric tests, transaction IDs |
| Flask APIs, validation, errors | Harini | Deepthi | Varshith | OpenAPI, pytest, API reports |
| PostgreSQL, migrations, backup | Harini | Deepthi | Varshith | Schema, migration checks, restore drill |
| IPFS and outbox integration | Harini | Deepthi | Varshith | Fault injection, CID verification |
| Physical scanner adapter/API | Harini | Deepthi device security | Varshith hardware test | Compatibility sheet, sandbox and spool checks |
| React UI and accessibility | Varshith | Harini | Deepthi | UI tests, screen recordings |
| Test matrix, CI and release reports | Varshith | All | Deepthi signs off security; Harini signs off API | CSV, test results, defect register |
| Final report, PPT and reviews | Deepthi | All | Guide/team | Versioned documents and references |
**Branch/merge policy:** PR per task; no direct commits to main; mandatory checks and at least one independent reviewer; security-sensitive PRs additionally reviewed by Deepthi (or independent Harini if Deepthi authored); secret scan + SAST + dependency audit + tests block merge. Per-task worklog fields: requirement ID, owner, implementation PR, interfaces touched, test IDs, execution date, evidence links, defect IDs, reviewer approval.

## 7. Test data design and execution plan
**Synthetic fixtures only:**
| Fixture | Data/setup | Reuse |
|---|---|---|
| U-ADMIN | admin-A with ADMIN role, no automatic download grant | Admin endpoints |
| U-OWN-A | owner-A in organization A, active MFA | Owner operations |
| U-OWN-B | owner-B in organization B, active MFA | Cross-scope denial |
| U-GRANT | requester-A with active grant to DOC-OK | Approved retrieval |
| U-NOGRANT | requester-A with no grant | Unauthorized retrieval |
| U-AUD | auditor-A, read-only assigned scope | Audit queries |
| U-LOCK | user with repeated wrong passwords | Rate limit and lockout |
| DEV-OK | approved and mTLS-authenticated adapter-A | Physical scan job |
| DEV-BAD | revoked/unknown adapter-B certificate | Scanner denial |
| DOC-OK | 2-page synthetic PDF <= configured limit, committed hash, valid GCM tag | Happy path |
| DOC-IMG | Small valid JPEG and PNG samples | File variants |
| DOC-BAD | Mismatched extension/signature, invalid structure, PDF bomb test fixture | Upload rejection |
| DOC-TAMP | DOC-OK ciphertext with one byte changed | Tamper detection |
| DOC-REVOKED | Approved grant then revoked/expired | Revocation |
| DOC-PENDING | IPFS saved but Fabric registration uncommitted | Partial failure |
| JOB-OK | A4_300DPI_GRAY, one to three pages, authorized user | Physical job success |
| JOB-REPLAY | Repeat same adapter upload nonce/idempotency key | Replay rejection/deduplication |
**Testing layers:** pytest unit tests (cryptographic format, auth, validation, policy); Flask API contract and fuzz/property tests; PostgreSQL migration and rollback tests; isolated IPFS and Fabric integrations; React component and browser E2E; scanner emulator contract tests, then certified hardware acceptance; CI SAST/dependency/secret/config/DAST in isolated environment; chaos/fault-injection; performance baselines; manual privacy assessment. Use only authorized test systems. Never attack public infrastructure.
**Traceability:** Every endpoint has five planned case families in CSV; security scenarios below add deeper cross-endpoint coverage. All expected statuses require test evidence before marking passed. Execution fields: actual_result, run_status, commit_sha, executed_at, environment, tester, artifact_uri, defect_id. Default is NOT_RUN.

### Critical security/end-to-end test set
| ID | Scenario | Fixture/action | Expected result |
|---|---|---|---|
| SEC-01 | Mass-assignment role escalation | Send `role=ADMIN` in registration/update | No privileged role granted; 400/403 |
| SEC-02 | IDOR cross-tenant metadata | U-OWN-B requests DOC-OK owned in org A | 404/403 with zero content |
| SEC-03 | Credential stuffing | Repeated invalid logins | 429/lockout; generic errors; recorded audit |
| SEC-04 | TOTP time and replay | Reuse TOTP code after success | Replay denied; no second session |
| SEC-05 | CSRF and fixation | Forged state change / prelogin session reuse | CSRF denied; session rotated |
| SEC-06 | SQLi input handling | Harmless SQL meta-character payload in filter | No query logic change; parameterized query |
| SEC-07 | XSS output handling | Encoded HTML/JS test string in title | Displayed as text; no execution |
| SEC-08 | Revoked permission retrieval | Revoke U-GRANT then fetch | 403; no plaintext bytes |
| SEC-09 | Ciphertext tamper | Flip a byte in DOC-TAMP | No decryption; alert and audit |
| SEC-10 | PII ledger/log leakage | Scan audit/IPFS/Fabric sample artifacts for synthetic marker | No raw markers, keys, OTP seeds |
| SEC-11 | Malicious file and resource limits | Wrong signature, oversized, decompression/page bomb | 415/413/422; bounded CPU and memory |
| SEC-12 | Revocation race | Revoke grant while concurrent retrieval begins | No new retrieval after committed revocation; established streams handled per specified policy |
| SEC-13 | Audit failure durability | Simulate DB commit/outbox and ledger delay | Monotonic event, retriable pending status |
| SEC-14 | Rogue adapter | Unknown or revoked mTLS identity invokes ingest | Handshake/authorization rejection |
| SEC-15 | Scan replay/wrong job | Resend completed job bytes with old nonce | Idempotent no duplicate; mismatch denied |
| SEC-16 | Plaintext exposure | Inspect temp dirs, HTTP headers and log fixture | No unintended stored plaintext; no-store |
| SEC-17 | Scanner SSRF | Send URL or unauthorized scanner_id | No network request to target; validation rejection |
| SEC-18 | Printer spool handling | Complete/cancel jammed hardware scan and inspect adapter spool | No residual plaintext in managed spool; firmware risk documented |
| SEC-19 | Fabric caller spoofing | Invoke chaincode with unauthorized service identity | Rejected by identity/policy |
| SEC-20 | Dependency outage | Disconnect IPFS/Fabric mid upload | Safe non-ACTIVE state; idempotent retry |
| SEC-21 | Key loss/rotation | Rotate wrapping key and recover document | Old docs decrypt with authorized version; new docs use new version |
| SEC-22 | Backup/restore privacy | Restore encrypted DB metadata and keys in isolated test | No plaintext leak; references consistent |
| SEC-23 | Session expiry | Idle/absolute timeout, then sensitive retrieval | 401; no data |
| SEC-24 | Hardware profile mismatch | Unsupported duplex/resolution on actual scanner | Clear error; no unsafe fallback |

## 8. Defect and evidence handling
**Defect fields:** defect_id, date, module, endpoint, requirement_id, test_id, build_sha, environment, severity (critical/high/medium/low), steps, sanitized sample payload, expected, actual, screenshot/log link, assignee, root cause, fix PR, retest run, status, reviewer. **Definition of Done:** code reviewed, endpoint contract validated, all security role tests pass, dependency faults handled, logs redacted, tests reproduced in CI, and evidence archived. **Release gate:** zero known unresolved critical/high; 100% endpoint inventory documented, 100% applicable role × endpoint tests executed, ≥90% security-critical code coverage target, no released real-PII fixtures, threat model and hardware risk decisions signed off. Coverage alone never proves absence of vulnerabilities.

## 9. Week 1 day-by-day ownership and acceptance gates
| Day | Deepthi | Harini | Varshith | Exit artifact |
|---|---|---|---|---|
| 1 | Confirm scope, define roles, kickoff | Map Flask/DB/API dependencies | Inventory React flows + demo tests | Scope and assumption log |
| 2 | STRIDE risks, key boundaries | Define canonical data model and upload contract | Write role-case inventory | Threat model + data model |
| 3 | MFA/RBAC/chaincode policy | Draft 28 API contracts and schemas | Prepare positive/negative fixtures | Draft OpenAPI + test register |
| 4 | Scanner security and Fabric approvals | Inspect vendor scanner support + adapter feasibility | Draft scanner emulator/hardware matrix | Device compatibility decision |
| 5 | Cross-review risk acceptance | Cross-review API/schema + retries | Audit every case and coverage | Signed Week 1 baseline and backlog |

## 10. Open decisions (must close before relevant implementation)
1. **Scanner hardware:** vendor/model, OS, TWAIN/WIA/SANE/eSCL/IPP/SMB/SFTP capability, physical availability, secure spool cleanup guarantees.
2. **Registration:** proceed with admin-controlled invitations as proposed, or formally approve another policy.
3. **Plaintext policy:** browser viewer only versus downloaded file; retention and post-revocation expectation.
4. **Operational limits:** max upload bytes, PDF pages, scanner DPI, processing CPU/time and storage quota.
5. **Organizational scope:** one organization prototype or multiple departments/tenants; audit authority.
6. **Key management:** dev env secret for local prototype versus KMS/Vault for deployment, backup and recovery ownership.
7. **Retention/compliance:** original and encrypted file expiry, deletion and spool policy, consent/authorization for real scanning.
8. **Fabric trust:** organization count, endorsement policy, user/device identity mapping and ledger data privacy.
9. **Performance acceptance:** measure and approve latency/throughput on actual scanner and reference deployment.
10. **Release artifacts:** planned CI platform, scanner emulator, test network, demonstration environment.

## 11. Source reference notes
- **Project design.pdf:** technology/architecture pp. 1–4; team/role matrix p.5; flows pp.6–11; chaincode pp.12–13; DB pp.14–16; 21 API routes pp.17–18; frontend pp.18–19; security p.22; deployment pp.23–24.
- **Project Planning.pdf:** original prototype uses simulated upload (p.1); member work assignments (pp.1–4); eight-week plan (pp.5–11); repo plan (pp.12–13); original 15 example tests (p.15).
- **B1-REVIEW-1.pptx:** project motivation/objectives, proposed workflow, and evaluation goals (abstract/objectives/methodology slides).
- This document explicitly labels additional scanner integration details and hardening decisions as **proposed**, not statements in the original files.