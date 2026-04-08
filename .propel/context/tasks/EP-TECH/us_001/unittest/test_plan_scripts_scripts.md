# Unit Test Plan - [us_001_scripts]

## Requirement Reference
- **User Story**: us_001
- **Story Location**: .propel/context/tasks/us_001/us_001.md
- **Layer**: Scripts
- **Related Test Plans**: test_plan_infra_repo_protection.md (Infra), test_plan_be_repo_content.md (BE/Repo-Content)
- **Acceptance Criteria Covered**:
  - AC-1: Repository skeleton presence (scripts validate presence of required directories/files during bootstrap).
  - AC-3: Pre-commit framework installed and configured; pre-commit hooks run lint/format/secret/large-file checks locally and reject commits.
  - AC-5: Bootstrap completes dev environment setup so new developer can run test suite and linters locally within documented steps (~10 minutes).
  - Edge AC (Large files/secrets): pre-commit/CI secret and large-file scanning implemented and reject offending commits.

## Test Plan Overview
This plan covers unit tests for repository scripting and developer-bootstrapping logic: scripts/bootstrap-dev.{sh,py}, pre-commit config validator, secret-scan and large-file detection logic, and small wrapper utilities used by those scripts. Purpose: ensure deterministic, idempotent, error-reporting behaviour for developer onboarding automation and client-side enforcement of linting/secret/large-file checks before CI runs. Scope is unit-level (no real network or package installs); external interactions are mocked.

Why these tests:
- They prevent noisy developer experience regressions (broken bootstrap or hooks).
- They ensure early detection of secrets/large files before CI runs.
- Unit tests are fast and deterministic; integration tests / CI validation are separate.

## Dependent Tasks
- Infra branch-protection tests to validate CI job naming (Infra plan).
- Repo-content tests to validate presence/structure of template files (BE/Repo-Content plan).

## Components Under Test

| Component | Type | File Path | Responsibilities |
|-----------|------|-----------|------------------|
| bootstrap script | script (bash or python) | scripts/bootstrap-dev.sh or scripts/bootstrap-dev.py | Detects toolchain, installs pre-commit, sets up environment, documents minimal tool versions, exits with clear codes/messages |
| pre-commit config validator | utility/function | scripts/validate_precommit.py | Validates .pre-commit-config.yaml contains expected hooks (formatters, linters, secret/large-file hooks) |
| secret scanner | library function | scripts/hooks/secret_scan.py | Detects secret patterns in files (API keys, private keys, token patterns) |
| large-file checker | library function | scripts/hooks/large_file_check.py | Detects files > configured threshold (default 5MB) |
| git wrapper (test-friendly) | utility/module | scripts/lib/git_wrapper.py | Abstraction for git operations used by bootstrap and hooks (reading staged files, aborting commit) |
| installer stub | helper | scripts/lib/installer.py | Encapsulates package install commands (pip/npm/pre-commit) — unit-under-test ensures correct commands chosen, not executed |

## Test Cases

| Test-ID | Type | Description | Given | When | Then | Assertions |
|---------|------|-------------|-------|------|------|------------|
| TC-001 | positive | Bootstrap detects missing pre-commit and installs it (idempotent) | FS has project files but pre-commit not installed (mocked) | Run bootstrap script | Pre-commit install path invoked; pre-commit configured | installer.install called with expected arguments; exit code 0; printed next-steps message |
| TC-002 | positive | Bootstrap is idempotent when pre-commit already installed | pre-commit installed (mocked) | Run bootstrap script twice | Second run does not attempt installation | installer.install not called on second run; exit code 0 |
| TC-003 | negative | Bootstrap fails with descriptive error when required tool missing and auto-install disabled | Tool (e.g., python/node) missing and auto-install flag off | Run bootstrap with --no-auto-install | Bootstrap exits non-zero with guidance | Exit code != 0; error message suggests install steps and escalations |
| TC-004 | positive | validate_precommit accepts expected hook config | .pre-commit-config.yaml contains formatter, linter, secret/large-file hooks | Run validator | Validator returns success | Boolean true; validation report lists required hooks present |
| TC-005 | negative | validate_precommit rejects missing hooks | config missing secret/large-file hook | Run validator | Validator returns failure with missing-hook list | Boolean false; message enumerates missing hooks |
| TC-006 | positive | secret scanner flags file with API key-like pattern | File content contains "api_key=AKIA..." | Run secret_scan on file | Detector returns true/raises detection | Detection returns issue object with line, pattern, severity |
| TC-007 | negative/edge | secret scanner does not false-positive on allowed benign tokens | File contains long base64 blob used for fixtures (whitelisted path) | Run secret_scan with whitelist | No detection | Returns empty findings when path whitelisted |
| TC-008 | positive | large-file checker flags file > threshold | Simulated staged file size = 6MB | Run large_file_check | Detection returned | Finding includes file path, size; hook signals reject |
| TC-009 | edge | large-file checker accepts file exactly at threshold | Simulated file size = 5MB | Run large_file_check | No detection | No findings |
| TC-010 | error | installer failures are surfaced with actionable message | installer.install raises subprocess.CalledProcessError (mocked) | Run bootstrap script | Bootstrap exits non-zero and logs remediation steps | Exit code != 0; log contains command attempted and suggestion to open issue/escalate |
| TC-011 | negative | Git wrapper handles absence of staged files gracefully | No staged files | Hooks call git_wrapper.get_staged_files | Hook exits with success (no-op) | Returns empty list; no exception |
| TC-012 | error | Hook abort path triggers correct non-zero exit code for commit | secret_scan returns findings | Simulate commit hook execution | Hook aborts commit | Exit code non-zero; printed reasons; git_wrapper.abort_commit called |

Notes:
- Each test maps to ACs:
  - AC-3: TC-001, TC-002, TC-004, TC-005, TC-012
  - AC-5: TC-001, TC-002, TC-003, TC-010
  - Large-file/secret edge: TC-006..TC-009, TC-012

## Expected Changes

| Action | File Path | Description |
|--------|-----------|-------------|
| CREATE | tests/scripts/test_bootstrap.py | Unit tests for bootstrap logic (pytest) |
| CREATE | tests/scripts/test_precommit_validator.py | Tests for pre-commit config validator |
| CREATE | tests/scripts/test_secret_scan.py | Tests for secret scanner (unit) |
| CREATE | tests/scripts/test_large_file_check.py | Tests for large-file checker |
| CREATE | tests/mocks/installer.mock.py | Mock installer to simulate package installs and failures |
| CREATE | tests/fixtures/* | Fixture files: valid/invalid .pre-commit-config.yaml, files with/without secrets, size-simulated files |

## Mocking Strategy

| Dependency | Mock Type | Mock Behavior | Return Value |
|------------|-----------|---------------|--------------|
| Filesystem | in-memory fs | Use pyfakefs (Python) or memfs (Node) to simulate files and sizes | Creates files with specified content/size |
| subprocess / installer | stub/mock | Replace actual installer commands with stub that records calls or raises errors | Success: return 0; Failure: raise CalledProcessError |
| git operations | mock | git_wrapper functions return controlled staged file lists or perform no-ops; abort_commit tracked | get_staged_files -> [] or list; abort_commit records call |
| pre-commit presence | monkeypatch | Simulate detection by toggling environment or path checks | present/absent |
| whitelists (paths) | fixture | Provide whitelist entries to secret scanner | Whitelisted path -> no findings |
| logging | spy | Capture logs to assert user-facing messages | Log entries captured for assertions |

Mocking approach (why): Use high-level mocks (public contracts) rather than patching internal implementation to avoid brittle tests. For Python, prefer pytest + pytest-mock + pyfakefs. For bash scripts, wrap shell logic in small Python utilities where possible; for pure-shell sections, use bats-core or test harnesses invoking scripts with controlled environment vars and temporary FS mounts (or use Docker containers in integration tests — not for unit tests).

## Test Data

| Scenario | Input Data | Expected Output |
|----------|------------|-----------------|
| Valid pre-commit config | fixtures/precommit_valid.yaml | Validator passes, identifies required hooks |
| Missing hooks config | fixtures/precommit_missing_secret_hook.yaml | Validator fails; lists missing hook names |
| Secret present | fixtures/file_with_api_key.txt | secret_scan returns finding: {path, line, pattern} |
| Benign base64 blob | fixtures/fixture_blob.bin | secret_scan returns no findings when whitelisted |
| Large file >5MB | simulated file size 6*1024*1024 bytes | large_file_check returns finding with size and path |
| Installer success | installer.mock success | bootstrap completes (exit 0) |
| Installer failure | installer.mock raises error | bootstrap exits non-zero; message contains remediation steps |

## Test Commands
- Run Python/unit tests (recommended): pytest tests/scripts -q
- Run with coverage (Python): pytest --cov=scripts --cov-report=term-missing tests/scripts
- Run a single test: pytest tests/scripts/test_bootstrap.py::test_bootstrap_installs_precommit -q
- If using bats for shell parts: bats tests/bats/*.bats
- CI note: Ensure tests run with pyfakefs and pytest-mock installed in CI environment; add to requirements-dev.txt or tox/venv.

## Coverage Target
- **Line Coverage**: 90%
- **Branch Coverage**: 80%
- **Critical Paths (MUST be 100%)**:
  - Tool-detection logic branches (detect vs install vs skip)
  - Secret detection decision branch that decides to abort commit
  - Large-file threshold boundary logic (<= threshold vs > threshold)
  - Git abort path invoked upon findings

Rationale: High line coverage ensures bootstrap logic and hooks are guarded; complete coverage for critical decisions prevents developer-blocking regressions.

## Documentation References
- pytest docs: https://docs.pytest.org/
- pyfakefs: https://pyfakefs.readthedocs.io/
- bats-core (for shell tests): https://bats-core.readthedocs.io/
- pre-commit: https://pre-commit.com/
- YAML validation (python): PyYAML + yamllint
- Secret detection patterns reference: company policy (link placeholder in repo docs)

## Implementation Checklist (maximum 8 items)
- [ ] Create unit test files per Expected Changes and add to tests/scripts/
- [ ] Add pyfakefs and pytest-mock to dev requirements and CI test environment
- [ ] Implement mocks for installer, git_wrapper, and filesystem fixtures
- [ ] Implement positive, negative, edge, and error tests listed in Test Cases
- [ ] Ensure each test is independent (use fixtures and monkeypatch; no shared state)
- [ ] Integrate unit tests into CI job used by branch protection (name must match CI job expected by Infra plan)
- [ ] Add test coverage collection and validate thresholds locally and in CI
- [ ] Document how to run the tests and interpret failures in CONTRIBUTING.md (bootstrap section)

Additional layer-specific plans required: Infra (branch-protection/CI naming validation) and BE/Repo-Content (presence of skeleton files) — see related test plans.

Previous Analysis and Reasoning:
- Decomposed us_001 into layer-specific plans (Infra, Scripts, BE) because the full scope exceeds checklist limits per policy. This plan covers the Scripts layer focused on bootstrap, pre-commit, secret and large-file checks. Tests are unit-level, fast, deterministic, and rely on public contracts with dependencies mocked.

Notes on framework choice (short why):
- Prefer pytest + pyfakefs for Python utilities because they provide simple, fast, and well-documented primitives for filesystem and subprocess mocking (keeps tests deterministic).
- For pure-shell logic that cannot be refactored, use bats-core to exercise script behavior in a controlled environment; however, prefer small Python wrappers to enable easier unit testing (aligns with KISS and testability).
- Mock external effects (installs, network, git pushes) to keep unit tests isolated and fast.

Test Independence Guarantee:
- All tests must create and tear down their own fixtures (use tmp_path/pyfakefs) and must not rely on global environment state. Tests must not perform real installs or modify the developer machine.

Test Run Example (local quick-start):
1. Create virtualenv and install dev requirements: pip install -r requirements-dev.txt
2. Run: pytest tests/scripts -q
3. For coverage: pytest --cov=scripts tests/scripts

File name for this plan: test_plan_scripts_bootstrap.md