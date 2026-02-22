### Aspire + Observer patterns — Fenster (2026-02-22)

**Context:** OTel Phase 4 — Issues #265, #268

**Decisions made:**

1. **Aspire command uses Docker by default** when dotnet Aspire workload isn't detected. Docker is more portable and doesn't require .NET SDK installation. The `--docker` flag forces Docker even when dotnet is available.

2. **SquadObserver uses `fs.watch` with recursive:true** instead of chokidar or other watchers. Zero additional dependencies, works on Windows/macOS natively. Linux users may need to increase inotify watchers for large .squad/ directories.

3. **File classification is string-based prefix matching** on the relative path from .squad/ root. Categories: agent, casting, config, decision, skill, unknown. Windows backslashes are normalized to forward slashes before classification.

4. **Observer emits `agent:milestone` EventBus events** for file changes rather than introducing a new event type. This keeps compatibility with existing EventBus subscribers (SquadOffice expects `agent:milestone`). The payload includes `action: 'file_change'` to distinguish from other milestones.

5. **Debounce at 200ms default** to avoid flooding spans on rapid file saves (e.g., editor autosave). Configurable via `debounceMs` option.
