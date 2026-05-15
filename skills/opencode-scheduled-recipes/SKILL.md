---
name: opencode-scheduled-recipes
description: "Use this skill when the user asks about recurring tasks (daily digest, weekly audit, monthly scan) or wants to schedule automated OpenCode tasks. Documents the 3 bundled recipes and how to customize them."
---

# opencode-scheduled-recipes

## Nature et portee
Skill documentaire qui explique a Cowork comment proposer et configurer les scheduled tasks bundlees avec le plugin. Le plugin fournit 3 recettes pretes a l'emploi dans `scheduled-tasks/` :

| Recipe | Frequence | Agent | Resultat |
|---|---|---|---|
| daily-repo-digest | Tous les jours a 7h | plan | Resume des commits des 24h |
| weekly-deps-audit | Lundi a 9h | build | Rapport markdown dans reports/ |
| monthly-refactor-scan | 1er du mois a 9h | plan | 3 zones a refactor identifiees |

## Activation des recettes
Cowork dispose nativement d'un systeme de scheduled tasks. Quand l'utilisateur installe ce plugin, les 3 recettes sont **disponibles mais non activees** par defaut. Pour activer :
1. L'utilisateur dit "active le digest quotidien" ou "schedule le deps audit"
2. Tu invoques le mecanisme Cowork de scheduled tasks (cf. `mcp__scheduled-tasks__create_scheduled_task` natif) avec :
   - Le cronExpression de la recette
   - Le prompt de la recette
   - Une note "via opencode-agent recipe X"

## Customisation
Si l'utilisateur veut ajuster :
- Frequence : modifier la cron expression
- Repo cible : adapter le prompt avec `cd /path/to/repo &&` en prefix (workaround directory cf. orchestrator §Notes techniques)
- Notification : `notifyOnComplete: true` envoie une notif desktop quand termine

## Inter-skill
- **orchestrator** : delegue a ce skill quand l'utilisateur evoque une tache recurrente
- **safe-prompts** : scan les prompts schedules comme n'importe quel autre dispatch
- **result-validator** : applique si la recette utilise `agent: build`
- **task-memory** : enrichit la memoire au fur et a mesure des executions schedulees

## Exemple : activer le daily digest
Utilisateur : "Active le digest quotidien sur Relkhon_VerticalSlice"
Cowork :
1. Lit `scheduled-tasks/daily-repo-digest.json` (read pour acceder a la recette)
2. Adapte le prompt : "cd /mnt/d/Projects/Relkhon_VerticalSlice && [prompt original]"
3. Appelle `mcp__scheduled-tasks__create_scheduled_task` avec cron `0 7 * * *` et le prompt adapte
4. Confirme a l'utilisateur : "Digest active, premier run demain a 7h."

## Limites
- Les scheduled tasks ne respectent pas la conversation context — chaque run est independant
- Necessite que OpenCode soit installe et accessible au moment du run scheduled
- Les notifications dependent de l'OS de l'utilisateur (Cowork desktop)
- Pas de cron sur Windows pure sans WSL — sur Windows utiliser le Task Scheduler natif ou installer WSL

## Inspirations
- Hermes cron scheduler (RELEASE_v0.8.0 notify_on_complete)
- Cowork mcp__scheduled-tasks__ natif
- Pattern : recettes JSON + skill instructionnel = facile a customiser sans recoder le plugin
