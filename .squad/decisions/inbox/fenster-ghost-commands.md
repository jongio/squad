# Decision: Ghost Command Aliasing Strategy

**Author:** Fenster
**Date:** 2026-02-24
**Issues:** #501, #503, #504, #507, #509

## Context

Five commands were documented but had no CLI handlers: `hire`, `heartbeat`, `shell`, `loop`, `run`. Users hitting these got "Unknown command" errors.

## Decision

Wire them as aliases to existing functionality rather than building new features:

| Ghost Command | Resolution | Rationale |
|---------------|-----------|-----------|
| `squad hire` | Alias → `squad init` | Team creation = initialization |
| `squad heartbeat` | Alias → `squad doctor` | Health check already exists |
| `squad shell` | Explicit launch → `runShell()` | Same as no-args behavior, but explicit |
| `squad loop` | Alias → `squad triage` | Work monitoring = triage |
| `squad run <agent>` | Stub with "coming soon" message | Non-trivial to implement properly; directs users to REPL |

## Rationale

- **Minimal code change:** Each alias is 1-2 lines in the command router.
- **No new dependencies:** All aliases reuse existing implementations.
- **`run` is deferred:** Proper agent dispatch outside the REPL requires session lifecycle changes. A stub with a helpful message is better than a broken implementation.
- **Backwards compatible:** No existing commands were changed.

## Impact

- CLI help text updated to show all five commands.
- Users can now use the documented names without confusion.
- `squad run` will need a real implementation in a future PR when non-interactive agent dispatch is ready.
