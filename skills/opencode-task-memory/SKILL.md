---
name: opencode-task-memory
description: "Use this skill to maintain operational memory across OpenCode sessions on the same project: average task durations, reliable providers, learned patterns, known antipatterns. Reads memory at task start to calibrate, writes at task end to accumulate."
---

# opencode-task-memory

## Nature and scope

Instructional skill that teaches Cowork to maintain a **persistent operational memory per project** in a local JSON file in the workspace. Read at task start to calibrate expectations (durations, reliable providers, antipatterns to avoid), written at task end to accumulate learning.

### Critical distinction (3 memory layers)

Do not mix these layers:

- **OPERATIONAL memory** (this skill) = technical stats per project, observed execution patterns
- **USER memory** (managed by native Cowork `consolidate-memory`) = preferences, work style, relationships
- **PROJECT context** (AGENTS.md, read by OpenCode) = code conventions, architecture spec

This skill covers only the first layer. Do not put user preferences or specs in it.

## Composability with user MCPs

If the user has an MCP exposing an ADR (Architecture Decision Records) system — typically a Relkhon-style vault with `manage_adr`, or a custom docs tool — that MCP should serve as the reference memory for the project's architectural decisions.

This skill only covers technical execution stats, not architectural decisions or domain learning.

On the first run on a project, if a user-side MCP with `manage_adr` or equivalent is detected, offer the user:
- (a) disable task-memory in favor of the user MCP (which already covers part of it)
- (b) chain both (task-memory for raw stats, user MCP for foundational decisions)
- (c) keep only task-memory

## File location

`<workspace>/.opencode-agent-memory.json`

Where `<workspace>` is the working repo folder (equivalent of the `directory` passed to OpenCode).

### First run: propose .gitignore

If the project has a `.git/`, offer to add `.opencode-agent-memory.json` to `.gitignore`:

> "I'll maintain a `.opencode-agent-memory.json` file at your project root to learn your patterns (durations, reliable providers, etc.). I recommend adding it to `.gitignore` to avoid accidental commits. Want me to do that?"

If yes, add the line to `.gitignore`. No hidden opt-out — the user knows you're writing to their repo.

### If the project is read-only

If writing fails (read-only repo, CI sandbox), operate in in-memory mode for the current session and warn the user:

> "I can't write `.opencode-agent-memory.json` (workspace is read-only). I'll keep the memory in-session, but it will be lost on next startup."

## JSON schema

```json
{
  "version": "1.0",
  "project": "/absolute/path/to/repo",
  "createdAt": "2026-05-15T14:00:00Z",
  "lastUpdated": "2026-05-15T15:30:00Z",
  "tasksTotal": 47,
  "byType": {
    "feature_implementation": {
      "count": 28,
      "avgDurationSec": 240,
      "successRate": 0.89,
      "preferredAgent": "build"
    },
    "bug_fix": {
      "count": 12,
      "avgDurationSec": 90,
      "successRate": 0.92,
      "preferredAgent": "build"
    },
    "refactor": {
      "count": 7,
      "avgDurationSec": 300,
      "successRate": 0.71,
      "preferredAgent": "build"
    },
    "exploration": {
      "count": 5,
      "avgDurationSec": 30,
      "successRate": 1.0,
      "preferredAgent": "plan"
    }
  },
  "providerStats": {
    "anthropic": { "calls": 35, "successRate": 0.94 },
    "openrouter": { "calls": 12, "successRate": 0.83 }
  },
  "patterns": [
    "Tasks on the factions module: ~5 min, requires cross-check with specs/combat.md",
    "Godot UI bugs: ~1-2 min, check _ready() and _process() first"
  ],
  "antipatterns": [
    "Cross-system refactor without explicit plan -> spec contradictions 3 out of 4 times",
    "Implementing without LightRAG validation -> bug surfaced in QA twice"
  ]
}
```

## When to read memory

At the start of each OpenCode task on a project:

1. Check if `<workspace>/.opencode-agent-memory.json` exists
2. If yes, parse the JSON and use it to:
   - **Calibrate `maxDurationSeconds`**: if the task resembles `feature_implementation` and `byType.feature_implementation.avgDurationSec` = 240, propose `maxDurationSeconds: 300` (with ~25% margin)
   - **Choose the initial provider**: if `providerStats.anthropic.successRate` > `providerStats.openrouter.successRate`, default to Anthropic
   - **Mention antipatterns in the ReAct**: "Note: on this project, [antipattern]. I'll [strategy to avoid it]"
   - **Reference patterns**: "Based on past sessions, [pattern]. I'm following this approach."
3. If not, start in neutral mode (orchestrator skill defaults)

## When to write memory

At the end of an OpenCode task (after `opencode_check` shows status `idle`/`finished`, or after an orphaned session recovery):

1. Determine the task type via heuristic on the initial prompt:
   - "implement", "add", "implemente", "ajoute" -> `feature_implementation`
   - "fix", "correct", "bug", "corrige" -> `bug_fix`
   - "refactor", "restructure", "improve", "ameliore" -> `refactor`
   - "explain", "find", "analyse", "explique", "trouve" -> `exploration`
   - Otherwise -> `other`
2. Update the stats:
   - Increment `tasksTotal` and `byType.<type>.count`
   - Recalculate `avgDurationSec` (running average: `newAvg = (oldAvg * (count-1) + thisDuration) / count`)
   - Update `successRate` (binary success/failure based on the result)
3. Update `providerStats` for the provider used
4. If a new lesson was learned (working pattern, observed antipattern), ADD to `patterns` or `antipatterns`
5. Update `lastUpdated`
6. Save the JSON

### Overflow management

- `patterns` and `antipatterns`: max 20 entries each, FIFO (eject oldest)
- The file should never exceed ~50KB. If it does, prune more aggressively (keep 10 + 10).

## User-facing format

### At task start (standard mode)

If you use memory to calibrate, mention it briefly:

> "Based on past sessions on this project (47 tasks), features take ~4 min on average and Anthropic is reliable at 94%. Dispatching with these parameters."

### At task end

Silent by default. Except when:
- You add a new pattern/antipattern: "I noted that [observation]. This enriches the project's operational memory."
- The user asks to see the memory: you can show the JSON or a summary.

### Dev mode override (view full memory)

If the user asks "show memory" or "task-memory status":

> "Operational memory for this project: 47 tasks total, 28 features (avg 4min, 89% success), 12 fixes (avg 1m30, 92%), 7 refactors (avg 5min, 71%). Anthropic provider at 94%. 3 learned patterns, 2 noted antipatterns."

## Inter-skill interactions

- **orchestrator**: called first, decides what to dispatch. Reads memory to calibrate maxDurationSeconds and choose provider.
- **safe-prompts** (v0.2.0): scans before dispatch, does not read memory.
- **result-validator** (v1.3 upcoming): validates after dispatch, writes to memory if validation fails (observed antipattern).

## Known limitations

- Memory is per PROJECT (repo path), not global. If you switch projects, memory doesn't follow.
- No cross-machine sync — the file lives in the local workspace.
- No versioning or rollback. If corrupted, delete to start fresh.
- No automatic purge of very old patterns (>6 months) in v1.2. To be evaluated in v1.x.
- Duration calculation in `avgDurationSec` may be skewed if you change model/provider between sessions — it's a raw average, not a rigorous comparison.

## Inspirations

- JSON schema: design doc §5.2 (our own design)
- "Operational vs user vs spec memory" distinction: native Cowork consolidate-memory audit (our Q1)
- Composability with user MCP (manage_adr style): design doc §13.5
- Persistent logging pattern: agent trajectories (§11.4 JSONL traces — task-memory is the summarized version, traces are the raw version not yet implemented)
