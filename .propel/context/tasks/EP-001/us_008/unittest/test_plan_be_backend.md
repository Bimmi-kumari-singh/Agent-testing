test_plan_be_registration.md

# Unit Test Plan - [TASK_us_008_BE]

## Requirement Reference
- **User Story**: [us_008] Handle Duplicate Registrations and Race Conditions
- **Story Location**: [.propel/context/tasks/us_008/us_008.md]
- **Layer**: BE
- **Related Test Plans**: test_plan_db_unique_constraints.md (DB), test_plan_infra_email_and_retry.md (Infra), test_plan_scripts_migrations.md (Scripts)
- **Acceptance Criteria Covered**:
  - AC-1: Given no existing account -> POST /api/v1/register returns 201 Created, persists single user row (canonical email), stores a single verification token with expiry, password hashed, exactly one verification email enqueued, audit log emitted.
  - AC-2: Given existing account -> re-submit returns 200 OK, no new user row, returns existing user_id and account_status, no new token/email.
  - AC-3: Concurrent requests for same canonical email -> exactly one user row/token/email; other requests return idempotent 200 OK after dedupe handling.
  - AC-4: DB unique constraint violation handling -> application looks up by canonical email, returns 200 OK with existing user, no partial duplicates, transaction rollback/compensation.
  - AC-5: Email provider temporarily unavailable -> registration still returns 201 Created, user & token persisted, failed send recorded and retry scheduled, metrics/alerts emitted.

## Test Plan Overview
Purpose: Unit tests for the backend registration flow (controller + service orchestration) to verify idempotency, canonicalization, error handling for unique-constraint races, password hashing, token creation, and post-commit enqueue behavior. Scope: pure unit tests that exercise RegistrationController and RegistrationService with all external dependencies mocked (repositories, transaction manager, mail enqueue client, metrics/audit). Integration-level DB/queue tests are out of scope and covered in DB/Infra plans.

Why: Unit tests validate the business rules and orchestration deterministically and fast; mocking external systems ensures isolation and reproducibility.

## Dependent Tasks
- US_DB_001: DB unique email index & migration (must exist or be mocked by repository behavior)
- US_EMAIL_003: Email delivery & retry integration (email enqueue interface shape needed)
- US_AUTH_004: Password hashing & normalization utilities (or their interfaces)

## Components Under Test

| Component | Type | File Path | Responsibilities |
|-----------|------|-----------|------------------|
| RegistrationController | controller/handler | src/controllers/registrationController.ts | HTTP input validation, payload canonicalization, delegates to RegistrationService, maps service result to HTTP status/headers/body |
| RegistrationService | service/orchestrator | src/services/registrationService.ts | Normalize email, validate policy, begin transaction, call UserRepository.createUser, generate token, persist token, schedule post-commit email enqueue, emit metrics/audit |
| UserRepository (mocked) | repository interface | src/repositories/userRepository.ts | Insert/select user by canonical_email, fetch user by id/status; in unit tests this will be mocked to simulate success, pre-existing record, and unique-constraint error |
| TokenService (mocked) | service | src/services/tokenService.ts | Generate verification token, compute expiry, persist token record (interface mocked) |
| PasswordHasher (mocked) | util | src/utils/passwordHasher.ts | Hash and verify password; ensure stored hash != plaintext |
| TransactionManager (mocked) | infra/utility | src/infra/transactionManager.ts | Begin/commit/rollback and post-commit hooks (email enqueue must run in post-commit) |
| EmailQueueClient (mocked) | infra/client | src/infra/emailQueueClient.ts | Enqueue verification email job; should be invoked only after commit and deduped |
| Metrics/AuditLogger (mocked) | infra/monitoring | src/monitoring/metrics.ts, src/logging/auditLogger.ts | Emit counters and audit events for success, duplicate attempts, unique constraints, email failures |

## Test Cases

| Test-ID | Type | Description | Given | When | Then | Assertions |
|---------|------|-------------|-------|------|------|------------|
| TC-001 | positive | Happy path: new registration results in 201 and side effects | UserRepository.createUser resolves with created user; Transaction commit succeeds; EmailQueueClient.enqueue succeeds | POST /register with valid payload (email not present) | 201 Created returned | Response contains user_id, account_status = PendingVerification, Location header; UserRepository.createUser called once with canonical email; TokenService.createToken called once; PasswordHasher.hash called; EmailQueueClient.enqueue called in post-commit hook once; Metrics/Audit logged for create |
| TC-002 | negative | Re-submit same registration returns idempotent 200 and no side effects | UserRepository.createUser throws UniqueConstraintError OR UserRepository.findByCanonicalEmail returns existing user before create | POST /register with same canonical email | 200 OK returned | Response contains existing user_id and account_status; No new token created (TokenService.createToken not called); EmailQueueClient.enqueue not called; DB insert not persisted again (createUser either not called or handled via caught error + lookup) |
| TC-003 | edge_case | Email canonicalization (case/whitespace) maps to same account | Payload email = " ALIce@Example.COM " ; repository has user with canonical "alice@example.com" | POST /register | 200 OK returned | RegistrationService normalizes input (trim + lowercase) before any repo call; UserRepository.findByCanonicalEmail called with "alice@example.com"; No new user created |
| TC-004 | error | DB unique-constraint race: create throws UniqueConstraintError -> service performs lookup and returns 200 | Simulate concurrent insert: first call would have created row elsewhere; createUser throws UniqueConstraintError | RegistrationService handles createUser UniqueConstraintError | 200 OK returned after service runs lookup | Service calls UserRepository.findByCanonicalEmail and returns existing user_id/status; TransactionManager ensures rollback/clean state; TokenService.createToken not called; Audit log emitted for unique-constraint resolution |
| TC-005 | positive | Post-commit ordering: email enqueue invoked only after commit | TransactionManager supports postCommit callback; EmailQueueClient.enqueue should only run in that callback | createUser succeeds but simulate failure before commit or simulate rollback | On rollback, EmailQueueClient.enqueue is not invoked | Assert EmailQueueClient.enqueue called only when TransactionManager.commit invoked and in post-commit hook; not called on rollback |
| TC-006 | error | Email send/enqueue fails transiently after commit -> registration returns 201 and failure recorded & retry scheduled | createUser & commit succeed; EmailQueueClient.enqueue throws transient error (e.g., network error) | RegistrationService attempts enqueue in post-commit | 201 Created returned | User persisted and token persisted; Email failure recorded via Metrics/AuditLogger with retry-schedule metadata; Service does not rollback user creation; retry scheduling method invoked (e.g., enqueue with retry/backoff parameters) |
| TC-007 | positive | Password hashing & verification | Valid password input | RegistrationService hashes password before passing to repository | Stored password not equal plaintext; PasswordHasher.verify returns true for plaintext | Assert PasswordHasher.hash called; UserRepository.createUser receives hashed password; PasswordHasher.verify returns true for stored hash |
| TC-008 | concurrency_sim (unit-level) | Simulate two concurrent registration attempts: first returns created, second throws UniqueConstraintError -> second returns 200 | First mocked createUser resolves and returns created user for call A; concurrent call B mocked to throw UniqueConstraintError | Two RegistrationService.register calls invoked sequentially in test simulating concurrency | Exactly one token created and one enqueue attempted | Assert one createUser resulted in commit path with TokenService.createToken and EmailQueueClient.enqueue; second path handled unique constraint and returned existing user without creating token/email; metrics logged for both paths |

Notes:
- Each test must be independent: reset mocks and fake timers between tests.
- Concurrency tests are simulated by mocking repository behaviors to avoid flaky multi-threaded execution.

## AI Component Test Cases [CONDITIONAL: If AIR-XXX in scope]
Skip — no AI/LLM behavior detected in this user story's backend responsibilities.

## Expected Changes

| Action | File Path | Description |
|--------|-----------|-------------|
| CREATE | tests/unit/controllers/registrationController.spec.ts | Unit tests for controller behavior and HTTP mappings |
| CREATE | tests/unit/services/registrationService.spec.ts | Unit tests for core orchestration and idempotency logic |
| CREATE | tests/mocks/userRepository.mock.ts | Mock/stub for UserRepository behaviors (success, unique constraint, find) |
| CREATE | tests/mocks/emailQueueClient.mock.ts | Mock for email enqueue client (success, transient error) |
| CREATE | tests/mocks/transactionManager.mock.ts | Mock for transaction lifecycle and post-commit hooks |
| UPDATE | package.json | Add jest (if not present) and test script entry (if required) |

## Mocking Strategy

| Dependency | Mock Type | Mock Behavior | Return Value / Throws |
|------------|-----------|---------------|-----------------------|
| UserRepository | stub/mock | - createUser resolves with created user or throws UniqueConstraintError depending on scenario. - findByCanonicalEmail resolves with existing user when needed. | createdUser object OR throw new UniqueConstraintError; existingUser object |
| TransactionManager | fake | Implements begin/commit/rollback and supports registering postCommit callbacks; test controls commit vs rollback | commit(): invokes registered post-commit callbacks; rollback(): ensures callbacks not invoked |
| TokenService | mock | createToken called only on successful create path; returns token object | { token: "abc", expiresAt: <Date> } |
| EmailQueueClient | spy/mock | enqueue called in post-commit; can be configured to succeed or throw transient error; verifies payload contains user_id & token | resolves or throws (e.g., new NetworkError()) |
| PasswordHasher | stub | hash returns deterministic hash for given password (use test vector) ; verify returns true | hashed string; verify(true) |
| Metrics | spy | increment/record called with tags: success/create/duplicate/email_failure | no-op but track calls |
| AuditLogger | spy | record event calls with expected messages | no-op but assertions on called arguments |
| TimeProvider | fake | Provides control over "now" for token expiry assertions | deterministic Date values |

Mocking Guidance (why): Use dependency-injection-friendly interfaces so unit tests assert orchestration without hitting DB or network. Use spies for metrics/audit to assert observability side effects.

## Test Data

| Scenario | Input Data | Expected Output |
|----------|------------|-----------------|
| New valid registration | { email: " Alice@example.com ", password: "P@ssw0rd!" } | 201 Created; persisted email "alice@example.com"; token with expiry; one email enqueue |
| Re-submit same email | { email: "alice@example.com", password: "P@ssw0rd!" } | 200 OK; existing user_id; no new token; no enqueue |
| Concurrent race (simulated) | Two calls with same canonical email | One 201, one 200; single persisted user & token & enqueue |
| Email enqueue transient failure | As new valid registration but enqueue client throws transient error | 201 Created; user & token persisted; metrics audit reflect failure and retry scheduled |
| Password hashing check | password "secret123" | Stored hash !== "secret123"; verify("secret123", storedHash) true |

## Test Commands
- Run unit tests (all): 
```bash
npm run test
# or if project uses npx jest
npx jest --config jest.config.js
```
- Run only registration service tests:
```bash
npx jest tests/unit/services/registrationService.spec.ts --runInBand
```
- Run with coverage:
```bash
npm run test -- --coverage --collectCoverageFrom='src/services/**' --collectCoverageFrom='src/controllers/**'
```

(Adjust commands to project's test runner if not Jest: e.g., pytest -k registration for Python.)

## Coverage Target
- **Line Coverage**: 90%
- **Branch Coverage**: 85%
- **Critical Paths (100% required)**:
  - Idempotency path (insert -> unique-constraint handling -> lookup)
  - Post-commit email enqueue logic (ensuring enqueue only after commit)
  - Canonicalization of email input
  - Password hashing & verification routine

Rationale: High coverage on orchestration and dedupe logic reduces risk of regressions in idempotency behavior.

## Documentation References
- Jest docs: https://jestjs.io/docs/getting-started
- Mocking patterns: https://jestjs.io/docs/mock-functions
- Testing transaction/post-commit patterns: internal design doc (link in repo) or patterns in src/infra/transactionManager.ts
- Project test examples: tests/unit/ (refer to existing test patterns in repo)

## Implementation Checklist
- [ ] Create unit test files for RegistrationController and RegistrationService as listed in Expected Changes
- [ ] Implement mocks for UserRepository, TransactionManager, TokenService, EmailQueueClient, PasswordHasher, Metrics/AuditLogger
- [ ] Implement positive tests: TC-001 (happy path), TC-007 (hashing)
- [ ] Implement negative and error tests: TC-002 (re-submit), TC-004 (unique constraint handling), TC-006 (email enqueue failure)
- [ ] Implement edge/concurrency simulation tests: TC-003 (canonicalization), TC-008 (concurrency simulation)
- [ ] Run test suite and ensure coverage targets met for the BE layer
- Note: Additional layer-specific plans required for DB, Infra (email/queue) and Scripts — implement those separately (test_plan_db_unique_constraints.md, test_plan_infra_email_and_retry.md, test_plan_scripts_migrations.md)

Previous Analysis and Reasoning:
- Decomposition: The user story spans persistence (DB unique index), infra (email/retry), and backend orchestration; to respect the 8-item checklist constraint and provide focused, verifiable unit tests, I generated a BE-focused unit test plan. DB/Infra/Scripts require separate layer-specific plans (referenced above). The above plan maps each acceptance criterion to at least one test case, includes positive/negative/edge/error scenarios, recommends Jest + TypeScript as the primary framework (adjust as needed if project uses a different stack), and prescribes mocking strategies to keep tests isolated and deterministic.