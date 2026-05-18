---
name: opencode-timeout-router
description: "Use this skill before EVERY OpenCode dispatch (alongside `opencode-orchestrator`, before `opencode-safe-prompts`) to estimate task duration, route to the right MCP tool (opencode_ask / opencode_run / opencode_run_streaming / opencode_fire), and dedupe accidental duplicate dispatches within 60s. Bilingual triggers: 'estime la durée', 'routage sync ou async', 'évite doublon dispatch', 'idempotency locale', 'estimate duration', 'route to ask run or fire', 'dispatch dedup', 'avoid duplicate dispatch'."
---

# opencode-timeout-router

## Nature and scope

Defensive routing skill that sits **between `opencode-orchestrator` (v1.0) and `opencode-safe-prompts` (v1.1)**. It does two things before any OpenCode dispatch leaves Cowork:

1. **Pre-flight duration estimate + tool routing**: pick the right MCP tool (`opencode_ask`, `opencode_run`, `opencode_run_streaming`, `opencode_fire` + `opencode_wait`) based on the estimated task duration, calibrated against `.opencode-agent-memory.json` when available.
2. **Local idempotency**: detect accidental duplicate dispatches within a 60-second window (`sha256(sessionId + prompt + timestamp_minute)`) and refuse the second one with a clear message — *before* the wrapper's own dedup helper sees it.

This skill is **defense-in-depth** against known opencode-mcp pathologies (retry destructive on POST, polling silent-fail, dedup over-application across endpoints). It is most valuable while the wrapper fixes MEK-281, MEK-282, MEK-283, MEK-284 are pending or not yet rolled out across every consumer. Once those wrapper fixes are universally deployed, the local idempotency layer becomes redundant — but the routing layer stays useful regardless (it is the operational embodiment of the Plan-Execute-Reflect pattern).

## When this skill triggers

Automatically, before any OpenCode dispatch performed under the `opencode-orchestrator` flow. Trigger phrases (handled by Cowork's description-based routing):

- FR: "estime la durée", "routage sync ou async", "évite doublon dispatch", "idempotency locale", "lance OpenCode sur"
- EN: "estimate duration", "route to ask run or fire", "dispatch dedup", "avoid duplicate dispatch", "run OpenCode on"

When NOT to apply:
- Reading a session (`opencode_check`, `opencode_conversation`, `opencode_review_changes`) — no dispatch, no estimate needed.
- The user explicitly requested raw control: "fire it directly, skip routing".

## 1. Duration estimation

### Baselines (no memory)

| Task type                | Baseline (s) |
|--------------------------|--------------|
| `question`               | 15           |
| `bug_fix`                | 60           |
| `exploration`            | 120          |
| `feature_implementation` | 180          |
| `multi_file_generation`  | 200          |
| `refactor`               | 240          |

Type detection mirrors `opencode-task-memory` ("Determine the task type via heuristic on the initial prompt"):

- "implement", "add", "implemente", "ajoute" → `feature_implementation`
- "fix", "correct", "bug", "corrige" → `bug_fix`
- "refactor", "restructure", "ameliore" → `refactor`
- "explain", "find", "analyse", "explique", "trouve" → `exploration`
- "generate N files", "scaffold", "crée plusieurs fichiers" → `multi_file_generation`
- "what / which / how (short)", "c'est quoi" → `question`
- Otherwise → `feature_implementation` (the most conservative default for routing)

### Memory-aware estimation

If `<workspace>/.opencode-agent-memory.json` exists (see `opencode-task-memory` skill for the JSON schema, especially `byType.<type>.avgDurationSec`):

```
estimatedSec = byType[type].avgDurationSec × confidence_factor
```

Where `confidence_factor` ∈ `[0.5, 2.0]` reflects similarity to past tasks on this project:

| Similarity signal                                                              | Factor |
|--------------------------------------------------------------------------------|--------|
| `byType[type].count >= 10` AND prompt keywords match a recent pattern          | 0.8    |
| `byType[type].count >= 5` AND same module / file area                          | 1.0    |
| `byType[type].count >= 5` BUT different module                                 | 1.3    |
| `byType[type].count < 5` (low sample)                                          | 1.5    |
| `byType[type].count == 0`                                                      | use baseline |
| Antipattern hit (prompt matches an entry in `antipatterns[]`)                  | 2.0    |

Cap the final estimate at 1800s (30 min) — beyond that, prefer scheduled-task or split the work.

## 2. Tool routing

| Estimated duration | Tool                                                       | Rationale                                                                                 |
|--------------------|------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| < 30 s             | `opencode_ask`                                             | One-shot, no session continuity needed.                                                   |
| 30 s – 60 s        | `opencode_run` (`maxDurationSeconds = est × 1.5`, min 60)  | Synchronous, well below the ~180s MCP transport timeout.                                  |
| 60 s – 180 s       | `opencode_run_streaming` (when available, since wrapper 1.11.0; else `opencode_run`) | Synchronous with live progress; emits `SESSION_HANG` on timeout instead of generic error. |
| > 180 s            | `opencode_fire` + announce monitoring plan (`opencode_check` next turn) | Avoids the MCP transport timeout entirely.                                                |
| > 600 s            | `opencode_fire` + recommend a scheduled-task wrap         | Long-running work should not block a conversation.                                        |

Edge case — wrapper version detection: if `opencode_run_streaming` is not in the tool surface (consult `opencode_tool_list` or `opencode_setup`), fall back to `opencode_run` for the 60–180s band and note this in dev-mode output.

## 3. Local idempotency (60-second dedup window)

Before dispatching, compute:

```
prompt_hash = sha256(sessionId + "\n" + prompt + "\n" + floor(now_ms / 60_000))
```

Maintain an in-conversation `Map<prompt_hash, dispatched_at>` (Cowork session-scoped, no persistence — it resets at conversation end, which is intentional).

- If the same `prompt_hash` already exists and `now - dispatched_at < 60s`: **refuse the dispatch**. Tell the user:
  > "I'm about to dispatch the exact same prompt to OpenCode session `<sessionId>` for the second time within 60 seconds. This is likely an accidental double-tap. The original dispatch is still in flight — check it with `opencode_check`. If you really want to re-dispatch, wait 60s or change one character in the prompt."
- Otherwise: record `prompt_hash` and proceed.

This layer is **independent of the wrapper's MEK-284 idempotency dedup** and runs first. It catches the Cowork-side accidental double-fire pattern (user clicks twice, hot-key fires twice, two parallel routings of the same chat turn). It does NOT replace MEK-284 — wrapper-level dedup also defends against HTTP retry storms.

## 4. Plan-phase announcement (ReAct)

Standard mode (one-liner inside the orchestrator's Plan block):

> Routing: `<type>`, estimated `<N>s` (baseline / memory-derived), tool=`<tool_name>`, sessionId=`<new|reused>`, idempotency=`OK / duplicate-refused`.

Dev mode (under "show details"):

```
Routing decision (opencode-timeout-router):
  type:                feature_implementation
  baseline:            180s
  memory factor:       1.0 (count=12, same module match)
  estimated:           180s
  tool selected:       opencode_run_streaming
  maxDurationSeconds:  270 (est × 1.5)
  sessionId:           ses_xxx (new — no parent)
  prompt_hash:         a83c…b91d (recorded)
  fallback if 429:     opencode-fallback-chain → next provider
```

Never dispatch silently. Always show the routing line.

## 5. Reflect-phase: write back to memory

After the dispatched task reaches `idle` / `finished` (see `opencode-result-validator` for the validation gate):

1. Compute `actualDurationSec = finishedAt - dispatchedAt` (use `opencode_session_get` for timestamps).
2. Update `byType[type].avgDurationSec` in `.opencode-agent-memory.json` using the running-average formula from `opencode-task-memory`:
   ```
   newAvg = (oldAvg * (count - 1) + actualDurationSec) / count
   ```
3. If `actualDurationSec > 2 × estimatedSec`: this skill made a bad call. Record it as an internal stat (planned: a new `routerStats.misestimates` field in v1.2) so future runs apply a higher confidence_factor to similar prompts.
4. If a timeout actually fired (see §6 below), increment `routerStats.timeouts.<type>` (when the field exists).

This is the Reflect step of Plan-Execute-Reflect. The memory writes are delegated to `opencode-task-memory` — this skill **calls** it, never reimplements the JSON writes.

## 6. Unexpected-timeout handling

If the dispatched tool returns a timeout that the routing did not anticipate (e.g. MCP-transport ~180s on a task estimated at 60s, or `SESSION_HANG` from `opencode_run_streaming`):

1. **Do not retry on the same session.** The session may be busy, half-completed, or holding stale state. Following `opencode-orchestrator`'s orphan-recovery section: call `opencode_sessions_overview` + `opencode_check` to determine if the work is still running server-side.
2. **If the work is still running**: switch to fire-and-forget mode (`opencode_fire` semantics) — announce to the user that the original session will be polled next turn. Do not duplicate the work.
3. **If the work is genuinely stuck**: create a **new sessionId** via `opencode_session_create` and resubmit. NEVER reuse the original sessionId for the retry — the wrapper's session state is suspect once a timeout fires.
4. **Record** the timeout against the original `(type, model, provider)` triple in memory so the next estimate inflates by 2× for the same shape of work.

The "new sessionId on timeout" rule defends against the dedup over-application bug class observed in opencode-mcp #27 and the MEK-284 family: a stale session combined with retries amplifies the wrong cached state.

## 7. Composability with user MCPs

If the user has an MCP that already routes by duration (a custom LLM gateway, a Relkhon-style dispatcher), follow `opencode-mcp-discovery`'s composability pattern: on the first run per project, offer:

- (a) **Substitution**: the user MCP replaces this skill's routing — `opencode-timeout-router` becomes pass-through (still applies the 60s local idempotency, which is gateway-orthogonal).
- (b) **Chaining**: this skill routes first; the user MCP refines if its heuristics differ. Persist the decision in `.opencode-agent-memory.json` under `mcpComposition.timeoutRouter`.
- (c) **Skill-only** (default if no user MCP detected).

## 8. When to retire this skill

- **Local idempotency (§3)**: becomes redundant once opencode-mcp's MEK-284 fix (issue #27 in the wrapper) is universally deployed and the dedup helper correctly keys on request body. Until then, this layer catches the gap. Keep as **defense-in-depth** even post-fix — the Cowork-side dedup catches user-side double-taps that the wrapper cannot see.
- **Routing (§2) and estimation (§1)**: stay valuable independent of wrapper fixes. They are the operational embodiment of Plan-Execute-Reflect for OpenCode dispatches. No retirement planned.

## Inter-skill interactions

- **`opencode-orchestrator`**: calls this skill in the Plan phase, before formulating the dispatch. The orchestrator owns the conversation, this skill owns the tool selection and idempotency check.
- **`opencode-safe-prompts`**: runs *after* this skill. The routing decision and idempotency check happen first; if the prompt is going to be dispatched, *then* safe-prompts scans it for risky patterns.
- **`opencode-task-memory`**: this skill reads `byType.*.avgDurationSec` to calibrate, and triggers memory writes after Reflect. All JSON I/O is delegated to task-memory.
- **`opencode-fallback-chain`**: this skill does NOT handle provider errors (429, 5xx, EMPTY_RESPONSE). Those are fallback-chain's job. If a dispatch fails with a provider error, fallback-chain retries on the next provider — and this skill is invoked again for the retry to compute a fresh `prompt_hash` (which will differ because of the timestamp_minute component) and routing.
- **`opencode-result-validator`**: runs *after* the dispatch returns. This skill's Reflect-phase write happens conditionally on the validator's verdict (only write memory if the result is sound, to avoid poisoning the averages with failed runs).

## Known limitations

- **Type detection is keyword-based and English/French only.** Other languages fall through to the conservative `feature_implementation` default — risking over-estimation. Acceptable for MVP.
- **Local idempotency is session-scoped**, not cross-session. Two distinct Cowork conversations could re-dispatch the same `(sessionId, prompt)` within 60s of each other. The wrapper's MEK-284 dedup is the safety net for that case.
- **Memory writes can race** if two parallel dispatches finish simultaneously. The task-memory skill's overflow management (FIFO, 50KB cap) prevents corruption; the resulting average may be slightly skewed by ordering. Acceptable for operational stats.
- **No automatic detection of `opencode_run_streaming` availability per server.** Falls back to `opencode_run` if the streaming tool is absent — but this requires a `opencode_tool_list` call on first dispatch per server. Cache the result for the conversation.

## References

- Issue: [opencode-agent #37](https://github.com/MekaretEriker/opencode-agent-for-cowork/issues/37) (migrated from Linear MEK-285).
- Mitigated wrapper bugs: [opencode-mcp #27](https://github.com/MekaretEriker/opencode-mcp/issues/27) (MEK-284 dedup over-application — the local idempotency layer is defense against accidental Cowork-side double-fires that compound the wrapper bug); [opencode-mcp #26](https://github.com/MekaretEriker/opencode-mcp/issues/26) (`EMPTY_RESPONSE` silent-fail — the timeout-handling section §6 ensures we surface and retry rather than silently consume the empty response).
- Twin idempotency concern: MEK-284 (wrapper-side dedup). This skill's §3 is the Cowork-side counterpart.
- Handoff: future preflight-provider work (MEK-286) — once that ships, the routing decision can also account for provider health, not just task duration.
- Design references: `DESIGN-opencode-agent-for-cowork.md` v8 §5.1 (routing pattern) and §5.2 (memory schema) — note: these files were not present in this repo at the time the skill was authored; the schema is reproduced from `opencode-task-memory/SKILL.md` and the routing pattern is reproduced from `opencode-orchestrator/SKILL.md` §2 recap table. If the DESIGN doc is added to this repo later, cross-link from this section.
- Spec: `SPEC-plugin.md` Composant 1 (not present in this repo; same note as above).

## Inspirations

- Activity-based timeout pattern from `NousResearch/hermes-agent` (issue #4815, PR #4864) — already cited in `opencode-orchestrator`'s stall heuristic; this skill applies the same philosophy at routing time rather than at polling time.
- Idempotency key + time-window pattern from `swarm-code-plugin` (apoapps) and the Stripe API conventions — adapted to a Cowork-session scope rather than HTTP request scope.
- Plan-Execute-Reflect framing from "Building Effective Agents" (Anthropic, late 2024): this skill is the Plan-phase enforcer for OpenCode dispatches.
