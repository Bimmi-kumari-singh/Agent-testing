test_plan_fe_registration_validation.md

# Unit Test Plan - TASK_us_002_FE

## Requirement Reference
- **User Story**: us_002
- **Story Location**: .propel/context/tasks/us_002/us_002.md
- **Layer**: FE (Frontend)
- **Related Test Plans**:
  - test_plan_be_register_api.md (Backend) — required for server-side validation & graceful degradation flows
  - test_plan_scripts_telemetry.md (Scripts/Telemetry) — required for telemetry assertions when validator fails
- **Acceptance Criteria Covered**:
  - AC-1: Inline RFC5322-aware email validation; show "Enter a valid email address"; aria-invalid="true"; submit disabled while invalid.
  - AC-2: Password policy checklist (min 12 chars, uppercase, lowercase, digit, special, no whitespace) updates visually and programmatically in real-time.
  - AC-3: Prevent network POST to /api/register when invalid; focus first invalid field; display accessible error summary with keyboard-focusable links; block submission until valid.
  - AC-4: Paste/autocomplete triggers same real-time validations and UI updates immediately.
  - AC-5: Graceful degradation: if client validator fails to initialize, UI shows banner "Local validation temporarily unavailable; server will validate your inputs.", allows POST, and does not perform inline validation.

## Test Plan Overview
Purpose: Unit-test client-side validation logic and Registration UI behaviors that provide immediate, accessible feedback and prevent invalid submissions. Scope covers validators (email + password rules), RegistrationForm component behaviors (inline messages, aria attributes, error summary, focus management), PasswordChecklist UI semantics, paste/autocomplete handling, and graceful-degradation UI when validator initialization fails. Tests will be isolated, deterministic, and independent; network interactions are mocked (MSW/jest mocks). Backend responses and telemetry behavior are addressed in separate layer plans.

Why: Ensures user-visible validation is correct, accessible, and non-blocking in degraded environments, reducing invalid server requests and improving UX.

## Dependent Tasks
- Complete BE test plan for /api/register behaviors (server-side validation responses).
- Complete Scripts/Telemetry test plan for telemetry events when validator initialization fails.

## Components Under Test

| Component | Type | File Path | Responsibilities |
|-----------|------|-----------|------------------|
| RegistrationForm | React component (form container) | src/components/RegistrationForm.tsx | Renders form, orchestrates validation, submit handler, error summary, focus management |
| EmailField | React component (input + inline validation) | src/components/EmailField.tsx | Email input, inline RFC5322-aware validation, aria-invalid state, inline message region |
| PasswordChecklist | React component (policy checklist) | src/components/PasswordChecklist.tsx | Displays 6 password rules with visual + programmatic states (aria-checked/role) |
| validators | Utility functions | src/utils/validators.ts | emailIsValid(email): boolean + reason codes; passwordChecks(password): {minLength, uppercase, ...} |
| useValidation | Hook / orchestrator | src/hooks/useValidation.ts | Attaches validators to inputs, debounce, paste/autocomplete handling, exposes formValid state |
| apiClient | Service (mocked in tests) | src/services/api.ts | postRegister(payload) — network POST; mocked in unit tests |
| telemetry | Telemetry client (mocked) | src/telemetry/telemetry.ts | logValidationFailure(metadata) — spied/mocked for init failure path |
| a11y helpers | Utilities | src/utils/a11y.ts | focusFirstInvalid(), buildErrorSummary() — tested via behavior on RegistrationForm |

## Test Cases

| Test-ID | Type | Description | Given | When | Then | Assertions |
|---------|------|-------------|-------|------|------|------------|
| TC-001 | positive | Valid email (RFC-valid, common) does not show inline error | Registration form mounted; email input empty | User types "user@example.com" | No inline error; aria-invalid false; submit not blocked (assuming password valid) | email inline message absent; input.getAttribute('aria-invalid') !== "true"; formValid true |
| TC-002 | negative | Invalid email examples per AC fail validation | Mount form | Type "user@@example.com", "user@", "user@example..com" | Inline message "Enter a valid email address"; aria-invalid="true"; submit control disabled | inline message text present; aria-invalid==="true"; submit button disabled |
| TC-003 | edge_case | RFC-valid unusual formats accepted | Mount form | Type "\"john..doe\"@example.com", "john.o'connor+tag@example.co.uk" | No inline error; aria-invalid not set | inline message absent; aria-invalid !== "true" |
| TC-004 | edge_case | Email length boundary at 254 accepted, 255 rejected | Mount form | Generate email local+domain length =254 and =255 | 254 -> accepted; 255 -> show "Email exceeds maximum length (254 characters)" and block submit | check messages; check submit disabled on 255; message equality; aria-invalid set for 255 |
| TC-005 | positive | Password checklist updates all six rules in real-time | Password field focused | Type password incrementally to satisfy rules one-by-one | Checklist items toggle satisfied/unsatisfied accordingly | For each rule: element has programmatic indicator (aria-checked or data-state) matching expected; visual class updated |
| TC-006 | accessibility | Password checklist is programmatic and discoverable | Mount PasswordChecklist | Query checklist items | Items expose role/aria-checked (or equivalent) and are reachable by screen-reader selectors | Each rule element has role (e.g., "listitem" with aria-checked) or data attribute; tests assert presence of ARIA attributes |
| TC-007 | positive / paste | Paste/autocomplete triggers immediate validation and UI update | Mount form | Fire paste event with valid/invalid values; simulate autofill by setting value + input event | Validation runs immediately; UI updates without blur | Inline error or checklist changes appear after event; no additional user keystroke required |
| TC-008 | negative / submit-prevent | Submit prevented when any input invalid; focus and error summary shown | Form with invalid email and/or password | Click submit | No POST emitted; focus moved to first invalid field; error summary is visible and links focus fields | Assert network mock not called; document.activeElement is first invalid input; error summary aria-live assertive; links focus corresponding fields when activated |
| TC-009 | error | Validator initialization failure -> banner shown and inline validation disabled | Simulate validation init throwing (mock useValidation/init module to throw) | Mount form | Banner "Local validation temporarily unavailable; server will validate your inputs." shown; inline validation not present; submit allowed | Banner text present; inline messages absent even after input; submit button enabled; apiClient.postRegister called when submit clicked |
| TC-010 | edge_case | Rapid input changes and debounced validation (if debounce used) | Hook configured with debounce | Type quickly and fast sequences | Validation behaves deterministically; last value validated after debounce | Use fake timers to advance; final validation result matches last input |
| TC-011 | accessibility | Error summary links are keyboard-focusable and move focus to fields | Form with two invalid fields | Click/activate error summary link via keyboard (Enter) | Focus moves to corresponding field | document.activeElement === expected field after activation |
| TC-012 | negative | Prevent POST when formValid===false (defensive guard) | Form handler instrumented | Programmatically call submit handler while formValid false | No network request | Assert apiClient.postRegister not called |
| TC-013 | negative | Special characters and whitespace in password correctly evaluated | Mount form | Type password containing whitespace and special characters | 'no whitespace' rule fails when whitespace present; other rules evaluated correctly | Checklist state reflects whitespace violation; other rules true/false as applicable |

Notes:
- Every AC is mapped: AC-1 covered by TC-002, TC-001, TC-003, TC-004; AC-2 by TC-005/TC-006/TC-013; AC-3 by TC-008/TC-012/TC-011; AC-4 by TC-007; AC-5 by TC-009.
- Tests are unit-level (component + hook + util). Interaction with network is mocked; backend behavior asserted via request presence/absence and request payload shape.

## AI Component Test Cases [CONDITIONAL: If AIR-XXX in scope]
- AI Impact: No (this user story does not include LLM/AI components). Skip AI section.

## Expected Changes

| Action | File Path | Description |
|--------|-----------|-------------|
| CREATE | tests/unit/RegistrationForm.test.tsx | Unit tests for RegistrationForm behaviors (submit prevention, error summary, focus) |
| CREATE | tests/unit/EmailField.test.tsx | Unit tests for email validation logic and ARIA states |
| CREATE | tests/unit/PasswordChecklist.test.tsx | Unit tests for checklist visual/programmatic states |
| CREATE | tests/unit/validators.test.ts | Unit tests for emailIsValid() and passwordChecks() functions (boundary and RFC subset) |
| UPDATE | jest.setup.ts | Configure testing-library, MSW, jest-axe, user-event setup, fake timers if needed |
| CREATE | tests/mocks/api.mock.ts | Mock for apiClient (or MSW handlers for /api/register) |
| CREATE | tests/fixtures/validationData.ts | Test data for valid/invalid emails and passwords |

## Mocking Strategy

| Dependency | Mock Type | Mock Behavior | Return Value |
|------------|-----------|---------------|--------------|
| apiClient.postRegister | mock (MSW or jest mock) | Intercept /api/register; allow control over responses (200, 400 with field errors, 500) | 200: { success:true }; 400: { errors: { email: "Invalid", password: "Too weak" } } |
| telemetry.logValidationFailure | spy/mock | Track invocation and sanitize payload; do not send network events during unit tests | jest.fn() (no-op) |
| validationInit / useValidation internals | stub/mock | Simulate normal initialization or throw to test graceful degradation | normal: returns validators; failure: throw new Error('init failed') |
| DOM clipboard (paste) | jsdom event stub | Mock clipboardData on paste events to supply text | clipboardData.getData('text') => provided string |
| focus utilities (a11y.focusFirstInvalid) | spy | Track focus calls without moving real focus beyond jsdom capability | jest.fn() |
| Timers/debounce | fake timers | Control debounce behavior deterministically in tests | jest.useFakeTimers(); jest.advanceTimersByTime() |

Mocking notes:
- Use MSW (Mock Service Worker) for network request interception to keep assertions close to real fetch/axios behavior.
- Keep mocks local to each test and reset (cleanup) between tests: afterEach(() => { jest.resetAllMocks(); server.resetHandlers(); cleanup(); }).
- Do not mock internal implementation of utilities under test; prefer mocking external dependencies only (network, telemetry, initialization entrypoints).
- Ensure no real network calls or telemetry are emitted.

## Test Data

| Scenario | Input Data | Expected Output |
|----------|------------|-----------------|
| Valid email | "user@example.com" | emailIsValid -> true |
| Invalid email examples | "user@@example.com", "user@", "user@example..com" | emailIsValid -> false; message "Enter a valid email address" |
| RFC unusual valid | "\"john..doe\"@example.com" | emailIsValid -> true |
| Email length 254 | generated 254-char email | accepted |
| Email length 255 | generated 255-char email | rejected with "Email exceeds maximum length (254 characters)" |
| Valid password | "Aa1!aaaaaaaa" (12 chars, others satisfied) | passwordChecks -> all rules true |
| Password with whitespace | "Aa1!aaaa aaaa" | passwordChecks -> whitespace rule false |
| Partial password progression | incremental typing: "A", "Aa1!", "Aa1!aaaaaaa" | checklist updates progressively |
| Paste content | clipboard text -> "pasted@example.com" | validation runs immediately |

## Test Commands
- Run all frontend unit tests:
  - npm test -- tests/unit -- --testPathPattern=tests/unit
  - or: yarn test tests/unit
- Run frontend tests with coverage:
  - npm run test:coverage -- --collectCoverageFrom='src/components/**,src/hooks/**,src/utils/**'
  - or: yarn test:coverage
- Run a single test file:
  - npm test -- tests/unit/RegistrationForm.test.tsx
  - or: yarn test tests/unit/RegistrationForm.test.tsx
- Run tests with jest-axe (a11y assertions included in test suites):
  - npm test -- -t "accessibility" --runInBand
Notes:
- CI should run test:coverage and fail if coverage < required thresholds (see below).
- If project uses pnpm or a different test script name, adapt commands to package.json test scripts.

## Coverage Target
- **Line Coverage**: 85% (overall for targeted modules)
- **Branch Coverage**: 80%
- **Critical Paths** (aim for 100% unit coverage):
  - validators.emailIsValid: invalid patterns, RFC-valid edge cases, length boundary
  - validators.passwordChecks: each rule true/false branch
  - RegistrationForm submit guard: branch preventing POST when invalid vs allowing when validator unavailable
  - useValidation init failure path and its effect on UI
  - focusFirstInvalid behavior invoked on submit failure

## Documentation References
- **Jest**: https://jestjs.io
- **React Testing Library**: https://testing-library.com/docs/react-testing-library/intro
- **MSW (Mock Service Worker)**: https://mswjs.io
- **@testing-library/user-event**: https://testing-library.com/docs/user-event/intro
- **jest-axe** (accessibility checks): https://github.com/nickcolley/jest-axe
- **Project Test Patterns**: (link to in-repo examples) .propel/context/docs/testing-patterns.md
- **Mocking Guide**: jest docs (https://jestjs.io/docs/mock-functions)

## Implementation Checklist
- [ ] Create unit test files: RegistrationForm.test.tsx, EmailField.test.tsx, PasswordChecklist.test.tsx, validators.test.ts
- [ ] Configure MSW handlers for /api/register and add helpers in jest.setup.ts
- [ ] Implement positive, negative, edge, and error-scenario tests as per Test Cases (TC-001 .. TC-013)
- [ ] Mock telemetry and validationInit paths; ensure no external network/telemetry calls
- [ ] Add accessibility assertions (aria attributes, aria-live, error summary link focus) using jest-axe and DOM queries
- [ ] Ensure tests use isolated setup/teardown (reset mocks, server.resetHandlers, cleanup) for test independence
- [ ] Run test suite and meet Coverage Targets; add tests to cover any uncovered critical branches
- [ ] Additional layer-specific plans required: backend tests for server validation responses and scripts/telemetry tests for telemetry assertions

Previous Analysis and Reasoning:
- The feature spans multiple layers; the frontend contains the bulk of UX/validation complexity and must be tested in isolation with mocked network and telemetry. Because the number of implementation checklist items across all layers would exceed 8, plans were decomposed by layer; this document is the FE plan. BE and Scripts plans must be implemented separately to cover server error mapping and telemetry behaviors.

Reasoning Summary (overview)
- Focus FE tests on user-visible behavior, validator correctness (including RFC-valid exceptions and length boundaries), accessibility requirements (aria attributes, live regions, focus management), paste/autocomplete behavior, prevention of invalid POSTs, and graceful degradation when client-side validation fails. Use Jest + React Testing Library + MSW + jest-axe for robust, deterministic unit tests.