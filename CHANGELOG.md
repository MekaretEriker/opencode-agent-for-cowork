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

- New skill `opencode-mcp-discovery` : detecte au 1er run par projet les MCPs user-side (RAG, validate, manage_adr) et propose substitution/chainage/skill-only
- Decision de composition persistee dans `.opencode-agent-memory.json` (champ `mcpComposition`)
- Re-discovery possible sur demande explicite

### Milestone

Cette version marque la **completion du Full scope** du design doc :
- 8 skills Full
- 3 scheduled-tasks bundlees (daily-repo-digest, weekly-deps-audit, monthly-refactor-scan)
- 1 agent custom (cowork-with-github)
- Composabilite avec MCPs utilisateur formalisee
- 11 decisions Q1-Q11 satisfaites

## [0.7.0] - 2026-05-15

### Added

- 3 scheduled-tasks bundlees : `daily-repo-digest`, `weekly-deps-audit`, `monthly-refactor-scan`
- Skill `opencode-scheduled-recipes` qui documente l'activation et la customisation
- Integration avec le mecanisme natif Cowork `mcp__scheduled-tasks__*`

## [0.6.0] - 2026-05-15

### Added

- Skill `opencode-agent-roster` qui livre l'agent custom `cowork-with-github` (extension `build` + GitHub MCP)
- Template `agent-templates/cowork-with-github.json`
- Ecriture AGENTS.md opt-in par projet avec section delimitee par balises HTML
- Stockage opt-in dans `.opencode-agent-memory.json` (champ `agentsMdWriteEnabled`)

## [0.5.0] - 2026-05-15

### Added

- Skill `opencode-fallback-chain` : retry automatique avec provider suivant en cas de 429/5xx
- Chaine par defaut `anthropic -> openrouter -> openai`
- Override via env `OPENCODE_AGENT_PROVIDER_CHAIN`
- 3 retries max, 5s backoff

## [0.4.0] - 2026-05-15

### Added

- Skill `opencode-result-validator` : pattern Plan-Execute-Reflect (3e step)
- 4 checks systematiques : diff sain, tests, build, coherence semantique
- 3 modes de sortie : succes complet, reserves, echec critique

## [0.3.0] - 2026-05-15

### Added

- Skill `opencode-task-memory` : memoire operationnelle JSON par projet (`<workspace>/.opencode-agent-memory.json`)
- Stats par type de tache, par provider, patterns/antipatterns appris
- Propose `.gitignore` au 1er run

## [0.2.0] - 2026-05-14

### Added

- Skill `opencode-safe-prompts` : advisor patterns dangereux (5 categories : filesystem destructif, SQL destructif, RCE, git destructif, secrets/exfiltration)
- Pattern advisor (pas firewall), coherent avec layered safety Q5

## [0.1.1] - 2026-05-14

### Fixed

- Patch handling timeout MCP transport ~180s
- Section 2 du SKILL.md orchestrator : abaissement seuil `opencode_run` de "1-15 min" a "moins de 2-3 min"
- Au-dela : `opencode_fire` + `opencode_check` au prochain tour
- Nouvelle sous-section "Recuperation de session orpheline" : pattern de recuperation quand le MCP a deconnecte mais la session OpenCode continue cote serveur
- Inspiration : hermes-agent activity-based timeout (issue #4815, PR #4864)

## [0.1.0] - 2026-05-14

### Added — MVP initial

- Plugin scaffolding (manifest, .mcp.json, README)
- Skill `opencode-orchestrator` complet avec 8 sections :
  - Cadre general (philosophie orchestrator-workers)
  - ReAct (explicitation avant dispatch)
  - Choix outil (ask / run / fire)
  - Choix agent (build / plan / @general)
  - Heuristique stall pendant pilotage actif
  - Politique de parallelisation (max 3 concurrentes)
  - Install assistee OpenCode au 1er run
  - Lecture AGENTS.md du repo cible
  - Format resultat (mode standard vs mode dev, progressive disclosure)
  - Notes techniques (workaround `directory` opencode-mcp)
- Connecteur opencode-mcp avec caret pin `^1.10.1`
- 11 decisions Q1-Q11 du design doc satisfaites pour le scope MVP
