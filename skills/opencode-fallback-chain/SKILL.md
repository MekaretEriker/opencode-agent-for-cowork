---
name: opencode-fallback-chain
description: "Use this skill when an OpenCode dispatch fails with a provider error (HTTP 429 rate limit or 5xx server error). Retries the task with the next provider in the configured fallback chain."
---

# opencode-fallback-chain

## Nature and scope

Instructional skill. When an OpenCode call (`opencode_run` / `_ask` / `_fire`) returns an error indicating a provider failure (429 rate limit, 5xx server error, "provider unavailable", etc.), you retry the task with the next provider in the configured chain, after a short 5-second backoff. No infinite retries: 3 attempts maximum, each with a different provider. If all 3 fail, you abort cleanly.

This skill does not handle MCP transport errors (180s timeout, broken connection) — those are the orchestrator's responsibility.

## Composability with user MCPs

If the user has an MCP with its own retry logic (typically a custom LLM gateway that already routes calls between providers), that MCP should handle the fallback, not this skill. On the first plugin launch per project, the skill offers the user three options:

1. **Substitution**: the custom MCP replaces the skill's fallback chain (the skill becomes pass-through)
2. **Chaining**: the skill's chain runs first, and the custom MCP acts as a safety net of last resort
3. **Skill-only**: disable the custom MCP's fallback and let the skill handle everything (default behavior if no custom MCP is detected)

This behavior is documented in the design doc at §6.2.2.

## Triggers

Error patterns to detect in OpenCode responses (in the `error` field or the textual response message):

- `"rate limit"`, `"429"`, `"TooManyRequests"`
- `"5xx"`, `"500"`, `"502"`, `"503"`, `"504"`, `"internal server error"`
- `"provider unavailable"`, `"service unavailable"`
- `"timeout to upstream"` (to distinguish from the MCP transport timeout of 180s, which is handled by the orchestrator)

Do **not** retry on client error codes:

- **400**: invalid or malformed prompt — retry is useless, the problem is in the request
- **401 / 403**: missing or refused authentication — switching providers won't solve this
- **MCP transport timeout (~180s)**: handled by the orchestrator with the orphaned session recovery pattern, not by this skill

When in doubt about the nature of the error (ambiguous message that could be a disguised 5xx), apply the precautionary principle and attempt the fallback — a useless retry is less costly than a premature abort.


## Structured error codes (opencode-mcp 1.11.0+)

Since opencode-mcp 1.11.0 (MEK-282), every error response from a tool includes both a human-readable line **and** a machine-parsable JSON block embedded in an HTML comment:

```
Error [RATE_LIMITED]: Rate limit exceeded (HTTP 429)

**Suggestion:** Wait and retry, or switch provider/model

<!-- structured-error
{
  "code": "RATE_LIMITED",
  "message": "Rate limit exceeded",
  "raw": { "httpStatus": 429, "body": "..." },
  "provider": "anthropic",
  "modelIdRequested": "claude-opus-4-6",
  "sessionId": "ses_xxx",
  "suggestedAction": "Wait and retry, or switch provider/model"
}
-->
```

When this JSON block is present, **prefer parsing it over regex-matching the error message** — codes are stable across locale and wording changes. Mapping for fallback decisions:

- `RATE_LIMITED` -> trigger fallback (chain to next provider)
- `PROVIDER_ERROR` with `raw.httpStatus >= 500` -> trigger fallback
- `AUTH_FAILED` -> **do not fallback** (re-auth needed, not a provider problem)
- `TIMEOUT` or `SESSION_HANG` -> orchestrator's responsibility (orphaned-session recovery), not this skill
- `EMPTY_RESPONSE` -> trigger fallback (degenerate model output)

Regex matching on the human-readable line remains as a safe fallback for any error without a structured block (older opencode-mcp versions or non-wrapped tool errors).

## Default chain

If no chain is explicitly configured by the user, the default chain is:

```
anthropic -> openrouter -> openai
```

If the user has configured providers via `opencode_provider_list`, the skill uses those providers in their listing order (generally by decreasing priority, as defined in the user's OpenCode configuration).

## User configuration

The environment variable `OPENCODE_AGENT_PROVIDER_CHAIN` allows overriding the default chain. Format: comma-separated list of provider names.

Example:

```
OPENCODE_AGENT_PROVIDER_CHAIN=anthropic,deepseek,openrouter
```

This variable is read at session startup via `opencode_setup` or `opencode_context`. If the variable is absent or empty, the default chain applies.

Another recognized environment variable (future iteration):

```
OPENCODE_AGENT_FALLBACK_BACKOFF_SEC=10
```

In MVP (v0.5.0), the backoff is fixed at 5 seconds. Configurable backoff is planned for a future iteration if field feedback justifies it.

## Procedure

1. **Detect** the error pattern in the OpenCode response (see Triggers section)
2. **Identify** the provider that failed via the current `sessionId` and a call to `opencode_session_get`
3. **Find** the next provider in the configured chain (user chain if defined, otherwise default chain)
4. **Announce** in standard mode: "Provider X returned [error type]. Retrying with Y, please wait 5 seconds..."
5. **Wait** 5 seconds (fixed backoff)
6. **Re-dispatch** the task with the new provider by passing an explicit `providerID` in the next `opencode_run`, `opencode_ask`, or `opencode_fire` call
7. **If 3 consecutive failures** across 3 different providers: abort and present a clear error to the user ("All 3 providers in the chain failed: [details]. Check your connectivity and API keys."). Do not automatically retry.
8. **On success**: update `task-memory` (if the task-memory skill is active, v0.3.0+) by incrementing `providerStats.<previousProvider>.failureCount` to adjust the orchestrator's future choices.

## Limitations

- **No mid-session switch for an active OpenCode session.** The fallback applies to the **next dispatched task**, not to an already active session. If a session is running and its provider fails mid-execution, the fallback cannot resume the session — the entire task must be re-launched with a new provider.
- **If all configured providers fail**, the error is surfaced to the user. No infinite fallback or silent loop.
- **The 5-second backoff is fixed in MVP.** A future iteration may make it configurable via `OPENCODE_AGENT_FALLBACK_BACKOFF_SEC` if user feedback justifies it.
- **The skill makes no distinction between providers.** It follows the chain order, without cost, latency, or regional preference logic. These optimizations are out of scope for the MVP.

## Inter-skill

- **orchestrator**: calls this skill when a provider error is detected after a dispatch (`opencode_run` / `_ask` / `_fire`). The orchestrator remains master of the flow: it decides to re-dispatch with the new provider.
- **task-memory**: this skill writes to the `providerStats` field (incrementing `failureCount` per provider) so the orchestrator can adjust its future choices — for example, temporarily avoiding a frequently failing provider.
- **result-validator**: no direct interaction. The result-validator intervenes after successful execution, i.e., after the fallback-chain has potentially switched providers. Both skills operate at different stages of the pipeline.

## Inspirations

- Provider fallback pattern from [hermes-agent](https://github.com/NousResearch/hermes-agent) (NousResearch): retry with next provider on transient error, configurable backoff
- x3 retry with exponential backoff from [swarm-code-plugin](https://github.com/apoapps/swarm-code-plugin)
- Cowork adaptation: skill-as-instruction, not auto-runtime. The skill provides the procedure, the Cowork orchestrator executes it — no invisible automation.
