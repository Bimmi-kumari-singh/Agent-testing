# 1) Title and Versioning
- Title: Authentication System — Business Requirements Document (BRD)
- Author: Product / IAM Team
- Reviewers: Security Team, Platform (SRE/DevOps), QA, Legal/Compliance, Product Owner
- Date: 2026-04-15
- Version: 1.0

# 2) Executive Summary
Purpose:
- Define business and technical requirements for a centralized, secure, and scalable Authentication System to provide registration, credential authentication, password recovery, multi-factor authentication (MFA), session management, SSO (OAuth2 / OpenID Connect), audit logging and administrative controls.

Business value:
- Reduce security risk and product maintenance overhead by centralizing identity; improve user experience via consistent auth flows; satisfy regulatory and compliance obligations (GDPR, audit trails); enable other services to rely on a single token-based identity provider.

Relevant source artifacts:
- High-level requirements and goals: .propel/context/docs/spec.md
- Architecture intent and non-functional targets: .propel/context/docs/design.md
- Existing BRD baseline: docs/Brd.md

# 3) Stakeholders
- Product Owner: accountable for product scope & prioritization (see EP-001..EP-007 in .propel/context/docs/epics.md)
- Platform / DevOps (SRE): deployment, availability, scaling (see EP-TECH and EP-007)
- Security Team: policy, pen-test sign-off, threat modelling (NFR-SEC responsibilities in .propel/context/docs/spec.md and .propel/context/docs/design.md)
- Compliance / Legal: GDPR, retention, data subject requests (retention mapped in FR-013)
- QA / Test: validation, automation and security testing (unit/integration test plans in .propel/context/docs/unit_test_plan.md and per-story test plans)
- Product Integrations / App Teams: web, mobile and internal services consuming tokens
- Admins / Support: administrative user management and unlock flows

# 4) Scope
4.1 In-scope (MUST / business-critical)
- User registration (email-based) with verification tokens — EP-001, user stories US_009..US_012 (.propel/context/tasks/EP-001/)
- Login with credentials and account state checks (pending / locked / active) — EP-002, US_013..US_016
- Password management: forgot-password, reset flow, token lifecycle — EP-004, US_020..US_021
- Multi-Factor Authentication (MFA): Email OTP, SMS OTP, TOTP (authenticator apps) — EP-005, US_022..US_025
- OAuth2 / OpenID Connect support for SSO integrations (IdP connectors) — TR-001/TR-002 references in .propel/context/docs/spec.md and design.md
- Session and token lifecycle: short-lived access tokens (JWT), refresh tokens with rotation and revocation, inactivity and absolute expiration, remember-me opt-in — EP-002/EP-003 (FR-SES-001..FR-SES-003)
- Logout and token revocation / blacklist — EP-003, US_019
- Account lockout (brute-force protection) with unlock (email/admin) — EP-006, US_028..US_030
- Audit logging & monitoring for all security-relevant events — DR-005, FR-LOG-002, tests in .propel/context/tasks/EP-001/us_010/us_010.md
- Administrative user management actions (unlock, force-reset, view audit) — EP-006 / US_030

4.2 Out-of-scope (for MVP / deferred)
- Biometric auth, social login/canonical federation beyond base OAuth/OIDC connectors (tagged as future enhancements in docs/Brd.md)
- Fully adaptive/risk-based ML-driven step-up beyond deterministic rules (FR-014 noted as AI-candidate)
- Cross-tenant multi-tenant policies (unless explicitly required per future epics)

# 5) Objectives and Success Metrics
Objectives:
- Provide a secure, single identity service for web, mobile and API clients.
- Reduce inconsistent auth implementations and support centralized policy enforcement (password policy, MFA, lockout).
- Improve observability and reduce support tickets for account recovery.

Primary Success Metrics (measurable; tie to NFRs):
- Login success rate > 95% for normal flows (spec: .propel/context/docs/spec.md)
- Login latency: 95th percentile < 2s under normal load (NFR-PERF-001)
- System availability: 99.9% uptime (NFR-AVAIL-001)
- No critical OWASP-Top-10 findings in external pen test (NFR-SEC-001)
- MFA adoption among privileged users > 20% in 6 months (spec goal)

Non-functional targets (NFRs):
- Strong password hashing: Argon2id preferred; fallback bcrypt with work factor documented (design.md: TR-004)
- TLS 1.2+ enforced for all endpoints (NFR-SEC-003)
- Rate limiting/default thresholds: per-IP and per-account rules (NFR-SEC-004 / FR-012)
- Token validation endpoint latency: introspection/JWKS verification < 200ms (FR-008 acceptance)
- Audit log retention and accessibility: default 1 year for audit events, configurable per compliance (FR-013)

# 6) Functional Requirements (numbered)
Each requirement is mapped to epics & user stories (file references). Acceptance criteria follow each FR.

FR-1: User Registration (FR-REG-001)
- Requirement: Support POST /api/v1/users/register accepting email, password, display name. Validate RFC-5322 email; enforce password policy; create user in PENDING_VERIFICATION; generate single-use time-limited verification token; enqueue verification email.
- Mapped to: EP-001 (see .propel/context/docs/epics.md), user stories US_009, US_010, US_011, US_012 (.propel/context/tasks/EP-001/us_009..us_012.md). Unit/integration test plan reference: .propel/context/tasks/EP-001/us_001/us_001.md and BE unit test plan in .propel/context/tasks/EP-001/us_001/unittest/test_plan_be_backend.md
- Acceptance Criteria:
  1. Valid registration -> 201 Created with userId, masked email, accountStatus=PENDING_VERIFICATION and verification token persisted (expires in 24h by default).
  2. Duplicate email -> 409 EMAIL_ALREADY_EXISTS, no new user created.
  3. Invalid email/password -> 400 with field-level errors.
  4. Email delivery failure -> 201 returned, token persisted, email queued for retry and metric emitted (verification_email_send_failure).

FR-2: Email Verification & Activation (FR-REG-006)
- Requirement: Token consumption endpoint to activate account; single-use tokens stored hashed; rotation/re-send logic.
- Mapped to: EP-001; user stories US_006, US_004, US_005 (.propel/context/tasks/EP-001/us_006.md, us_004.md, us_005.md)
- Acceptance Criteria:
  1. GET/POST /api/v1/verify?token=... -> activates account if token valid, marks used_at, returns 200/302.
  2. Expired token -> 410 Gone; UI shows resend CTA.
  3. Reuse or replay -> 410 or 400 "token already used".

FR-3: Login with Credentials (FR-LOGIN-001)
- Requirement: POST /api/v1/auth/login accepts email + password; validate, enforce account state, issue access + refresh tokens if successful; trigger MFA challenge if enabled.
- Mapped to: EP-002 and user stories US_013, US_014, US_015 (.propel/context/docs/epics.md and .propel/context/docs/user_stories.md)
- Acceptance Criteria:
  1. Valid credentials -> HTTP 200, JSON { access_token (JWT), refresh_token (opaque), expires_in }.
  2. MFA enabled -> return 200 with mfa_required flag and require second factor before issuing tokens.
  3. Invalid creds -> 401 with generic "Invalid credentials"; increment failed-login counter.
  4. Login response time: <2s for 95% of successful logins (NFR-PERF-001).

FR-4: Password Reset (FR-PASS-001)
- Requirement: Forgot-password flow sends time-limited (default 1h) single-use reset token to email; reset endpoint validates token and enforces password policy.
- Mapped to: EP-004, US_020, US_021 (.propel/context/docs/spec.md and .propel/context/tasks/EP-004)
- Acceptance Criteria:
  1. Request reset -> 202 Accepted; email sent (or queued) without revealing whether email exists.
  2. Reset token single-use, TTL default 1 hour.
  3. New password must pass policy (FR-004); reset invalidates previous refresh tokens and sessions.

FR-5: Password Policy (FR-PASS-002 / FR-004)
- Requirement: Enforce configurable password policy; default minimum 10 characters (or 12 as spec variations), upper/lower/digit/special character, no reuse of last 5 passwords, deny-list integration.
- Mapped to: .propel/context/docs/spec.md and FR-004 definitions.
- Acceptance Criteria:
  1. Password failing policy -> 400 with list of violations.
  2. New passwords checked against breach denylist (haveibeenpwned integration or local denylist) -> reject if compromised.

FR-6: Multi-Factor Authentication (FR-MFA-001)
- Requirement: Optional MFA with Email OTP, SMS OTP, TOTP authenticator apps; enrollment and recovery codes.
- Mapped to: EP-005 and user stories US_022..US_025 (.propel/context/docs/epics.md and tasks)
- Acceptance Criteria:
  1. Enroll flow provides QR/secret for TOTP and test OTP for Email/SMS.
  2. OTP TTL default 5 minutes, 6-digit codes.
  3. Recovery codes (10 single-use) generated and shown once on enrollment.
  4. Failed OTP attempts counted and subject to FR-012 rate limiting.

FR-7: OAuth2 / OpenID Connect SSO Support
- Requirement: Support OAuth2/OIDC connectors (IdP side) for SSO and federated login; expose standard endpoints (authorize, token, userinfo) or provide connectors to external IdPs.
- Mapped to: TR-001/TR-002 in .propel/context/docs/spec.md (integration requirements) and design.md
- Acceptance Criteria:
  1. External IdP connector supports standard OIDC flows (code, implicit where allowed).
  2. Token claims mapping documented; user provisioning/Just-In-Time (JIT) flows defined.
  3. SSO configuration stored encrypted, admin-only access.

FR-8: Session & Token Lifecycle (FR-SESS-001..FR-SESS-003, FR-006)
- Requirement: Issue short-lived access tokens (JWT) with exp claim and refresh tokens opaque & stored hashed server-side with rotation and revocation; provide refresh endpoint and revocation endpoint; remember-me opt-in extends session via refresh but subject to security rules.
- Mapped to: EP-002, EP-003, TR-002, TR-003; implementation guidance in design.md (JWT vs opaque) and TR-003/DR-004.
- Acceptance Criteria:
  1. Access token TTL default 15m; refresh token TTL default 30d; refresh rotation invalidates previous refresh token.
  2. Logout endpoint revokes both access (blacklist or token versioning) and refresh tokens immediately (FR-SES-003).
  3. Token introspection or JWKS endpoint provided for API Gateway validation (FR-008) and responds within <200ms under normal load.

FR-9: Account Lockout & Recovery (FR-ACC-001 / FR-007)
- Requirement: Lock account after configurable failed attempts (default 5 within 15 minutes); auto-unlock after configurable timeout or unlock via email/admin.
- Mapped to: EP-006, US_028, US_029, US_030 (.propel/context/docs/epics.md & tasks)
- Acceptance Criteria:
  1. After threshold, account set to LOCKED; login returns 423 Locked with neutral message.
  2. Unlock via email link (TTL 1h) or admin console; auto-unlock after 15m (configurable).

FR-10: Administrative User Management
- Requirement: Admin APIs for unlock, force-password-reset, list sessions, revoke sessions, role assignment; RBAC enforced.
- Mapped to: EP-006, US_030
- Acceptance Criteria:
  1. Admin endpoints accessible only to authorized admin principal; actions audited in audit logs.

FR-11: Audit Logging & Monitoring (FR-LOG-002 / DR-005)
- Requirement: Structured logs and immutable audit events for login success/failures, password reset, token issuance/revocation, lock/unlock, MFA events, admin actions. Centralized logging and metrics instrumented.
- Mapped to: .propel/context/tasks/EP-001/us_010/us_010.md and design.md
- Acceptance Criteria:
  1. Events logged with timestamp, user_id (or hashed), ip, event_type, correlation_id; retained per retention policy (default 1 year but configurable).
  2. Alerts for suspicious patterns (e.g., many failures from single IP) wired to PagerDuty/Slack.

FR-12: Token Validation Endpoint (FR-SES-002 / FR-008)
- Requirement: Provide a token introspection endpoint and JWKS for JWT verification; introspection requires mTLS or API key for API Gateway trust.
- Mapped to: EP-002 and TR-002
- Acceptance Criteria:
  1. Introspection/JWKS responds within <200ms; revoked tokens return active=false.

FR-13: Remember-me (Optional)
- Requirement: Allow secure "remember me" behavior using refresh tokens with limited extension; UI/clients consent required.
- Acceptance Criteria:
  1. Remember-me tokens issued only on explicit opt-in, limited lifespan and multi-device revocation available.

# 7) Non-Functional Requirements
7.1 Security
- Encryption in transit: TLS 1.2+ mandatory across all public/internal comms (NFR-SEC-003).
- Encryption at rest: database and MFA secrets encrypted; keys managed via KMS (design.md KMS references).
- Password hashing: Argon2id preferred; fallback bcrypt with paramization documented (TR-004 and design.md).
- OWASP hardening: follow OWASP Top 10; SAST + DAST + pen tests scheduled before production cutover (NFR-SEC-001).
- Secrets: do not log plaintext passwords, tokens, or secrets; use HashiCorp Vault/KMS for secrets management (design.md TR-009).

7.2 Performance
- Login response time: 95th percentile < 2s under typical load (NFR-PERF-001).
- Token introspection endpoint: <200ms typical latency.
- Concurrency: Support minimum 10,000 concurrent active sessions (NFR-PERF-002). Load tests and capacity planning tracked in EP-007 / US_035 / US_037.

7.3 Availability & Reliability
- Target availability: 99.9% uptime (NFR-AVAIL-001). Redundancy / HA across AZs (design.md deployment architecture).
- Automated failover for DB and caches; recovery time objective (RTO) targets in runbook (design.md NFR-REL-001).

7.4 Compliance & Privacy
- GDPR: Provide data subject export and deletion within 24 hours; soft-delete grace period default 30 days, audit retention default 1 year (FR-013).
- Audit trail retention, access controls and admin logging must comply with legal requirements (FR-013, .propel/context/docs/spec.md).

7.5 Logging & Monitoring
- Centralized logs (ELK or cloud logging), structured JSON logs, correlation-id propagation, Prometheus metrics and Grafana dashboards (design.md NFR-REL-002).
- Key metrics: login_success_rate, login_latency_ms, MFA_success_rate, failed_login_counts, token_revocation_events, audit_event_backlog.

# 8) Security Controls & Threat Model
8.1 Authentication flows
- Standard flows: username/password -> optional MFA -> token issuance (JWT + refresh).
- Token types: access token (JWT with exp claim) for API auth; refresh token opaque hashed server-side for revocation.
- Alternative: support opaque access tokens + introspection for highly-sensitive contexts; design allows pluggable token type.

8.2 Tokens, storage & signing
- Preferred: JWT signed with RS256 (asymmetric) for public verification by API Gateway; publish JWKS endpoint for distributed verification (design.md TR-002).
- Refresh tokens: opaque long-lived single-use tokens stored hashed in database or Redis; rotation on refresh (TR-003).
- Revocation: blacklisting store (Redis/Postgres) with token hash and expiry, or token versioning user-side to invalidate prior tokens.

8.3 Storage of secrets
- MFA secrets, IdP client secrets and signing keys stored encrypted via KMS/Vault; least privilege access.
- Regular key rotation policy for signing keys and secrets.

8.4 Rate limiting & brute-force protection
- Multi-layer rate-limiting at API Gateway (per-IP) and at application-level per-account (Redis counters) per FR-012 and design.md TR-008.
- CAPTCHA step-up after configurable threshold; account lockout enforced after configurable failed attempts.

8.5 CSRF / XSS / Other
- For browser-based flows favor HttpOnly, Secure, SameSite cookies for tokens to reduce XSS/CSRF risk.
- If bearer tokens used in SPA, recommend secure storage (not localStorage) and CSRF mitigations for cookie-based flows.
- Harden APIs against injection, parameter validation, and input sanitization per OWASP.

8.6 Threat mitigations summary
- Brute-force: rate-limiting, account lockout, CAPTCHA.
- Token theft: short-lived access tokens, refresh rotation, HTTPS-only, secure cookie flags.
- Credential leaks: Argon2id hashing, deny-lists, breach-detection processes.
- Replay: single-use tokens for password reset and verification.

# 9) API Contract Sketch
(High-level; implement OpenAPI spec as next deliverable)

9.1 Endpoints (examples)
- POST /api/v1/users/register
  - Request: { email, password, name }
  - Responses: 201 Created (user meta), 400 Bad Request, 409 Conflict
  - Maps: .propel/context/tasks/EP-001/us_009.md

- POST /api/v1/auth/login
  - Request: { email, password, remember_me? }
  - Responses: 200 { access_token, refresh_token, expires_in } OR 200 { mfa_required: true } OR 401 Invalid credentials, 423 Account locked
  - Maps: .propel/context/tasks/EP-002/us_013.md

- POST /api/v1/auth/mfa/verify
  - Request: { user_id, method, otp, challenge_id }
  - Responses: 200 tokens, 400 invalid/expired otp

- POST /api/v1/auth/refresh
  - Request: { refresh_token }
  - Responses: 200 new access + refresh tokens (rotation), 401 invalid/expired

- POST /api/v1/auth/logout
  - Request: Authorization: Bearer access_token
  - Responses: 200 OK, tokens revoked

- POST /api/v1/password/forgot
  - Request: { email } -> 202 Accepted (generic response), rate-limited

- POST /api/v1/password/reset
  - Request: { token, new_password } -> 200 OK OR 400 invalid/expired

- GET /.well-known/jwks.json
  - Public JWKS for JWT verification (cached rotation policy)

- POST /introspect
  - Auth: mTLS or API Key for API Gateway
  - Request: { token } -> { active: true/false, sub, exp, scopes... }

- Admin endpoints (RBAC):
  - POST /admin/users/{id}/unlock
  - GET /admin/users?query...
  - POST /admin/users/{id}/revoke-sessions

9.2 Error codes and status mapping (examples)
- 200 OK (success)
- 201 Created (resource created)
- 202 Accepted (async accepted — email queued)
- 400 Bad Request (validation, malformed)
- 401 Unauthorized (invalid token/credentials)
- 403 Forbidden (insufficient permissions)
- 409 Conflict (email already exists)
- 410 Gone (token used/expired)
- 423 Locked (account locked)
- 429 Too Many Requests (rate limit)
- 500 Internal Server Error (generic; correlation-id returned)

9.3 Request / Response examples
- Login success:
  {
    "access_token": "eyJhbGci...",
    "refresh_token": "r_abcdef...",
    "token_type": "Bearer",
    "expires_in": 900
  }

- Introspection success:
  {
    "active": true,
    "sub": "user-uuid",
    "scope": "openid profile",
    "exp": 1610000000
  }

# 10) Data Model
10.1 Core entities (as in design.md)
- users
  - user_id (UUID PK), email (unique, normalized), password_hash, password_salt (if used), name, status (PENDING_VERIFICATION, ACTIVE, LOCKED, DISABLED), created_at, updated_at, last_login_at, failed_login_attempts, lockout_expires_at
  - File reference: design.md DR-001

- verification_tokens
  - token_id (UUID), user_id FK, token_hash (store only hash), type (EMAIL_VERIFICATION, PASSWORD_RESET), expires_at, created_at, is_used

- mfa_config
  - mfa_id, user_id, mfa_method (EMAIL_OTP, SMS_OTP, TOTP), mfa_secret_encrypted, is_enabled, phone_number, recovery_codes_encrypted

- sessions / refresh_tokens
  - session_id (UUID), user_id, refresh_token_hash, issued_at, expires_at, last_used_at, device_info, revoked_at

- token_blacklist (for explicit revocation)
  - token_value_hash, expires_at, revoked_at

- audit_logs
  - log_id (UUID), timestamp, event_type, user_id (nullable), ip_address, details (JSONB)

10.2 Retention policy (default)
- Audit logs: 1 year (configurable) — FR-013 and us_010 plan
- Verification tokens: retained until used/expires + short grace for troubleshooting (config)
- Soft-delete user grace: 30 days default

# 11) Acceptance Criteria (traceable & testable)
For each major feature include testable acceptance criteria and mapping to unit/integration/test plans & user stories.

11.1 Registration
- AC-REG-1: POST /register with valid payload returns 201 and verification token persisted (.propel/context/tasks/EP-001/us_009.md, unit test: .propel/context/tasks/EP-001/us_001/unittest/test_plan_be_backend.md)
- AC-REG-2: Duplicate email returns 409 (test in us_001 plan)
- AC-REG-3: Email queued; if email provider failing, API still returns 201 and retry scheduled (us_005 email send plan)

11.2 Login
- AC-LOGIN-1: Valid credentials yield JWT + refresh (US_013, US_015)
- AC-LOGIN-2: MFA flow short-circuits token issuance until OTP verified (US_013, US_023)
- AC-LOGIN-3: Failed credentials increment counters and lock account after threshold; lockout behavior testable via US_028/US_029

11.3 Password Reset
- AC-PASS-1: Forgot password returns 202 and does not reveal account existence (US_020)
- AC-PASS-2: Reset with valid token updates password and invalidates sessions (US_021)

11.4 MFA
- AC-MFA-1: Enrollment flow issues TOTP secret and validation succeeds (US_024)
- AC-MFA-2: OTP TTL and delivery SLA (email/SMS OTP delivered within configured SLA) (US_022/US_023)

11.5 Tokens & Session Management
- AC-SES-1: Access token contains exp claim and fails after expiry; refresh endpoint rotates refresh token and issues new access (US_015, US_016)
- AC-SES-2: Logout revokes tokens immediately (US_019)

11.6 Audit & Logging
- AC-AUD-1: Security events appear in the centralized log within configured SLA; audits provide required fields (us_010)

Mapping to test artifacts:
- Unit/integration tests: .propel/context/tasks/EP-001/unittest and .propel/context/docs/unit_test_plan.md and per-story test plans referenced in tasks directories (examples: .propel/context/tasks/EP-001/us_001/unittest/test_plan_be_backend.md, .propel/context/tasks/EP-TECH/us_001/unittest/test_plan_scripts_scripts.md)

# 12) Testing & Validation
12.1 Tests to implement & map
- Unit tests for controllers, services, repositories (see per-story unit test plans in .propel/context/tasks/EP-001/*/unittest)
- Integration tests for DB persistence, migrations, token lifecycle, refresh rotation (see EP-TECH migrations story us_007)
- End-to-end smoke tests for registration/login/reset (US_009..US_021)
- Load testing: scripted scenarios for 5k and 10k concurrent session targets (US_035)
- Security testing:
  - SAST integrated into CI pipeline (EP-TECH CI stories)
  - DAST against staging, prior to prod cutover
  - Third-party penetration testing & remediation before launch (NFR-SEC-001)
- Test coverage targets:
  - Unit: 90% coverage for auth core modules; branch coverage 80% on critical flows (see unit test plans)
- Automation CI:
  - Include unit, integration and performance pipelines in CI/CD (EP-TECH US_003 / US_004)
12.2 Security validation
- Regular dependency scanning; secret leaks pre-commit & CI hooks (scripts and unit tests per EP-TECH us_001 plans).
- Pen tests and remediation tracking; no critical OWASP findings required for release.

# 13) Rollout & Migration Plan
13.1 Existing users migration
- Migrate password hashes from legacy if needed:
  - Provide migration plan to re-hash legacy bcrypt -> Argon2id on login or in background job (design.md TR-004).
- Data migration steps:
  1. Provision authentication service infra in staging (IaC templates in EP-TECH us_005).
  2. DB migration: create new tables and data pipeline to import user table (use migration tooling described in EP-TECH us_007).
  3. Validate token flows and encryption keys on staging.
13.2 Backwards compatibility
- Support legacy tokens or implement dual-validation during migration window.
- Feature flags to toggle new behaviors: token enforcement, mandatory MFA for privileged roles, etc. (centralized config in TR-009 design.md)
13.3 DB migrations
- Use versioned migrations (Flyway/Alembic); migration scripts recorded under /migrations (EP-TECH us_007).
- Ensure rollout includes rollback plans and pre-migration backups.
13.4 Phased rollout
- Canary in staging; deploy to limited production subset; monitor metrics and audit events; gradual ramp-up controlled via feature flags.

# 14) Operational Considerations
14.1 Monitoring & Alerting
- Dashboards for login success rate, error rate, token issuance, MFA failures, email send failures, refresh token error rate.
- Alerts:
  - High failed login rate (>X/min) -> security paged.
  - Email provider send failure rate exceeding threshold -> ops paged.
  - Token introspection latency >200ms 90th percentile -> SRE alert.
14.2 Runbook & incident procedures
- Runbooks stored under /docs/runbook (EP-TECH us_009); include playbooks:
  - Token-signing key compromise
  - Mass login failures (possible credential stuffing)
  - Email/SMS provider outage
  - DB failover and recovery steps
14.3 On-call responsibilities
- Security on-call for incidents related to compromise; SRE on-call for availability/performance incidents.
14.4 Operational metrics / thresholds
- Login success rate > 95% (target)
- Email OTP delivery success > 98% per configured SLA
- Queue/backlog depth for email/sms delivery < configured threshold; alert above threshold

# 15) Dependencies & Risks
15.1 External dependencies
- Email providers: AWS SES / SendGrid (us_005), SMS providers: Twilio / AWS SNS (design.md TR-005/TR-006)
- External IdPs for SSO: customer-provided OIDC providers
- Key libraries: Argon2 / bcrypt implementations, JWT libs, OAuth2 client library
- Infrastructure: PostgreSQL, Redis, Kafka (design.md)
15.2 Risks & mitigations (top)
- Brute-force attacks -> rate limiting, lockout, CAPTCHA (FR-012)
- Password breach -> strong hashing, denylist, periodic password policy updates
- Token replay/hijack -> short TTLs, HTTPS, Secure cookies, refresh rotation, revocation lists
- Email/SMS provider failures -> multi-provider strategy, retries, backoff queues
- Regulatory compliance -> configurable retention, "right to be forgotten" workflows (FR-013)

# 16) Open Questions & Decisions Needed
- Token format choice for primary access tokens: JWT (stateless) vs opaque (introspect) — recommended: JWT + JWKS; confirm decision and public key rotation policy (design.md recommends JWT but supports alternatives).
- Mandatory MFA policy for privileged roles: product/security to decide timeline and enforcement scope.
- Exact password policy defaults (min length 10 vs 12) — legal/security to sign off.
- Audit retention default and legal retention windows by region — legal to confirm for GDPR requirements.
- IdP support priorities (which providers to include at GA): Product to prioritize (Google, Microsoft, enterprise SAML support).
- Key management solution choice (HashiCorp Vault vs cloud KMS) — platform decision.

# 17) Appendix: References, Links & Glossary
17.1 Key repository files & maps (traceability)
- Master requirements & goals: .propel/context/docs/spec.md
- Architecture & design: .propel/context/docs/design.md
- Epics mapping: .propel/context/docs/epics.md
- User stories consolidation: .propel/context/docs/user_stories.md
- BRD baseline: docs/Brd.md
- Registration backend story & tests:
  - Create User Account story: .propel/context/tasks/EP-001/us_001/us_001.md
  - Registration tests & plans: .propel/context/tasks/EP-001/us_001/unittest/test_plan_be_backend.md
  - Email verification token generation & send: .propel/context/tasks/EP-001/us_004/us_004.md and .propel/context/tasks/EP-001/us_005/us_005.md
- Authentication & token service stories:
  - Login & token issuance: .propel/context/tasks/EP-002/* (see user_stories mapping US_013..US_016 in .propel/context/docs/user_stories.md)
- MFA stories: .propel/context/tasks/EP-005/* (US_022..US_025)
- Session & logout stories: EP-003 and US_017..US_019
- Security & compliance stories: EP-006 and tasks (US_026..US_032)
- Operations & infra tasks: EP-TECH tasks under .propel/context/tasks/EP-TECH/us_001..us_009.md (scaffold, CI/CD, migrations)
- Unit test plans and scripts: .propel/context/docs/unit_test_plan.md and .propel/context/tasks/**/unittest/*

17.2 Diagrams / Sequence diagrams
- PlantUML diagrams: included in .propel/context/docs/design.md component & sequence diagrams (search container diagrams section)
- Sequence diagrams for core flows available in .propel/context/docs/spec.md use-case PlantUML blocks

17.3 Glossary (selected)
- JWT: JSON Web Token signed token carrying claims
- OIDC: OpenID Connect
- MFA: Multi-Factor Authentication
- TOTP: Time-based One-Time Password (RFC 6238)
- IdP: Identity Provider
- KMS: Key Management Service
- EP / US: Epic / User Story IDs referenced in .propel/context/docs/epics.md and .propel/context/docs/user_stories.md

17.4 Next actions (short)
- Product/security: confirm open questions (MFA enforcement, password policy, token format) and sign off on defaults.
- Platform: prepare IaC and staging environments per EP-TECH tasks and migrate DB schema per us_007 migration plan.
- Dev: produce OpenAPI spec for the API surface and implement auth microservice scaffold (EP-TECH us_006).
- Security/QA: schedule SAST/DAST and third-party pen test prior to production launch (timeline in EP-006).

End of document.