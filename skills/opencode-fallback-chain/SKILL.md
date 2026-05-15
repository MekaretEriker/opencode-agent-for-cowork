---
name: opencode-fallback-chain
description: "Use this skill when an OpenCode dispatch fails with a provider error (HTTP 429 rate limit or 5xx server error). Retries the task with the next provider in the configured fallback chain."
---

# opencode-fallback-chain

## Nature et portee

Skill instructionnel. Quand un appel OpenCode (`opencode_run` / `_ask` / `_fire`) renvoie une erreur indiquant une defaillance du provider (429 rate limit, 5xx server error, "provider unavailable", etc.), tu retry la tache avec le provider suivant dans la chaine configuree, apres un court backoff de 5 secondes. Pas de retry infini : 3 tentatives maximum, chacune avec un provider different. Si les 3 echouent, on abandonne proprement.

Ce skill ne gere pas les erreurs de transport MCP (timeout 180s, connexion rompue) — celles-ci sont du ressort de l'orchestrator.

## Composabilite avec MCPs utilisateur

Si l'utilisateur dispose d'un MCP avec sa propre logique de retry (typiquement un gateway LLM custom qui route deja les appels entre providers), c'est ce MCP qui doit gerer le fallback, pas ce skill. Au premier lancement du plugin par projet, le skill propose a l'utilisateur trois options :

1. **Substitution** : le MCP custom remplace la chaine de fallback du skill (le skill devient pass-through)
2. **Chainage** : la chaine du skill s'execute d'abord, et le MCP custom sert de filet de securite en dernier recours
3. **Skill-only** : desactiver le fallback du MCP custom et laisser le skill gerer entierement (comportement par defaut si aucun MCP custom n'est detecte)

Ce comportement est documente dans le design doc au §6.2.2.

## Declencheurs

Patterns d'erreur a detecter dans les reponses d'OpenCode (dans le champ `error` ou le message textuel de la reponse) :

- `"rate limit"`, `"429"`, `"TooManyRequests"`
- `"5xx"`, `"500"`, `"502"`, `"503"`, `"504"`, `"internal server error"`
- `"provider unavailable"`, `"service unavailable"`
- `"timeout to upstream"` (a distinguer du timeout MCP transport de 180s, qui lui est gere par l'orchestrator)

Ne **pas** retry sur les codes d'erreur client :

- **400** : prompt invalide ou mal forme — retry inutile, le probleme est dans la requete
- **401 / 403** : authentification absente ou refusee — changer de provider ne resoudra pas le probleme
- **Timeout MCP transport (~180s)** : gere par l'orchestrator avec le pattern de recuperation de session orpheline, pas par ce skill

En cas de doute sur la nature de l'erreur (message ambigu qui pourrait etre un 5xx deguise), appliquer le principe de precaution et tenter le fallback — un retry inutile est moins couteux qu'un abandon premature.

## Chaine par defaut

Si aucune chaine n'est explicitement configuree par l'utilisateur, la chaine par defaut est :

```
anthropic -> openrouter -> openai
```

Si l'utilisateur a configure des providers via `opencode_provider_list`, le skill utilise ces providers dans l'ordre de leur listing (generalement par priorite decroissante, telle que definie dans la configuration OpenCode de l'utilisateur).

## Configuration utilisateur

La variable d'environnement `OPENCODE_AGENT_PROVIDER_CHAIN` permet de surcharger la chaine par defaut. Format : liste de noms de providers separes par des virgules.

Exemple :

```
OPENCODE_AGENT_PROVIDER_CHAIN=anthropic,deepseek,openrouter
```

Cette variable est lue au demarrage de session via `opencode_setup` ou `opencode_context`. Si la variable est absente ou vide, la chaine par defaut s'applique.

Autre variable d'environnement reconnue (future iteration) :

```
OPENCODE_AGENT_FALLBACK_BACKOFF_SEC=10
```

En MVP (v0.5.0), le backoff est fixe a 5 secondes. La configurabilite du backoff est prevue pour une iteration ulterieure si le besoin emerge du terrain.

## Marche a suivre

1. **Detecter** le pattern d'erreur dans la reponse OpenCode (cf. section Declencheurs)
2. **Identifier** le provider qui a echoue via le `sessionId` courant et un appel a `opencode_session_get`
3. **Chercher** le provider suivant dans la chaine configuree (chaine utilisateur si definie, sinon chaine par defaut)
4. **Annoncer** en mode standard : "Le provider X a renvoye [type d'erreur]. Je retry avec Y, patiente 5 secondes..."
5. **Attendre** 5 secondes (backoff fixe)
6. **Relancer** la tache avec le nouveau provider en passant un `providerID` explicite dans le prochain appel `opencode_run`, `opencode_ask` ou `opencode_fire`
7. **Si 3 echecs successifs** sur 3 providers differents : abandonner et presenter une erreur claire a l'utilisateur ("Les 3 providers de la chaine ont echoue : [details]. Verifie ta connectivite et tes cles API."). Ne pas relancer automatiquement.
8. **Si succes** : mettre a jour `task-memory` (si le skill task-memory est actif, v0.3.0+) en incrementant `providerStats.<previousProvider>.failureCount` pour ajuster les choix futurs de l'orchestrator.

## Limites

- **Pas de switch en cours de session OpenCode existante.** Le fallback s'applique a la **prochaine tache dispatchee**, pas a une session deja active. Si une session est en cours et que son provider echoue en milieu d'execution, le fallback ne peut pas reprendre la session — il faut relancer la tache entiere avec un nouveau provider.
- **Si tous les providers configures echouent**, l'erreur est remontee a l'utilisateur. Pas de fallback infini ni de boucle silencieuse.
- **Le backoff de 5 secondes est fixe en MVP.** Une future iteration pourra le rendre configurable via `OPENCODE_AGENT_FALLBACK_BACKOFF_SEC` si les retours utilisateur le justifient.
- **Le skill ne fait pas de distinction entre les providers.** Il suit l'ordre de la chaine, sans logique de cout, de latence ou de preference regionale. Ces optimisations sont hors scope du MVP.

## Inter-skill

- **orchestrator** : appelle ce skill lorsqu'une erreur provider est detectee apres un dispatch (`opencode_run` / `_ask` / `_fire`). L'orchestrator reste maitre du flux : c'est lui qui decide de re-dispatcher avec le nouveau provider.
- **task-memory** : ce skill ecrit dans le champ `providerStats` (incrementation de `failureCount` par provider) pour que l'orchestrator puisse ajuster ses choix futurs — par exemple, eviter temporairement un provider qui echoue frequemment.
- **result-validator** : pas d'interaction directe. Le result-validator intervient apres execution reussie, donc apres que le fallback-chain a potentiellement change de provider. Les deux skills operent a des etapes differentes du pipeline.

## Inspirations

- Pattern de fallback providers de [hermes-agent](https://github.com/NousResearch/hermes-agent) (NousResearch) : retry avec provider suivant sur erreur transitoire, backoff configurable
- Retry x3 avec exponential backoff de [swarm-code-plugin](https://github.com/apoapps/swarm-code-plugin)
- Adaptation Cowork : skill-as-instruction, pas auto-runtime. Le skill fournit la procedure, l'orchestrator Cowork l'execute — pas d'automatisme invisible.
