---
name: opencode-agent-roster
description: "Use this skill when dispatching to OpenCode and needing to choose between native agents (build, plan, @general) or the plugin's custom agent (cowork-with-github for PR/issue tasks). Also handles opt-in writing to AGENTS.md in the target repo."
---

# opencode-agent-roster

## Nature and scope

This skill extends orchestrator §3 (agent selection) with three new capabilities delivered in v0.6.0:

1. A custom agent `cowork-with-github`, an extension of `build` augmented with the GitHub MCP, delivered by the plugin
2. An enriched routing heuristic that includes this agent in the decision matrix
3. An opt-in mechanism for writing to the target repo's `AGENTS.md`, conforming to design doc decision Q9 §6.5

The final roster now covers four agents with complementary profiles, eliminating grey areas where agent selection was ambiguous (GitHub tasks with no clear routing precedent).

## Final roster (after v0.6.0)

| Agent | Origin | When to use |
|---|---|---|
| build | native OpenCode | Modification task (refactor, feature, fix) — default for any task with side effects |
| plan | native OpenCode | Exploration, review, analysis — read-only, zero side effects |
| @general | native OpenCode sub-agent | Complex multi-step search requiring several rounds of discovery |
| cowork-with-github | DELIVERED by plugin (v0.6.0) | Tasks involving PR, issues, branches, or any GitHub API interaction |

Each agent has a distinct tool profile and behavior. Routing must choose the most specialized for the current task, with explicit fallback if prerequisites (MCP, installed agent) are not met.

## Updated selection heuristic

The routing heuristic (orchestrator §3) is extended with a new test at the head of the chain:

1. **GitHub detection**: if the prompt contains "PR", "pull request", "issue", "#\d+", "GitHub", "branch", "branche", "merge" → route to `cowork-with-github`
   - Check for GitHub MCP presence on OpenCode side via `opencode_mcp_status`
   - If GitHub MCP absent: fallback to `build` with explicit note "manual PR needed afterwards"
   - If `cowork-with-github` agent is not yet installed on OpenCode side: propose opt-in installation (see Installation section)
2. **Read-only detection**: if the task is purely exploratory without code modification → `plan`
3. **Complexity detection**: if the task requires multiple rounds of research before action → `@general`
4. **Default**: all other tasks → `build`

This chain is evaluated sequentially. The first true condition wins. Conditions 2–4 are unchanged from orchestrator v0.5.0; only condition 1 is new.

### Concrete routing examples

| User prompt | Routed agent | Justification |
|---|---|---|
| "Refactor le module factions pour utiliser EventBus" / "Refactor the factions module to use EventBus" | build | Modification, no GitHub keyword |
| "Analyse les dependances circulaires dans src/" / "Analyze circular dependencies in src/" | plan | Read-only, exploration |
| "Trouve tous les appels a deprecatedMethod et propose un plan de migration" / "Find all calls to deprecatedMethod and propose a migration plan" | @general | Multi-step research before action |
| "Cree une PR pour le fix du bug #342" / "Create a PR for the bug fix #342" | cowork-with-github | PR + issue keywords |
| "Ouvre une issue pour tracker le refactor du parser" / "Open an issue to track the parser refactor" | cowork-with-github | Issue + GitHub keyword |
| "Push la branche feat/auth et ouvre une PR" / "Push the feat/auth branch and open a PR" | cowork-with-github | Branch + PR keywords |

## cowork-with-github agent

### Definition

The agent is defined in `agent-templates/cowork-with-github.json` (template to install on the OpenCode side via `opencode_agent_list` + native OpenCode configuration). Technical characteristics:

- **Base**: native OpenCode `build` agent
- **Required MCPs**: GitHub MCP (typically `@modelcontextprotocol/server-github`)
- **Additional system prompt**: the agent prefers creating a dedicated branch rather than working on main/master, opens a PR instead of pushing directly, and links the issue if a reference is detected in the prompt

This is not a completely separate agent — it is a specialization of `build` with an enriched GitHub context. This minimalist approach avoids duplicating code modification logic (already in `build`) while adding GitHub workflow-specific behavior.

### Specific behavior

When the `cowork-with-github` agent is activated, it adopts the following default behaviors (encoded in the system prompt):

1. **Dedicated branch**: creates a branch named after the task (e.g., `feat/auth-jwt`, `fix/issue-342-null-pointer`, `refactor/eventbus-migration`). Never works directly on `main` or `master`.
2. **Systematic PR**: opens a pull request with a clear title and description, rather than pushing directly to the main branch.
3. **Issue link**: if the prompt contains a reference to an issue number (`#123`), the PR includes `Closes #123` in its description.
4. **Structured description**: the PR includes a summary of changes, the list of modified files, and tests passed.

These rules are preferences, not hard constraints. If the user explicitly requests a direct push, the agent complies.

### Fallback if GitHub MCP absent

If the GitHub MCP is not configured on the OpenCode side, the `cowork-with-github` agent cannot function. In that case:
- Routing falls back to `build`
- A note is added to the task report: "Manual PR needed afterwards — GitHub MCP is not configured on OpenCode side"
- The user can configure the GitHub MCP and retry

## First-run installation on a project (opt-in)

Installation of the `cowork-with-github` agent on the OpenCode side follows a strict opt-in protocol:

### Detection

On the first run per project where a `cowork-with-github`-eligible task is detected, the skill checks whether the agent already exists on the OpenCode side via `opencode_agent_list`. If the agent is absent:

### Proposal

The skill offers the user:
> "This task would benefit from the custom `cowork-with-github` agent. Want me to install it on OpenCode? It extends `build` with the GitHub MCP to handle PRs and issues automatically."

### Installation

If the user accepts:
1. Write the agent file via OpenCode configuration (native OpenCode format)
2. Add the GitHub MCP (`@modelcontextprotocol/server-github`) to OpenCode configuration if absent
3. Verify the agent appears in `opencode_agent_list`

### Refusal

If the user refuses: fallback to `build` with the standard note "manual PR needed afterwards". No re-prompting on subsequent tasks for the same project — refusal is definitive for this project. The user can activate manually later with: "install the cowork-with-github agent".

## AGENTS.md writing (decision Q9)

### General principle

On the first run on a repo, in addition to the mandatory reading of `AGENTS.md` (orchestrator §7), the skill proposes **explicit opt-in** writing of an automatically maintained section in this file. This section feeds the project's operational memory beyond the current session.

### Opt-in protocol

1. **Single proposal**: "Do you want me to maintain a section in the target repo's `AGENTS.md` with cross-session learnings (observed durations, successful patterns, antipatterns)?"
2. **If OK**:
   - If `AGENTS.md` is absent from the repo, offer to create it (with the standard OpenCode header)
   - If `AGENTS.md` already exists, add a section delimited by HTML tags: `<!-- Maintained by Cowork OpenCode Agent plugin -->` ... `<!-- end Cowork section -->`
   - Store the opt-in in `.opencode-agent-memory.json` with the field `agentsMdWriteEnabled: true`
3. **After each successful task**: update the section with an aggregated summary of learnings. Content must be readable and relevant — not a raw stats dump. Aggregate data from `task-memory` into trends.
4. **Never auto-commit**: `AGENTS.md` modifications remain unstaged. It's up to the user to commit them when they see fit.
5. **Never overwrite**: the skill only touches the delimited section. All content outside the tags is preserved as-is.
6. **Definitive refusal**: if the user refuses on the first run, never propose again. They can activate later via the command "enable AGENTS.md write".

### Format of the managed AGENTS.md section

```markdown
<!-- Maintained by Cowork OpenCode Agent plugin -->
## Observed patterns (auto-maintained)

- Tests typically take ~120s on this repo (sample: 12 runs)
- Refactor tasks: ~5min average (sample: 7 runs)
- Anthropic provider reliable (94% success), OpenAI as fallback
- Pattern: prefer EventBus over direct module imports for cross-system communication

## Known antipatterns

- Cross-system refactors without explicit plan -> spec contradictions (3 cases)
- Implementing without LightRAG validation -> bugs found in QA (2 cases)

<!-- end Cowork section -->
```

### Update rules

- The `Observed patterns` field is updated after each successful task. Numeric stats use a rolling average (no reset).
- The `Known antipatterns` field is updated when `result-validator` detects a repeated failure (same pattern on >= 2 tasks).
- Entries are in English for readability by LLMs that will read the file.
- If the section grows beyond 20 lines, the oldest entries are removed (FIFO). No more aggressive automatic purge — it's up to the user to clean up if desired.

## Inter-skill

This skill is called by other plugin skills and interacts with them in a well-defined way:

| Calling skill | Interaction point |
|---|---|
| **orchestrator** | Calls this skill for agent routing (choice between build/plan/@general/cowork-with-github) and to decide if a task warrants using `cowork-with-github` |
| **task-memory** | Feeds the content of the managed `AGENTS.md` section via its operational metrics (durations, providers, patterns) |
| **safe-prompts** | No direct interaction — security scan happens upstream of routing |
| **result-validator** | Feeds antipatterns via its failure verdicts (repeated pattern = recorded antipattern) |
| **fallback-chain** | No direct interaction — provider fallback is transparent to agent routing |

## Limitations

- **GitHub MCP dependency**: the `cowork-with-github` agent requires the GitHub MCP on the OpenCode side. Without this MCP, routing falls back to `build` without GitHub features. The user must have pre-configured GitHub authentication.
- **Writing to user repo**: the `AGENTS.md` section modifies a file in the user's repo. The mechanism is strictly opt-in, delimited by HTML tags, and never triggers an automatic commit. The user retains full control.
- **No aggressive auto-purge**: the `AGENTS.md` section uses a simple FIFO at ~20 lines max. If it grows despite this, it's up to the user to clean it. No automatic deletion based on age or relevance.
- **Section uniqueness**: only one `<!-- Maintained by Cowork ... -->` section is managed per file. If multiple plugin instances run on the same repo, they share the same section (last writer wins). No intelligent merging.

## Inspirations

- **Agent roster pattern**: native OpenCode (build, plan, @general) extended by custom agents (cf. `swarm-code-plugin` agents/, where Claude Code adds specializations on top of the base model)
- **AGENTS.md convention**: natively adopted by OpenCode for project context (see design doc §6.5 + §2.4), extended here with an auto-maintained section for cross-session operational memory
- **Strict opt-in**: Anthropic AUP best practice — no silent writing to user repo. Every file modification in the user's workspace is preceded by explicit consent.
- **Simple FIFO**: inspired by circular logs (no infinite retention, no complex purge heuristic). The user can always manually clean the section if needed.
