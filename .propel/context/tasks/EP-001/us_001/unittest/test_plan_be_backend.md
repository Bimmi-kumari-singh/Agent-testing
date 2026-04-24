# Unit Test Plan - [US_001_BE_Registration]

## Requirement Reference
- **User Story**: [us_001]
- **Story Location**: [.propel/context/tasks/us_001/us_001.md]
- **Layer**: BE
- **Related Test Plans**: test_plan_db_registration.md (DB), test_plan_infra_registration.md (Infra)
- **Acceptance Criteria Covered**:
  - [AC-1: Valid registration -> 201 Created, masked email, accountStatus=PENDING_VERIFICATION, createdAt ISO, verification email queued; user row persisted with password salted+hashed; verification token persisted with expiry = created_at + 24h]
  - [AC-2: Duplicate email -> 409 Conflict { code: "EMAIL_ALREADY_EXISTS" }, no user/token created]
  - [AC-3: Invalid input -> 400 Bad Request with field-level validation errors, no user/token created]
  - [AC-4: Email provider transient failure after persistence -> API still returns 201, token persisted, email-send job queued/retried, ERROR log with correlation id emitted, metric verification_email_send_failure emitted]
  - [AC-5: Security (password hashed with configured algorithm, no plaintext logged) and performance NFR (server processing time target <= 500ms at 95th percentile) — unit tests verify hashing usage and no plaintext logging; full perf validated by separate perf tests]

## Test Plan Overview
This BE unit test plan covers controller and service-layer unit tests for the registration flow (POST /api/v1/users/register). Scope includes validation, orchestration (uniqueness check, hashing, user + token persistence), token creation parameters (expiry), interactions with the email-enqueue adapter, and mapping domain errors to HTTP responses. Tests are pure unit tests: all external dependencies (DB repositories, email provider, metrics, logger, correlation id provider) must be mocked to ensure test independence and determinism.

Technology stack recommendation (detected / advised):
- Primary recommendation: Node.js + TypeScript backend.
  - Unit test frameworks: Jest (v29+) or Vitest
  - HTTP route/integration helper: supertest (for lightweight controller handler testing if needed)
  - Mocking: jest.fn(), jest.mock(); sinon optional
  - Rationale: common industry stack for REST APIs, strong TypeScript typing improves testability and DI-friendly services.
- Alternative (if project is Python): FastAPI + pytest + pytest-mock + httpx TestClient.

Why these frameworks: standard, widely used, DI-friendly, good mocking support; tests stay small and fast (unit-level).

Test independence: each test must mock external adapters and avoid shared state; use beforeEach to reset mocks and inject fresh stubs.

## Dependent Tasks
- test_plan_db_registration.md (DB layer) — repository behavior and transactions must be unit-tested to complement BE tests.
- test_plan_infra_registration.md (Infra adapters) — password-hasher and email-retry logic validated at infra layer.
- Ensure configuration for hashing algorithm and max password length is available (env/config mocks).

## Components Under Test

| Component | Type | File Path | Responsibilities |
|-----------|------|-----------|------------------|
| RegistrationController | controller/route handler | src/controllers/registration.controller.ts | Accepts POST /api/v1/users/register, delegates to RegistrationService, maps service errors to HTTP responses, returns JSON body + status |
| RegistrationService | service/orchestrator | src/services/registration.service.ts | Orchestrates validation, uniqueness check, password hashing, user + token persistence, enqueues verification email, emits metrics |
| validateRegistrationPayload | function/validator | src/lib/validation.ts | Validates RFC-5322 email, password policy (min, complexity, max length) and returns field-level errors |
| PasswordHasher | adapter/service | src/lib/password-hasher.ts | Wraps hashing library (bcrypt/argon2) with configured params; returns salted+hashed password |
| TokenService | service | src/services/token.service.ts | Generates verification token (random secure string), calculates expires_at = now + 24h |
| UserRepository | repository (interface) | src/repositories/user.repository.ts | create/findByEmail; persisted user rows |
| VerificationTokenRepository | repository (interface) | src/repositories/token.repository.ts | create token linked to user |
| EmailEnqueuer | queue adapter | src/queues/email-enqueuer.ts | Enqueue verification email job; surface transient failures to service via exceptions or return status |
| MetricsEmitter | telemetry adapter | src/lib/metrics.ts | Emit registration_success, registration_duplicate_email, verification_email_send_failure |
| Logger | logging adapter | src/lib/logger.ts | Structured logging; must not log plaintext passwords |
| CorrelationIdProvider | helper | src/lib/correlation.ts | Provide request correlation id for logs/metrics |

## Test Cases

| Test-ID | Type | Description | Given | When | Then | Assertions |
|---------|------|-------------|-------|------|------|------------|
| TC-BE-001 | positive | Happy path registration | Valid email/password meeting policy; UserRepository.findByEmail returns null | Call controller/register endpoint | Returns 201 Created and service persisted user+token and enqueued email | Response: status 201; body contains userId (uuid regex), maskedEmail (e.g., j***@domain.com), accountStatus = "PENDING_VERIFICATION", createdAt ISO8601, message contains "Verification email queued"; Asserts: UserRepository.create called with email and hashed password; TokenService.create called with expiry ≈ now+24h; EmailEnqueuer.enqueue called once with correct payload; MetricsEmitter.registration_success emitted |
| TC-BE-002 | negative | Duplicate email -> 409 | UserRepository.findByEmail returns existing user | Call controller/register | Returns 409 Conflict | Response: status 409; body { code: "EMAIL_ALREADY_EXISTS", message: "An account with this email already exists." }; Asserts: No new UserRepository.create or TokenRepository.create calls; EmailEnqueuer not called; MetricsEmitter.registration_duplicate_email emitted |
| TC-BE-003 | negative (validation) | Invalid inputs -> 400 with field errors | Invalid email formats; password too short/weak; password over max length | Call controller/register | Returns 400 Bad Request | Response: status 400; body contains field-level errors e.g., { email: "Invalid email format" } or { password: "PASSWORD_TOO_LONG" }; Asserts: No repo creates; no enqueues |
| TC-BE-004 | positive | Password hashing integration (no plaintext persisted) | Valid registration; PasswordHasher mocked | Call registration service | Persists only hashed password | Assert PasswordHasher.hash called with raw password and configured params; UserRepository.create receives password_hash only; Logger.info/Logger.error spy verifies plaintext password not logged |
| TC-BE-005 | edge_case | Extremely long password beyond allowed -> 400 PASSWORD_TOO_LONG | Password length > configured MAX_PASSWORD_LENGTH (e.g., 128) | Call controller/register | Returns 400 | Response body indicates { password: "PASSWORD_TOO_LONG" }; Asserts: No hashing attempted; no persistence |
| TC-BE-006 | error | Email provider transient failure after persistence -> API returns 201, token persisted, retry queued & metrics/logs | UserRepository.create and TokenRepository.create succeed; EmailEnqueuer.enqueue throws transient error or returns failure code | Call controller/register | Returns 201 Created | Asserts: User and token persisted (UserRepository.create & TokenRepository.create called); EmailEnqueuer.enqueue attempted and error handled: service enqueues retry job or records retry request; Logger.error called with correlation id; MetricsEmitter.verification_email_send_failure emitted; Response still 201 with expected body |
| TC-BE-007 | edge / non-functional check | Lightweight processing-time micro-check for hashing call (unit-level) | Valid request; PasswordHasher.hash simulated to take deterministic small time | Call registration service | Completes within a conservative unit-test threshold (e.g., 200ms) | Assert that registration service completes within threshold in unit test bounds (note: this is a guard, full perf causal verification requires integration/perf tests). Also assert Token expiry correctness (expires_at ≈ now + 24h within allowed delta) |

Notes:
- Every AC maps: AC-1->TC-BE-001, AC-2->TC-BE-002, AC-3->TC-BE-003/TC-BE-005, AC-4->TC-BE-006, AC-5->TC-BE-004 & TC-BE-007.
- Tests must be isolated: each test uses fresh mocks and avoids shared in-memory DB state.

## AI Component Test Cases [CONDITIONAL: If AIR-XXX in scope]
Skip — story has no AI/LLM components.

## Expected Changes

| Action | File Path | Description |
|--------|-----------|-------------|
| CREATE | tests/unit/registration.controller.spec.ts | Unit tests for RegistrationController (happy, validation, duplicate, error mapping) |
| CREATE | tests/unit/registration.service.spec.ts | Unit tests for RegistrationService orchestration and error-handling |
| CREATE | tests/mocks/user.repository.mock.ts | Mock/stub for UserRepository methods (findByEmail, create) |
| CREATE | tests/mocks/token.repository.mock.ts | Mock for TokenRepository.create |
| CREATE | tests/mocks/email.enqueuer.mock.ts | Mock for EmailEnqueuer with success and transient-failure behaviors |
| CREATE | tests/mocks/password-hasher.mock.ts | Mock for PasswordHasher.hash |
| MODIFY | jest.config.ts | Ensure test match patterns include tests/unit and coverage config |

## Mocking Strategy

| Dependency | Mock Type | Mock Behavior | Return Value / Behavior |
|------------|-----------|---------------|-------------------------|
| UserRepository | stub/mock | findByEmail: configurable to return null or existing user; create: resolves to created user object | findByEmail -> null / existingUser; create -> { id, email, password_hash, status, created_at } |
| VerificationTokenRepository | mock | create: returns token object with token string and expires_at | { token: "xxxx", user_id, expires_at } |
| PasswordHasher | spy/mock | hash: returns deterministic hashed string; optionally simulate latency or error | "hashed::<salt>::<digest>" |
| EmailEnqueuer | stub/mock | enqueue: success path resolves; transient failure throws specific TransientEmailError or rejects | success: resolve(true); failure: reject(new TransientEmailError("SMTP 421")) |
| MetricsEmitter | spy | track calls to emit specific metric names | record calls for assertions |
| Logger | spy | spies on info/error calls; captures messages and meta | capture messages but do not output during tests |
| CorrelationIdProvider | stub | returns deterministic correlation id per test | "test-cid-1234" |

Mocking approach:
- Use dependency injection where possible. For controllers instantiate with mock service; for service instantiate with mock repositories/adapters.
- Avoid mocking internal implementation details; mock external contracts (public methods).
- Reset mocks in beforeEach.

## AI Mocking Strategy [CONDITIONAL: If AIR-XXX in scope]
Skipped.

## Test Data

| Scenario | Input Data | Expected Output |
|----------|------------|-----------------|
| Valid case | { email: "jane.doe@example.com", password: "P@ssw0rd!1" } | 201, body with userId UUID, masked email "j***@example.com", accountStatus PENDING_VERIFICATION |
| Duplicate email | { email: "existing@example.com", password: "P@ssw0rd!1" } | 409, { code: "EMAIL_ALREADY_EXISTS" } |
| Invalid email | { email: "bad-email", password: "P@ssw0rd!1" } | 400, { email: "Invalid email format" } |
| Weak password | { email: "jane@example.com", password: "1234" } | 400, { password: "Password does not meet complexity requirements" } |
| Too long password | { email: "jane@example.com", password: "<129 chars>" } | 400, { password: "PASSWORD_TOO_LONG" } |
| Email provider transient failure | valid input; EmailEnqueuer.rejects | 201; user/token persisted; Metric verification_email_send_failure emitted; Logger.error called with correlation id |

## Test Commands
- Run all unit tests (Jest): npm test
- Run BE unit tests only: npm test -- --testPathPattern=tests/unit/registration
- Run with coverage: npm run test:coverage
  - Example (package.json): "test:coverage": "jest --coverage"
- Run single test by name: npm test -- -t "TC-BE-001" (or a descriptive test name)
- Watch mode during dev: npm run test:watch

(Adjust commands if project uses yarn/pnpm or alternative test runner; replace with pytest commands if Python stack.)

## Coverage Target
- **Line Coverage**: 90%
- **Branch Coverage**: 80%
- **Critical Paths (100% required)**:
  - Password hashing call and that only hashed password is persisted
  - Uniqueness check branch leading to 409 and prevention of persistence
  - Token generation with correct expires_at calculation
  - Email enqueue transient-failure handling branch (ensuring API still returns 201 and metrics/logs emitted)
- Rationale: high coverage on security and branching that affects data persistence and external interactions; overall targets align with language-agnostic standards.

## Documentation References
- **Framework Docs**:
  - Jest: https://jestjs.io/docs/getting-started
  - supertest (controller handler tests): https://github.com/visionmedia/supertest
- **Project Test Patterns**: (link to project examples) — replace with internal links to existing tests in repository
- **Mocking Guide**: Jest mocking docs: https://jestjs.io/docs/mock-functions

## Implementation Checklist
- [ ] Create unit test files: tests/unit/registration.controller.spec.ts and tests/unit/registration.service.spec.ts
- [ ] Implement mocks/stubs for UserRepository, VerificationTokenRepository, PasswordHasher, EmailEnqueuer, MetricsEmitter, Logger, CorrelationIdProvider
- [ ] Implement positive tests for happy path (TC-BE-001) asserting response body and correct calls to repositories/adapters
- [ ] Implement negative/validation tests (TC-BE-002, TC-BE-003, TC-BE-005) ensuring no persistence on invalid input
- [ ] Implement error-handling test for email provider transient failure ensuring 201 response, persistence, and metrics/logging (TC-BE-006)
- [ ] Implement hashing-only tests verifying PasswordHasher used and no plaintext logging (TC-BE-004)
- [ ] Run tests and enforce coverage targets; add test coverage config and gating in CI if not present
- [ ] Additional layer-specific plans required: implement and run test_plan_db_registration.md and test_plan_infra_registration.md to cover repository transactions and infra adapters

Previous Analysis and Reasoning:
- Decomposition required because full scope covers DB and infra responsibilities; this file focuses on the BE/service/controller layer only. DB and Infra layer plans must be implemented separately to keep each checklist ≤ 8 items and ensure clarity of responsibilities.

Please proceed to implement these unit tests in the indicated test files. If the project uses a non-Node stack, I can regenerate this plan tailored to that stack (pytest/pytest-mock) — indicate the preferred stack and I will adapt test commands and mock details.