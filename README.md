# Agent-testing — README

One-sentence project summary
- The repository is labeled "Agent-testing" but that specific purpose is unspecified in docs; the files present a full Product/BRD and implementation plan for a Secure Authentication System that centralizes user registration, secure login (JWT + refresh), password management, MFA (Email/SMS/TOTP), session management, and account protection.

What this repo contains (main components / where to look)
- Design & architecture: .propel/context/docs/design.md
- Epics and backlog: .propel/context/docs/epics.md
- Requirements spec: .propel/context/docs/spec.md and artifacts/spec/spec.md
- Business Requirements Document: docs/Brd.md
- User stories & decomposition: .propel/context/docs/user_stories.md
- Unit test plan index: .propel/context/docs/unit_test_plan.md
- Story-level tasks & test plans: .propel/context/tasks/ (per-epic user stories and per-story unit test plans, e.g. .propel/context/tasks/EP-TECH/us_001/unittest/)
- Example task artifacts (detailed user stories / acceptance criteria): .propel/context/tasks/EP-001/*

Key highlights from the spec and epics (goals, main features, capabilities)
- Central goal: build one auditable, secure Authentication System to replace ad-hoc auth across apps (registration, login, password reset, MFA, tokens).
- User-visible flows: sign-up with email verification, login with email+password, password reset via email link, optional MFA (Email OTP, SMS OTP, Authenticator/TOTP).
- Session model: short-lived access tokens (JWT) + longer-lived refresh tokens with rotation and revocation.
- Security controls: secure password hashing (Argon2id preferred / bcrypt fallback), HTTPS/TLS-only, OWASP mitigations, rate limiting and brute-force protections, account lockout policies.
- Operational/Non-functional targets: login < 2s for 95% under normal load, support 10,000+ concurrent users, availability target ~99.9% (design lists 99.95% target as desirable), monitoring (Prometheus/Grafana) and centralized logging (ELK).
- Architecture: microservices (User, Auth, MFA, Notification), API Gateway, Postgres DB, Redis cache, Kafka for async events, containerized deployments on Kubernetes (detailed in design.md).

Where to find test plans and how testing is organised
- Top-level unit test plan index: .propel/context/docs/unit_test_plan.md
- Unit tests are organized per user story and per task (see .propel/context/tasks/*/unittest/* for concrete unit test plans and examples).
- Testing layers and examples: plans show layered testing (BE / scripts / infra) — for instance .propel/context/tasks/EP-TECH/us_001/unittest/test_plan_scripts_scripts.md covers bootstrap/script unit tests; .propel/context/tasks/EP-001/... includes BE unit test plans.
- CI/CD and pipeline guidance is scoped under EP-TECH (see .propel/context/tasks/EP-TECH/us_003.md, us_004.md) — unit tests are expected to integrate into the CI pipeline described there.
- Note: integration & infra tests (DB migrations, email enqueue retries, deployment smoke-tests) are referenced in per-story test plans and marked as separate plans (DB / Infra) where applicable.

Recommended next steps for contributors
1. Read the BRD and spec first:
   - docs/Brd.md
   - artifacts/spec/spec.md
   - .propel/context/docs/spec.md
2. Review architecture and epics to pick scope and priorities:
   - .propel/context/docs/design.md
   - .propel/context/docs/epics.md
3. Inspect user stories and pick a small story to start (user stories + acceptance criteria):
   - .propel/context/docs/user_stories.md
   - .propel/context/tasks/EP-001/* (example story breakdowns)
4. Follow the unit test plans linked from:
   - .propel/context/docs/unit_test_plan.md and the per-story unittest plans under .propel/context/tasks/*/unittest/
5. Use EP-TECH onboarding tasks to set up dev environment and CI:
   - .propel/context/tasks/EP-TECH/us_002.md (dev environment), us_003.md (CI), us_006.md (service scaffold)
6. If information needed to start (owner, CI endpoints, credentials, repository-specific runbooks) is missing, open an Issue in this repository requesting the missing detail.

Contact / owner
- There is no explicit contact, author, or repository owner specified in the documents I reviewed — "unspecified in docs". If you need an owner/contact, please open an Issue in the repo or raise this with your team's repository/ops administrators.

Notes and constraints
- Statements in this README are drawn from the BRD/spec/epics/design and task files in the repository. Where the docs are silent (for example, the specific "Agent-testing" purpose or contact details), I have marked that as "unspecified in docs" rather than guessing.
- If you plan to implement features, follow the epics priority guidance: EP-TECH (foundation) is highest priority to unblock other epics (see .propel/context/docs/epics.md).

Relevant paths recap
- BRD: docs/Brd.md
- Product spec: artifacts/spec/spec.md and .propel/context/docs/spec.md
- Architecture: .propel/context/docs/design.md
- Epics: .propel/context/docs/epics.md
- User stories: .propel/context/docs/user_stories.md
- Unit test plans index: .propel/context/docs/unit_test_plan.md
- Tasks & per-story tests: .propel/context/tasks/

If anything above needs to be expanded into a developer onboarding checklist or an ISSUE template, create an Issue referencing this README and tag it EP-TECH / documentation.