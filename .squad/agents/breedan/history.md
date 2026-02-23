# Breedan — History

## Project Context
- **Project:** Squad — programmable multi-agent runtime for GitHub Copilot
- **Owner:** Brady
- **Stack:** TypeScript (strict, ESM), Node.js ≥20, Ink 6 (React for CLI), Vitest
- **Existing tests:** test/*.test.ts using Vitest, ink-testing-library for component tests
- **Key files:** vitest.config.ts, test/repl-ux.test.ts (existing ink component tests)

## Learnings

### E2E Terminal Harness Design (2026-02-23)

**Architecture Decision:** Built terminal harness using `child_process.spawn` with pipes instead of node-pty. This pragmatic approach avoids native compilation issues on Windows and provides cross-platform compatibility. The harness API is designed to be compatible with node-pty, so we can upgrade later when CI has build tools without breaking tests.

**Key Components:**
- `test/acceptance/harness.ts` — TerminalHarness class for spawning CLI and capturing output
- `test/acceptance/support/gherkin.ts` — Minimal Gherkin parser (no external deps)
- `test/acceptance/support/runner.ts` — Test runner that maps Gherkin steps to vitest
- `test/acceptance/steps/cli-steps.ts` — Given/When/Then step definitions
- `test/acceptance/features/*.feature` — 5 Gherkin feature files (version, help, status, error-handling, doctor)
- `test/ux-gates.test.ts` — UX quality gates (overflow, error remediation, clean output)

**Patterns:**
- CLI spawned with `node packages/squad-cli/dist/cli-entry.js [args]`
- Environment variables used to control terminal size (COLUMNS, LINES) and disable interactivity (TERM=dumb, NO_COLOR=1)
- Output accumulated in append-only buffer for waitForText assertions
- ANSI codes stripped using simple regex (no extra dependencies)
- Tests focus on non-interactive commands (--version, --help, status, doctor) since interactive shell requires Copilot SDK unavailable in test environment

**UX Gates Philosophy:**
- Quality gates document existing behavior and catch regressions
- Pragmatic thresholds (120 char max, not 80) balance ideals with reality
- Informational logging for soft constraints (lines exceeding 80 chars)
- Hard assertions for critical issues (error messages include remediation)

**Test Results:** All 13 tests passing (7 acceptance scenarios + 6 UX gates)

- **Stack:** TypeScript (strict, ESM), Node.js >=20, Ink 6 (React for CLI), Vitest
- **Key files:** vitest.config.ts, test/acceptance/ (E2E harness + Gherkin scenarios)

## Learnings

### E2E Coverage Expansion (2026-02-23)

**Scenarios added (14 new, 21 total):**
- init-command.feature: re-init in existing project, exit code
- status-extended.feature: resolution details, no-squad directory isolation
- doctor-extended.feature: diagnostic header, summary pass counts
- help-comprehensive.feature: all core commands listed, flags section
- error-paths.feature: special characters, invalid flags, help suggestion
- exit-codes.feature: success (0) and failure (1) validation

**Test infrastructure:**
- test/acceptance/harness.ts: TerminalHarness with cwd option
- test/acceptance/support/gherkin.ts: Minimal Gherkin parser
- test/acceptance/support/runner.ts: Step registry + vitest mapping
- test/acceptance/steps/cli-steps.ts: Step definitions
- test/acceptance/features/*.feature: 11 feature files
- test/ux-gates.test.ts: 6 UX quality gate tests

**Key patterns:**
- Absolute path resolution for cwd-isolated tests
- mkdtempSync for no-squad scenarios
- ANSI stripped, NO_COLOR=1, TERM=dumb for deterministic output
- Negative assertions via 'does not contain' step

**PR:** #348 (closes #326)
