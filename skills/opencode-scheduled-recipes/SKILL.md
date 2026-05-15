---
name: opencode-scheduled-recipes
description: "Use this skill when the user asks about recurring tasks (daily digest, weekly audit, monthly scan) or wants to schedule automated OpenCode tasks. Documents the 3 bundled recipes and how to customize them."
---

# opencode-scheduled-recipes

## Nature and scope
Documentary skill that teaches Cowork how to propose and configure the scheduled tasks bundled with the plugin. The plugin provides 3 ready-to-use recipes in `scheduled-tasks/`:

| Recipe | Frequency | Agent | Output |
|---|---|---|---|
| daily-repo-digest | Every day at 7am | plan | Summary of commits from the last 24h |
| weekly-deps-audit | Monday at 9am | build | Markdown report in reports/ |
| monthly-refactor-scan | 1st of month at 9am | plan | 3 identified refactor areas |

## Activating recipes
Cowork has a native scheduled tasks system. When the user installs this plugin, the 3 recipes are **available but not activated** by default. To activate:
1. The user says "active le digest quotidien" / "activate the daily digest" or "schedule the deps audit"
2. You invoke the Cowork scheduled tasks mechanism (see native `mcp__scheduled-tasks__create_scheduled_task`) with:
   - The recipe's cronExpression
   - The recipe's prompt
   - A note "via opencode-agent recipe X"

## Customization
If the user wants to adjust:
- Frequency: modify the cron expression
- Target repo: adapt the prompt with `cd /path/to/repo &&` as prefix (directory workaround, see orchestrator §Technical notes)
- Notification: `notifyOnComplete: true` sends a desktop notification when done

## Inter-skill
- **orchestrator**: delegates to this skill when the user mentions a recurring task
- **safe-prompts**: scans scheduled prompts like any other dispatch
- **result-validator**: applies if the recipe uses `agent: build`
- **task-memory**: enriches memory as scheduled executions accumulate

## Example: activating the daily digest
User: "Active le digest quotidien sur Relkhon_VerticalSlice" / "Activate the daily digest on Relkhon_VerticalSlice"
Cowork:
1. Reads `scheduled-tasks/daily-repo-digest.json` (read to access the recipe)
2. Adapts the prompt: "cd /mnt/d/Projects/Relkhon_VerticalSlice && [original prompt]"
3. Calls `mcp__scheduled-tasks__create_scheduled_task` with cron `0 7 * * *` and the adapted prompt
4. Confirms to the user: "Digest activated, first run tomorrow at 7am."

## Limitations
- Scheduled tasks do not share conversation context — each run is independent
- Requires OpenCode to be installed and accessible at the time of the scheduled run
- Notifications depend on the user's OS (Cowork desktop)
- No cron on pure Windows without WSL — on Windows use the native Task Scheduler or install WSL

## Inspirations
- Hermes cron scheduler (RELEASE_v0.8.0 notify_on_complete)
- Native Cowork mcp__scheduled-tasks__
- Pattern: JSON recipes + instructional skill = easy to customize without recoding the plugin
