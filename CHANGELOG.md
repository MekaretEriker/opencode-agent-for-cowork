# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.2] - 2026-05-17

### Changed

- `.mcp.json` now requires `@mekareteriker/opencode-mcp@^1.11.2-mekareteriker.0` (was `^1.11.1-mekareteriker.0`). Picks up the **MEK-295 hotfix** for `opencode_run_streaming`: v1.11.1 (MEK-294) had fixed the broken `/prompt` endpoint by switching to `/message`, but `/message` is synchronous (blocks until the agent loop completes) — the SSE subscription in `workflow.ts` was opened AFTER the await, systematically missing `session.idle`. Sessions executed correctly (assistant message non-empty, cost recorded) but the tool returned `SESSION_HANG`. v1.11.2 switches to `/prompt_async` (canonical async endpoint, returns 204 immediately, emits SSE events) and reorders the subscription to happen BEFORE the POST — the canonical pattern from the official `@opencode-ai/sdk`. Without this bump, `opencode_run_streaming` returns `SESSION_HANG` despite the LLM actually executing. See [opencode-mcp v1.11.2-mekareteriker.0 release](https://github.com/MekaretEriker/opencode-mcp/releases/tag/v1.11.2-mekareteriker.0).

## [1.1.1] - 2026-05-17

### Changed

- `.mcp.json` now requires `@mekareteriker/opencode-mcp@^1.11.1-mekareteriker.0` (was `^1.11.0-mekareteriker.0`). Picks up the **MEK-294 hotfix** for `opencode_run_streaming`: v1.11.0 shipped the tool with a `POST /session/{sid}/prompt` that is silently accepted by opencode server 1.14.50 but does NOT trigger LLM execution — every dispatch produced an empty session and timed out with `SESSION_HANG`. v1.11.1 switches to `POST /session/{sid}/message` (the canonical endpoint also used by `opencode_run` / `opencode_ask`). Without this bump, `opencode_run_streaming` is non-functional in production despite the v1.1.0 skill wiring. See [opencode-mcp v1.11.1-mekareteriker.0 release](https://github.com/MekaretEriker/opencode-mcp/releases/tag/v1.11.1-mekareteriker.0).

## [1.1.0] - 2026-05-17

### Changed

- `.mcp.json` now requires `@mekareteriker/opencode-mcp@^1.11.0-mekareteriker.0` (was `^1.10.2-mekareteriker.0`). The previous range did not match `1.11.0` due to npm's strict caret-semver-with-prerelease rules. The upstream fork release bundles MEK-282 (structured error codes), MEK-283 (new `opencode_run_streaming` tool), MEK-284 (idempotency dedup for POST/PUT/PATCH), MEK-289 (WSL/Windows path translation). See [opencode-mcp v1.11.0-mekareteriker.0 release](https://github.com/MekaretEriker/opencode-mcp/releases/tag/v1.11.0-mekareteriker.0).

### Added

- `skills/opencode-orchestrator/SKILL.md`: new subsection **`opencode_run_streaming` — SSE-driven synchronous task**, positioned between `opencode_run` and `opencode_fire`. Documents when to prefer the streaming variant (60-180s tasks where the user is actively waiting and would benefit from live progress) and clarifies the ~180s MCP transport timeout still applies. New row added to the recap table. Orphaned-session recovery section now mentions the new `SESSION_HANG` structured error code distinct from the Cowork-side MCP timeout.
- `skills/opencode-fallback-chain/SKILL.md`: new section **Structured error codes (opencode-mcp 1.11.0+)** documenting how to parse the `<!-- structured-error -->` JSON block to decide on fallback. Maps each code (`RATE_LIMITED`, `PROVIDER_ERROR`, `AUTH_FAILED`, `TIMEOUT`, `SESSION_HANG`, `EMPTY_RESPONSE`) to a fallback decision. Regex fallback on message preserved for older opencode-mcp versions.
- `skills/opencode-safe-prompts/SKILL.md`: `opencode_run_streaming` added to the list of dispatch tools scanned for risky patterns (frontmatter description + body). Closes a latent security gap where a user could have bypassed the safety scan by using the new streaming tool.
- `skills/opencode-result-validator/SKILL.md`: `opencode_run_streaming` added as a valid trigger alongside `opencode_run` and the `opencode_check` phase following `opencode_fire`.

## [1.0.3] - 2026-05-16

### Changed
- `skills/opencode-orchestrator/SKILL.md`: section "Workaround - opencode-mcp directory parameter" replaced by "Directory parameter — resolved via @mekareteriker/opencode-mcp fork". The `cd $path && …` instruction is no longer the active behavior; it stays only as a fallback note for users who force the plugin onto upstream `opencode-mcp <= 1.10.1`.
- `DESIGN-opencode-agent-for-cowork.md` §10 R9 (in the parent project repo): status updated from "open, mitigation skill" to "✅ resolved via fork on 2026-05-16".

## [1.0.2] - 2026-05-16

### Changed
- `.mcp.json` now uses `@mekareteriker/opencode-mcp@^1.10.2-mekareteriker.0` (hardened fork) instead of upstream `opencode-mcp@^1.10.1`. Resolves R9 — the `directory` parameter no longer rejects valid absolute paths on Windows. See [@mekareteriker/opencode-mcp on npm](https://www.npmjs.com/package/@mekareteriker/opencode-mcp) and `D:\Projects\opencode-mcp\SPEC-fork.md` for full audit trail.

## [1.0.1] - 2026-05-15

### Changed
- All skill bodies translated to English (frontmatter triggers remain bilingual)
- Frontmatter descriptions enriched with bilingual trigger examples (FR + EN)
- User-facing French usage unchanged (Cowork routes skills based on description, language-agnostic)

## [1.0.0] - 2026-05-15

### Added — Full scope complete

- New skill `opencode-mcp-discovery`: detects user-side MCPs (RAG, validate, manage_adr) on first run per project and proposes substitution/chaining/skill-only
- Composition decision persisted in `.opencode-agent-memory.json` (field `mcpComposition`)
- Re-discovery possible on explicit request

### Milestone

This version marks the **completion of the Full scope** of the design doc:
- 8 Full skills
- 3 bundled scheduled-tasks (daily-repo-digest, weekly-deps-audit, monthly-refactor-scan)
- 1 custom agent (cowork-with-github)
- User MCP composability formalized
- 11 decisions Q1-Q11 satisfied

## [0.7.0] - 2026-05-15

### Added

- 3 bundled scheduled-tasks: `daily-repo-digest`, `weekly-deps-audit`, `monthly-refactor-scan`
- Skill `opencode-scheduled-recipes` documenting activation and customization
- Integration with native Cowork `mcp__scheduled-tasks__*` mechanism

## [0.6.0] - 2026-05-15

### Added

- Skill `opencode-agent-roster` delivering the custom agent `cowork-with-github` (`build` extension + GitHub MCP)
- Template `agent-templates/cowork-with-github.json`
- Opt-in AGENTS.md writing per project with HTML-tag-delimited section
- Opt-in stored in `.opencode-agent-memory.json` (field `agentsMdWriteEnabled`)

## [0.5.0] - 2026-05-15

### Added

- Skill `opencode-fallback-chain`: automatic retry with next provider on 429/5xx errors
- Default chain `anthropic -> openrouter -> openai`
- Override via env `OPENCODE_AGENT_PROVIDER_CHAIN`
- 3 retries max, 5s backoff

## [0.4.0] - 2026-05-15

### Added

- Skill `opencode-result-validator`: Plan-Execute-Reflect pattern (3rd step)
- 4 systematic checks: clean diff, tests, build, semantic coherence
- 3 output modes: full success, caveats, critical failure

## [0.3.0] - 2026-05-15

### Added

- Skill `opencode-task-memory`: JSON operational memory per project (`<workspace>/.opencode-agent-memory.json`)
- Stats by task type, by provider, learned patterns/antipatterns
- Proposes `.gitignore` entry on first run

## [0.2.0] - 2026-05-14

### Added

- Skill `opencode-safe-prompts`: dangerous pattern advisor (5 categories: destructive filesystem, destructive SQL, RCE, destructive git, secrets/exfiltration)
- Advisor pattern (not a firewall), consistent with layered safety Q5

## [0.1.1] - 2026-05-14

### Fixed

- Patched MCP transport timeout handling (~180s)
- Orchestrator SKILL.md section 2: lowered `opencode_run` threshold from "1-15 min" to "under 2-3 min"
- Beyond that: `opencode_fire` + `opencode_check` on the next turn
- New subsection "Orphaned session recovery": recovery pattern when MCP disconnects but the OpenCode session keeps running server-side
- Inspiration: hermes-agent activity-based timeout (issue #4815, PR #4864)

## [0.1.0] - 2026-05-14

### Added — Initial MVP

- Plugin scaffolding (manifest, .mcp.json, README)
- Complete skill `opencode-orchestrator` with 8 sections:
  - General framework (orchestrator-workers philosophy)
  - ReAct (explicit reasoning before dispatch)
  - Tool selection (ask / run / fire)
  - Agent selection (build / plan / @general)
  - Stall heuristic during active monitoring
  - Parallelization policy (max 3 concurrent)
  - Assisted OpenCode install on first run
  - Reading target repo's AGENTS.md
  - Result format (standard mode vs dev mode, progressive disclosure)
  - Technical notes (opencode-mcp `directory` workaround)
- opencode-mcp connector with caret pin `^1.10.1`
- 11 design doc decisions Q1-Q11 satisfied for MVP scope
