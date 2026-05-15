---
name: opencode-mcp-discovery
description: "Use this skill at the start of each new project to detect user-side MCPs (RAG, validate, manage_adr, etc.) and propose composability options with the plugin's Full skills. Concretizes the composability principle from design doc §13.5."
---

# opencode-mcp-discovery

## Nature et portee
Skill meta-coordinateur qui s'active **au 1er run du plugin sur un projet**. Il detecte les MCPs disponibles cote Cowork et cote OpenCode, identifie ceux qui couvrent (totalement ou partiellement) des besoins des skills Full du plugin, et propose des modes de composition a l'utilisateur. Le but : ne JAMAIS doublonner ce que l'utilisateur a deja construit.

C'est l'incarnation du principe directeur §13.5 du design doc : "le plugin enrichit, ne remplace pas".

## Detection des MCPs
Au 1er run par projet :
1. Liste les MCPs cote Cowork (via introspection des tools `mcp__*` disponibles dans la session)
2. Liste les MCPs cote OpenCode via `opencode_mcp_status`
3. Pour chaque MCP detecte, regarde ses tools exposes (recherche heuristique sur les noms : `search`, `query`, `validate`, `manage_adr`, `analyze`, `trace`, etc.)
4. Mappe ces tools aux skills Full du plugin qu'ils pourraient couvrir

## Mapping MCP -> skill plugin

| Tool MCP detecte | Couvre potentiellement | Action proposee |
|---|---|---|
| `search_vault`, `search_code_graph`, `ask` | Contexte projet enrichi (au-dela de AGENTS.md) | Enrichir orchestrator avec ce MCP avant dispatch |
| `validate`, `check_consistency` | result-validator (v0.4.0) | Substitution ou chainage avec result-validator |
| `manage_adr`, `list_decisions` | task-memory (v0.3.0) — partie architecturale | Chainer : task-memory pour stats, MCP pour decisions |
| `analyze_entity_impact`, `trace_call_path` | Heuristique d'agent + contexte | Enrichir l'analyse pre-dispatch |
| RAG generique (Chroma, LlamaIndex) | Contexte projet | Lecture avant dispatch pour prompt enrichi |

## Marche a suivre au 1er run par projet
1. Verifier dans `<workspace>/.opencode-agent-memory.json` si la decouverte a deja ete faite (champ `mcpDiscoveryDone: true`)
2. Si non : lancer la decouverte. Sinon : skip.
3. Si MCPs pertinents detectes, presenter a l'utilisateur :

```
J'ai detecte que tu utilises [liste MCPs detectes].

Certains couvrent deja ce que mes skills Full font :
- [MCP X.validate] -> remplace ou complete mon result-validator
- [MCP Y.manage_adr] -> complete mon task-memory pour les decisions architecturales

Tu preferes :
(a) substitution : je desactive mes skills pour ces fonctions, j'utilise les tiens
(b) chainage : mes skills d'abord (checks generiques), tes MCPs ensuite (checks domaine)
(c) plugin seul : je garde mes skills, je n'utilise pas tes MCPs

Choix par defaut si tu ne reponds pas dans 30s : (b) chainage.
```

4. Stocker la decision dans `.opencode-agent-memory.json` :
```json
{
  "mcpDiscoveryDone": true,
  "mcpDiscoveryAt": "2026-05-15T15:00:00Z",
  "mcpComposition": {
    "result-validator": { "mode": "chain", "delegate": "relkhon-vault-agent.validate" },
    "task-memory": { "mode": "skill-only", "note": "task-memory enough, no MCP user-side ADR" }
  }
}
```

5. Aux taches suivantes, l'orchestrator et les skills Full lisent cette config et ajustent leur comportement.

## Heuristiques de detection

### Vault Relkhon-style
Si tu detectes des tools comme `search_vault`, `search_code_graph`, `manage_adr`, `validate`, `analyze_entity_impact` sous un meme MCP -> c'est un vault domaine-specifique. Propose chainage avec orchestrator, task-memory, result-validator.

### RAG simple (Chroma, LlamaIndex)
Si tu detectes uniquement `search` ou `query` + similarity sans graphe ou ADR -> c'est un RAG simple. Propose enrichissement du prompt avant dispatch (lecture before pour ajouter contexte).

### Validator custom
Si tu detectes un tool `validate` isole sans le reste -> propose substitution du result-validator si l'utilisateur est confiant qu'il couvre mieux les checks de son domaine.

## Reverification
La decouverte n'est pas immuable. L'utilisateur peut demander :
- "Refais la decouverte MCP" -> reset `mcpDiscoveryDone: false` dans la memoire et relance
- "Change ma composition pour [skill] en [mode]" -> mise a jour ciblee

## Inter-skill
- **orchestrator** : appelle ce skill une fois par projet (au 1er run detecte via task-memory)
- **task-memory** : stocke la config mcpComposition
- **result-validator** : consulte la config avant de faire ses checks (peut deleguer au MCP user)
- **task-memory** elle-meme : peut etre court-circuitee si manage_adr couvre deja

## Limites
- La detection se base sur les noms des tools (heuristique). Faux negatifs possibles si MCP custom avec naming non-standard.
- L'utilisateur peut surcharger n'importe quelle decision via commande explicite.
- La decouverte ajoute ~5-10s au 1er run d'un projet, c'est acceptable.
- Pas de reverification automatique si l'utilisateur ajoute un nouveau MCP en cours de route — il doit re-lancer la decouverte explicitement.

## Inspirations
- Pattern "configurable composability" : design doc §13.5
- "Dispatch double" (cote Cowork vs cote OpenCode) : §6.2.2
- Decouverte MCP au runtime : equivalent fonctionnel des `check_fn` Hermes (mais cote skill, pas cote tool registration)
- swarm-code-plugin n'a pas cet equivalent — c'est notre contribution unique

## Note sur la v1.0.0
Cette version marque la **completion du Full scope** du design doc. Toutes les capacites prevues sont implementees :
- 7 skills Full : orchestrator, safe-prompts, task-memory, result-validator, fallback-chain, agent-roster, scheduled-recipes, mcp-discovery (8 en fait avec le meta-coordinateur)
- 3 scheduled-tasks bundlees
- 1 agent custom (cowork-with-github)
- Composabilite avec MCPs utilisateur formalisee

Toutes les decisions Q1-Q11 du design doc sont satisfaites. Le plugin est feature-complete pour son MVP-Full roadmap.
