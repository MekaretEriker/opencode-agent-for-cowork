---
name: opencode-safe-prompts
description: "Use this skill before dispatching any prompt to OpenCode that involves shell commands, file modifications, SQL operations, git destructive ops, or remote execution. Scans prompts for dangerous patterns (rm -rf, DROP TABLE, curl | sh, --force, etc.) and warns the user before execution."
---

# opencode-safe-prompts

## Nature and scope

This skill is an **advisor** on the Cowork side. Before dispatching a prompt to OpenCode via `opencode_run` / `opencode_run_streaming` / `opencode_ask` / `opencode_fire`, you scan the prompt (and any reformulation you made) for risky patterns. If you detect any, you **clearly announce it to the user** and ask for explicit confirmation.

**This is NOT a firewall.** You do not block automatically. You inform and let the user decide. Final responsibility remains with the user. Consistent with design doc decision Q5 (layered safety: opencode-mcp `permission: allow` + agent `plan` + safe-prompts advisor).

## Composability with user MCPs

If a user MCP exposes a `validate` tool or equivalent (typically a Relkhon-style vault with semantic cross-spec validation), that MCP should handle the fine-grained audit. This skill only covers generic patterns (destructive shell, destructive SQL, remote execution). See DESIGN doc section 6.2.2 for the composition pattern.

On the first run on a project, if you detect a user-side MCP with a `validate` tool, offer the user:
- (a) disable safe-prompts in favor of the user MCP (more precise for their domain)
- (b) chain both (safe-prompts first for generic patterns, user MCP next for domain checks)
- (c) keep only safe-prompts

## When this skill triggers

Before sending a prompt to OpenCode that potentially contains:
- An explicit or expected shell command
- A file write operation
- A database operation
- A destructive git operation
- A remote script execution

If the prompt is purely read/analysis (`opencode_ask` with `plan` agent, or phrases like "explain", "find", "analyse") -> you can skip this skill.

## Pattern scan table

### Destructive filesystem

| Pattern (case-insensitive regex) | Category | Action |
|---|---|---|
| `rm\s+-rf?\s+/(?!\w)` | recursive root deletion | Critical warning + explicit confirmation |
| `rm\s+-rf?\s+~` | home deletion | Critical warning + explicit confirmation |
| `rm\s+-rf?\s+\$HOME` | home variable deletion | Critical warning + explicit confirmation |
| `>\s*/dev/sd[a-z]` | raw disk write | Critical warning + block by default |
| `mkfs\.\w+` | reformatting | Critical warning + explicit confirmation |
| `dd\s+.*of=/dev/` | dd disk write | Critical warning + explicit confirmation |

### Destructive SQL

| Pattern | Category | Action |
|---|---|---|
| `DROP\s+(TABLE\|DATABASE\|SCHEMA)` | schema destruction | Warning + confirmation |
| `TRUNCATE\s+TABLE` | table wipe | Warning + confirmation |
| `DELETE\s+FROM\s+\w+\s*;?\s*$` (without WHERE) | delete without where | Critical warning + confirmation |

### Remote execution

| Pattern | Category | Action |
|---|---|---|
| `curl\s+.*\|\s*(bash\|sh\|zsh)` | RCE pattern | Warning + confirmation |
| `wget\s+.*\|\s*(bash\|sh\|zsh)` | RCE pattern | Warning + confirmation |
| `(curl\|wget)\s+.*\|\s*python` | remote python exec | Warning + confirmation |

### Destructive git

| Pattern | Category | Action |
|---|---|---|
| `git\s+push\s+.*--force(\s\|$)` | force push | Warning + confirmation |
| `git\s+push\s+.*--force-with-lease` | force-with-lease (lower risk) | Note only, no blocking |
| `git\s+reset\s+--hard` | hard reset | Warning |
| `git\s+clean\s+-fd` | force clean | Warning |
| `git\s+branch\s+-D` | force branch deletion | Warning |
| `git\s+rebase\s+.*(master\|main)` | rebase main | Warning + suggestion to use PR |

### Secrets and exfiltration

| Pattern | Category | Action |
|---|---|---|
| `cat\s+.*\.env` (in copy/exposure context) | secrets read | Warning |
| `cat\s+~/.ssh/id_` | SSH key read | Critical warning |
| `(echo\|printf)\s+.*\|\s*(curl\|nc\s)` | potential exfiltration | Warning |

## Procedure when a pattern matches

1. **Do not dispatch immediately** to OpenCode
2. Announce to the user (standard mode):

   ```
   Warning: your prompt contains a risky pattern ([category]).
   Detected: "[matched pattern excerpt]"

   What this could do: [non-technical explanation of the potential impact].

   Do you still want me to dispatch to OpenCode? Type 'yes' to confirm, 'no' to cancel, or rephrase the prompt.
   ```

3. Wait for explicit confirmation ('yes' or clear equivalent)
4. If confirmed -> dispatch
5. If no or rephrasing -> follow the new direction

## Edge cases

### Pattern in a spec, not a command
If the user says "implement the migration `DELETE FROM users WHERE last_login < '2020'`", the `DELETE` is *in the spec*, not a direct command to execute immediately. You can skip the warning if:
- The pattern is explicitly framed as intent/specification
- The user has already run a pre-check on their query (mentions tests, dry-run, etc.)
- Context indicates it's documentation or planning

When in doubt, a false positive is better than a disaster -> warn anyway.

### Repeated tasks in the same conversation
If the user has already confirmed a pattern in the current conversation, you don't need to re-ask every time for the exact same pattern. But always re-warn for critical patterns (`rm -rf /`, `DELETE FROM` without WHERE, `/dev/sd*` writes) at each occurrence.

### Dev mode override
If the user explicitly activates dev mode and says "skip safety", you can disable non-critical warnings for the current session. You confirm ONCE at the start:

> "OK, I'm disabling non-critical safe-prompts for this conversation. Critical patterns (rm -rf /, dd to /dev/, DELETE without WHERE) will still be warned. You're in control."

## Canonical inspirations

Patterns inspired by:
- `apoapps/swarm-code-plugin`: ToS-compliant audit, non-blocking advisor pattern
- `tools/approval.py` from hermes-agent (NousResearch): dangerous pattern table with regex
- But reformulated to be **advisors** on the Cowork side, not blocking hooks on the server side (see design doc Q11 - no plugin hooks in MVP, skill-as-instruction)

## Known limitations

- Regexes don't cover everything. A determined user can bypass with obfuscations (`r''m -rf /` or env variables).
- No security guarantee — this is a safety net, not a wall.
- The user remains responsible. This skill reduces accidents, it doesn't prevent them.
- Warnings consume tokens and time. If the user frequently hits borderline patterns (legitimate SQL analysis for example), dev mode override is the solution.
