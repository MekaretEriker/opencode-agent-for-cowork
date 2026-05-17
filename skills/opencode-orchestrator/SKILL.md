---
name: opencode-orchestrator
description: "Use this skill when delegating coding tasks to OpenCode from Cowork. Triggers on phrases like 'demande a OpenCode', 'lance OpenCode sur', 'build/refactor/debug/test ... in my project'."
---

# opencode-orchestrator

## General framework

This skill teaches Cowork (you, Claude) to drive OpenCode as an intelligent code worker. You don't write the code yourself here — you decide WHEN to delegate, WHAT to delegate, HOW to formulate the request, and you interpret the result for the user.

When this skill triggers:
- The user wants to write/modify/analyze code in a repo
- Typical phrases: "demande a OpenCode" / "ask OpenCode", "lance OpenCode sur" / "run OpenCode on", "refactor module X", "implement feature Y", "explain how Z works"
- Any substantial code task (more than a few lines) that benefits from a dedicated agent

When NOT to use it:
- Conceptual question without code to produce (Claude answers directly)
- Minimal file modification you can do yourself with Edit (fewer than 5 lines)
- The user explicitly requested "do it yourself, don't use OpenCode"

Philosophy: you are the orchestrator, OpenCode is the executor. You carry the conversation with the user, OpenCode carries the analytical load and code writing. You validate, synthesize, present.

---

## 1. ReAct - explicit reasoning before dispatch

The ReAct pattern (Reasoning + Acting) requires you to explain your reasoning before acting. For this skill: before dispatching a task to OpenCode, briefly announce to the user:

1. Detected task type: refactor / debug / explore / feature / multi-module / etc.
2. Chosen OpenCode agent and reason (build / plan / cowork-with-github)
3. Execution mode: synchronous (_run) / background (_fire) / parallel (multiple _fire)
4. Duration estimate if relevant (based on experience / defaults)

Why: canonical reference "Building Effective Agents" (Anthropic, late 2024) — making your behavior explicit makes it predictable. The user can stop you before you go in the wrong direction.

Never dispatch silently. Always show a mini plan block.

Standard mode template:

I'm going to ask OpenCode to [short task summary]. This is a [task type], so I'm using the [build/plan] agent and [syncing/launching in background]. Estimate: [X min].

[Dispatch]

Dev mode template:

Detection:
- Task type: multi-file refactor
- Keywords: "refactor", "auth module", "use JWT"
- Chosen agent: build (modifies code) - anthropic provider by default
- Tool: opencode_run (duration 1-15 min)
- maxDurationSeconds: 600 (medium refactor)
- No parallelization (single target module)

[Dispatch]

## 2. Choosing the OpenCode tool (ask / run / fire)

opencode-mcp exposes 79 tools, but for 90% of tasks you use these 3 workflow tools.

### IMPORTANT - MCP timeout constraint

The MCP transport between Cowork and opencode-mcp has an internal timeout of approximately 180 seconds (3 minutes) that overrides the maxDurationSeconds parameter you pass. This means an opencode_run lasting more than 3 minutes risks disconnecting on the Cowork side even if OpenCode continues running on the server side (creating an orphaned session). Adjust your thresholds accordingly (see "Orphaned session recovery" subsection below).

### opencode_ask - one-shot question

When to use:
- Short question (estimated under 30 seconds)
- No file writing expected
- No session continuity needed

Example: "Which modules import X?" → opencode_ask.

### opencode_run - synchronous task with wait

When to use:
- Real modification task (short refactor, bug fix, content addition)
- Estimated duration LESS THAN 2-3 MIN (beyond that, switch to opencode_fire to avoid MCP transport timeout — see above)
- You can afford to wait within the conversation turn

maxDurationSeconds parameter to adjust:
- 120 (2 min) for simple fix — sweet spot
- 180 (3 min) synchronous upper limit, beyond that switch to fire
- More than 180: DO NOT use opencode_run, use opencode_fire (see below)

Example: "Fix the authentication bug in login.ts" → opencode_run with maxDurationSeconds 180.

### opencode_run_streaming - SSE-driven synchronous task (new in v1.1)

Available since opencode-mcp 1.11.0 (MEK-283). Functionally equivalent to `opencode_run` but consumes the OpenCode server's `text/event-stream` instead of polling `/session/{id}` every 3s. Two advantages over `opencode_run`:

- **One persistent connection** instead of N roundtrips — measurably lighter on the MCP transport, especially for tasks in the 60-180s range.
- **Live progress** in Cowork if it forwards a `progressToken` in the call params (per MCP progress spec). Each intermediate event (`tool.start`, `tool.end`, `message.update`) is forwarded as `notifications/progress`. If no `progressToken` is provided, the tool behaves like a silent `opencode_run` — fully backward compatible.

When to prefer `opencode_run_streaming` over `opencode_run`:

- The user is actively waiting and would benefit from seeing progress
- Task estimated 60-180s (long enough for progress to matter, short enough to stay below the MCP transport timeout)
- You want a structured `SESSION_HANG` error code on timeout instead of a generic "Request timed out"

When `opencode_run` is still fine:

- Very short task (under 30-60s) where progress overhead isn't worth it
- The client (e.g. an automated dispatch) doesn't display progress notifications

`opencode_run_streaming` is still subject to the same ~180s MCP transport timeout as `opencode_run`. For tasks over 3 min, use `opencode_fire` regardless.

Example: "Refactor login.ts to use the new auth module" (estimate 90s, user waiting) -> `opencode_run_streaming` with maxDurationSeconds 180.

### opencode_fire - fire-and-forget (background) - PREFERRED DEFAULT for 3+ min

When to use:
- Task estimated over 2-3 min (preferred default from 3 min, NOT only for 15+ min as in v1.0)
- Parallelizable tasks (multiple modules to process simultaneously — see section 5)
- The user wants to continue the conversation while OpenCode works
- Any case where you want to avoid blocking on the MCP transport timeout

Recommended pattern:
1. opencode_fire dispatches the task, returns sessionId immediately
2. You respond to the user "OK, launching in background, I'll update you next turn"
3. On the next conversation turn (or when the user comes back to ask), you call opencode_check(sessionId) for status
4. When status is "idle" / "finished", you call opencode_conversation or opencode_review_changes to retrieve the result

Monitoring: opencode_check (compact status), opencode_wait (blocking wait — subject to the same MCP timeout, use in 120-180s chunks max), opencode_sessions_overview (global view).

Example: "Refactor the factions module to use EventBus" → opencode_fire (estimate 5 min) → check + reply next turn.

### Orphaned session recovery (MCP timeout)

If you launched opencode_run or opencode_run_streaming and receive an MCP timeout error (e.g., "Request timed out after Xs" or "MCP server disconnected"), DO NOT ASSUME the task failed on OpenCode's side. The OpenCode server often keeps running.

Note on structured errors: as of opencode-mcp 1.11.0, `opencode_run_streaming` returns a structured `SESSION_HANG` code (parseable from the `<!-- structured-error -->` JSON block in the error message) when its own SSE timeout fires without seeing `session.idle`. This is distinct from the Cowork-side MCP transport timeout described above. If you see `SESSION_HANG`, the session is still alive on the server but failed to emit `session.idle` — the recovery procedure below applies regardless.

Procedure:

1. Call opencode_sessions_overview to list active sessions
2. Identify the one corresponding to your task (by title or recency)
3. Run opencode_check on that session to see its state:
   - busy: still running, switch to fire+check mode (announce to user "OpenCode is still running, I'll check next turn")
   - idle: task is done, retrieve result via opencode_conversation or opencode_review_changes
   - error: task truly failed, present the error to the user

Goal: never re-launch a task that is already running on the server side (token waste) and never lose a result because the MCP transport was cut.

Announce to user (ReAct): "The MCP tool has an internal timeout of ~180s. The task is probably still running on OpenCode's side. I'm checking session state and switching to fire-and-forget + check mode if needed."

### Recap table (v1.0.1)

| Situation | Tool | Reason |
|---|---|---|
| Question under 30s | opencode_ask | No session to maintain |
| Task under 2-3 min, synchronous | opencode_run with maxDur 120-180 | Wait included, below MCP timeout threshold (180s) |
| Task under 2-3 min, user actively waiting | opencode_run_streaming with maxDur 120-180 | Same as _run but emits live progress + structured SESSION_HANG on timeout |
| Task over 3 min OR parallel | opencode_fire + check next turn | Avoids MCP timeout, preferred default |
| MCP timeout received on opencode_run | sessions_overview + check (recover orphan) | Don't re-launch a running task |
| Follow-up on a _run task | opencode_reply | Continues the existing session |
| Follow-up on a _fire task | opencode_check for status, opencode_reply when user returns | Same |

---

## 3. Choosing the OpenCode agent (build / plan / @general)

OpenCode (see anomalyco/opencode) natively includes two main agents + one sub-agent.

### Native agents

build - full-access agent (default)
- Read/write/bash without confirmation
- For any real code modification task
- This is the right default for the vast majority of tasks

plan - read-only / exploration agent
- Does not modify files by default
- Asks confirmation before running bash
- For: "explain", "review", "find", "analyse", "where is X defined"
- Safer for unfamiliar domains

### Sub-agent

@general - for complex multi-step searches
- Invoked via mention in the prompt (not via the agent: param)
- For: "find all usages of X across the 5 services", "trace this call path"
- Useful combined with build or plan

### Custom agents (delivered by the plugin)

In MVP: none. Starting from v1.5, the plugin will deliver cowork-with-github (build extension with GitHub MCP). For now, if a task requires GitHub, fall back to build and inform the user that the PR will be manual.

### Routing heuristic

| Prompt type | Agent | Indicative keywords |
|---|---|---|
| Exploration / review / question | plan | "explain", "review", "how does", "find", "where is", "analyse", "explique", "trouve" |
| Modification / refactor / fix | build | "implement", "add", "fix", "refactor", "modify", "write", "implemente", "ajoute" |
| Complex multi-file search | (build or plan) + @general in the prompt | "find all", "trace", "cross-service", "trouve tous les" |
| GitHub / PR / issues | build (MVP) or cowork-with-github (v1.5+) | "PR", "pull request", "issue", "GitHub" |

### Concrete examples

| User prompt | Agent + tool |
|---|---|
| "Explique-moi le flow d'authentification" / "Explain the authentication flow" | plan + opencode_ask |
| "Implemente la pagination sur GET /users" / "Implement pagination on GET /users" | build + opencode_run (maxDur 300) |
| "Trouve tous les endroits ou on appelle decryptToken et liste-les" / "Find all places where decryptToken is called and list them" | plan + opencode_ask with @general in the prompt |
| "Refactor le module factions et cree une PR" / "Refactor the factions module and create a PR" | build (MVP) + opencode_run (maxDur 600), with note "manual PR since cowork-with-github not yet delivered" |

### Fallback if requested agent doesn't exist

Before any dispatch, you can verify the list of available agents via opencode_agent_list. If the target agent (e.g., cowork-with-github) is not available locally, fall back to build and inform the user.

## 4. Stall heuristic during active monitoring

The stall detection pattern distinguishes an intelligent orchestrator from a naive poller. When you're driving an OpenCode session via opencode_run or opencode_wait, you monitor progress between polls. If OpenCode is spinning without progress, you exit cleanly instead of waiting for the timeout.

### Progress indicators (from opencode_check)

- filesChanged: number of files modified since session start
- todosCompleted: number of completed todos
- tokenUsage: can indicate activity even without output (less reliable)

### Heuristic

1. Between 2 consecutive calls to opencode_check, compare these counters to the previous turn
2. If NO counter has increased over 2 consecutive polls (typically 30-60s apart), stall detected
3. Action:
   - opencode_session_summarize with id, providerID, modelID to capture a summary of work done
   - opencode_session_abort with id to exit cleanly
   - Present the summary to the user + the option "want me to restart with a more precise prompt?"

### Pseudo-code

```
files_prev = 0
todos_prev = 0
stall_count = 0

while session_busy:
  status = opencode_check(sessionId)
  if status.filesChanged > files_prev OR status.todosCompleted > todos_prev:
    files_prev = status.filesChanged
    todos_prev = status.todosCompleted
    stall_count = 0
  else:
    stall_count += 1
    if stall_count >= 2:
      summary = opencode_session_summarize(sessionId, ...)
      opencode_session_abort(sessionId)
      return present_to_user(summary, "stall detected")
  wait 30-60s
```

### Known limitation

If the user moves to another topic while OpenCode is running, this skill does not re-poll on its own in the background. Stall detection only works within the active conversation turn (see design doc R5).

### Inspiration: activity-based timeout (Hermes pattern)

This heuristic is aligned with the activity-based timeout approach described in hermes-agent (NousResearch): instead of killing a task after N wall-clock seconds, we track the timestamp of the last activity signal (change in filesChanged / todosCompleted / new message) and only trigger the stall if NO activity has occurred since the threshold. This distinguishes "complex task that is working" from "task stuck on a dead call". See hermes-agent issue #4815 and PR #4864 for canonical context.

### When NOT to apply

- opencode_fire tasks that you are not actively supervising
- Very short tasks (under 1 min timeout) where the natural timeout suffices

## 5. Parallelization policy (max 3 concurrent)

Cowork can dispatch multiple OpenCode sessions in parallel via opencode_fire, but without limits this consumes RAM, provider quota, and can create file conflicts on the same repo.

### Default policy

- Max 3 concurrent sessions on the same project (same directory)
- Beyond that: serialize (launch the first 3, wait for one to finish before launching the 4th)
- No intra-skill limit between distinct projects — OpenCode and the LLM provider manage their own rate limiting

### User override

Environment variable OPENCODE_AGENT_MAX_PARALLEL (read at startup via opencode_setup or opencode_context). Reasonable values:
- 1 = strictly sequential (modest machines or tight rate limit)
- 3 = default
- 5+ = for powerful machines with tolerant providers

### Proactive detection of file conflicts

When the user requests 2+ tasks on the same repo, detect whether they might touch the same files before parallelizing:

| Case | Strategy |
|---|---|
| Tasks on distinct modules/folders | Parallel (up to 3) |
| Tasks on the same module | Serial |
| Global-scope tasks (architecture, full lint) | Serial |
| Doubt | Serial + ask user |

### Concrete examples

| User request | Strategy |
|---|---|
| "refactor the auth module AND add tests to the auth module" | Same files likely → serial |
| "refactor the auth module AND refactor the payments module" | Distinct files → parallel OK |
| "refactor the entire codebase" | Broad scope → serial |

### Parallel dispatch example

```
fire(prompt: "cd /repo && refactor the auth module to use JWT") -> sessionId A
fire(prompt: "cd /repo && add Zod validation to payments endpoints") -> sessionId B
fire(prompt: "cd /repo && document the public endpoints") -> sessionId C

# Later, user asks for status
sessions_overview() -> state of the 3
check(A); check(B); check(C)
```

### When the user requests more than 3

"You want 5 refactors in parallel. By default I cap at 3 concurrent to avoid saturating your LLM provider. I'm launching the first 3 now and the next 2 as soon as a slot frees up. Confirm, or do you want me to override OPENCODE_AGENT_MAX_PARALLEL?"

## 6. Assisted OpenCode install on first run

On the first run of the skill on a user's machine, OpenCode may not be installed. The skill detects this and proposes installation instead of failing opaquely.

### Detection

1. At the start of each orchestrator session, call opencode_setup
2. If the tool reports "opencode binary not found" or connection error, we're in the absent case
3. If OK, continue normally (no re-detection on each task)

### OS detection

Several methods (use whichever works first):
- Cowork knows its host OS (Windows/macOS/Linux/WSL)
- Via bash sandbox: uname -a, contents of /etc/os-release
- Last resort: ask the user

### Install commands by platform

Windows (PowerShell, primary for Mekaret):
```
irm https://opencode.ai/install.ps1 | iex
```
Alternatives: scoop install opencode (if scoop installed), choco install opencode (if choco)

macOS:
```
brew install anomalyco/tap/opencode
```
Alternative: curl -fsSL https://opencode.ai/install | bash

Linux:
```
curl -fsSL https://opencode.ai/install | bash
```
Alternatives: npm i -g opencode-ai (if Node.js available), nix run nixpkgs#opencode (NixOS), sudo pacman -S opencode (Arch)

WSL (Linux in Windows): same as Linux above.

### UX flow (standard mode)

You tell the user:

"OpenCode is not installed on your machine. Want me to install it? The command will be:

[command adapted to your OS]

Confirm with 'yes' and I'll run it, or decline to stay without OpenCode."

If the user confirms:
1. Run the command via Cowork bash sandbox
2. Wait for the download to complete (typically 30s-1min)
3. Re-test opencode_setup to confirm
4. If OK: "OpenCode installed! I can start on your task."
5. If failure: present the error and propose manual install or alternative method

### Explicit decisions

- No silent install — user validation required
- No re-detection on each session — once installed, assume it's there
- No automatic updates — if the user wants to update, they do it manually

## 7. Reading target repo's AGENTS.md

OpenCode (see anomalyco/opencode) natively reads an AGENTS.md file at the root of the target repo to inject project context into all its sessions. It's the OpenCode equivalent of CLAUDE.md on the Cowork side.

### At the start of each OpenCode task on a new project

1. Check if AGENTS.md exists at the root of the target repo (test via opencode_file_read or a cat in the prompt)
2. If present:
   - Read the content
   - Summarize in 2-3 lines to the user: "This project has an AGENTS.md documenting: X, Y, Z. I'll take it into account."
   - No need to manually include it in the OpenCode prompt — OpenCode already reads it natively
3. If absent: nothing special, just continue

### What to do if the user wants to create an AGENTS.md (in MVP)

The plugin does NOT automatically create AGENTS.md in MVP. Writing to the target repo comes in v1.5 with explicit opt-in (see design doc Q9).

If the user asks "can you create an AGENTS.md for this project?":
- Propose creation as a normal task via OpenCode (build agent, opencode_run)
- The user validates the content before the commit
- No automatic HTML section tags in MVP — that's manual

### MVP limitations

- Read-only: no automatic writing
- No auto-bootstrap on new projects
- No sync from task-memory (not yet in MVP, coming in v1.2)
- All of this arrives in v1.5 with explicit per-project opt-in

## 8. Result format (standard mode vs dev mode)

Cowork serves two audiences (see design doc Q3): general public users AND developers. You adapt your output format based on context. Approach: progressive disclosure.

### Standard mode (default)

Natural language, synthetic summary. No technical jargon unless the user used it first.

Response template after a successful task:

Done. I [1-sentence summary of what was done].

Modified files:
- relative/path/file1.ext (1-line summary)
- relative/path/file2.ext (1-line summary)

[If relevant: tests pass / build OK / other validation]

Want me to [natural next step proposal]?

Template after failure or truncated task (stall):

Task partially completed.

Here's where we stand:
- [What was done]
- [What remains]

Reason: [short summary, no technical details]

Shall we continue, retry differently, or do you want to see the details?

### Dev mode (activated on demand)

The user activates dev mode by explicitly requesting: "show details", "dev mode", "explain technically", "give me the session ID", etc. Once activated, you include:
- The OpenCode sessionId (for drill-down via opencode_conversation)
- The agent used (build / plan / custom)
- The effective provider and model (if different from the default)
- Elapsed time and number of iterations
- OpenCode tools called internally (summary)

Dev mode template:

Task completed - technical details

Session: ses_xxxxx (agent: build, provider: anthropic, model: claude-sonnet-4-6)
Duration: 4min12s (8 OpenCode iterations)
Tools invoked: opencode_find_text (3x), opencode_file_read (12x), opencode_message_send (2x)

Diff summary:
- auth.ts: +47 lines, -23 lines
- middleware.ts: +12 lines, -4 lines
- types.ts: +8 lines

For drill-down:
- opencode_conversation with sessionId ses_xxxxx for full history
- opencode_review_changes with sessionId ses_xxxxx for raw diff

You don't impose dev mode — it's progressive disclosure. You start in standard mode, the user opens the technical drawer if they want.

---

## Technical notes

### Directory parameter — resolved via @mekareteriker/opencode-mcp fork

The `directory` parameter works natively on Windows clients as of `@mekareteriker/opencode-mcp >= 1.10.2-mekareteriker.0` (the hardened fork bundled with this plugin). Pass absolute paths like `D:\Projects\myproject` or `/mnt/d/Projects/myproject` directly — both forms are accepted.

Previous workaround (no longer needed): on upstream `opencode-mcp <= 1.10.1`, the validator required the resolved path to start with `"/"`, rejecting Windows absolute paths with "is not an absolute path". The fork switched to platform-aware `path.isAbsolute` (upstream commit `e8e6cfe`, never released on the upstream npm). If you are forced to run on the upstream package (e.g. via a custom MCP config), prepend the prompt with `cd /path && ` as a fallback.

Reference: `D:\Projects\opencode-mcp\SPEC-fork.md` for the full audit trail and the patch plan.
