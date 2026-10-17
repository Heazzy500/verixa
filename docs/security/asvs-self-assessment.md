# OWASP ASVS 4.0.3 Platform Self-Assessment

## Scope and confidence

This assessment supersedes the credentials-only assessment below. It evaluates
the repository at the revision containing this document, with emphasis on the
application code in `apps/api` and `packages/*`. It does not certify a deployed
system: deployment configuration, identity-provider settings, network policy,
backups, and operator procedures must be verified by each deployment owner.

The target is **OWASP ASVS 4.0.3, Level 1**, with Level 2 controls included
where Verixa handles authentication, authorization, identity evidence, or
audit records. ASVS 4.0.3 is used to preserve the chapter mapping expected by
the existing V2 assessment and this project's Phase 11 requirements. The
standard's official [release and requirement identifiers](https://github.com/OWASP/ASVS/tree/v4.0.3) are version-pinned because control numbers change across releases.

This is a source review, not a penetration test. A **PASS** means the cited
code or automated check provides evidence for the specific control in the
repository; it does not assert production configuration. **FAIL** identifies a
missing or incomplete control. **N-A** means the requirement does not apply to
the API-only surface in this repository, with the reason stated. Partial
implementations are failures. The linked phase specification is the tracked
follow-up area for each gap; maintainers should split individual work items
when scheduling them.

## Chapter assessment

| ASVS chapter | Status | Evidence and rationale |
| --- | --- | --- |
| V1 Architecture, Design and Threat Modeling | PASS | Context-specific threat models and architecture constraints exist under `docs/security/` and `planning/ARCHITECTURE.md`; the dependency direction is enforced in code and lint configuration. This is a source-level review, not an independent threat-model workshop. |
| V2 Authentication | PARTIAL | Password hashing, password policy, account lockout, verification, reset, and MFA implementations are documented below. Rate limiting remains a no-op in composition, and production mail delivery is not configured here. Track hardening in [Phase 11](../../planning/issues/phase-11-security-hardening.md) and [Phase 15](../../planning/issues/phase-15-rate-limiting.md). |
| V3 Session Management | PARTIAL | Signed access tokens, hashed refresh tokens, expiry, revocation and reuse detection are implemented and described in `docs/security/threat-model-sessions.md`. Browser cookie policy and deployment key rotation are not established by this API repository. Track in [Phase 11](../../planning/issues/phase-11-security-hardening.md) and [Phase 20](../../planning/issues/phase-20-docker-deployment.md). |
| V4 Access Control | PARTIAL | RBAC/ABAC policy evaluation and guarded authorization routes exist. Tenant-boundary behavior and route-by-route authorization coverage still require deployment-specific integration review. Track in [Phase 07](../../planning/issues/phase-07-rbac.md), [Phase 08](../../planning/issues/phase-08-abac-policy-engine.md), and [Phase 12](../../planning/issues/phase-12-rest-api.md). |
| V5 Validation, Sanitization and Encoding | PARTIAL | Fastify JSON schemas reject unknown request properties, and domain value objects validate inputs. A platform-wide inventory of every input/output boundary and context-specific encoding is not complete. Track in [Phase 12](../../planning/issues/phase-12-rest-api.md). |
| V6 Stored Cryptography | PARTIAL | Passwords and bearer tokens are stored as hashes using the dedicated credential/session implementations. Database encryption-at-rest and key custody are external deployment controls, not proven by this repository. Track in [Phase 11](../../planning/issues/phase-11-security-hardening.md) and [Phase 20](../../planning/issues/phase-20-docker-deployment.md). |
| V7 Error Handling and Logging | PARTIAL | Shared error response mapping, structured logging, redaction, and audit-chain verification are implemented. A complete assurance that every error path and deployment log sink excludes secrets requires an operational review. Track in [Phase 11](../../planning/issues/phase-11-security-hardening.md) and [Phase 18](../../planning/issues/phase-18-observability.md). |
| V8 Data Protection | PARTIAL | Credential/token serialization avoids exposing secret material, and audit metadata is schema-constrained. Retention, backup deletion, and data-subject lifecycle depend on deployment policy and are not fully verified here. Track in [Phase 24](../../planning/issues/phase-24-compliance-privacy.md). |
| V9 Communications | PARTIAL | The application uses HTTPS-capable server deployment and validates configured origins in relevant security-sensitive flows. TLS termination, certificate lifecycle, and secure transport between all production components are operator controls. Track in [Phase 20](../../planning/issues/phase-20-docker-deployment.md). |
| V10 Malicious Code | PARTIAL | Dependency lockfiles and CI checks provide a reproducible dependency set, but continuous vulnerability alerting and a documented response SLA are not established by the application source alone. Track in [Phase 19](../../planning/issues/phase-19-cicd-automation.md). |
| V11 Business Logic Security | PARTIAL | Invariants live in domain value objects/use cases and have focused tests. This assessment did not execute a complete abuse-case review of every workflow or concurrency boundary. Track gaps with the relevant context phase specification. |
| V12 Files and Resources | N-A | The assessed API does not currently accept user-uploaded files or expose file-serving endpoints. Reassess if a context adds uploads, exports, or user-controlled file retrieval. |
| V13 API and Web Service | PARTIAL | Fastify schemas and explicit route handlers define API boundaries; OpenAPI work is in progress. Authentication, authorization, pagination, versioning, and error contracts are not yet assessed as a single complete surface. Track in [Phase 12](../../planning/issues/phase-12-rest-api.md). |
| V14 Configuration | PARTIAL | Environment parsing validates required secrets and fails closed for selected security-sensitive settings. A complete secure-default review of every deployment option, header, CORS/CSRF policy, and secret rotation process remains. Track in [Phase 11](../../planning/issues/phase-11-security-hardening.md) and [Phase 20](../../planning/issues/phase-20-docker-deployment.md). |

## Evidence by security area

### Authentication and credentials (V2)

The previous V2 assessment remains useful as implementation evidence, but its
scope and version claims are superseded by this document. The code uses Argon2
for password hashing, central password policy validation, account lockout,
email verification and password-reset tokens stored as digests, and
single-use recovery flows. MFA implementations include WebAuthn verification.
Relevant sources include:

- `packages/credentials/domain/value-objects/raw-password.ts`
- `packages/credentials/infrastructure/crypto/argon2-password-hasher.ts`
- `packages/credentials/application/use-cases/authenticate-with-password.ts`
- `packages/credentials/application/use-cases/confirm-password-reset.ts`
- `packages/mfa/infrastructure/webauthn/`
- `docs/security/authentication-flows.md`

A no-op rate limiter is wired in `apps/api/src/composition-root.ts`. It is a
known gap, not a production defense. Until a real adapter is configured,
operators need a gateway limit and monitoring. Account lockout alone is not a
substitute: attackers can use it to deny service to targeted users.

### Sessions and authorization (V3–V4)

Session token signing and verification, hashed refresh-token persistence,
expiry, revocation, and replay detection are covered in
`packages/sessions/` and `docs/security/threat-model-sessions.md`. The API
composition root requires a configured signing secret rather than silently
using a development key. This repository does not establish browser cookie
flags, reverse-proxy trust, or production secret rotation; those need deployment
verification.

RBAC and policy evaluation live in `packages/authorization/`. A library-level
guard is not evidence that every exposed route uses it correctly. Review each
route with a principal and tenant boundary test before relying on the result in
a multi-tenant deployment. Known architecture and phase work is tracked in
the Phase 07, 08, and 12 specifications above.

### Input, output, errors, and logs (V5, V7, V11, V13)

The Fastify application sets `removeAdditional: false`; malformed requests
with undeclared fields are rejected instead of silently rewritten. Route
schemas validate request shape, while domain objects enforce business
invariants. This division avoids maintaining the same policy twice. It does
not by itself prove safe output encoding in every future consumer, and this
API has no browser templating layer to assess for HTML encoding.

Shared error mapping avoids leaking internal exception details through
responses. Structured logs redact configured sensitive fields, while audit
records use action-specific metadata schemas. Operators must still protect
log access, retention, backups, and alerting. An audit log is not a substitute
for operational logs, and neither should record passwords, tokens, or private
keys.

### Cryptography, data, and communications (V6, V8, V9)

Passwords and refresh tokens are not stored in recoverable form. Cryptographic
keys used for tokens and Stellar anchoring have dedicated configuration and
key-management guidance. These code-level decisions do not demonstrate
storage-volume encryption, TLS between services, certificate rotation, backup
protection, or key custody in a deployed environment. Deployment owners must
record those controls and verify their evidence separately.

### Supply chain, configuration, and operations (V10, V14)

The repository pins dependency resolution in `pnpm-lock.yaml` and has CI
workflows for quality checks. A lockfile proves a resolution, not that every
transitive package is free of known vulnerabilities or that alerts are
reviewed. The Phase 19 specification tracks pipeline automation. Configuration
is parsed through `packages/config`; selected dangerous settings fail closed,
but this source review did not validate every production environment or
secret-rotation procedure. See the Phase 11 and Phase 20 specifications.

## Earlier credentials-only findings

The old V2 assessment described password maximum length as a failure because
the configured cap exceeded the cited ASVS threshold. Treat that as an
unresolved policy decision until the credential owner explicitly maps the
current rule to the chosen standard requirement. Do not change a password
limit solely to make this table look better: compare denial-of-service
protection, password-manager compatibility, and the precise ASVS wording first.

## Verification and maintenance

For each PASS, reviewers should be able to follow the cited path and inspect
its tests. For each PARTIAL, the referenced phase specification is the tracked
follow-up area; maintainers should open a narrowly scoped issue when the work
is scheduled. This document must be re-reviewed after changes to authentication,
sessions, authorization, API routes, cryptographic storage, dependency policy,
or deployment configuration.

A future revision should include a requirement-by-requirement checklist keyed
to the pinned ASVS release and attach test or deployment evidence. This
chapter-level mapping is an honest first platform-wide assessment, not a
claim that every individual ASVS requirement has been verified.
