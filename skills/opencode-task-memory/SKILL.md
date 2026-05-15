---
name: opencode-task-memory
description: "Use this skill to maintain operational memory across OpenCode sessions on the same project: average task durations, reliable providers, learned patterns, known antipatterns. Reads memory at task start to calibrate, writes at task end to accumulate."
---

# opencode-task-memory

## Nature et portee

Skill instructionnel qui apprend a Cowork a maintenir une **memoire operationnelle persistante par projet** dans un fichier JSON local au workspace. Lecture en tete de tache pour calibrer les attentes (durees, providers fiables, antipatterns a eviter), ecriture en fin de tache pour accumuler l'apprentissage.

### Distinction critique (3 couches de memoire)

Tu ne melanges pas ces couches :

- **Memoire OPERATIONNELLE** (ce skill) = stats techniques par projet, patterns d'execution observes
- **Memoire UTILISATEUR** (gere par `consolidate-memory` natif Cowork) = preferences, style de travail, relations
- **Contexte PROJET** (AGENTS.md, lu par OpenCode) = conventions de code, spec architecture

Ce skill couvre uniquement la premiere couche. Ne pas y mettre de prefs utilisateur ou de spec.

## Composabilite avec MCPs utilisateur

Si l'utilisateur dispose d'un MCP exposant un systeme d'ADR (Architecture Decision Records) — typiquement un vault Relkhon-style avec `manage_adr`, ou un outil custom de docs — c'est lui qui doit servir de memoire de reference pour les decisions architecturales du projet.

Ce skill ne couvre que les stats techniques d'execution, pas les decisions architecturales ou apprentissages metier.

Au 1er run sur un projet, si un MCP user-side avec `manage_adr` ou equivalent est detecte, propose a l'utilisateur :
- (a) desactiver task-memory au profit du MCP user (qui couvre deja une partie)
- (b) chainer les deux (task-memory pour stats brutes, MCP user pour decisions de fond)
- (c) garder seulement task-memory

## Localisation du fichier

`<workspace>/.opencode-agent-memory.json`

Ou `<workspace>` est le dossier du repo de travail (equivalent du `directory` passe a OpenCode).

### Premier run : proposer .gitignore

Si le projet a un `.git/`, propose a l'utilisateur d'ajouter `.opencode-agent-memory.json` au `.gitignore` :

> "Je vais maintenir un fichier `.opencode-agent-memory.json` a la racine de ton projet pour apprendre tes patterns (durees, providers fiables, etc.). Je te recommande de l'ajouter au `.gitignore` pour ne pas le commiter par erreur. Tu veux que je le fasse ?"

Si oui, ajoute la ligne au `.gitignore`. Pas d'opt-out cache — l'utilisateur sait que tu ecris dans son repo.

### Si le projet est en lecture seule

Si l'ecriture echoue (repo en lecture seule, sandbox CI), opere en mode in-memory pour la session courante et previens l'utilisateur :

> "Je n'arrive pas a ecrire `.opencode-agent-memory.json` (workspace en lecture seule). Je garde la memoire en interne pour cette conversation, mais elle sera perdue au prochain demarrage."

## Schema JSON

```json
{
  "version": "1.0",
  "project": "/absolute/path/to/repo",
  "createdAt": "2026-05-15T14:00:00Z",
  "lastUpdated": "2026-05-15T15:30:00Z",
  "tasksTotal": 47,
  "byType": {
    "feature_implementation": {
      "count": 28,
      "avgDurationSec": 240,
      "successRate": 0.89,
      "preferredAgent": "build"
    },
    "bug_fix": {
      "count": 12,
      "avgDurationSec": 90,
      "successRate": 0.92,
      "preferredAgent": "build"
    },
    "refactor": {
      "count": 7,
      "avgDurationSec": 300,
      "successRate": 0.71,
      "preferredAgent": "build"
    },
    "exploration": {
      "count": 5,
      "avgDurationSec": 30,
      "successRate": 1.0,
      "preferredAgent": "plan"
    }
  },
  "providerStats": {
    "anthropic": { "calls": 35, "successRate": 0.94 },
    "openrouter": { "calls": 12, "successRate": 0.83 }
  },
  "patterns": [
    "Taches sur le module factions : ~5 min, necessite verif croisee avec specs/combat.md",
    "Bugs UI Godot : ~1-2 min, regarder _ready() et _process() en priorite"
  ],
  "antipatterns": [
    "Refactor cross-systemes sans plan explicite -> contradictions specs 3 fois sur 4",
    "Implementer sans valider via LightRAG -> bug remonte en QA 2 fois"
  ]
}
```

## Quand lire la memoire

En tete de chaque tache OpenCode sur un projet :

1. Verifier si `<workspace>/.opencode-agent-memory.json` existe
2. Si oui, parser le JSON et utiliser pour :
   - **Calibrer `maxDurationSeconds`** : si la tache ressemble a `feature_implementation` et `byType.feature_implementation.avgDurationSec` = 240, propose `maxDurationSeconds: 300` (avec marge ~25%)
   - **Choisir le provider initial** : si `providerStats.anthropic.successRate` > `providerStats.openrouter.successRate`, defaut Anthropic
   - **Mentionner les antipatterns dans le ReAct** : "A noter, sur ce projet, [antipattern]. Je vais [strategie pour eviter]"
   - **Referencer les patterns** : "D'apres les sessions passees, [pattern]. Je suis cette approche."
3. Si non, demarrer en mode neutre (defauts du skill orchestrator)

## Quand ecrire la memoire

En fin de tache OpenCode (apres `opencode_check` montre status `idle`/`finished`, ou apres une recuperation de session orpheline) :

1. Determiner le type de tache via heuristique sur le prompt initial :
   - "implemente", "ajoute" -> `feature_implementation`
   - "fix", "corrige", "bug" -> `bug_fix`
   - "refactor", "restructure", "ameliore" -> `refactor`
   - "explique", "trouve", "analyse" -> `exploration`
   - Sinon -> `other`
2. Mettre a jour les stats :
   - Incrementer `tasksTotal` et `byType.<type>.count`
   - Recalculer `avgDurationSec` (moyenne courante : `newAvg = (oldAvg * (count-1) + thisDuration) / count`)
   - Mettre a jour `successRate` (binaire succes/echec selon le resultat)
3. Mettre a jour `providerStats` pour le provider utilise
4. Si nouvelle lecon apprise (pattern qui marche, antipattern observe), AJOUTER a `patterns` ou `antipatterns`
5. Mettre a jour `lastUpdated`
6. Sauvegarder le JSON

### Gestion du debordement

- `patterns` et `antipatterns` : max 20 entrees chacune, FIFO (eject les plus anciennes)
- Le fichier ne devrait jamais depasser ~50KB. Si oui, prune plus aggressivement (garder 10 + 10).

## Format de presentation a l'utilisateur

### Au debut d'une tache (mode standard)

Si tu utilises la memoire pour calibrer, mentionne-le brievement :

> "D'apres les sessions passees sur ce projet (47 taches), les features prennent ~4 min en moyenne et Anthropic est fiable a 94%. Je dispatche avec ces parametres."

### En fin de tache

Silencieux par defaut. Sauf si :
- Tu ajoutes un pattern/antipattern nouveau : "J'ai note que [observation]. Ca enrichit la memoire operationnelle du projet."
- L'utilisateur demande a voir la memoire : tu peux montrer le JSON ou son resume.

### Mode dev override (voir memoire complete)

Si l'utilisateur demande "montre la memoire" ou "task-memory status" :

> "Memoire operationnelle de ce projet : 47 taches au total, 28 features (avg 4min, 89% succes), 12 fix (avg 1m30, 92%), 7 refactors (avg 5min, 71%). Provider anthropic a 94%. 3 patterns appris, 2 antipatterns notes."

## Inter-fonctionnement avec d'autres skills

- **orchestrator** : appele en premier, decide quoi dispatcher. Lit la memoire pour calibrer maxDurationSeconds et choisir provider.
- **safe-prompts** (v0.2.0) : scan avant dispatch, ne lit pas la memoire.
- **result-validator** (v1.3 a venir) : valide apres dispatch, ecrit dans la memoire si validation echoue (antipattern observe).

## Limites assumees

- La memoire est par PROJET (chemin du repo), pas globale. Si tu changes de projet, la memoire ne suit pas.
- Pas de cross-machine sync — le fichier vit dans le workspace local.
- Pas de versioning ou rollback. Si corrompu, supprimer pour repartir a zero.
- Pas de purge automatique des tres vieux patterns (>6 mois) en v1.2. A voir si pertinent en v1.x.
- Le calcul de duree dans `avgDurationSec` peut etre fausse si tu changes de modele/provider entre les sessions — c'est une moyenne brute, pas une comparaison rigoureuse.

## Inspirations

- Schema schema : design doc §5.2 (notre propre design)
- Distinction "memoire operationnelle vs user vs spec" : audit consolidate-memory Cowork natif (notre Q1)
- Composabilite avec MCP user (manage_adr style) : §13.5 du design doc
- Pattern de logging persistant : trajectoires d'agent (§11.4 traces JSONL — task-memory est la version resumee, traces sont la version brute non encore implementee)
