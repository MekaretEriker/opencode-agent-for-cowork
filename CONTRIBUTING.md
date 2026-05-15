# Contributing to opencode-agent-for-cowork

Merci de t'interesser au projet ! Ce plugin Cowork transforme Cowork en orchestrateur d'OpenCode. Toute contribution est bienvenue.

## Reporting bugs

Ouvre une [issue GitHub](../../issues/new) avec :
- Version du plugin (`cat .claude-plugin/plugin.json | grep version`)
- Version d'opencode-mcp utilisee (cf. `.mcp.json`)
- Version d'OpenCode CLI (`opencode --version`)
- OS / WSL version
- Etapes pour reproduire
- Comportement attendu vs observe

## Proposing a new skill

Le plugin est structure en skills Markdown (cf. `skills/`). Chaque skill est instructionnel — il apprend a Cowork comment se comporter dans un cas precis, ce n'est pas du code executable.

### Anatomie d'un skill

```
skills/<kebab-case-name>/
├── SKILL.md          # OBLIGATOIRE, avec frontmatter YAML
├── scripts/          # OPTIONNEL, scripts annexes
└── ...               # autres ressources si besoin
```

### Frontmatter SKILL.md

```yaml
---
name: <kebab-case-name>
description: "Use this skill when [trigger]. [Brief explanation]."
---
```

La description doit etre **a la 3e personne** et contenir les phrases de declenchement entre guillemets (c'est ce que le router de Claude utilise).

### Style du corps

- Imperatif, instructionnel POUR Claude (pas pour le humain qui lit)
- Section "Nature et portee" en intro
- Section "Composabilite" si pertinent (cf. design doc §13.5)
- Limites assumees a la fin
- En francais ou en anglais coherent avec le reste du plugin

### Tests

Les skills sont teste par usage reel. Ouvre une PR avec :
1. Le nouveau skill
2. Un scenario E2E dans la PR description (prompt utilisateur -> comportement attendu)
3. Bump de version dans `plugin.json` (semver)
4. Note dans CHANGELOG.md

## Style des commits

Convention :
- `feat(vX.Y.Z): description` pour les nouvelles features
- `fix: description` pour les bugfixes
- `docs: description` pour la doc
- `chore: description` pour la maintenance (CI, deps, etc.)

Reference les issues : `Fixes #123` ou `Refs #456` en fin de message.

## Versioning

Le plugin suit [semver](https://semver.org/) :
- MAJOR : breaking change dans les contrats des skills
- MINOR : nouveau skill ou nouvelle capacite
- PATCH : bugfix ou amelioration sans ajout

## Design doc

Le design doc complet (`DESIGN-opencode-agent-for-cowork.md`) explique les decisions architecturales (Q1-Q11), les patterns adoptes (orchestrator-workers, ReAct, Plan-Execute-Reflect), et le scope vise. **Lis-le avant de proposer un changement structurel.**

## Code of Conduct

Sois respectueux. Discussions techniques OK, attaques personnelles non. Cf. [Contributor Covenant](https://www.contributor-covenant.org/).

## License

En contribuant, tu acceptes que ta contribution soit publiee sous la meme license que le projet (MIT).
