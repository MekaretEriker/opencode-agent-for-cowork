# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
