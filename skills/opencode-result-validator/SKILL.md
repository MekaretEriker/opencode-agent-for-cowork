---
name: opencode-result-validator
description: "Use this skill after an OpenCode task with the build agent reports completion. Validates that the result is sound (diff coherent, tests pass, build OK, semantically aligned with intent) before declaring success to the user. Implements the Plan-Execute-Reflect pattern's third step."
---

# opencode-result-validator

## Nature and scope

Instructional skill that activates **after** an OpenCode session (typically `opencode_run` or the `opencode_check` phase following `opencode_fire`) reports "completed" on a modification task. Before declaring success to the user, you systematically validate that the result holds up.

This is the **3rd step of the Plan-Execute-Reflect pattern** (see design doc §13.1 and original §5.6). OpenCode's native `plan` agent handles the "Plan" for exploratory tasks. The `build` agent handles the "Execute". This skill handles the "Reflect".

**Without this skill**: Cowork could tell you "Done" while tests are red, the diff contains out-of-scope modifications, or the feature doesn't compile. False positive.

**With this skill**: you verify before handing back the result. If an issue is detected, you present it with "caveats" rather than a raw success.

## Composability with user MCPs

If the user has an MCP exposing a `validate` tool or equivalent (typically a Relkhon-style vault with semantic cross-spec validation, or a custom domain linter), that MCP is **more precise** for their project. This skill only covers generic checks.

On the first run on a project, if a user-side MCP with `validate` is detected, offer:

- (a) Replace plugin validation with a call to the MCP tool (substitution)
- (b) Chain both: plugin = generic checks (diff/tests), user MCP = domain checks (semantic)
- (c) Keep only the plugin

See design doc §6.2.2.

## When to activate

Always activate after:

- `opencode_run` or `opencode_check` showing `idle/finished` on a `build` agent task
- Recovery of an orphaned session (see orchestrator §2)
- Any task whose initial prompt was `feature_implementation`, `bug_fix`, or `refactor`

Do NOT activate after:

- `opencode_ask` (read-only)
- `plan` agent tasks (exploration / analysis, no files modified)
- Very short tasks (under 30s duration) — cost/benefit not favorable

Skippable on user request:

- If the user says "skip validation" or "no checks", skip for the current session
- Confirm once: "OK, I'm skipping validation checks for this conversation. You'll see OpenCode results directly."

## Systematic checks (in order)

### 1. Clean diff (always)

Call `opencode_review_changes` (or `opencode_session_diff`) to retrieve the full session diff.

Verify:

- Are the modified files within the scope announced in Plan?
- Are there unexpected modifications (sensitive config files, secrets, etc.)?
- Is the diff of the expected reasonable size?

If an anomaly is detected, annotate in the final summary.

### 2. Tests (if detectable)

Detect the project's test command via heuristic:

- Presence of `package.json` with `test` script -> `npm test`
- Presence of `pyproject.toml`/`setup.py` with pytest -> `pytest`
- Presence of `Cargo.toml` -> `cargo test`
- Presence of `build.gradle` -> `gradle test`
- Otherwise -> skip (no detectable tests)

Run via OpenCode a short command like:

```
opencode_ask(prompt: "Run the project tests once and tell me pass/fail with a summary. Don't fix anything, just report.")
```

Parse the response; if tests are red, note it in the summary.

### 3. Build / compilation (if relevant)

For TypeScript, Rust, C++ and other compiled languages:

- TS: `npm run build` or `tsc --noEmit`
- Rust: `cargo build`
- etc.

If the build breaks, this is a critical validation failure.

### 4. Semantic coherence (soft check via Claude)

Direct question to the sub-LLM (yourself): "Does the result truly answer the initial request? Is there a divergence between what was asked and what was done?"

This is qualitative. If you detect a divergence (e.g., "I asked to refactor the auth module but OpenCode also modified the payments module"), annotate it.

## Output format

### On full success

```
Task completed

[1-sentence summary of what was done]

Modified files:
- path/file1.ext (summary)
- path/file2.ext (summary)

Validation:
- OK Diff coherent with the request
- OK Tests: 47/47 pass
- OK Build: OK
- OK No semantic divergence

Want me to [natural next step proposal]?
```

### On caveats (one red check)

```
Task completed with caveats

[1-sentence summary]

Modified files:
- path/file1.ext (summary)

Validation:
- OK Diff coherent
- KO Tests: 2 failing (auth.test.ts:142, auth.test.ts:198)
- OK Build: OK
- OK Semantic: aligned

Suggestion: "Ask OpenCode to fix the failing tests"
```

### On critical failure (broken build)

```
Task completed with critical issue

[1-sentence summary]

Modified files:
- path/file1.ext (summary)

Validation:
- KO Build: compilation failure (5 TypeScript errors)
- (other checks skipped since build is broken)

The code doesn't compile as-is. I recommend:
- (1) Revert the changes (git restore)
- (2) Ask OpenCode to fix the errors

What do you want to do?
```

## Inter-skill interactions

- **orchestrator**: calls this skill at the end of a `build` task
- **safe-prompts**: no direct interaction
- **task-memory**: if validation fails, it's an *antipattern* to note in `antipatterns` ("Task X dispatched as Y failed validation Z")

## Known limitations

- Tests are run **in addition to** the main execution -> consumes tokens and time. On very short tasks, cost/benefit is unfavorable.
- If OpenCode itself failed to make the tests pass, this skill reports only, it does not fix.
- Semantic coherence is qualitative — Claude can miss a subtle divergence.

## Inspirations

- Plan-Execute-Reflect pattern: design doc §13.1 (canonical reference: Reflexion 2023, ReAct framework)
- Composability with user-side validate: §6.2.2
- swarm-code-plugin has a conceptually similar `opencode-result-handling` skill (but on Claude Code side, not Cowork)
