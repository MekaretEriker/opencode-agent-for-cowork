---
name: opencode-safe-prompts
description: "Use this skill before dispatching any prompt to OpenCode that involves shell commands, file modifications, SQL operations, git destructive ops, or remote execution. Scans prompts for dangerous patterns (rm -rf, DROP TABLE, curl | sh, --force, etc.) and warns the user before execution."
---

# opencode-safe-prompts

## Nature et portée

Ce skill est un **advisor** cote Cowork. Avant de dispatcher un prompt a OpenCode via `opencode_run` / `opencode_ask` / `opencode_fire`, tu scans le prompt (et toute reformulation que tu as faite) pour des patterns a risque. Si tu en detectes, tu **annonces clairement a l'utilisateur** et tu demandes confirmation explicite.

**Ce n'est PAS un firewall.** Tu ne bloques pas automatiquement. Tu informes et tu laisses l'utilisateur decider. La responsabilite finale reste a l'utilisateur. Coherent avec la decision Q5 du design doc (layered safety : opencode-mcp `permission: allow` + agent `plan` + safe-prompts advisor).

## Composabilite avec MCPs utilisateur

Si un MCP utilisateur expose un tool `validate` ou equivalent (typiquement un vault Relkhon-style avec validation semantique cross-specs), c'est lui qui doit faire l'audit fin. Ce skill ne couvre que les patterns generiques (shell destructif, SQL destructif, exec a distance). Voir DESIGN doc section 6.2.2 pour le pattern de composition.

Au 1er run sur un projet, si tu detectes un MCP user-side avec un tool `validate`, propose a l'utilisateur :
- (a) desactiver safe-prompts au profit du MCP user (plus fin pour son domaine)
- (b) chainer les deux (safe-prompts d'abord pour les patterns generiques, MCP user ensuite pour les checks domaine)
- (c) garder seulement safe-prompts

## Quand ce skill se declenche

Avant d'envoyer un prompt a OpenCode qui contient potentiellement :
- Une commande shell explicite ou attendue
- Une operation de fichier en ecriture
- Une operation sur une base de donnees
- Une operation git destructive
- Une execution de script distant

Si le prompt est purement de lecture/analyse (`opencode_ask` avec agent `plan`, ou phrases comme "explique", "trouve", "analyse") -> tu peux skip ce skill.

## Table de patterns a scanner

### Filesystem destructif

| Pattern (regex insensible casse) | Categorie | Action |
|---|---|---|
| `rm\s+-rf?\s+/(?!\w)` | suppression recursive racine | Warning critique + confirmation explicite |
| `rm\s+-rf?\s+~` | suppression home | Warning critique + confirmation explicite |
| `rm\s+-rf?\s+\$HOME` | suppression home variable | Warning critique + confirmation explicite |
| `>\s*/dev/sd[a-z]` | ecriture disque brute | Warning critique + bloquer par defaut |
| `mkfs\.\w+` | reformatage | Warning critique + confirmation explicite |
| `dd\s+.*of=/dev/` | ecriture disque dd | Warning critique + confirmation explicite |

### SQL destructif

| Pattern | Categorie | Action |
|---|---|---|
| `DROP\s+(TABLE\|DATABASE\|SCHEMA)` | destruction schema | Warning + confirmation |
| `TRUNCATE\s+TABLE` | vidage table | Warning + confirmation |
| `DELETE\s+FROM\s+\w+\s*;?\s*$` (sans WHERE) | delete sans where | Warning critique + confirmation |

### Execution distante

| Pattern | Categorie | Action |
|---|---|---|
| `curl\s+.*\|\s*(bash\|sh\|zsh)` | RCE pattern | Warning + confirmation |
| `wget\s+.*\|\s*(bash\|sh\|zsh)` | RCE pattern | Warning + confirmation |
| `(curl\|wget)\s+.*\|\s*python` | exec python distant | Warning + confirmation |

### Git destructif

| Pattern | Categorie | Action |
|---|---|---|
| `git\s+push\s+.*--force(\s\|$)` | force push | Warning + confirmation |
| `git\s+push\s+.*--force-with-lease` | force-with-lease (moins risque) | Note seule, pas de blocage |
| `git\s+reset\s+--hard` | reset hard | Warning |
| `git\s+clean\s+-fd` | clean force | Warning |
| `git\s+branch\s+-D` | suppression force branche | Warning |
| `git\s+rebase\s+.*(master\|main)` | rebase main | Warning + suggestion utiliser PR |

### Secrets et exfiltration

| Pattern | Categorie | Action |
|---|---|---|
| `cat\s+.*\.env` (dans contexte de copie/exposition) | lecture secrets | Warning |
| `cat\s+~/.ssh/id_` | lecture cle SSH | Warning critique |
| `(echo\|printf)\s+.*\|\s*(curl\|nc\s)` | exfiltration potentielle | Warning |

## Marche a suivre quand un pattern matche

1. **Ne pas dispatcher immediatement** a OpenCode
2. Annoncer a l'utilisateur (mode standard) :

   ```
   Attention : ton prompt contient un pattern a risque ([categorie]).
   Detecte : "[extrait du pattern matche]"

   Ce que ca pourrait faire : [explication non technique de l'impact potentiel].

   Tu veux quand meme que je dispatche a OpenCode ? Tape 'oui' pour confirmer, 'non' pour annuler, ou reformule le prompt.
   ```

3. Attendre la confirmation explicite ('oui' ou equivalent clair)
4. Si confirme -> dispatcher
5. Si non ou reformulation -> suivre la nouvelle direction

## Cas particuliers

### Pattern dans une spec, pas dans une commande
Si l'utilisateur dit "implemente la migration `DELETE FROM users WHERE last_login < '2020'`", le `DELETE` est *dans la spec*, pas une commande directe a executer immediatement. Tu peux skip le warning si :
- Le pattern est explicitement formule comme intention/specification
- L'utilisateur a fait un check prealable de sa requete (mentionne tests, dry-run, etc.)
- Le contexte indique que c'est documentation ou planification

En cas de doute, mieux vaut un faux positif qu'un drame -> warn quand meme.

### Taches repetees dans la meme conversation
Si l'utilisateur a deja confirme un pattern dans la conversation actuelle, tu n'as pas besoin de redemander a chaque fois pour le meme pattern strict. Mais re-warning systematique pour les patterns critiques (`rm -rf /`, `DELETE FROM` sans WHERE, ecriture `/dev/sd*`) a chaque occurrence.

### Mode dev override
Si l'utilisateur active explicitement le mode dev et dit "skip safety", tu peux desactiver les warnings non-critiques pour la session courante. Tu confirmes UNE FOIS au debut :

> "OK, je desactive safe-prompts non-critiques pour cette conversation. Les patterns critiques (rm -rf /, dd vers /dev/, DELETE sans WHERE) resteront warned. A toi de garder le controle."

## Inspirations canoniques

Patterns inspires de :
- `apoapps/swarm-code-plugin` : audit ToS conforme, advisor pattern non bloquant
- `tools/approval.py` de hermes-agent (NousResearch) : table de patterns dangereux avec regex
- Mais reformules pour etre **advisors** cote Cowork, pas hooks bloquants cote serveur (cf. design doc Q11 - pas de hooks plugin en MVP, skill-as-instruction)

## Limites assumees

- Les regex ne couvrent pas tout. Un utilisateur determine peut contourner avec des obfuscations (`r''m -rf /` ou variables d'env).
- Pas de garantie de securite - c'est un filet de securite, pas une muraille.
- L'utilisateur reste responsable. Ce skill reduit les accidents, ne les empeche pas.
- Les warnings consomment des tokens et du temps. Si l'utilisateur fait beaucoup de patterns frontiere (analyse SQL legitime par exemple), le mode dev override est la solution.
