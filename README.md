# opencode-agent

> Plugin Cowork qui transforme Cowork en orchestrateur d'OpenCode ([anomalyco/opencode](https://github.com/anomalyco/opencode)).

**Statut** : v0.7.0 — Full scope complete sauf MCP discovery (v1.7 a venir).

## Changelog

### v0.7.0 (scheduled tasks)

- 3 scheduled-tasks bundlees : `daily-repo-digest`, `weekly-deps-audit`, `monthly-refactor-scan`
- Skill documentaire `opencode-scheduled-recipes` qui explique l'activation et la customisation
- Integration avec le mecanisme natif Cowork `mcp__scheduled-tasks__*`
- Recettes non actives par defaut : opt-in explicite par l'utilisateur

### v0.6.0 (agent-roster + AGENTS.md write)

- Nouveau skill `opencode-agent-roster` : utilise build/plan natifs OpenCode + livre l'agent custom `cowork-with-github` (extension build + GitHub MCP)
- Template `agent-templates/cowork-with-github.json` pour install cote OpenCode au 1er run (opt-in)
- Ecriture AGENTS.md opt-in par projet avec section delimitee par balises HTML
- Stockage de l'opt-in dans `.opencode-agent-memory.json` (champ `agentsMdWriteEnabled`)
- Aucune ecriture silencieuse dans le repo utilisateur

### v0.5.0 (skill fallback-chain)

- Nouveau skill `opencode-fallback-chain` : retry automatique avec provider suivant en cas de 429/5xx
- Chaine par defaut anthropic -> openrouter -> openai (override via env `OPENCODE_AGENT_PROVIDER_CHAIN`)
- Max 3 tentatives, backoff 5s entre chaque
- Inter-skill avec task-memory : `providerStats.failureCount` mis a jour
- Composable avec MCPs utilisateur (gateway custom) cf. §6.2.2

### v0.4.0 (skill result-validator)

- Nouveau skill `opencode-result-validator` : implemente le 3e step du pattern Plan-Execute-Reflect
- Active apres taches d'agent `build` (skip pour `ask`/`plan` et taches < 30s)
- 4 checks : diff sain, tests (si detectables), build/compilation, coherence semantique
- 3 formats de sortie : succes complet, reserves (un check rouge), echec critique
- Composable avec MCPs utilisateur exposant `validate` (cf. design doc §6.2.2)
- Inter-skill avec task-memory : antipattern note si validation rouge

### v0.3.0 (skill task-memory)

- Nouveau skill `opencode-task-memory` : memoire operationnelle persistante par projet (`<workspace>/.opencode-agent-memory.json`)
- Stats par type de tache (feature, fix, refactor, exploration), par provider, patterns/antipatterns appris
- Propose `.gitignore` au 1er run pour eviter le commit accidentel
- Distinction sémantique explicitée : memoire operationnelle (ce skill) vs utilisateur (consolidate-memory natif) vs contexte projet (AGENTS.md)
- Composable avec MCPs utilisateur exposant `manage_adr` ou equivalent (cf. design doc §6.2.2)

### v0.2.0 (skill safe-prompts)

- Nouveau skill `opencode-safe-prompts` : advisor qui scanne les prompts avant dispatch a OpenCode et warning l'utilisateur sur les patterns dangereux (5 categories : filesystem destructif, SQL destructif, RCE, git destructif, secrets/exfiltration)
- Composable avec MCPs utilisateur exposant un tool `validate` (cf. design doc §6.2.2)
- Pattern advisor (pas firewall) coherent avec decision Q5 layered safety
- Inspirations : swarm-code-plugin + tools/approval.py de hermes-agent

### v0.1.1 (patch timeout MCP)

- §2 du SKILL.md : seuil opencode_run abaisse de "1-15 min" a "moins de 2-3 min" pour respecter le timeout MCP transport (~180s)
- Au-dela de 3 min : fire+check par defaut (au lieu de attendre une exception)
- Nouvelle sous-section "Recuperation de session orpheline" : pattern pour recuperer une session OpenCode toujours active cote serveur quand le MCP a deconnecte
- §4 : reference au pattern activity-based timeout de hermes-agent (issue #4815)

### v0.1.0 (MVP initial)

- Skill orchestrator (8 sections), MCP opencode-mcp en caret pin ^1.10.1, decisions Q1-Q11 cf. design doc

## Quick start

### Pré-requis

- Cowork installé (desktop app Anthropic)
- OpenCode CLI (auto-installable au 1er run du plugin, voir ci-dessous)
- Connexion internet pour npm (téléchargement de `opencode-mcp` à la volée)

### Installation du plugin

1. Récupère le fichier `opencode-agent.plugin`
2. Drop-le dans le chat Cowork (sur le dossier `outputs/` ou via drag-and-drop)
3. Cowork affiche un rich preview cliquable. Appuie sur **Accept** pour installer.

C'est tout. Au prochain redémarrage de la conversation, le skill `opencode-orchestrator` et le serveur MCP `opencode-mcp` sont disponibles.

### Premier lancement

Demande quelque chose comme :

> "Demande à OpenCode d'analyser le module auth de mon projet."

Si OpenCode n'est pas encore installé sur ta machine, Cowork te le proposera et exécutera la commande après ta validation. Pas d'installation silencieuse.

## Premier usage

### Mode standard (par défaut)

Tu parles en langage naturel, Cowork fait le reste :

> **Toi** : "OpenCode, refactor le module factions pour utiliser un EventBus."
>
> **Cowork** : "Je vais demander à OpenCode de refactor le module factions. C'est une tâche de modification (~5 min estimées), j'utilise l'agent `build` et je synchronise. Lancement..."
>
> *[OpenCode travaille...]*
>
> **Cowork** : "✓ J'ai refactor le module factions vers un pattern EventBus. Fichiers modifiés : `factions.gd` (séparation event/handler), `event_bus.gd` (nouveau), `combat.gd` (subscriptions). Les tests passent. Tu veux que je documente le pattern dans un README ?"

### Mode dev (activé sur demande)

Si tu veux voir le détail technique, demande-le explicitement :

> **Toi** : "Mode dev, montre-moi la session."
>
> **Cowork** : "Session : `ses_xxxxx` (agent: build, provider: anthropic, model: claude-sonnet-4-6). Durée : 4min12s (8 itérations). Outils OpenCode : opencode_find_text (3x), opencode_file_read (12x), opencode_message_send (2x). Diff : factions.gd +47/-23, event_bus.gd +89/-0, combat.gd +12/-4. Pour drill-down : `opencode_conversation({sessionId: \"ses_xxxxx\"})`."

Tu peux activer le mode dev à n'importe quel moment avec : "montre les détails", "mode dev", "explique-toi techniquement", "donne le session ID", etc.

### Tâches en parallèle

Si tu veux lancer plusieurs analyses ou refactors en même temps :

> **Toi** : "Lance en parallèle un refactor sur auth, un audit de sécurité sur payments, et une review de notifications."
>
> **Cowork** : "Trois tâches indépendantes sur modules distincts — parallélisable. Je lance 3 sessions OpenCode en background. Tu peux continuer à me parler, et je te rends le statut quand tu demandes 'où en sont mes sessions OpenCode ?'."

Par défaut le plugin limite à 3 sessions concurrentes (variable d'env `OPENCODE_AGENT_MAX_PARALLEL` pour override).

## Positionnement vs OpenCode Desktop

OpenCode a aussi son propre app desktop ([opencode.ai/download](https://opencode.ai/download)). Ce plugin **n'est pas un remplacement** — c'est un cas d'usage différent.

| Tu veux... | Choisis... |
|---|---|
| Piloter OpenCode en TUI direct, avec raccourcis clavier rapides | **OpenCode Desktop** |
| Déléguer des tâches OpenCode depuis Cowork (langage naturel, multi-sessions, dans une conversation plus large) | **Ce plugin** |
| Cohabiter les deux | Aucun conflit — OpenCode `serve` tourne en local pour les deux |

**Cohabitation** : si OpenCode Desktop tourne déjà, le plugin réutilise le même serveur OpenCode local. Pas de double-instance.

## Limites du MVP (v1.0) et roadmap

Le plugin est en MVP. Beaucoup de capacités sont à venir incrémentalement :

| Capacité | Version cible | Statut |
|---|---|---|
| Délégation avec ReAct, routing build/plan, stall detection | **v1.0** | ✅ MVP |
| Multi-sessions parallèles (max 3 par défaut) | **v1.0** | ✅ MVP |
| Install assistée OpenCode au 1er run | **v1.0** | ✅ MVP |
| Lecture AGENTS.md du repo cible | **v1.0** | ✅ MVP |
| `safe-prompts` (advisor sur patterns dangereux) | v1.1 | À venir |
| `task-memory` (mémoire persistante par projet) | v1.2 | À venir |
| `result-validator` (validation Reflect post-exécution) | v1.3 | À venir |
| `fallback-chain` (retry provider sur 429/5xx) | v0.5.0 | ✅ Fait |
| `agent-roster` (`cowork-with-github`) + écriture AGENTS.md | v0.6.0 | ✅ Fait |
| Scheduled tasks (digests, audits, scans) | v1.6 | À venir |
| Composabilité MCPs utilisateur (RAG, validators, ADR) | v1.7 | À venir |

## Conformité ToS Anthropic

Ce plugin fait que Cowork (basé sur Claude) orchestre OpenCode (un autre AI). Cette direction — Claude délègue à OpenCode pour de la productivité développeur — est explicitement permise comme cas d'usage agentique légitime par la [Anthropic Acceptable Use Policy](https://www.anthropic.com/legal/aup) et les [Agentic Use Guidelines](https://support.anthropic.com/en/articles/12005017-using-agents-according-to-our-usage-policy).

Précédent : [apoapps/swarm-code-plugin](https://github.com/apoapps/swarm-code-plugin) a fait l'audit ToS et est compliant pour un pattern analogue (Claude Code orchestre OpenCode).

## Bug connu — paramètre `directory` (opencode-mcp ≤ 1.10.x)

Le paramètre `directory` des outils `opencode_run` / `opencode_ask` / `opencode_fire` peut rejeter des chemins absolus valides avec le message :

```
Error: Invalid directory: "/chemin/absolu/valide" is not an absolute path.
```

**Workaround actif** dans le skill orchestrator : préfixer le prompt par `cd /chemin && ` au lieu de passer `directory`. Transparent pour l'utilisateur. À retirer du skill quand corrigé upstream.

## License

MIT

## Crédits

- [opencode-mcp](https://github.com/AlaeddineMessadi/opencode-mcp) par AlaeddineMessadi (le bridge MCP vers OpenCode)
- [opencode](https://github.com/anomalyco/opencode) par anomalyco (l'agent de code open source)
- Pattern d'architecture inspiré de [swarm-code-plugin](https://github.com/apoapps/swarm-code-plugin) (variant Claude Code)
