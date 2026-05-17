# Cowork operational guide — `opencode-agent` (Cowork plugin)

This file documents the **release management, versioning, plugin packaging,
and issue tracking conventions** for Claude operating in Cowork mode on this
repo. For engineering invariants (skill conventions, `.mcp.json` structure,
OpenRouter model roster, SDK source-of-truth rule), see `AGENTS.md`.

## Quick reference — engineering rule that applies to Cowork too

When YOU (Cowork's Claude) write or refactor skills, scheduled tasks, or
any code that touches the opencode HTTP API in this repo, the **"Source of
truth — opencode SDK first, Hermes reference second"** rule applies. The
full rule lives in `AGENTS.md` (this repo) and `../opencode-mcp/AGENTS.md`
(the wrapper). Before inventing an HTTP route, an SSE pattern, or a new
opencode-mcp tool wrapper, consult the SDK (https://opencode.ai/docs/sdk.md)
and the wider ecosystem (`zaycruz/hermes-opencode-plugin`, `awesome-opencode`).

## Release workflow

**No CI automation** — releases are manual. A reference end-to-end script lives
in the working tree (untracked, gitignored) as `release-vX.Y.Z.sh` — keep the
most recent one as a template for the next bump.

The ritual:

1. **Bump `.claude-plugin/plugin.json`** version. Semver:
   - Minor (`1.0.x` → `1.1.0`) — new skill, new exposed `opencode-mcp` tool
     wired into a skill, new bundled scheduled task. Anything visible to users.
   - Patch (`1.0.3` → `1.0.4`) — dependency bump only, skill content fix,
     documentation. Anything transparent.
2. **Bump `.mcp.json`** dep range if consuming a new `opencode-mcp` minor —
   see "Semver gotcha" below.
3. **Update `CHANGELOG.md`** with a dated entry (`## [X.Y.Z] - YYYY-MM-DD`).
4. **Update skills** when a new `opencode-mcp` tool ships — see "Skill update
   checklist" in `AGENTS.md`.
5. **One commit**: `release: vX.Y.Z — short subject`.
6. **Tag and push**:
   ```bash
   git tag vX.Y.Z
   git push origin main
   git push origin vX.Y.Z
   ```
   `--follow-tags` skips lightweight tags. Push the tag explicitly.
7. **Build the `.plugin`**:
   ```bash
   zip -r opencode-agent-vX.Y.Z.plugin \
     .claude-plugin .mcp.json skills scheduled-tasks
   ```
8. **Create the GitHub Release** manually at
   `https://github.com/MekaretEriker/opencode-agent-for-cowork/releases/new?tag=vX.Y.Z`.
   Title: `vX.Y.Z — <subject>`. Body: copy-paste the `## [X.Y.Z]` section from
   `CHANGELOG.md`. Attach `opencode-agent-vX.Y.Z.plugin`. Publish.

The `.plugin` file is in `.gitignore` (built from sources). Do not commit the
zip.

**TODO**: write a `release.yml` GitHub Actions workflow that auto-builds the
zip and creates the release on tag push (analogous to the one in
`opencode-mcp`). Eliminates steps 7-8 manual work.

## Semver gotcha (`.mcp.json`)

`opencode-mcp` uses prerelease tags (`-mekareteriker.N`). npm's caret
semantics with prereleases are **strict**: `^1.10.2-mekareteriker.0` matches
other prereleases of `1.10.2` only — it does NOT match `1.11.0-mekareteriker.0`.

When `opencode-mcp` ships a new minor, `.mcp.json` must be explicitly bumped:

```json
"args": ["-y", "@mekareteriker/opencode-mcp@^1.11.0-mekareteriker.0"]
```

This caught us on `v1.0.3 → v1.1.0`. The plugin had been pinned to the old
minor and was not picking up the new `opencode_run_streaming` tool until the
range was updated.

## Issue tracking — GitHub Issues + GitHub Projects v2

Issues live on the GitHub repo (`MekaretEriker/opencode-agent-for-cowork/issues`)
and roll up on the user-level project board:
**https://github.com/users/MekaretEriker/projects/2**.

The sibling `opencode-mcp` repo feeds the same board. Both repos tracked
work in Linear under `MEK-XXX` identifiers prior to the migration; historical
references in CHANGELOG entries, skills, and commit history are preserved
as archive — do not rewrite them.

**Conventions for new work:**

- **Issue refs**: `#N` for issues in this repo, `MekaretEriker/opencode-mcp#N`
  for the wrapper.
- **Magic keywords for auto-close**: `Closes #N`, `Fixes #N`, or
  `Resolves #N` anywhere in the commit body. GitHub native — fires on push
  to the default branch OR on PR merge. No third-party babysit.
- **Bare `#N` mentions** create a link but do NOT close the issue.
- **Skill bumps / dep range bumps** that reference a wrapper-side fix
  should cite the GitHub commit or release URL in the CHANGELOG entry,
  not the issue number (issue numbers may not exist in the cross-repo
  context if work was scoped only on the other side).

**Project board automation** (configured in the Project UI):

- New issues opened in either feeding repo → auto-added with status `Todo`.
- Closed issues → status auto-`Done`.
- Custom fields (Phase, Priority, Effort) on the board, not in issue bodies.

**Historical migration note**: a Linear ↔ GitHub bidirectional integration
was used before this workflow. It is no longer active. If you encounter a
`Closes MEK-XXX` reference in a recent commit, treat it as legacy and
adjust the corresponding GitHub Issue's project status by hand.
