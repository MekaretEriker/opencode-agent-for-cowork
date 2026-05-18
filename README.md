# opencode-agent

> Cowork plugin that turns Cowork into an intelligent orchestrator for [OpenCode](https://github.com/anomalyco/opencode). Delegate coding tasks, monitor sessions, validate results — all from natural-language conversations.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.1.4-blue.svg)](CHANGELOG.md)
[![Status](https://img.shields.io/badge/status-feature--complete-success.svg)](#roadmap)
[![Built on opencode-mcp](https://img.shields.io/badge/built%20on-%40mekareteriker%2Fopencode--mcp%20%5E1.12.1-orange.svg)](https://github.com/MekaretEriker/opencode-mcp)
[![For Cowork](https://img.shields.io/badge/for-Cowork-purple.svg)](https://anthropic.com)
[![Plugin format](https://img.shields.io/badge/format-.plugin-lightgrey.svg)](#installation)

---

```
┌────────────────────────────────────────────────┐
│  Cowork  (Claude desktop)                      │
│  ┌──────────────────────────────────────────┐  │
│  │  opencode-agent plugin                   │  │
│  │  • 8 orchestration skills (catalog ↓)    │  │
│  │  • 3 scheduled task recipes              │  │
│  │  • 1 custom agent template               │  │
│  │  • opencode-mcp connector (auto-managed) │  │
│  └────────────────┬─────────────────────────┘  │
└───────────────────┼────────────────────────────┘
                    │  MCP (stdio)
                    ▼
       ┌─────────────────────┐   HTTP   ┌─────────────────────┐
       │   opencode-mcp      │ ───────► │   opencode serve    │
       │  (npx, auto-start)  │ ◄─────── │  (local OpenCode)   │
       └─────────────────────┘          └─────────┬───────────┘
                                                  │
                                       ┌──────────┴──────────┐
                                       │                     │
                              ┌────────▼────────┐   ┌────────▼────────┐
                              │  Native agents  │   │  User MCPs      │
                              │  • build        │   │  • GitHub       │
                              │  • plan         │   │  • Postgres     │
                              │  • @general     │   │  • Notion       │
                              └─────────────────┘   │  • RAG / vault  │
                                                    │  • validators   │
                                                    └─────────────────┘
```

## Table of contents

- [Why this plugin](#why-this-plugin)
- [Installation](#installation)
- [Prerequisites](#prerequisites)
- [Quick usage](#quick-usage)
- [Skills catalog (8)](#skills-catalog-8)
- [Configuration](#configuration)
- [Architecture](#architecture)
- [Composability with user MCPs](#composability-with-user-mcps)
- [Coexistence with OpenCode Desktop](#coexistence-with-opencode-desktop)
- [Related projects](#related-projects)
- [Known limitations](#known-limitations)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [Anthropic ToS compliance](#anthropic-tos-compliance)
- [License](#license)
- [Credits](#credits)

---

## Why this plugin

OpenCode is a powerful agentic coding tool. Using it from Cowork natively means writing prompts blind — you decide everything yourself: which agent to use, how long to wait, what to validate.

This plugin teaches Cowork how to do that intelligently:

- **Delegate** — picks the right OpenCode agent (`build` vs `plan` vs `@general`) and the right invocation tool (`ask` vs `run` vs `fire`) based on your prompt
- **Monitor** — detects stalled sessions, recovers orphaned ones after MCP transport timeouts
- **Validate** — before declaring "done", verifies diff coherence, tests, build, semantic alignment with intent
- **Remember** — accumulates operational stats per project (durations, reliable providers, learned patterns)
- **Stay safe** — warns on dangerous patterns before dispatch (`rm -rf /`, `DROP TABLE`, etc.)
- **Recover** — automatically retries with the next provider on rate-limit/server errors
- **Schedule** — runs daily digests, weekly audits, monthly refactor scans
- **Compose** — defers to your existing MCPs (RAG, ADR, validators) when relevant rather than duplicating their work

Result: Cowork becomes a **team lead** for OpenCode, while you stay in conversation.

## Installation

**3 steps, < 3 minutes** (assuming Cowork and OpenCode are already installed — see [Prerequisites](#prerequisites) if not).

1. **Download** `opencode-agent.plugin` from the latest [release](../../releases/latest).
2. **Drag-and-drop** the file into a Cowork conversation. A rich preview card appears.
3. **Click "Accept"** when prompted. Restart any open conversation to activate the plugin.

That's it. The next time you ask Cowork something like *"OpenCode, refactor my auth module"*, it will route through this plugin.

## Prerequisites

| Requirement | Version | How to check / install |
|---|---|---|
| **[Cowork](https://anthropic.com)** | Latest desktop app | You're reading this from Cowork? You have it ✓ |
| **[OpenCode CLI](https://github.com/anomalyco/opencode)** | 1.14.x or later | `opencode --version`. Install: `curl -fsSL https://opencode.ai/install \| bash` (macOS/Linux/WSL) or `irm https://opencode.ai/install.ps1 \| iex` (Windows PowerShell) |
| **Node.js** | 18+ (for `npx`) | `node --version`. Needed for `opencode-mcp` auto-fetch at first run |
| **OS** | Windows / macOS / Linux / WSL2 | Primary tested platform: Windows + WSL2. Others: best-effort, no known blockers |
| **Internet** | Required at first run | `npm` fetches `opencode-mcp` (~10 MB, cached after first run) |

The plugin proposes assisted install of OpenCode on first run if missing — no manual setup required for typical users.

## Quick usage

### Standard mode (default)

Just talk to Cowork in natural language:

> **You:** "OpenCode, refactor the auth module to use JWT."
>
> **Cowork:** "I'll ask OpenCode to refactor the auth module. This is a code modification task (~5 min estimated), so I'll use the `build` agent and dispatch it in the background. Starting now..."
>
> *[OpenCode works in background]*
>
> **Cowork:** "✓ Done. Modified 3 files (auth.ts, middleware.ts, types.ts). Tests pass (47/47). Want me to also update the documentation?"

The orchestrator skill announces its plan (ReAct pattern), dispatches intelligently, and validates the result before reporting back.

### Dev mode (on request)

Activate technical details on demand:

> **You:** "Show me the details."
>
> **Cowork:** "Session: `ses_xxxxx` (agent: build, provider: anthropic, model: claude-sonnet-4-6). Duration: 4m12s (8 iterations). Tools invoked: opencode_find_text (3x), opencode_file_read (12x). Diff: auth.ts +47/-23, middleware.ts +12/-4, types.ts +8/-0. For drill-down: `opencode_conversation({sessionId: \"ses_xxxxx\"})`."

Triggers: `"show me details"`, `"dev mode"`, `"session id"`, `"explain technically"`.

### Parallel sessions

> **You:** "Run a refactor on auth, a security audit on payments, and a doc review on notifications — in parallel."
>
> **Cowork:** "Three independent tasks on distinct modules — parallelizable. Dispatching 3 OpenCode sessions in background. Ask me 'where are my OpenCode sessions?' anytime for status."

Default cap: 3 concurrent sessions per repo (configurable via `OPENCODE_AGENT_MAX_PARALLEL`).

## Skills catalog (8)

The plugin ships 8 instructional skills. Each is a markdown file under `skills/<name>/SKILL.md`. They're loaded by Cowork automatically; you don't invoke them by name.

| Skill | Triggered when | What it does |
|---|---|---|
| **[opencode-orchestrator](skills/opencode-orchestrator/SKILL.md)** | You ask Cowork to delegate code work to OpenCode | Core piloting: ReAct, agent routing (`build`/`plan`/`@general`), tool choice (`ask`/`run`/`fire`), stall detection, parallel sessions, install assistance, AGENTS.md reading, MCP timeout workaround |
| **[opencode-safe-prompts](skills/opencode-safe-prompts/SKILL.md)** | Before dispatching prompts that include shell, SQL, git destructive ops, or remote execution | Advisor that warns on 5 categories of dangerous patterns (filesystem destructive, SQL destructive, RCE, git destructive, secrets) before dispatch |
| **[opencode-task-memory](skills/opencode-task-memory/SKILL.md)** | At task start (read) and task end (write) on the same project | Maintains per-project operational memory in `<workspace>/.opencode-agent-memory.json` (stats by task type, by provider, learned patterns, antipatterns) |
| **[opencode-result-validator](skills/opencode-result-validator/SKILL.md)** | After OpenCode reports completion of a `build` agent task | 4 systematic checks before declaring success: diff coherence, tests, build/compile, semantic alignment with intent (Plan-Execute-Reflect 3rd step) |
| **[opencode-fallback-chain](skills/opencode-fallback-chain/SKILL.md)** | When OpenCode returns 429/5xx from a provider | Retries with the next provider in the configured chain (anthropic → openrouter → openai by default), up to 3 attempts with 5s backoff |
| **[opencode-agent-roster](skills/opencode-agent-roster/SKILL.md)** | When choosing an OpenCode agent, especially for GitHub-related tasks | Extends native build/plan with custom `cowork-with-github` agent (build + GitHub MCP). Also handles opt-in AGENTS.md writing with delimited section |
| **[opencode-scheduled-recipes](skills/opencode-scheduled-recipes/SKILL.md)** | When the user asks about recurring tasks or scheduled automation | Documents 3 bundled recipes (`daily-repo-digest`, `weekly-deps-audit`, `monthly-refactor-scan`) and how to activate them via Cowork's native scheduler |
| **[opencode-mcp-discovery](skills/opencode-mcp-discovery/SKILL.md)** | At first run on a new project | Detects user-side MCPs (RAG, validate, ADR, vault, etc.) and proposes substitution/chaining/skill-only composability options with the plugin's Full skills |

## Configuration

All optional. The plugin works with sensible defaults out of the box.

### Environment variables

| Variable | Default | Description |
|---|---|---|
| `OPENCODE_AGENT_MAX_PARALLEL` | `3` | Max concurrent OpenCode sessions per project. Set to `1` for strict sequential, higher for powerful machines with tolerant providers |
| `OPENCODE_AGENT_PROVIDER_CHAIN` | `anthropic,openrouter,openai` | Comma-separated provider fallback chain. Used by `opencode-fallback-chain` skill on 429/5xx errors |
| `OPENCODE_DEFAULT_PROVIDER` | *(none)* | Default LLM provider for OpenCode dispatches when not specified per-tool. Example: `anthropic` |
| `OPENCODE_DEFAULT_MODEL` | *(none)* | Default model ID. Example: `claude-sonnet-4-5` |
| `OPENCODE_BASE_URL` | `http://127.0.0.1:4096` | OpenCode server URL (inherited from `opencode-mcp`) |

### Per-project memory

The `task-memory` skill creates a file at `<workspace>/.opencode-agent-memory.json` on first run. It's proposed for addition to your `.gitignore`. Schema:

```json
{
  "version": "1.0",
  "project": "/absolute/path/to/repo",
  "tasksTotal": 47,
  "byType": {
    "feature_implementation": {
      "count": 28,
      "avgDurationSec": 240,
      "successRate": 0.89,
      "preferredAgent": "build"
    }
  },
  "providerStats": { "anthropic": { "calls": 35, "successRate": 0.94 } },
  "patterns": ["..."],
  "antipatterns": ["..."],
  "mcpComposition": { /* set by mcp-discovery skill */ }
}
```

## Architecture

The plugin sits **Cowork-side** as orchestration intelligence. It does not modify OpenCode itself.

```
Cowork (you, Claude) — orchestrator
  │
  └─ orchestrator skill decides:
       ├─ tool: ask / run / fire
       ├─ agent: build / plan / @general / cowork-with-github
       ├─ duration: estimate from task-memory
       └─ provider: best from providerStats or env default
            │
            ▼
    opencode-mcp (npx bridge)
            │
            ▼
    opencode serve (local HTTP server)
            │
            ├─ executes via native agents (build/plan/@general)
            ├─ uses user MCPs (GitHub, Postgres, ...) if configured
            └─ reads AGENTS.md from target repo for project context
            │
            ▼
    Result returns to Cowork
            │
            └─ result-validator skill checks: diff, tests, build, semantics
                 ├─ ✓ all green → "Task complete"
                 ├─ ⚠ caveats → "Done with reserves"
                 └─ ✗ critical → "Critical issue, here are options"
```

This is the **orchestrator-workers pattern** described in [Anthropic's "Building Effective Agents"](https://www.anthropic.com/research/building-effective-agents): Cowork directs and synthesizes, OpenCode executes the heavy work.

## Composability with user MCPs

If you already have domain-specific MCPs (a RAG/vault, a validator, an ADR manager), this plugin **defers to them** rather than duplicating their work.

At first run per project, the `mcp-discovery` skill detects MCPs exposing tools like `search_vault`, `validate`, `manage_adr`, and proposes:

- **(a) Substitution** — disable the plugin's overlapping skill, use yours
- **(b) Chaining** — plugin checks first (generic), your MCP next (domain-specific)
- **(c) Plugin-only** — keep my skills, ignore your MCPs for this concern

The decision is persisted in `task-memory`. Re-discovery on demand: `"redo MCP discovery"`.

| Your MCP exposes... | Plugin skill it can replace/augment |
|---|---|
| `search_vault`, `search_code_graph` | (none — augments orchestrator context) |
| `validate` | `result-validator` |
| `manage_adr`, `list_decisions` | `task-memory` (architectural decisions side) |
| Generic RAG (`search`, `query`) | (none — augments orchestrator context) |

See the `mcp-discovery` skill for the full matching logic.

## Coexistence with OpenCode Desktop

OpenCode publishes its own [desktop app (beta)](https://opencode.ai/download). **This plugin is not a replacement.**

| You want to... | Use... |
|---|---|
| Pilot OpenCode in a fast TUI with keyboard shortcuts | **OpenCode Desktop** |
| Delegate OpenCode tasks from Cowork conversations (natural language, multi-session, with memory) | **This plugin** |
| Both | No conflict — `opencode serve` runs locally for both |

If OpenCode Desktop is already running, this plugin reuses the same local server. No duplicate instance.

## Known limitations

| # | Limitation | Mitigation in plugin |
|---|---|---|
| L1 | ~~`opencode-mcp` validation can reject valid absolute paths in the `directory` parameter (false positive on `/mnt/d/...` paths)~~ | ✅ Resolved in v1.0.2 via `@mekareteriker/opencode-mcp` fork (MEK-289 WSL/Windows path translation). The `cd $path &&` workaround is no longer needed. |
| L2 | MCP transport has a ~180s timeout that overrides `maxDurationSeconds` | Orchestrator defaults to `fire+check` for tasks > 3 min; recovers orphaned sessions via `sessions_overview` |
| L3 | Skills are markdown instructions, not event-driven hooks. Stall detection works only while Cowork is actively piloting the task | Documented in each skill's "Limits" section |
| L4 | WSL ↔ Windows mount can have cache lag for newly written files | Plugin packaging uses dual-naming (`opencode-agent.plugin` + `opencode-agent-v<version>.plugin`) to bypass cache |
| L5 | `task-memory` is per-project, not cross-project | By design (per Q1 decision in DESIGN doc). Cross-project memory deferred to v2 if needed |

## Roadmap

Full scope from the design document is **complete in v1.0**. All 11 decisions (Q1-Q11) shipped.

| Version | Milestone | Status |
|---|---|---|
| v0.1.0 | MVP — orchestrator skill, opencode-mcp connector | ✓ shipped |
| v0.1.1 | Patch: MCP timeout 180s handling | ✓ shipped |
| v0.2.0 | safe-prompts | ✓ shipped |
| v0.3.0 | task-memory | ✓ shipped |
| v0.4.0 | result-validator (Plan-Execute-Reflect) | ✓ shipped |
| v0.5.0 | fallback-chain | ✓ shipped |
| v0.6.0 | agent-roster + AGENTS.md write | ✓ shipped |
| v0.7.0 | scheduled-tasks | ✓ shipped |
| **v1.0.0** | **mcp-discovery (Full scope complete)** | ✓ shipped |
| v1.0.1 | i18n: skill bodies translated to English | ✓ shipped |
| v2.x | TBD — informed by community feedback | open |

Full per-version notes: [CHANGELOG.md](CHANGELOG.md).

## Contributing

Contributions welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for:

- How to propose a new skill (frontmatter spec, body style, testing)
- Commit conventions
- Semver policy
- Code of conduct

Tracker: [issues](../../issues) · Discussions: [discussions](../../discussions).

## Anthropic ToS compliance

This plugin makes Cowork (Claude) orchestrate OpenCode (another AI agent). This direction — Claude delegating to another AI for developer productivity — is explicitly permitted under Anthropic's [Acceptable Use Policy](https://www.anthropic.com/legal/aup) and [Agentic Use Guidelines](https://support.anthropic.com/en/articles/12005017-using-agents-according-to-our-usage-policy).

Precedent: [swarm-code-plugin](https://github.com/apoapps/swarm-code-plugin) (Claude Code → OpenCode variant) ran the same legal audit and is compliant.

## License

MIT — see [LICENSE](LICENSE).

## Related projects

Other orchestration approaches in the OpenCode ecosystem:

| Project | Layer | Approach |
|---|---|---|
| **This plugin** | Cowork-side | Skills inside Cowork tell Claude how to delegate to OpenCode. Single OpenCode `build`/`plan` agent per task. |
| [tempont/small-opencode-orchestrator](https://github.com/tempont/small-opencode-orchestrator) | **OpenCode-side** | Native OpenCode config with ~10 custom agents (`orchestrator`, `plan-runner`, `code-executor`, `code-explorer`, reviewers…). Model tiering per role (strong models for orchestrator, cheap models for subagents). Plans persisted to disk with explicit approval gates. |
| [apoapps/swarm-code-plugin](https://github.com/apoapps/swarm-code-plugin) | Claude Code-side | Claude Code orchestrates OpenCode as a worker via tmux pipe-pane + keyword watcher. ~80% token savings reported. |
| [code-yeongyu/oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) | **Terminal-side** | Shell integration layer (zsh/bash/fish) for developers who live in the terminal. Brings OpenCode into your shell workflow with keyboard shortcuts, live streaming output, and shell hooks. 54.9k★ |

These are **complementary, not competing**. In theory, you can compose:

```
Cowork (this plugin) ─► OpenCode (tempont config) ─► subagents (tempont)
       ↓                          ↓                            ↓
  decides what to            decides how to              executes plan,
  delegate (build/plan,      execute (orchestrator       reviews, tests
  parallel, when to fire)    → plan-runner →
                             code-executor →
                             reviewers)
```

Three levels of orchestration — overkill for most cases, but theoretically powerful for large/critical projects. Pick the layer that matches your workflow.

### Coexistence with oh-my-openagent

[oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) and this plugin live at **different layers** of the same stack — they coexist without conflict:

- **oh-my-openagent** owns the *terminal experience*: shell integration, keyboard shortcuts, live streaming output in your zsh/bash/fish session. Ideal for developers who want OpenCode tightly woven into their terminal workflow.
- **This plugin** owns the *Cowork orchestration layer*: natural-language delegation, multi-session parallelism, per-project memory, result validation, provider fallback. Ideal for developers who work from Cowork conversations.

You can use both simultaneously — oh-my-openagent for your own direct OpenCode sessions in the terminal, this plugin for everything Cowork delegates to OpenCode. Both talk to the same local `opencode serve` instance with no conflict.

## Credits

| Project | Role |
|---|---|
| [anomalyco/opencode](https://github.com/anomalyco/opencode) | The open source coding agent we orchestrate (129k★, 100+ providers, native LSP) |
| [AlaeddineMessadi/opencode-mcp](https://github.com/AlaeddineMessadi/opencode-mcp) | The MCP bridge this plugin builds on |
| [apoapps/swarm-code-plugin](https://github.com/apoapps/swarm-code-plugin) | Architectural inspiration (Claude Code → OpenCode variant of the orchestrator-workers pattern) |
| [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) | Patterns adopted: activity-based timeout, provider fallback, ADR-style memory |
| [Anthropic Cowork](https://anthropic.com) | The host platform |

About OpenCode programmatic integration: in addition to the MCP path this plugin uses, OpenCode publishes [official SDKs](https://opencode.ai/docs/sdk/) for JavaScript and Python — useful if you want to build a non-MCP integration (CLI, dashboard, scheduled workers outside Cowork).

---

**Made by [Mekaret](https://github.com/<your-username>)** · *Cowork directs, OpenCode executes.*
