# Contributing to opencode-agent-for-cowork

Thanks for your interest in the project! This Cowork plugin turns Cowork into an OpenCode orchestrator. All contributions are welcome.

## Reporting bugs

Open a [GitHub issue](../../issues/new) with:
- Plugin version (`cat .claude-plugin/plugin.json | grep version`)
- opencode-mcp version used (see `.mcp.json`)
- OpenCode CLI version (`opencode --version`)
- OS / WSL version
- Steps to reproduce
- Expected vs observed behavior

## Proposing a new skill

The plugin is structured as Markdown skills (see `skills/`). Each skill is instructional — it teaches Cowork how to behave in a specific situation; it is not executable code.

### Skill anatomy

```
skills/<kebab-case-name>/
├── SKILL.md          # REQUIRED, with YAML frontmatter
├── scripts/          # OPTIONAL, auxiliary scripts
└── ...               # other resources if needed
```

### SKILL.md frontmatter

```yaml
---
name: <kebab-case-name>
description: "Use this skill when [trigger]. [Brief explanation]."
---
```

The description must be **in the third person** and contain trigger phrases in quotes (this is what Claude's router uses).

### Body style

- Imperative, instructional FOR Claude (not for the human reader)
- "Nature and scope" section as intro
- "Composability" section if relevant (see design doc §13.5)
- Known limitations at the end
- In English, consistent with the rest of the plugin

### Testing

Skills are validated through real usage. Open a PR with:
1. The new skill
2. An E2E scenario in the PR description (user prompt → expected behavior)
3. Version bump in `plugin.json` (semver)
4. Entry in CHANGELOG.md

## Commit style

Convention:
- `feat(vX.Y.Z): description` for new features
- `fix: description` for bug fixes
- `docs: description` for documentation
- `chore: description` for maintenance (CI, deps, etc.)

Reference issues: `Fixes #123` or `Refs #456` at the end of the message.

## Versioning

The plugin follows [semver](https://semver.org/):
- MAJOR: breaking change in skill contracts
- MINOR: new skill or new capability
- PATCH: bug fix or improvement without addition

## Design doc

The full design doc (`DESIGN-opencode-agent-for-cowork.md`) explains the architectural decisions (Q1-Q11), adopted patterns (orchestrator-workers, ReAct, Plan-Execute-Reflect), and the intended scope. **Read it before proposing a structural change.**

## Code of Conduct

Be respectful. Technical discussions are welcome, personal attacks are not. See [Contributor Covenant](https://www.contributor-covenant.org/).

## License

By contributing, you agree that your contribution will be published under the same license as the project (MIT).
