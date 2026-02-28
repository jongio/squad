# PRD: Personal Squad Consult Mode

**Author:** James Sturtevant  
**Date:** 2026-02-27  
**Status:** Draft  
**Wave:** M6 (Personal Squad Enhancement)

---

## Problem Statement

You have a personal squad at your global squad path (resolved via `resolveGlobalSquadPath()` — e.g. `~/.config/squad/.squad` on Linux, `~/Library/Application Support/squad/.squad` on macOS, `%APPDATA%/squad/.squad` on Windows) with agents, skills, and decisions refined over time. You want to use this team on projects you don't own (OSS contributions, client work, temporary collaborations) without:

1. **Polluting the project** — no `.gitignore` changes, no committed `.squad/` folder
2. **Polluting your squad** — no project-specific knowledge bleeding into your global squad

Currently, `squad init` creates a project-owned squad. There's no way to "bring your team" to a project invisibly, work with them, and bring back only the generic learnings.

---

## Solution: Consult Mode

Your team **consults** on a project. They bring their expertise, do the work, learn things. When done, they extract what's reusable and return home. The project never knows Squad was there.

### Key Behaviors

| Aspect | Normal Mode | Consult Mode |
|--------|-------------|--------------|
| Squad location | `.squad/` in project | Global squad path (personal) |
| Git visibility | Committed or `.gitignore` | Invisible via `.git/info/exclude` |
| Writes go to | Project `.squad/` | Project `.squad/` (local only) |
| After session | Stays in project | Extract generic → global squad, discard rest |

---

## Commands

### Entry: `squad consult`

```bash
cd ~/projects/their-oss-project
squad consult              # Enter consult mode
squad consult --status     # Check if in consult mode, show pending learnings
squad consult --check      # Dry-run: show what would happen without creating files
```

**Creates:**
```
.squad/
├── config.json     # { "version": 1, "teamRoot": "<resolveGlobalSquadPath()>/.squad", "projectKey": "consult", "consult": true }
└── sessions/       # Local session history
```

**Also:**
- Appends `.squad/` to `.git/info/exclude` (git-internal, never visible)
- If project already has committed `.squad/`: **error out**

### Exit: `squad extract`

```bash
squad extract                    # Review and extract generic learnings
squad extract --dry-run          # Preview what would be extracted (no changes)
squad extract --clean            # Also delete project .squad/ after (prompts for confirmation)
squad extract --clean --yes      # Delete without confirmation
squad extract --accept-risks     # Allow extraction despite license or other risks
```

**Flow:**
1. Read project LICENSE file
2. Warn if copyleft (GPL, AGPL) — license contamination risk
3. Propose generic learnings (skills + decisions)
4. User approves, edits, or rejects
5. Merge approved items to global squad
6. Log to `<globalSquadPath>/consultations/{project}.md`

---

## Classification Intelligence

The extraction review uses **LLM-assisted classification**. An agent (Scribe or Lead) analyzes each learning and proposes whether it's generic or project-specific.

**Classification heuristics the agent applies:**

| Signal | Classification |
|--------|----------------|
| Contains file paths from this project | Project-specific |
| References project-specific config/env | Project-specific |
| Pattern applies to any project (validation, testing, architecture) | Generic |
| Decision about a specific library/framework choice for THIS project | Project-specific |
| Decision about coding standards that apply everywhere | Generic |

**User always has final say.** The agent proposes, user approves/rejects/edits. No extraction happens without explicit confirmation.

---

## Extraction Review

At session end (Scribe) or via `squad extract`:

```
📤 Learnings from this session:

⚠️  License: MIT (safe to extract)

✅ Suggest bringing home:
  - Skill: "Use Zod for API validation" (observed 3x)
  - Skill: "Cursor-based pagination pattern"
  - Decision: "Never use `any` in TypeScript"

❌ Project-specific (leaving here):
  - "Auth middleware lives at src/auth/guard.ts"
  - "Database uses Prisma with custom middleware"

Bring these learnings home to your personal squad? [Y/n/edit]
```

### License Handling

**Permissive licenses (MIT, Apache, BSD, ISC):** Proceed normally with extraction review.

**Copyleft licenses (GPL, AGPL, LGPL):** **Blocked by default.** Extraction refuses unless user explicitly opts in:

```
🚫 License: GPL-3.0 (copyleft)
   Extraction blocked. Patterns from copyleft projects may carry
   license obligations that affect your future work.
   
   See: https://squad.dev/docs/license-risk
   
   To proceed anyway: squad extract --accept-risks
```

**Risk:** If you extract patterns from a GPL-licensed project, your personal squad could "infect" future work on differently-licensed projects. Blocking by default ensures users make an informed, explicit choice.

**`--accept-risks` flag:** Acknowledges all extraction risks (license, data sensitivity, etc.) and allows extraction. Logged in consultation history for audit trail.

---

## Consultation Log

Track all consultations in `<globalSquadPath>/consultations/`:

**`<globalSquadPath>/consultations/kubernetes-dashboard.md`:**
```markdown
# kubernetes-dashboard

**Repository:** github.com/kubernetes/dashboard  
**First consulted:** 2026-02-27  
**Last session:** 2026-03-15  
**License:** Apache-2.0

## Extracted Learnings

### 2026-02-27
- Skill: "Zod validation pattern"
- Skill: "Cursor pagination"

### 2026-03-15
- Decision: "Use structured logging"
```

---

## Technical Design

### Config Schema

Consult mode extends the existing `SquadDirConfig` interface from `packages/squad-sdk/src/resolution.ts` (not the runtime `SquadConfig`). The `.squad/config.json` in consult mode must conform to the existing schema so that `resolveSquadPaths()` recognizes it:

```typescript
// Existing schema from resolution.ts:
interface SquadDirConfig {
  version: number;       // Required — must be 1
  teamRoot: string;      // Path to personal squad (resolved via resolveGlobalSquadPath())
  projectKey: string | null; // "consult" for consult mode
}

// New field added for consult mode:
interface ConsultDirConfig extends SquadDirConfig {
  consult: boolean;      // true = consult mode
}
```

### Detection

```typescript
function isConsultMode(config: SquadDirConfig & { consult?: boolean }): boolean {
  return config.consult === true;
}
```

### Invisibility Mechanism

`.git/info/exclude` is:
- Git-internal exclude file (same syntax as `.gitignore`)
- Lives in `.git/`, so never committed
- Project owner never sees it
- `git status` shows nothing

> **Important:** Do not hard-code `resolve(cwd, '.git/info/exclude')`. In git worktrees
> and submodules, `.git` is a *file* pointing at the real git dir. Use `git rev-parse`
> to resolve the correct path:

```bash
# Resolve the correct exclude path (works with worktrees/submodules)
EXCLUDE_PATH=$(git rev-parse --git-path info/exclude)
echo ".squad/" >> "$EXCLUDE_PATH"
```

### Existing Infrastructure

Building on `packages/squad-sdk/src/sharing/`:

| Module | Purpose | Consult Mode Usage |
|--------|---------|-------------------|
| `splitHistory()` | Separates shareable vs private | Propose generic learnings |
| `mergeHistory()` | Merge + deduplicate | Add to global squad |
| `export.ts` | Export to JSON | Consultation log format |
| `conflicts.ts` | Handle conflicts | Merge approved items |

---

## SDK Implementation

### New Functions

#### `packages/squad-sdk/src/sharing/consult.ts`

```typescript
import { AgentHistory, HistoryEntry } from '../types.js';
import { SquadDirConfig, resolveGlobalSquadPath } from '../resolution.js';
import { splitHistory, mergeHistory } from './history-split.js';
import { resolve } from 'path';

/**
 * Check if current config is in consult mode.
 */
export function isConsultMode(config: SquadDirConfig & { consult?: boolean }): boolean {
  return config.consult === true;
}

/**
 * Resolve the personal squad root using the existing platform-aware
 * resolver (XDG on Linux, AppData on Windows, etc.).
 */
export function getPersonalSquadRoot(): string {
  return resolve(resolveGlobalSquadPath(), '.squad');
}

/**
 * Classification result for a single learning.
 */
export interface ClassifiedLearning {
  entry: HistoryEntry;
  classification: 'generic' | 'project-specific';
  confidence: number;  // 0-1
  reason: string;
}

/**
 * Classification heuristics (before LLM pass).
 * Returns undefined if heuristic is inconclusive.
 */
function heuristicClassify(entry: HistoryEntry, projectRoot: string): 'generic' | 'project-specific' | undefined {
  const content = entry.content.toLowerCase();
  
  // Project-specific signals
  // Escape projectRoot before building a RegExp — paths can contain
  // regex metacharacters (e.g. dots, backslashes on Windows).
  const escapedRoot = projectRoot.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
  const projectSpecificPatterns = [
    /src\//,                           // File paths
    /\.\//,                       // Relative paths
    new RegExp(escapedRoot, 'i'),      // Project root in content
    /this (project|codebase|repo)/,   // Explicit project references
  ];
  
  for (const pattern of projectSpecificPatterns) {
    if (pattern.test(content)) {
      return 'project-specific';
    }
  }
  
  // Generic signals
  const genericPatterns = [
    /always use/i,
    /never use/i,
    /prefer \w+ over/i,
    /best practice/i,
    /pattern:/i,
  ];
  
  for (const pattern of genericPatterns) {
    if (pattern.test(content)) {
      return 'generic';
    }
  }
  
  return undefined; // Needs LLM classification
}

/**
 * Classify learnings as generic or project-specific.
 * Uses heuristics first, then LLM for ambiguous cases.
 */
export async function classifyLearnings(
  history: AgentHistory,
  projectRoot: string,
  options?: {
    llmClassifier?: (entries: HistoryEntry[]) => Promise<ClassifiedLearning[]>;
  }
): Promise<ClassifiedLearning[]> {
  const results: ClassifiedLearning[] = [];
  const needsLLM: HistoryEntry[] = [];
  
  // First pass: heuristics
  for (const entry of history.entries) {
    const heuristic = heuristicClassify(entry, projectRoot);
    if (heuristic) {
      results.push({
        entry,
        classification: heuristic,
        confidence: 0.8,
        reason: 'Heuristic classification',
      });
    } else {
      needsLLM.push(entry);
    }
  }
  
  // Second pass: LLM for ambiguous entries
  if (needsLLM.length > 0 && options?.llmClassifier) {
    const llmResults = await options.llmClassifier(needsLLM);
    results.push(...llmResults);
  } else {
    // Default to project-specific if no LLM available
    for (const entry of needsLLM) {
      results.push({
        entry,
        classification: 'project-specific',
        confidence: 0.5,
        reason: 'Default (no LLM classifier)',
      });
    }
  }
  
  return results;
}

/**
 * License detection result.
 */
export interface LicenseInfo {
  type: 'permissive' | 'copyleft' | 'unknown';
  spdxId?: string;
  name?: string;
  filePath?: string;
}

const COPYLEFT_LICENSES = [
  'GPL', 'AGPL', 'LGPL', 'MPL', 'EPL', 'CDDL', 'CC-BY-SA',
];

const PERMISSIVE_LICENSES = [
  'MIT', 'Apache', 'BSD', 'ISC', 'Unlicense', 'CC0', 'WTFPL',
];

/**
 * Escape a string so it can be safely used inside a RegExp pattern.
 */
function escapeRegex(value: string): string {
  return value.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

/**
 * Detect license from LICENSE file.
 */
export function detectLicense(licenseContent: string): LicenseInfo {
  const content = licenseContent;
  const upperContent = licenseContent.toUpperCase();

  // 1. Prefer SPDX identifiers when present.
  const spdxMatch = content.match(/SPDX-License-Identifier:\s*([^\s*]+)/i);
  if (spdxMatch) {
    const spdxId = spdxMatch[1];
    const spdxIdUpper = spdxId.toUpperCase();

    const copyleftUpper = COPYLEFT_LICENSES.map(id => id.toUpperCase());
    const permissiveUpper = PERMISSIVE_LICENSES.map(id => id.toUpperCase());

    if (copyleftUpper.includes(spdxIdUpper)) {
      return { type: 'copyleft', spdxId, name: spdxId };
    }
    if (permissiveUpper.includes(spdxIdUpper)) {
      return { type: 'permissive', spdxId, name: spdxId };
    }
    return { type: 'unknown', spdxId, name: spdxId };
  }

  // 2. Fallback: word-boundary regex, longest-first to avoid
  //    misclassifying e.g. "LGPL" as "GPL".
  const detectFromList = (
    licenses: readonly string[],
    type: LicenseInfo['type'],
  ): LicenseInfo | null => {
    const sorted = [...licenses].sort((a, b) => b.length - a.length);
    for (const license of sorted) {
      const pattern = new RegExp(
        `\\b${escapeRegex(license.toUpperCase())}\\b`, 'i',
      );
      if (pattern.test(upperContent)) {
        return { type, spdxId: license, name: license };
      }
    }
    return null;
  };

  const copyleftMatch = detectFromList(COPYLEFT_LICENSES, 'copyleft');
  if (copyleftMatch) return copyleftMatch;

  const permissiveMatch = detectFromList(PERMISSIVE_LICENSES, 'permissive');
  if (permissiveMatch) return permissiveMatch;

  return { type: 'unknown' };
}

/**
 * Extraction result.
 */
export interface ExtractionResult {
  extracted: ClassifiedLearning[];
  skipped: ClassifiedLearning[];
  license: LicenseInfo;
  projectName: string;
  timestamp: string;
}

/**
 * Merge extracted learnings into personal squad.
 */
export async function mergeToPersonalSquad(
  learnings: ClassifiedLearning[],
  personalSquadRoot: string,
): Promise<void> {
  // Filter to only generic learnings
  const genericLearnings = learnings.filter(l => l.classification === 'generic');
  
  // Group by type (decisions vs skills vs history)
  const decisions = genericLearnings.filter(l => l.entry.type === 'decision');
  const patterns = genericLearnings.filter(l => l.entry.type === 'pattern');
  
  // TODO: Implement actual merge logic
  // - Append decisions to <globalSquadPath>/decisions.md
  // - Create/update skills in <globalSquadPath>/skills/
  // - Update agent histories in <globalSquadPath>/agents/*/history.md
}

/**
 * Write consultation log entry.
 */
export async function logConsultation(
  personalSquadRoot: string,
  result: ExtractionResult,
): Promise<string> {
  const logPath = resolve(
    personalSquadRoot,
    'consultations',
    `${result.projectName}.md`
  );
  
  const entry = `
### ${result.timestamp}
- License: ${result.license.spdxId || result.license.type}
- Extracted: ${result.extracted.length} learnings
${result.extracted.map(l => `  - ${l.entry.type}: "${l.entry.content.slice(0, 50)}..."`).join('\n')}
`;
  
  // TODO: Append to existing log or create new
  return logPath;
}
```

#### `packages/squad-cli/src/cli/commands/consult.ts`

> **Note:** The current CLI does not use `commander`. Commands are wired as
> functions in `cli-entry.ts` via if-chains and dynamic imports (e.g. `runLink`,
> `doctorCommand`). The `consult` and `extract` commands should follow this
> existing pattern. The sketch below uses function exports compatible with
> `cli-entry.ts` integration.

```typescript
import { existsSync, appendFileSync, mkdirSync, writeFileSync, readFileSync } from 'fs';
import { resolve, basename } from 'path';
import { execSync } from 'child_process';
import { 
  isConsultMode, 
  getPersonalSquadRoot,
  detectLicense,
  classifyLearnings,
  mergeToPersonalSquad,
  logConsultation,
} from '@bradygaster/squad-sdk';

export async function runConsult(cwd: string, args: string[]): Promise<void> {
    const showStatus = args.includes('--status');
    const dryRun = args.includes('--check');
    const squadDir = resolve(cwd, '.squad');
    // Resolve exclude path via git rev-parse (handles worktrees/submodules)
    const gitExclude = execSync('git rev-parse --git-path info/exclude', { cwd, encoding: 'utf-8' }).trim();
    const personalSquad = getPersonalSquadRoot();
    
    // Check prerequisites
    if (!existsSync(resolve(cwd, '.git'))) {
      console.error('❌ Not a git repository.');
      process.exit(1);
    }
    
    if (!existsSync(personalSquad)) {
      console.error('❌ No personal squad found.');
      console.error('   Run `squad init --global` first.');
      process.exit(1);
    }
    
    if (showStatus) {
      // Show status
      if (existsSync(squadDir)) {
        const config = JSON.parse(
          readFileSync(resolve(squadDir, 'config.json'), 'utf-8')
        );
        if (config.consult) {
          console.log('✅ Consult mode active');
          console.log(`   Team root: ${config.teamRoot}`);
          // TODO: Show pending learnings count
        } else {
          console.log('ℹ️  Project has .squad/ but not in consult mode');
        }
      } else {
        console.log('ℹ️  Not in consult mode (no .squad/ directory)');
      }
      return;
    }
    
    // Check if project already has .squad/
    if (existsSync(squadDir)) {
      console.error('❌ This project already has a .squad/ directory.');
      console.error('   Cannot use consult mode on squadified projects.');
      console.error('');
      console.error('   Options:');
      console.error('   - Use their squad: work normally with project squad');
      console.error('   - Remove theirs: delete .squad/ and retry');
      process.exit(1);
    }
    
    if (dryRun) {
      console.log('📋 Dry-run: squad consult would:');
      console.log(`   1. Create ${squadDir}/config.json with consult: true`);
      console.log(`   2. Add .squad/ to ${gitExclude}`);
      console.log(`   3. Link to personal squad at ${personalSquad}`);
      return;
    }
    
    // Create consult mode .squad/
    mkdirSync(squadDir, { recursive: true });
    mkdirSync(resolve(squadDir, 'sessions'), { recursive: true });
    
    const config = {
      version: 1,
      teamRoot: personalSquad,
      projectKey: 'consult',
      consult: true,
    };
    
    writeFileSync(
      resolve(squadDir, 'config.json'),
      JSON.stringify(config, null, 2)
    );
    
    // Add to .git/info/exclude
    const excludeContent = existsSync(gitExclude) 
      ? readFileSync(gitExclude, 'utf-8') 
      : '';
    
    if (!excludeContent.includes('.squad/')) {
      appendFileSync(gitExclude, '\n# Squad consult mode\n.squad/\n');
    }
    
    console.log('✅ Consult mode activated');
    console.log(`   Team: ${personalSquad}`);
    console.log(`   Project: ${config.projectName}`);
    console.log('');
    console.log('   Your squad is now consulting on this project.');
    console.log('   Run `squad extract` when done to bring learnings home.');
}
```

#### `packages/squad-cli/src/cli/commands/extract.ts` (new)

```typescript
import { existsSync, readFileSync, rmSync } from 'fs';
import { resolve } from 'path';
import {
  isConsultMode,
  getPersonalSquadRoot,
  detectLicense,
  classifyLearnings,
  mergeToPersonalSquad,
  logConsultation,
} from '@bradygaster/squad-sdk';
import { confirm, checkbox } from '@inquirer/prompts';

export async function runExtract(cwd: string, args: string[]): Promise<void> {
    const dryRun = args.includes('--dry-run');
    const clean = args.includes('--clean');
    const yes = args.includes('--yes');
    const acceptRisks = args.includes('--accept-risks');
    const squadDir = resolve(cwd, '.squad');
    const configPath = resolve(squadDir, 'config.json');
    
    // Check we're in consult mode
    if (!existsSync(configPath)) {
      console.error('❌ No .squad/config.json found.');
      console.error('   Run `squad consult` first.');
      process.exit(1);
    }
    
    const config = JSON.parse(readFileSync(configPath, 'utf-8'));
    if (!config.consult) {
      console.error('❌ Not in consult mode.');
      console.error('   This command only works after `squad consult`.');
      process.exit(1);
    }
    
    // Check license
    const licensePath = resolve(cwd, 'LICENSE');
    let license = { type: 'unknown' as const };
    
    if (existsSync(licensePath)) {
      const licenseContent = readFileSync(licensePath, 'utf-8');
      license = detectLicense(licenseContent);
    }
    
    if (license.type === 'copyleft' && !acceptRisks) {
      console.error('🚫 License: ' + (license.spdxId || 'copyleft'));
      console.error('   Extraction blocked. Patterns from copyleft projects may carry');
      console.error('   license obligations that affect your future work.');
      console.error('');
      console.error('   See: https://squad.dev/docs/license-risk');
      console.error('');
      console.error('   To proceed anyway: squad extract --accept-risks');
      process.exit(1);
    }
    
    if (license.type === 'unknown') {
      console.warn('⚠️  No LICENSE file found.');
      console.warn('   Cannot determine license risk for extraction.');
      console.warn('   Proceed with caution.');
      console.warn('');
    }
    
    // TODO: Gather learnings from session history
    // TODO: Classify learnings
    // TODO: Present interactive review
    // TODO: Merge approved learnings
    // TODO: Log consultation
    
    if (dryRun) {
      console.log('📋 Dry-run: would extract learnings (not implemented yet)');
      return;
    }
    
    console.log('📤 Extraction complete (stub)');
    
    // Clean up if requested
    if (clean) {
      if (!yes) {
        const confirmed = await confirm({
          message: 'Delete project .squad/ directory?',
          default: false,
        });
        if (!confirmed) {
          console.log('Keeping .squad/ directory.');
          return;
        }
      }
      
      rmSync(squadDir, { recursive: true, force: true });
      console.log('🗑️  Deleted .squad/');
    }
}
```

### Type Changes

#### `packages/squad-sdk/src/resolution.ts` — extend `SquadDirConfig`

> **Note:** Consult mode extends `SquadDirConfig` (the `.squad/config.json` schema
> in `resolution.ts`), **not** the runtime `SquadConfig` in `runtime/config.ts`.
> The runtime config requires `version`, `models`, `routing`, etc. and is unrelated
> to the project `.squad/` directory pointer.

```typescript
export interface SquadDirConfig {
  version: number;
  teamRoot: string;
  projectKey: string | null;
  
  /** True when in consult mode */
  consult?: boolean;
}
```

#### `packages/squad-cli/src/cli-entry.ts` — wire new commands

```typescript
// In the command routing section of cli-entry.ts:
  case 'consult': {
    const { runConsult } = await import('./cli/commands/consult.js');
    await runConsult(process.cwd(), argv.slice(1));
    break;
  }
  case 'extract': {
    const { runExtract } = await import('./cli/commands/extract.js');
    await runExtract(process.cwd(), argv.slice(1));
    break;
  }
```

### Integration Points

| Component | Integration |
|-----------|-------------|
| **Scribe** | Add extraction review to session-end tasks when `isConsultMode()` |
| **Coordinator** | Pass `teamRoot` to agents when in consult mode |
| **Agents** | Read charters/skills from `teamRoot`, write sessions locally |
| **CLI** | Register `consult` and `extract` commands |

---

## Scribe Integration

Update session-end workflow:

```markdown
## Session-End Tasks

1. Orchestration log
2. Session log  
3. Decision inbox merge
4. Cross-agent updates
5. **Extraction review** ← NEW (if consult mode)
6. Git commit (local `.squad/`)
7. History summarization
```

---

## Edge Cases

### Project Already Has `.squad/`

**Current decision:** Error out

```
❌ This project already has a .squad/ directory.
   Cannot use consult mode on squadified projects.
   
   Options:
   - Use their squad: `squad init` (join their team)
   - Remove theirs: delete .squad/ and retry
```

**Future consideration:** `--force` flag or `.squad-consult/` namespace.

### No LICENSE File

```
⚠️  No LICENSE file found.
    Cannot determine license risk for extraction.
    Proceed with caution.
```

### User Declines All Extractions

No changes to global squad. Consultation still logged (with "No learnings extracted").

---

## Work Items

### Phase 1: Core Command

| Item | Description |
|------|-------------|
| `squad consult` command | Creates `.squad/` with `consult: true`, writes to `.git/info/exclude` |
| Config schema update | Add `consult` boolean field |
| Detection helper | `isConsultMode()` function |

### Phase 2: Extraction

| Item | Description |
|------|-------------|
| `squad extract` command | Manual extraction trigger |
| License check | Read LICENSE, warn on copyleft |
| Extraction review UI | Interactive prompt for approving learnings |
| Scribe integration | Auto-prompt at session end when `consult: true` |

### Phase 3: Tracking

| Item | Description |
|------|-------------|
| Consultation log | Write to `<globalSquadPath>/consultations/{project}.md` |
| `--clean` flag | Delete project `.squad/` after extraction |

---

## Success Criteria

1. **Invisible by default:** `git status` shows nothing in consult mode
2. **No pollution upstream:** Project `.squad/` never modifies global squad without explicit approval
3. **No pollution downstream:** Project-specific learnings stay in project or are discarded
4. **Audit trail:** `<globalSquadPath>/consultations/` tracks what was extracted from where
5. **License safety:** User warned about copyleft extraction risks

---

## Non-Goals (v1)

- Cross-machine sync of global squad (users can use git or `squad export/import`)
- Consulting on squadified projects (error out for now)
- Automatic extraction (always requires user approval)

---

## Open Questions

1. **Override for existing `.squad/`?** Allow `--force` or namespace (`.squad-consult/`)?
2. **License bypass?** Allow `--no-license-check` for power users?
3. **Extraction suggestions?** How smart should the "generic vs specific" classification be?

---

## References

- [Personal Squad Guide](../docs/guide/personal-squad.md)
- [SDK Sharing Module](../packages/squad-sdk/src/sharing/)
- [Export Command](../packages/squad-cli/src/cli/commands/export.ts)
