# Unit Test Plan - [TASK_us_001_BE_user_registration]

## Requirement Reference
- **User Story**: [us_001]
- **Story Location**: [.propel/context/tasks/us_001/us_001.md]
- **Layer**: BE
- **Related Test Plans**:
  - .propel/context/tasks/us_001/unittest/test_plan_db_user_persistence.md (DB)
  - .propel/context/tasks/us_001/unittest/test_plan_infra_email_enqueue.md (Infra)
- **Acceptance Criteria Covered**:
  - AC-1: Valid registration request → 201 Created; response JSON contains userId (uuid), masked email, accountStatus "PENDING_VERIFICATION", createdAt, message; DB user row with status PENDING_VERIFICATION; password stored salted+hashed; verification token persisted with expires_at = issued_at + 24h.
  - AC-2: Duplicate email → 409 Conflict, body { code: "EMAIL_ALREADY_EXISTS", message }, no new user or token.
  - AC-3: Invalid input (email/password) → 400 Bad Request with field-level validation errors, no user or token.
  - AC-4: Email provider unavailable after persistence → API returns 201, token persisted, email-send job queued for retry/logged, ERROR log with correlation id, metric "verification_email_send_failure" emitted.
  - AC-5: Performance/security: password stored hashed+salted; no plaintext logged or stored; processing path non-blocking (unit-level smoke checks toward perf target).

## Test Plan Overview
This BE-layer unit test plan covers the user registration API orchestration and business logic: controller input validation mapping, service orchestration (validation → hashing → persistence → token generation → enqueue), transaction handling, and observability calls (logs/metrics). Scope excludes DB integration and external email provider behavior (those are covered in separate DB and Infra plans). Tests are unit-level, isolated by DI and mocks, deterministic (time/UUID injected). Purpose: verify functional correctness, error handling, security assertions (no plaintext), and key edge conditions described in acceptance criteria.

Technology detection & recommended frameworks
- No repository provided. Common backend choices and recommendations:
  - Primary recommendation: Node.js + TypeScript (Express/Nest) — use Jest (v29+) with ts-jest or native ts support. Mocking: jest.fn(), jest.mock(), jest.spyOn.
  - Alternative: Python (FastAPI) — use pytest + pytest-mock / unittest.mock.
  - Java (Spring Boot) — use JUnit 5 + Mockito.
- This plan uses Node/TypeScript nomenclature in file paths and examples. If the project uses a different stack, adopt the corresponding framework and equivalent mocking patterns described.

## Dependent Tasks
- DB schema migrations and user/verification token tables must exist for integration (DB plan).
- Email enqueue integration adapters must exist (Infra plan).
- Secret management and configured hash algorithm (Argon2/Bcrypt) must be available/injected.

## Components Under Test

| Component | Type | File Path | Responsibilities |
|-----------|------|-----------|------------------|
| usersController.register | controller (function) | src/controllers/usersController.ts | Parse request, call registration service, map domain errors to HTTP responses, attach correlation id. |
| userRegistrationService | service | src/services/userRegistrationService.ts | Validate input, hash password, create user & token within transaction, enqueue verification email (async), emit metrics, handle errors/rollback. |
| validation | module | src/lib/validation.ts | Email RFC-5322 check, password policy (min, complexity, max length). |
| hashService | service/adapter | src/services/hashService.ts | Hashing wrapper (bcrypt/argon2) — returns salted+hashed value. |
| userRepository | repository | src/repositories/userRepository.ts | Insert user row; enforce normalized email and unique constraint handling. |
| verificationTokenRepository | repository | src/repositories/verificationTokenRepository.ts | Create verification token linked to user with expires_at. |
| transactionManager | infra adapter | src/lib/transactionManager.ts | begin/commit/rollback transaction abstraction. |
| emailQueue | infra adapter | src/queues/emailQueue.ts | enqueueVerification(payload) — returns immediately (async). |
| clock | util | src/lib/clock.ts | Injected time provider for deterministic now(). |
| logger | util | src/lib/logger.ts | Log info/error with correlation id; must not log plaintext passwords. |
| metricEmitter | util | src/lib/metricEmitter.ts | Emit registration_success, registration_duplicate_email, verification_email_send_failure metrics. |

## Test Cases

| Test-ID | Type | Description | Given | When | Then | Assertions |
|---------|------|-------------|-------|------|------|------------|
| TC-BE-01 | positive | Happy path registration | Valid email+password meeting policy; userRepository reports no existing email | Controller receives POST to /api/v1/users/register | Service persists user & token, enqueues email async, controller returns 201 | Response contains userId (uuid format), maskedEmail, accountStatus "PENDING_VERIFICATION", createdAt (ISO-8601), message; userRepository.createUser called with hashed password; verificationTokenRepository.createToken called with expires_at = issued_at + 24h; emailQueue.enqueueVerification called with expected payload (userId, token) |
| TC-BE-02 | negative | Duplicate email results in 409 | userRepository.findByEmail returns existing user | Controller receives same email registration | Service maps repository duplicate to domain error | Controller returns 409 and body { code: "EMAIL_ALREADY_EXISTS", message }; verify repository.createUser not called; metricEmitter.emit called with registration_duplicate_email |
| TC-BE-03 | negative | Input validation failures (email, password) | Malformed email or password failing complexity/min | Controller receives payload | Validation short-circuits service | Controller returns 400 with field-level errors (email/password messages); no repository or token calls; no enqueue |
| TC-BE-04 | edge_case | Password too long -> 400 PASSWORD_TOO_LONG | Password length > MAX_PASSWORD_LEN (e.g., 128) | Controller receives payload | Validation returns PASSWORD_TOO_LONG | Controller returns 400 with { password: "PASSWORD_TOO_LONG" }; no hashing or persistence attempted |
| TC-BE-05 | positive/security | Hashing called; no plaintext persisted or logged | Valid input; hashService mocked to return deterministic hash | Registration flow | userRepository receives stored record with password_hash field (no raw password); logger.info/error not called with plaintext | Assert hashService.hash called once with raw password; repository.createUser called with hashedPassword; logger captured messages do not contain raw password substring |
| TC-BE-06 | edge_case/error | Transaction rollback on persistence failure | userRepository.createUser throws after transaction begun | Registration invoked | transactionManager.rollback called; controller returns 500 with correlation id; emailQueue not called | Assert transactionManager.rollback invoked; logger.error logged with correlation id; no verificationTokenRepository.createToken or emailQueue.enqueueVerification calls |
| TC-BE-07 | negative/async | Email enqueue failure after persistence → API still 201 and metrics/logging emitted | Repositories succeed; emailQueue.enqueueVerification throws transient error (simulated) | Registration invoked | Service persists user and token, attempts enqueue which fails; API returns 201 | Assert response 201; verification token persisted; metricEmitter.emit called with verification_email_send_failure; logger.error called with correlation id; service schedules a retry/enqueues to retry queue (or calls retry API) — verify retry call invoked (mocked) |

Notes on test independence: each test must inject fresh mocks/stubs and a fresh deterministic clock and UUID provider. Tests must not share state.

## Expected Changes

| Action | File Path | Description |
|--------|-----------|-------------|
| CREATE | tests/unit/controllers/usersController.spec.ts | Unit tests for controller mapping and input handling |
| CREATE | tests/unit/services/userRegistrationService.spec.ts | Unit tests for service orchestration and error handling |
| CREATE | tests/unit/mocks/userRepository.mock.ts | Mock/stub for userRepository |
| CREATE | tests/unit/mocks/verificationTokenRepository.mock.ts | Mock/stub for verificationTokenRepository |
| CREATE | tests/unit/mocks/hashService.mock.ts | Mock for hashService |
| CREATE | tests/unit/mocks/emailQueue.mock.ts | Mock for emailQueue |
| CREATE | tests/unit/fixtures/testData.ts | Deterministic fixtures (UUIDs, tokens, timestamps) |

## Mocking Strategy

| Dependency | Mock Type | Mock Behavior | Return Value / Notes |
|------------|-----------|---------------|----------------------|
| userRepository | mock/stub | findByEmail returns null or existing user; createUser returns created user object or throws unique constraint error | For happy path return { id: uuid, email, password_hash, status, created_at } |
| verificationTokenRepository | mock | createToken returns token object with issued_at and expires_at based on injected clock | Use deterministic token string |
| hashService | spy/mock | hash(password) returns deterministic hashed string; can be configured to throw for too-long passwords | Example return "hashed::deterministic" |
| transactionManager | spy/mock | begin returns tx context; commit/rollback are tracked | Assert commit on success, rollback on failure |
| emailQueue | mock (async) | enqueueVerification returns resolved Promise for success, rejected Promise for transient failure | For transient failure, ensure service handles by scheduling retry |
| clock | fake | now() returns fixed ISO timestamp for deterministic expires_at | e.g., 2026-04-08T00:00:00Z |
| uuidProvider | fake | generate() returns deterministic UUID | e.g., "11111111-1111-1111-1111-111111111111" |
| logger | spy | capture info/error calls for assertions; must be checked for no plaintext | Provide redaction expectation |
| metricEmitter | spy | track emitted metrics with tags | Verify expected metrics and tags emitted |

Mock patterns and strategy reasoning:
- Use dependency injection to inject mocks into service/controller.
- Use spies for side-effects (logger, metricEmitter) rather than stubs that swallow calls.
- For Node/TS: prefer Jest with jest.fn() mocks and jest.spyOn. For Python: use pytest fixtures and unittest.mock.Mock/AsyncMock.
- Keep mocks deterministic and reset between tests (beforeEach/teardown).

## Test Data

| Scenario | Input Data | Expected Output |
|----------|------------|-----------------|
| Valid case | { email: "user@example.com", password: "P@ssw0rd1!" } | 201 with masked email "u***@example.com", userId uuid, status PENDING_VERIFICATION |
| Duplicate email | { email: "existing@example.com", password: "P@ssw0rd1!" } | 409 { code: "EMAIL_ALREADY_EXISTS" } |
| Invalid email | { email: "bad-email", password: "P@ssw0rd1!" } | 400 { email: "Invalid email format" } |
| Weak password | { email: "user@example.com", password: "short" } | 400 { password: "Password does not meet complexity requirements" } |
| Too long password | { email: "user@example.com", password: "A".repeat(129) } | 400 { password: "PASSWORD_TOO_LONG" } |
| Token expiry check | deterministic clock now = 2026-04-08T00:00:00Z | token.expires_at == 2026-04-09T00:00:00Z |

Fixtures must include deterministic UUIDs, token strings, and fixed timestamps. Do not include real secrets.

## Test Commands
- Run all BE unit tests (Node.js / Jest example):
  - `npm test -- tests/unit/services/userRegistrationService.spec.ts tests/unit/controllers/usersController.spec.ts`
- Run all unit tests with coverage:
  - `npm run test:coverage -- --collectCoverageFrom='src/services/**,src/controllers/**'`
- Run single test by name (Jest):
  - `npm test -- -t "TC-BE-01 Happy path registration"`
- If project is Python/pytest:
  - `pytest tests/unit/services/test_user_registration_service.py::test_happy_path -q`
Adjust commands to project test runner configuration.

## Coverage Target
- **Line Coverage**: 90% for userRegistrationService and usersController modules.
- **Branch Coverage**: 80% for business logic branches (validation failures, duplicate email, enqueue failure, transaction rollback).
- **Critical Paths** (must have 100% coverage or explicit unit tests):
  - Validation logic branches (email invalid, password weak, password too long)
  - Transaction success/rollback paths
  - HashService invocation path (ensure called)
  - Token expiry calculation
  - Mapping of repository duplicate errors → 409 response

## Documentation References
- Jest docs: https://jestjs.io
- Node best practices: https://nodejs.org/en/docs/
- OWASP Secure Coding: https://owasp.org/
- Project test patterns: refer to existing tests under tests/unit/ (if present)
- Hashing library docs (bcrypt/argon2) per configured adapter

## Implementation Checklist
- [ ] Create unit test files for controller and service per Expected Changes
- [ ] Add deterministic fixtures (UUIDs, tokens, clock) and test data
- [ ] Implement mocks/stubs for repositories, hashService, emailQueue, transactionManager, logger, metricEmitter
- [ ] Implement positive test: happy path registration (TC-BE-01)
- [ ] Implement negative tests: duplicate email (TC-BE-02) and validation failures (TC-BE-03/TC-BE-04)
- [ ] Implement error/edge tests: transaction rollback (TC-BE-06) and email enqueue failure handling (TC-BE-07)
- [ ] Run suite and verify coverage targets; iterate until targets met

Additional layer-specific plans required: DB persistence tests and Infra email enqueue tests are required and must be implemented separately (.propel/context/tasks/us_001/unittest/test_plan_db_user_persistence.md and test_plan_infra_email_enqueue.md) to cover persistence constraints, unique constraint behavior, and external-provider retry semantics.

Previous Analysis and Reasoning:
- This BE test plan was decomposed from the full story because the complete set of required tests exceeded the 8-item checklist constraint. DB and Infra layer plans are referenced and must be created to satisfy full acceptance coverage.

Notes on test independence and anti-patterns:
- Each unit test must be isolated: reset mocks and DI container state between tests (beforeEach). Avoid shared mutable singletons for request-scoped data.
- Do not mock internal implementation details; mock external contracts (repositories, queues, hash adapter).
- Avoid copying fixtures across tests; centralize fixtures in tests/unit/fixtures/testData.ts to follow DRY guidelines.

(End of BE unit test plan)