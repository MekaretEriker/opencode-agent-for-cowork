---
name: opencode-result-validator
description: "Use this skill after an OpenCode task with the build agent reports completion. Validates that the result is sound (diff coherent, tests pass, build OK, semantically aligned with intent) before declaring success to the user. Implements the Plan-Execute-Reflect pattern's third step."
---

# opencode-result-validator

## Nature et portee

Skill instructionnel qui s'active **apres** qu'une session OpenCode (typiquement `opencode_run` ou la phase `opencode_check` suite a `opencode_fire`) rapporte "terminé" sur une tache de modification. Avant de declarer le succes a l'utilisateur, tu valides systematiquement que le resultat tient debout.

C'est le **3e step du pattern Plan-Execute-Reflect** (cf. design doc §13.1 et §5.6 originel). L'agent `plan` natif d'OpenCode fait le "Plan" pour les taches exploratoires. L'agent `build` fait l'"Execute". Ce skill fait le "Reflect".

**Sans ce skill** : Cowork pourrait te dire "Termine" alors que les tests sont rouges, que le diff contient des modifications hors scope, ou que la fonctionnalite ne compile pas. Faux positif.

**Avec ce skill** : tu verifies avant de remettre le resultat. Si pb detecte, tu presentes avec "reserves" plutot que succes brut.

## Composabilite avec MCPs utilisateur

Si l'utilisateur a un MCP exposant un tool `validate` ou equivalent (typiquement un vault Relkhon-style avec validation semantique cross-specs, ou un linter custom domaine), ce MCP est **plus fin** pour son projet. Ce skill couvre uniquement les checks generiques.

Au 1er run sur un projet, si un MCP user-side avec `validate` est detecte, propose :

- (a) Remplacer la validation plugin par l'appel au tool MCP (substitution)
- (b) Chainer les deux : plugin = checks generiques (diff/tests), MCP user = checks domaine (semantique)
- (c) Garder seulement le plugin

Voir design doc §6.2.2.

## Quand activer

Active toujours apres :

- `opencode_run` ou `opencode_check` montrant `idle/finished` sur une tache d'agent `build`
- Recuperation d'une session orpheline (cf. orchestrator §2)
- Toute tache dont le prompt initial etait `feature_implementation`, `bug_fix`, ou `refactor`

NE PAS activer apres :

- `opencode_ask` (lecture seule)
- Taches d'agent `plan` (exploration / analyse, aucun fichier modifie)
- Taches tres courtes (moins de 30s de duree) — pas le cout/benefice

Skip-able sur demande utilisateur :

- Si l'utilisateur dit "skip validation" ou "no checks", tu skip pour la session courante
- Confirme une fois : "OK, je skippe les checks de validation pour cette conv. Tu vois directement les resultats d'OpenCode."

## Checks systematiques (par ordre)

### 1. Diff sain (toujours)

Appelle `opencode_review_changes` (ou `opencode_session_diff`) pour recuperer le diff complet de la session.

Verifie :

- Les fichiers modifies sont-ils dans le scope annonce en Plan ?
- Y a-t-il des modifications inattendues (fichiers de config sensibles, secrets, etc.) ?
- Le diff est-il de la taille raisonnable attendue ?

Si anomalie detectee, annote dans le resume final.

### 2. Tests (si detectables)

Detecte la commande de test du projet via heuristique :

- Presence de `package.json` avec script `test` -> `npm test`
- Presence de `pyproject.toml`/`setup.py` avec pytest -> `pytest`
- Presence de `Cargo.toml` -> `cargo test`
- Presence de `build.gradle` -> `gradle test`
- Sinon -> skip (pas de tests detectables)

Lance via OpenCode une commande courte type :

```
opencode_ask(prompt: "Run the project tests once and tell me pass/fail with a summary. Don't fix anything, just report.")
```

Analyse la reponse, si tests rouges, note dans le resume.

### 3. Build / compilation (si pertinent)

Pour TypeScript, Rust, C++ et autres langages compiles :

- TS : `npm run build` ou `tsc --noEmit`
- Rust : `cargo build`
- etc.

Si le build casse, c'est un echec critique de la validation.

### 4. Coherence semantique (soft check via Claude)

Question directe au sub-LLM (toi-meme) : "Le resultat repond-il vraiment a la demande initiale ? Y a-t-il une divergence entre ce qui a ete demande et ce qui a ete fait ?"

C'est qualitatif. Si tu detectes une divergence (ex: "j'ai demande de refactor le module auth mais OpenCode a aussi modifie le module payments"), tu l'annotes.

## Format de sortie

### En cas de succes complet

```
Tache terminee

[resume en 1 phrase de ce qui a ete fait]

Fichiers modifies :
- chemin/fichier1.ext (resume)
- chemin/fichier2.ext (resume)

Validation :
- OK Diff coherent avec la demande
- OK Tests : 47/47 passent
- OK Build : OK
- OK Pas de divergence semantique

Tu veux que je [proposition d'etape suivante naturelle] ?
```

### En cas de reserves (un check rouge)

```
Tache terminee avec reserves

[resume en 1 phrase]

Fichiers modifies :
- chemin/fichier1.ext (resume)

Validation :
- OK Diff coherent
- KO Tests : 2 echouent (auth.test.ts:142, auth.test.ts:198)
- OK Build : OK
- OK Semantique : aligne

Suggestion : "Cowork, demande a OpenCode de corriger les tests rouges"
```

### En cas d'echec critique (build casse)

```
Tache terminee avec probleme critique

[resume en 1 phrase]

Fichiers modifies :
- chemin/fichier1.ext (resume)

Validation :
- KO Build : echec compilation (5 erreurs TypeScript)
- (autres checks skip car build casse)

Le code ne compile pas en l'etat. Je recommande de :
- (1) Annuler les changements (git restore)
- (2) Demander a OpenCode de corriger les erreurs

Tu fais quoi ?
```

## Inter-fonctionnement avec d'autres skills

- **orchestrator** : appelle ce skill en fin de tache `build`
- **safe-prompts** : pas d'interaction directe
- **task-memory** : si la validation echoue, c'est un *antipattern* a noter dans `antipatterns` ("Tache X dispatchee comme Y a echoue la validation Z")

## Limites assumees

- Les tests sont lances **en plus** de l'execution principale -> consomme tokens et temps. Sur les taches tres courtes, ratio cout/benefice defavorable.
- Si OpenCode lui-meme n'a pas reussi a faire passer les tests, ce skill ne corrige pas, il rapporte seulement.
- La coherence semantique est qualitative — Claude peut louper une divergence subtile.

## Inspirations

- Pattern Plan-Execute-Reflect : design doc §13.1 (reference canonique : Reflexion 2023, ReAct framework)
- Composabilite avec validate user-side : §6.2.2
- swarm-code-plugin a un skill `opencode-result-handling` proche conceptuellement (mais cote Claude Code, pas Cowork)
