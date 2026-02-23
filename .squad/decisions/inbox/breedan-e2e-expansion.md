# Decision: E2E Test Coverage Expansion

**Date:** 2026-02-23
**Author:** Breedan (E2E Test Engineer)
**Status:** Accepted
**PR:** #348 (closes #326)

## Context
Issue #326 requested expanding E2E acceptance test coverage to 10+ scenarios. The existing harness (child_process-based) and Gherkin parser were functional but only had 7 scenarios.

## Decision
Added 14 new Gherkin scenarios across 6 new feature files, bringing total to 21 acceptance scenarios + 6 UX gate tests (27 total).

### New capabilities added to harness:
- `cwd` option on `spawnWithArgs()` for running CLI in isolated directories
- Absolute path resolution for CLI entry point (required for non-repo-root cwd)
- `mkdtempSync` temp dir creation step for testing without .squad/
- Negative assertion step ("does not contain")

### Coverage areas:
| Feature | Scenarios | Tests |
|---------|-----------|-------|
| init-command | 2 | re-init behavior, exit code |
| status-extended | 2 | resolution details, no-squad dir |
| doctor-extended | 2 | header display, summary counts |
| help-comprehensive | 2 | core commands, flags section |
| error-paths | 3 | special chars, invalid flags, help hint |
| exit-codes | 3 | version/help success, unknown failure |

## Alternatives Considered
- **node-pty based tests**: Deferred — native compilation issues on Windows CI. Current child_process approach gives sufficient coverage for non-interactive commands.
- **Playwright/Cypress-style tools**: Overkill for CLI testing; adds large dependency footprint.

## Team Impact
- New step definitions available for other test authors
- Harness cwd support enables future tests that need project isolation
