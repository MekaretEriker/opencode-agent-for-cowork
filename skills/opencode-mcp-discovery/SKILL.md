---
name: opencode-mcp-discovery
description: "Use this skill at the start of each new project to detect user-side MCPs (RAG, validate, manage_adr, etc.) and propose composability options with the plugin's Full skills. Concretizes the composability principle from design doc §13.5."
---

# opencode-mcp-discovery

## Nature and scope
Meta-coordinator skill that activates **on the first plugin run for a project**. It detects MCPs available on the Cowork side and the OpenCode side, identifies those that fully or partially cover the needs of the plugin's Full skills, and proposes composition modes to the user. The goal: NEVER duplicate what the user has already built.

This is the embodiment of the guiding principle §13.5 of the design doc: "the plugin enriches, it does not replace".

## MCP detection
On the first run per project:
1. List MCPs on the Cowork side (via introspection of `mcp__*` tools available in the session)
2. List MCPs on the OpenCode side via `opencode_mcp_status`
3. For each detected MCP, look at its exposed tools (heuristic search on names: `search`, `query`, `validate`, `manage_adr`, `analyze`, `trace`, etc.)
4. Map these tools to the plugin's Full skills they could potentially cover

## MCP -> plugin skill mapping

| Detected MCP tool | Potentially covers | Proposed action |
|---|---|---|
| `search_vault`, `search_code_graph`, `ask` | Enriched project context (beyond AGENTS.md) | Enrich orchestrator with this MCP before dispatch |
| `validate`, `check_consistency` | result-validator (v0.4.0) | Substitution or chaining with result-validator |
| `manage_adr`, `list_decisions` | task-memory (v0.3.0) — architectural part | Chain: task-memory for raw stats, MCP for decisions |
| `analyze_entity_impact`, `trace_call_path` | Agent heuristic + context | Enrich pre-dispatch analysis |
| Generic RAG (Chroma, LlamaIndex) | Project context | Read before dispatch for enriched prompt |

## Procedure on first run per project
1. Check in `<workspace>/.opencode-agent-memory.json` whether discovery has already been done (field `mcpDiscoveryDone: true`)
2. If not: run discovery. Otherwise: skip.
3. If relevant MCPs detected, present to the user:

```
I detected that you're using [list of detected MCPs].

Some of them already cover what my Full skills do:
- [MCP X.validate] -> replaces or complements my result-validator
- [MCP Y.manage_adr] -> complements my task-memory for architectural decisions

Do you prefer:
(a) substitution: I disable my skills for these functions, I use yours
(b) chaining: my skills first (generic checks), your MCPs next (domain checks)
(c) plugin only: I keep my skills, I don't use your MCPs

Default choice if you don't respond within 30s: (b) chaining.
```

4. Store the decision in `.opencode-agent-memory.json`:
```json
{
  "mcpDiscoveryDone": true,
  "mcpDiscoveryAt": "2026-05-15T15:00:00Z",
  "mcpComposition": {
    "result-validator": { "mode": "chain", "delegate": "relkhon-vault-agent.validate" },
    "task-memory": { "mode": "skill-only", "note": "task-memory enough, no MCP user-side ADR" }
  }
}
```

5. On subsequent tasks, the orchestrator and Full skills read this config and adjust their behavior.

## Detection heuristics

### Relkhon-style vault
If you detect tools like `search_vault`, `search_code_graph`, `manage_adr`, `validate`, `analyze_entity_impact` under the same MCP -> it's a domain-specific vault. Propose chaining with orchestrator, task-memory, result-validator.

### Simple RAG (Chroma, LlamaIndex)
If you detect only `search` or `query` + similarity without graph or ADR -> it's a simple RAG. Propose prompt enrichment before dispatch (read before to add context).

### Custom validator
If you detect an isolated `validate` tool without the rest -> propose substitution of result-validator if the user is confident it better covers their domain checks.

## Re-discovery
Discovery is not immutable. The user can request:
- "Refais la decouverte MCP" / "Redo MCP discovery" -> reset `mcpDiscoveryDone: false` in memory and re-run
- "Change ma composition pour [skill] en [mode]" / "Change my composition for [skill] to [mode]" -> targeted update

## Inter-skill
- **orchestrator**: calls this skill once per project (on first run detected via task-memory)
- **task-memory**: stores the mcpComposition config
- **result-validator**: consults the config before running its checks (can delegate to user MCP)
- **task-memory** itself: can be short-circuited if manage_adr already covers it

## Limitations
- Detection is based on tool names (heuristic). False negatives possible with custom MCPs using non-standard naming.
- The user can override any decision via explicit command.
- Discovery adds ~5-10s to the first run of a project; this is acceptable.
- No automatic re-discovery if the user adds a new MCP mid-project — they must explicitly re-run discovery.

## Inspirations
- "Configurable composability" pattern: design doc §13.5
- "Double dispatch" (Cowork side vs OpenCode side): §6.2.2
- Runtime MCP discovery: functional equivalent of Hermes `check_fn` (but skill-side, not tool registration side)
- swarm-code-plugin has no equivalent — this is our unique contribution

## Note on v1.0.0
This version marks the **completion of the Full scope** of the design doc. All planned capabilities are implemented:
- 8 Full skills: orchestrator, safe-prompts, task-memory, result-validator, fallback-chain, agent-roster, scheduled-recipes, mcp-discovery
- 3 bundled scheduled-tasks
- 1 custom agent (cowork-with-github)
- User MCP composability formalized

All Q1-Q11 decisions from the design doc are satisfied. The plugin is feature-complete for its MVP-Full roadmap.
