---
name: opencode-agent-roster
description: "Use this skill when dispatching to OpenCode and needing to choose between native agents (build, plan, @general) or the plugin's custom agent (cowork-with-github for PR/issue tasks). Also handles opt-in writing to AGENTS.md in the target repo."
---

# opencode-agent-roster

## Nature et portee

Ce skill etend l'orchestrator §3 (choix d'agent) avec trois capacites nouvelles livrees en v0.6.0 :

1. Un agent custom `cowork-with-github`, extension de `build` augmente du MCP GitHub, livre par le plugin
2. Une heuristique de routing enrichie qui inclut cet agent dans la matrice de decision
3. Un mecanisme opt-in pour ecrire dans le `AGENTS.md` du repo cible, conforme a la decision Q9 design doc §6.5

Le roster final couvre desormais quatre agents aux profils complementaires, eliminant les zones grises ou le choix d'agent etait ambigu (taches GitHub sans precedent clair dans le routing).

## Roster final (apres v0.6.0)

| Agent | Origine | Quand l'utiliser |
|---|---|---|
| build | natif OpenCode | Tache de modification (refactor, feature, fix) — defaut pour toute tache avec side effects |
| plan | natif OpenCode | Exploration, review, analyse — read-only, zero side effect |
| @general | sous-agent natif OpenCode | Recherche complexe multi-etapes necessitant plusieurs rounds de discovery |
| cowork-with-github | LIVRE par le plugin (v0.6.0) | Taches impliquant PR, issues, branches, ou toute interaction avec l'API GitHub |

Chaque agent a un profil d'outils et un comportement distinct. Le routing doit choisir le plus specialise pour la tache courante, avec fallback explicite si les prerequis (MCP, agent installe) ne sont pas satisfaits.

## Heuristique de choix mise a jour

L'heuristique de routing (orchestrator §3) est etendue avec un nouveau test en tete de chaine :

1. **Detection GitHub** : si le prompt contient "PR", "pull request", "issue", "#\d+", "GitHub", "branche", "merge" → router vers `cowork-with-github`
   - Verifier la presence du MCP GitHub cote OpenCode via `opencode_mcp_status`
   - Si MCP GitHub absent : fallback `build` avec note explicite "PR manuelle a faire ensuite"
   - Si l'agent `cowork-with-github` n'est pas encore installe cote OpenCode : proposer l'installation opt-in (voir section Installation)
2. **Detection read-only** : si la tache est purement exploratoire sans modification de code → `plan`
3. **Detection complexite** : si la tache necessite plusieurs rounds de recherche avant l'action → `@general`
4. **Defaut** : toute autre tache → `build`

Cette chaine est evaluee sequentiellement. La premiere condition vraie gagne. Les conditions 2 a 4 sont inchangees par rapport a l'orchestrator v0.5.0 ; seule la condition 1 est nouvelle.

### Exemples de routage concret

| Prompt utilisateur | Agent route | Justification |
|---|---|---|
| "Refactor le module factions pour utiliser EventBus" | build | Modification, pas de mot-cle GitHub |
| "Analyse les dependances circulaires dans src/" | plan | Read-only, exploration |
| "Trouve tous les appels a deprecatedMethod et propose un plan de migration" | @general | Recherche multi-etapes avant action |
| "Cree une PR pour le fix du bug #342" | cowork-with-github | Mots-cles PR + issue |
| "Ouvre une issue pour tracker le refactor du parser" | cowork-with-github | Mot-cle issue + GitHub |
| "Push la branche feat/auth et ouvre une PR" | cowork-with-github | Mots-cles branche + PR |

## Agent cowork-with-github

### Definition

L'agent est defini dans `agent-templates/cowork-with-github.json` (template a installer cote OpenCode via `opencode_agent_list` + configuration OpenCode native). Caracteristiques techniques :

- **Base** : agent `build` natif OpenCode
- **MCPs requis** : GitHub MCP (typiquement `@modelcontextprotocol/server-github`)
- **System prompt additionnel** : l'agent prefere creer une branche dediee plutot que de travailler sur main/master, ouvre une PR au lieu de pusher directement, et lie l'issue si une reference est detectee dans le prompt

Ce n'est pas un agent completement distinct — c'est une specialisation de `build` avec un contexte GitHub enrichi. Cette approche minimaliste evite la duplication de la logique de modification de code (deja dans `build`) tout en ajoutant le comportement specifique aux workflows GitHub.

### Comportement specific

Quand l'agent `cowork-with-github` est active, il adopte les comportements suivants par defaut (le system prompt les encode) :

1. **Branche dediee** : cree une branche nommee d'apres la tache (ex: `feat/auth-jwt`, `fix/issue-342-null-pointer`, `refactor/eventbus-migration`). Ne travaille jamais directement sur `main` ou `master`.
2. **PR systematique** : ouvre une pull request avec un titre et une description clairs, plutot que de pusher directement sur la branche principale.
3. **Lien d'issue** : si le prompt contient une reference a un numero d'issue (`#123`), la PR inclut `Closes #123` dans sa description.
4. **Description structuree** : la PR inclut un resume des changements, la liste des fichiers modifies, et les tests passes.

Ces regles sont des preferences, pas des contraintes dures. Si l'utilisateur demande explicitement un push direct, l'agent obeit.

### Fallback si MCP GitHub absent

Si le MCP GitHub n'est pas configure cote OpenCode, l'agent `cowork-with-github` ne peut pas fonctionner. Dans ce cas :
- Le routing tombe en fallback `build`
- Une note est ajoutee au rapport de tache : "PR manuelle a faire ensuite — le MCP GitHub n'est pas configure cote OpenCode"
- L'utilisateur peut configurer le MCP GitHub et reessayer

## Installation au 1er run sur un projet (opt-in)

L'installation de l'agent `cowork-with-github` cote OpenCode suit un protocole opt-in strict :

### Detection

Au 1er run par projet ou une tache `cowork-with-github`-eligible est detectee, le skill verifie si l'agent existe deja cote OpenCode via `opencode_agent_list`. Si l'agent est absent :

### Proposition

Le skill propose a l'utilisateur :
> "Cette tache beneficierait de l'agent custom `cowork-with-github`. Tu veux que je l'installe cote OpenCode ? Il etend `build` avec le MCP GitHub pour gerer les PR et issues automatiquement."

### Installation

Si l'utilisateur accepte :
1. Ecriture du fichier agent via la configuration OpenCode (format natif OpenCode)
2. Ajout du MCP GitHub (`@modelcontextprotocol/server-github`) a la configuration OpenCode si absent
3. Verification que l'agent apparait dans `opencode_agent_list`

### Refus

Si l'utilisateur refuse : fallback `build` avec la note standard "PR manuelle a faire ensuite". Aucune relance sur les taches suivantes du meme projet — le refus est definitif pour ce projet. L'utilisateur peut activer manuellement plus tard avec : "installe l'agent cowork-with-github".

## Ecriture AGENTS.md (decision Q9)

### Principe general

Au 1er run sur un repo, en plus de la lecture obligatoire de `AGENTS.md` (orchestrator §7), le skill propose l'ecriture **opt-in explicite** d'une section maintenue automatiquement dans ce fichier. Cette section alimente la memoire operationnelle du projet au-dela de la session courante.

### Protocole opt-in

1. **Proposition unique** : "Tu veux que je maintienne une section dans `AGENTS.md` du repo cible avec les apprentissages cross-sessions (durees observees, patterns reussis, antipatterns) ?"
2. **Si OK** :
   - Si `AGENTS.md` est absent du repo, proposer de le creer (avec l'entete standard OpenCode)
   - Si `AGENTS.md` existe deja, ajouter une section delimitee par des balises HTML : `<!-- Maintained by Cowork OpenCode Agent plugin -->` ... `<!-- end Cowork section -->`
   - Stocker l'opt-in dans `.opencode-agent-memory.json` avec le champ `agentsMdWriteEnabled: true`
3. **A chaque tache reussie** : mettre a jour la section avec un resume agrege des apprentissages. Le contenu doit etre lisible et pertinent — pas un dump de stats brutes. Agreger les donnees de `task-memory` en tendances.
4. **Jamais de commit automatique** : les modifications de `AGENTS.md` restent unstaged. C'est a l'utilisateur de les commiter quand il le juge pertinent.
5. **Jamais d'ecrasement** : le skill ne touche qu'a la section delimitee. Tout contenu hors balises est preserve a l'identique.
6. **Refus definitif** : si l'utilisateur refuse au 1er run, ne plus proposer. Il peut activer plus tard via la commande "active AGENTS.md write".

### Format de la section AGENTS.md geree

```markdown
<!-- Maintained by Cowork OpenCode Agent plugin -->
## Observed patterns (auto-maintained)

- Tests typically take ~120s on this repo (sample: 12 runs)
- Refactor tasks: ~5min average (sample: 7 runs)
- Anthropic provider reliable (94% success), OpenAI as fallback
- Pattern: prefer EventBus over direct module imports for cross-system communication

## Known antipatterns

- Cross-system refactors without explicit plan -> spec contradictions (3 cases)
- Implementing without LightRAG validation -> bugs found in QA (2 cases)

<!-- end Cowork section -->
```

### Regles de mise a jour

- Le champ `Observed patterns` est mis a jour apres chaque tache reussie. Les stats numeriques utilisent une moyenne glissante (pas de reset).
- Le champ `Known antipatterns` est mis a jour quand `result-validator` detecte un echec repete (meme pattern sur >= 2 taches).
- Les entrees sont en anglais pour la lisibilite par les LLMs qui liront le fichier.
- Si la section grossit au-dela de 20 lignes, les entrees les plus anciennes sont retirees (FIFO). Pas de purge automatique plus agressive — c'est a l'utilisateur de nettoyer s'il le souhaite.

## Inter-skill

Ce skill est appele par d'autres skills du plugin et interagit avec eux de maniere bien definie :

| Skill appelant | Point d'interaction |
|---|---|
| **orchestrator** | Appelle ce skill pour le routing d'agent (choix entre build/plan/@general/cowork-with-github) et pour decider si une tache justifie l'usage de `cowork-with-github` |
| **task-memory** | Alimente le contenu de la section `AGENTS.md` geree via ses metriques operationnelles (durees, providers, patterns) |
| **safe-prompts** | Pas d'interaction directe — le scan de securite s'effectue en amont du routing |
| **result-validator** | Alimente les antipatterns via ses verdicts d'echec (pattern repete = antipattern enregistre) |
| **fallback-chain** | Pas d'interaction directe — le fallback provider est transparent pour le routing d'agent |

## Limites

- **Dependance MCP GitHub** : l'agent `cowork-with-github` necessite le MCP GitHub cote OpenCode. Sans ce MCP, le routing tombe en fallback `build` sans les fonctionnalites GitHub. L'utilisateur doit avoir configure l'authentification GitHub au prealable.
- **Ecriture dans le repo utilisateur** : la section `AGENTS.md` modifie un fichier dans le repo de l'utilisateur. Le mecanisme est strictement opt-in, delimite par balises HTML, et ne declenche jamais de commit automatique. L'utilisateur garde le controle total.
- **Pas de purge automatique agressive** : la section `AGENTS.md` utilise un FIFO simple a ~20 lignes max. Si elle grossit malgre cela, c'est a l'utilisateur de la nettoyer. Pas de suppression automatique basee sur l'age ou la pertinence.
- **Unicite de la section** : une seule section `<!-- Maintained by Cowork ... -->` est geree par fichier. Si plusieurs instances du plugin tournent sur le meme repo, elles partagent la meme section (dernier qui ecrit gagne). Pas de merge intelligent.

## Inspirations

- **Pattern roster d'agents** : OpenCode natif (build, plan, @general) etendus par des agents custom (cf. `swarm-code-plugin` agents/, ou Claude Code ajoute des specialisations au-dessus du modele de base)
- **Convention AGENTS.md** : adoption native par OpenCode pour le contexte projet (cf. design doc §6.5 + §2.4), etendue ici avec une section auto-maintenue pour la memoire operationnelle cross-session
- **Opt-in strict** : best practice Anthropic AUP — aucune ecriture silencieuse dans le repo utilisateur. Chaque modification de fichier dans l'espace de travail de l'utilisateur est precedee d'un consentement explicite.
- **FIFO simple** : inspire des logs circulaires (pas de retention infinie, pas d'heuristique complexe de purge). L'utilisateur peut toujours nettoyer manuellement la section si besoin.
