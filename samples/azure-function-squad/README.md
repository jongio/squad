# azure-function-squad

**Azure Function + Content Review Squad** — a Squad SDK sample that wires an HTTP-triggered Azure Function to a multi-agent content review pipeline.

## What It Demonstrates

| SDK Feature | What You'll See |
|---|---|
| `defineSquad()` | Top-level config composing team, agents, and routing |
| `defineTeam()` | Team metadata with project context |
| `defineAgent()` | Three specialist agents with capabilities and models |
| `defineRouting()` | Pattern-based routing with tiers and priorities |
| Azure Functions v4 | HTTP trigger → squad orchestration → JSON response |

## How It Works

1. An HTTP POST hits `/api/squad-prompt` with a `{ "prompt": "..." }` body
2. The function loads the squad config (3 review agents defined via SDK builders)
3. The review squad wakes up and each agent analyzes the content:
   - **Tone Reviewer** — audience fit, engagement, emotional resonance
   - **Technical Reviewer** — factual accuracy, code validation, link checking
   - **Copy Editor** — grammar, sentence structure, readability
4. Results are aggregated into a structured JSON response with per-agent scores and findings

## Prerequisites

- Node.js ≥ 20
- [Azure Functions Core Tools](https://learn.microsoft.com/azure/azure-functions/functions-run-tools?tabs=v4) (for `func start`)
- The SDK must be built first: `cd ../../ && npm run build`

## Run It

### With Azure Functions Core Tools

```bash
npm install
func start
```

Then send a request:

```bash
curl -X POST http://localhost:7071/api/squad-prompt \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Azure Functions now supports Node.js v20 with ES modules. This is a game-changer for TypeScript developers who want first-class module support without transpilation hacks!"}'
```

### Quick Validation (no Azure Functions runtime needed)

```bash
npm install
npm test
```

## Example Request

```bash
curl -X POST http://localhost:7071/api/squad-prompt \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Building multi-agent systems with the Squad SDK is straightforward. Define your agents with defineAgent(), compose them into a team with defineTeam(), and wire up routing with defineRouting(). The SDK validates everything at runtime — no schema files needed. Here is an example:\n\n```typescript\nconst reviewer = defineAgent({ name: \"reviewer\", role: \"Code Reviewer\" });\n```\n\nDeploy to Azure Functions and you have a serverless agent pipeline!"
  }'
```

## Example Response

```json
{
  "prompt": "Building multi-agent systems with the Squad SDK is straightforward...",
  "timestamp": "2026-03-06T10:30:00.000Z",
  "reviews": [
    {
      "agent": "tone-reviewer",
      "role": "Tone & Voice Analyst",
      "score": 7,
      "findings": [
        {
          "severity": "info",
          "message": "Code blocks detected. Ensure surrounding prose provides adequate context for non-technical readers."
        },
        {
          "severity": "suggestion",
          "message": "Tone is neutral/flat. Consider adding variety to maintain reader interest."
        }
      ],
      "summary": "Analyzed 42 words. Tone is informational, energetic."
    },
    {
      "agent": "technical-reviewer",
      "role": "Technical Accuracy Checker",
      "score": 9,
      "findings": [
        {
          "severity": "info",
          "message": "Code blocks present. Verify snippets compile and match described behavior."
        }
      ],
      "summary": "Technical review complete. Code blocks verified."
    },
    {
      "agent": "copy-editor",
      "role": "Copy Editor",
      "score": 8,
      "findings": [
        {
          "severity": "suggestion",
          "message": "Passive voice detected. Consider rewriting in active voice for clarity."
        }
      ],
      "summary": "Reviewed 6 sentences across 3 paragraph(s). Avg sentence length: 12 words."
    }
  ],
  "overallScore": 8,
  "consensus": "✅ Content is publication-ready with minor suggestions."
}
```

## Project Structure

| File | Purpose |
|---|---|
| `src/functions/squad-prompt.ts` | Azure Function HTTP trigger — entry point |
| `src/squad/config.ts` | Squad config using `defineSquad()`, `defineTeam()`, `defineAgent()`, `defineRouting()` |
| `src/squad/handlers.ts` | Agent review handlers (mock logic for demo) |
| `host.json` | Azure Functions host configuration |
| `local.settings.json` | Local development settings |
| `package.json` | Dependencies and scripts |
| `tsconfig.json` | TypeScript configuration |

## Extending This Sample

To make this production-ready, you'd:

1. Replace the mock handlers in `handlers.ts` with real Squad runtime calls using `SquadClient`
2. Add streaming responses via `StreamingPipeline` for long-running reviews
3. Wire up `CostTracker` to monitor token usage per review agent
4. Add `defineHooks()` for PII scrubbing on submitted content
5. Deploy to Azure with `func azure functionapp publish`
