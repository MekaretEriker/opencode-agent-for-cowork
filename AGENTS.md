# Agent guidelines — `opencode-agent` (Cowork plugin)

Cowork plugin (`.plugin` zip bundle, not an npm package) that orchestrates
OpenCode via `@mekareteriker/opencode-mcp`. Distribution: GitHub Releases with
the `.plugin` file attached.

This file documents **engineering invariants** any build agent (OpenCode,
Cowork's Claude, Cline, etc.) must respect when working on this plugin: SDK
source-of-truth rule, skill update checklist, `.mcp.json` structure, OpenRouter
model roster pointers, skill conventions.

For **operational workflow** (release management, plugin packaging, GitHub
release creation, issue tracking), see `CLAUDE.md` in the same directory.
Cowork's Claude typically drives those workflows; they're kept separate so
the engineering rules stay focused.

## Source of truth — opencode SDK first

Anything that touches the opencode HTTP API (skill content that documents a
dispatch tool, scheduled-task script, new tool wired into a skill) must be
verified against the official `@opencode-ai/sdk` documentation at
https://opencode.ai/docs/sdk.md **before** trusting either the wrapper's
existing tool list or training-data intuition. The wrapper occasionally lags
the upstream API; the SDK is auto-generated from the server's OpenAPI spec
and is the canonical reference.

When proposing a new tool for `@mekareteriker/opencode-mcp` to consume, also
check how `zaycruz/hermes-opencode-plugin` and the ecosystem in
`awesome-opencode` handle the same case — they consume the SDK at scale and
surface canonical patterns. Diverging from that prior art needs a written
justification.

Full rule + anti-patterns: see `D:\Projects\opencode-mcp\AGENTS.md` section
"Source of truth — opencode SDK first, Hermes reference second". The same
rule applies here.

## Skill update checklist (when a new `opencode-mcp` tool ships)

When `opencode-mcp` exposes a new public tool (e.g. `opencode_run_streaming`
in MEK-283), update these skills **in this order**, and ALL of them:

1. **`skills/opencode-safe-prompts/SKILL.md`** — add the new tool name to the
   scanned tool list in BOTH the frontmatter `description` AND the body
   ("via `opencode_run` / `opencode_run_streaming` / `opencode_ask` /
   `opencode_fire`"). **This is the critical one** — forgetting it means the
   new tool bypasses the safety scan entirely. It's a latent security gap.
2. **`skills/opencode-orchestrator/SKILL.md`** — add a subsection describing
   when to use the tool, position it appropriately in the routing table
   (recap table around L128-130). If the tool can produce new structured
   error codes, mention them in the "Orphaned session recovery" subsection.
3. **`skills/opencode-result-validator/SKILL.md`** — add the new tool as a
   valid trigger alongside `opencode_run` / `opencode_check`.
4. **`skills/opencode-fallback-chain/SKILL.md`** — ONLY if the new tool
   introduces new `StructuredErrorCode` values. Update the "Structured error
   codes" section mapping to specify the fallback decision for each new code.

Do not skip safe-prompts. The other three are about discoverability and
correctness; safe-prompts is about security.

## OpenRouter workspace (model roster)

The OpenRouter workspace bound to this deployment exposes a **curated subset
of models** (the "Allowed Models" list in the operator's OpenRouter
dashboard). Dispatching a model NOT in this list silent-fails (no LLM call,
session created but never started — observed during e2e tests of the
1.11.x line in May 2026).

**User-local config, not version-controlled.** Each operator maintains their
own `OPENROUTER-MODELS.md` at the repo root, gitignored. It pins which models
are reachable on that operator's account, which one is the default code
dispatch, fallback chains by use case, and known caveats per model
(empty-content under load, etc.). The file is meant to be the single point
of truth for runtime model decisions — when the orchestrator picks a model
for `opencode_run` / `opencode_run_streaming` / etc., it should consult that
file first.

**Recommended structure** (the operator's own file, kept local):

- Allowed-models table (display name, OpenRouter ID, pricing, role).
- Known caveats per model (e.g. silent empty-content under quota pressure).
- Dispatch heuristics (default, heavy reasoning, code completion, multimodal,
  fallback, free-batch).
- Fallback chains (chain code, chain reasoning, chain multimodal).
- Out-of-roster models — DO NOT DISPATCH (negative list to avoid silent fails).

The plugin code itself never embeds model IDs — they live in
`OPENROUTER-MODELS.md` (per-operator) and in env vars like
`OPENCODE_AGENT_PROVIDER_CHAIN`. AGENTS.md only points at the convention.

## `.mcp.json` structure

```json
{
  "mcpServers": {
    "opencode": {
      "command": "npx",
      "args": ["-y", "@mekareteriker/opencode-mcp@^X.Y.Z-mekareteriker.0"],
      "env": {
        "OPENCODE_DEFAULT_PROVIDER": "${OPENCODE_DEFAULT_PROVIDER}"
      }
    }
  }
}
```

If `opencode-mcp` adds new env vars that the plugin should expose (e.g.
`OPENCODE_MCP_TRANSLATE_PATHS`, `OPENCODE_MCP_IDEMPOTENCY_WINDOW_MS`),
forward them via `${...}` so the user can configure them through Cowork's
plugin settings without editing this file.

## Conventions for the 8 skills

- **Frontmatter description triggers** are bilingual (FR + EN). Cowork routes
  skills based on description, language-agnostic — keeping both languages in
  the triggers covers users in either UI language without duplicating skills.
- **Skill bodies** are in English (translated as part of `v1.0.1`).
- **Inter-skill references** in the body should use bare skill names
  (`fallback-chain`, `orchestrator`) without the `opencode-` prefix, for
  readability.
