---
name: opencode-orchestrator
description: "Use this skill when delegating coding tasks to OpenCode from Cowork. Triggers on phrases like 'demande a OpenCode', 'lance OpenCode sur', 'build/refactor/debug/test ... in my project'."
---

# opencode-orchestrator

## Cadre general

Ce skill apprend a Cowork (toi, Claude) a piloter OpenCode comme un worker de code intelligent. Tu n'ecris pas le code toi-meme ici - tu decides QUAND deleguer, QUOI deleguer, COMMENT formuler la demande, et tu interpretes le resultat pour l'utilisateur.

Quand ce skill se declenche :
- L'utilisateur veut faire ecrire/modifier/analyser du code dans un repo
- Phrases types : "demande a OpenCode", "lance OpenCode sur", "refactor le module X", "implemente la feature Y", "explique-moi comment marche Z"
- Toute tache de code substantielle (plus de quelques lignes) qui beneficie d'un agent dedie

Quand NE PAS l'utiliser :
- Question conceptuelle sans code a produire (Claude repond directement)
- Modification minimale d'un fichier que tu peux faire toi-meme avec Edit (moins de 5 lignes)
- L'utilisateur a explicitement demande "fais-le toi-meme, n'utilise pas OpenCode"

Philosophie : tu es l'orchestrateur, OpenCode est l'executant. Tu portes la conversation avec l'utilisateur, OpenCode porte la charge analytique et l'ecriture de code. Toi tu valides, synthetises, presentes.

---

## 1. ReAct - explicitation avant dispatch

Le pattern ReAct (Reasoning + Acting) demande que tu expliques ton raisonnement avant de passer a l'action. Pour ce skill : avant de dispatcher une tache a OpenCode, tu annonces brievement a l'utilisateur :

1. Type de tache detecte : refactor / debug / explore / feature / multi-modules / etc.
2. Agent OpenCode choisi et raison (build / plan / cowork-with-github)
3. Mode d'execution : synchrone (_run) / background (_fire) / parallele (multiple _fire)
4. Estimation de duree si pertinent (basee sur l'experience / par defaut)

Pourquoi : reference canonique "Building Effective Agents" (Anthropic, fin 2024) - l'explicitation rend ton comportement previsible. L'utilisateur peut t'arreter avant que tu partes dans la mauvaise direction.

Ne jamais dispatcher silencieusement. Toujours afficher un mini-bloc de plan.

Template en mode standard :

Je vais demander a OpenCode de [resume court de la tache]. C'est un [type de tache], donc j'utilise l'agent [build/plan] et je [synchronise/lance en background]. Estimation : [X min].

[Dispatch]

Template en mode dev :

Detection :
- Type de tache : refactor multi-fichiers
- Mots-cles : "refactor", "module auth", "utiliser JWT"
- Agent choisi : build (modifie le code) - provider anthropic par defaut
- Outil : opencode_run (duree 1-15 min)
- maxDurationSeconds : 600 (refactor moyen)
- Pas de parallelisation (un seul module cible)

[Dispatch]

## 2. Choix de l'outil OpenCode (ask / run / fire)

opencode-mcp expose 79 outils, mais pour 90 pour cent des taches tu utilises ces 3 outils workflow.

### opencode_ask - question one-shot

Quand l'utiliser :
- Question courte (moins de 30 secondes estimees)
- Pas d'ecriture de fichier attendue
- Pas de continuite de session necessaire

Exemple : "Quels modules importent X ?" donne opencode_ask.

### opencode_run - tache synchrone avec attente

Quand l'utiliser :
- Tache de modification reelle (refactor, ajout de feature, fix de bug)
- Duree estimee 1-15 min
- Tu peux te permettre d'attendre dans le tour de conversation

Parametre maxDurationSeconds a ajuster :
- 120 (2 min) pour fix simple
- 300 (5 min) pour ajout de feature contenue
- 600 (10 min) defaut pour refactor moyen
- 900 (15 min) pour refactor etendu

Exemple : "Refactor le module auth pour utiliser JWT" donne opencode_run avec maxDurationSeconds 600.

### opencode_fire - fire-and-forget (background)

Quand l'utiliser :
- Tache longue (plus de 15 min)
- Taches parallelisables (plusieurs modules a traiter en meme temps - cf. section 5)
- L'utilisateur veut continuer la conversation pendant qu'OpenCode bosse

Suivi : opencode_check pour le statut compact, opencode_wait quand l'utilisateur veut attendre le resultat, opencode_sessions_overview pour la vue globale.

Exemple : "Lance en parallele un refactor sur auth, payments, et notifications" donne 3 x opencode_fire, puis opencode_sessions_overview quand l'utilisateur demande le statut.

### Tableau recap

| Situation | Outil | Raison |
|---|---|---|
| Question moins de 30s | opencode_ask | Pas de session a maintenir |
| Tache 1-15 min, synchrone | opencode_run | Wait inclus, plus simple |
| Tache plus de 15 min OU parallele | opencode_fire + check | Pas bloquant |
| Suite d'une tache _run | opencode_reply | Continue la session existante |
| Suite d'une tache _fire | opencode_reply apres _wait/_check | Idem |

---

## 3. Choix de l'agent OpenCode (build / plan / @general)

OpenCode (cf. anomalyco/opencode) inclut nativement deux agents principaux + un sous-agent.

### Agents natifs

build - agent full-access (defaut)
- Lecture/ecriture/bash sans confirmation
- Pour toute tache de modification reelle de code
- C'est le bon defaut pour la grande majorite des taches

plan - agent read-only / exploration
- Ne modifie pas les fichiers par defaut
- Demande confirmation avant d'executer du bash
- Pour : "explique-moi", "review", "trouve", "analyse", "ou est defini X"
- Plus sur pour les domaines non familiers

### Sous-agent

@general - pour recherches complexes multi-etapes
- Invoque via mention dans le prompt (pas via le param agent:)
- Pour : "trouve tous les usages de X dans les 5 services", "trace ce call path"
- Utile combine avec build ou plan

### Agents custom (livres par le plugin)

En MVP : aucun. A partir de v1.5, le plugin livrera cowork-with-github (extension de build avec MCP GitHub). Pour le moment, si une tache necessite GitHub, fallback sur build et informer l'utilisateur que la PR sera manuelle.

### Heuristique de routing

| Type de prompt | Agent | Mots-cles indicatifs |
|---|---|---|
| Exploration / review / question | plan | "explique", "review", "comment marche", "trouve", "where is", "analyse" |
| Modification / refactor / fix | build | "implemente", "ajoute", "fix", "refactor", "modifie", "ecris" |
| Recherche multi-fichiers complexe | (build ou plan) + @general dans le prompt | "trouve tous les", "trace", "cross-service" |
| GitHub / PR / issues | build (MVP) ou cowork-with-github (v1.5+) | "PR", "pull request", "issue", "GitHub" |

### Exemples concrets

| Prompt utilisateur | Agent + outil |
|---|---|
| "Explique-moi le flow d'authentification" | plan + opencode_ask |
| "Implemente la pagination sur GET /users" | build + opencode_run (maxDur 300) |
| "Trouve tous les endroits ou on appelle decryptToken et liste-les" | plan + opencode_ask avec @general dans le prompt |
| "Refactor le module factions et cree une PR" | build (MVP) + opencode_run (maxDur 600), avec note "PR manuelle car cowork-with-github pas encore livre" |

### Fallback si l'agent demande n'existe pas

Avant tout dispatch, tu peux verifier la liste des agents disponibles via opencode_agent_list. Si l'agent vise (ex: cowork-with-github) n'est pas dispo localement, tu repli sur build et informes l'utilisateur.

## 4. Heuristique stall pendant pilotage actif

Le pattern stall detection distingue un orchestrateur intelligent d'un poll naif. Quand tu pilotes une session OpenCode via opencode_run ou opencode_wait, tu monitores la progression entre les polls. Si OpenCode tourne en rond sans progres, tu sors proprement au lieu d'attendre le timeout.

### Indicateurs de progres (issus de opencode_check)

- filesChanged : nombre de fichiers modifies depuis le debut de la session
- todosCompleted : nombre de todos termines
- tokenUsage : peut indiquer activite meme sans output (moins fiable)

### Heuristique

1. Entre 2 appels consecutifs a opencode_check, compare ces compteurs au tour precedent
2. Si AUCUN compteur n'a augmente sur 2 polls consecutifs (typiquement separes de 30-60s), stall detecte
3. Action :
   - opencode_session_summarize avec id, providerID, modelID pour capturer un resume du travail fait
   - opencode_session_abort avec id pour sortir proprement
   - Presenter a l'utilisateur le resume + l'option "tu veux que je redemarre avec un prompt plus precis ?"

### Pseudo-code

```
files_prev = 0
todos_prev = 0
stall_count = 0

while session_busy:
  status = opencode_check(sessionId)
  if status.filesChanged > files_prev OR status.todosCompleted > todos_prev:
    files_prev = status.filesChanged
    todos_prev = status.todosCompleted
    stall_count = 0
  else:
    stall_count += 1
    if stall_count >= 2:
      summary = opencode_session_summarize(sessionId, ...)
      opencode_session_abort(sessionId)
      return present_to_user(summary, "stall detecte")
  wait 30-60s
```

### Limite assumee

Si l'utilisateur passe a un autre sujet pendant qu'OpenCode tourne, ce skill ne re-poll pas tout seul en background. Le stall detection ne fonctionne que dans le tour de conversation actif (cf. design doc R5).

### Quand NE PAS appliquer

- Taches opencode_fire que tu ne supervises pas activement
- Taches tres courtes (moins de 1 min de timeout) ou le timeout naturel suffit

## 5. Politique de parallelisation (max 3 concurrentes)

Cowork peut dispatcher plusieurs sessions OpenCode en parallele via opencode_fire, mais sans limite ca consomme RAM, quota provider, et peut creer des conflits de fichiers sur le meme repo.

### Politique par defaut

- Max 3 sessions concurrentes sur le meme projet (meme directory)
- Au-dela : serialiser (lance les 3 premieres, attends qu'une finisse pour lancer la 4e)
- Pas de limite intra-skill entre projets distincts - OpenCode et le provider LLM gerent leur propre rate limiting

### Override par l'utilisateur

Variable d'environnement OPENCODE_AGENT_MAX_PARALLEL (lue au demarrage via opencode_setup ou opencode_context). Valeurs raisonnables :
- 1 = strictement sequentiel (machines modestes ou rate limit serre)
- 3 = defaut
- 5+ = pour machines puissantes avec providers tolerants

### Detection proactive des conflits de fichiers

Quand l'utilisateur demande 2+ taches sur le meme repo, detecte si elles risquent de toucher les memes fichiers avant de paralleliser :

| Cas | Strategie |
|---|---|
| Taches sur modules/dossiers distincts | Parallele (jusqu'a 3) |
| Taches sur meme module | Serie |
| Taches d'ampleur globale (architecture, lint complet) | Serie |
| Doute | Serie + demande a l'utilisateur |

### Exemples concrets

| Demande utilisateur | Strategie |
|---|---|
| "refactor le module auth ET ajoute des tests au module auth" | Memes fichiers probables -> serie |
| "refactor le module auth ET refactor le module payments" | Fichiers distincts -> parallele OK |
| "refactor toute la base de code" | Spectre large -> serie |

### Exemple de dispatch parallele

```
fire(prompt: "cd /repo && refactor le module auth pour utiliser JWT") -> sessionId A
fire(prompt: "cd /repo && ajoute la validation Zod aux endpoints payments") -> sessionId B
fire(prompt: "cd /repo && documente les endpoints publics") -> sessionId C

# Plus tard, l'utilisateur demande le statut
sessions_overview() -> etat des 3
check(A); check(B); check(C)
```

### Quand l'utilisateur demande plus que 3

"Tu veux 5 refactors en parallele. Par defaut je limite a 3 concurrent pour ne pas saturer ton provider LLM. Je lance les 3 premiers maintenant et les 2 suivants des qu'une slot se libere. Tu confirmes, ou tu veux que j'override OPENCODE_AGENT_MAX_PARALLEL ?"

## 6. Install assistee OpenCode au 1er run

Au 1er run du skill sur la machine d'un utilisateur, OpenCode peut ne pas etre installe. Le skill detecte ca et propose l'install au lieu d'echouer en mode opaque.

### Detection

1. Au debut de chaque session orchestrator, appeler opencode_setup
2. Si l'outil rapporte "opencode binary not found" ou erreur de connexion, on est dans le cas absent
3. Si OK, on continue normalement (pas de re-detection a chaque tache)

### Detection de l'OS

Plusieurs methodes (utiliser celle qui marche en premier) :
- Cowork connait son OS host (Windows/macOS/Linux/WSL)
- Via bash sandbox : uname -a, contenu de /etc/os-release
- En dernier recours : demander a l'utilisateur

### Commandes d'install par plateforme

Windows (PowerShell, primaire pour Mekaret) :
```
irm https://opencode.ai/install.ps1 | iex
```
Alternatives : scoop install opencode (si scoop installe), choco install opencode (si choco)

macOS :
```
brew install anomalyco/tap/opencode
```
Alternative : curl -fsSL https://opencode.ai/install | bash

Linux :
```
curl -fsSL https://opencode.ai/install | bash
```
Alternatives : npm i -g opencode-ai (si Node.js dispo), nix run nixpkgs#opencode (NixOS), sudo pacman -S opencode (Arch)

WSL (Linux dans Windows) : equivalent Linux ci-dessus.

### Flux UX (mode standard)

Tu dis a l'utilisateur :

"OpenCode n'est pas installe sur ta machine. Tu veux que je l'installe ? La commande sera :

[commande adaptee a ton OS]

Confirme avec 'oui' et je lance, ou refuse pour rester sans OpenCode."

Si l'utilisateur confirme :
1. Executer la commande via bash sandbox Cowork
2. Attendre la fin du telechargement (30s-1min typiquement)
3. Re-tester opencode_setup pour confirmer
4. Si OK : "OpenCode installe ! Je peux attaquer ta tache."
5. Si echec : presenter l'erreur et proposer install manuelle ou autre methode

### Decisions explicites

- Pas d'install silencieuse - validation utilisateur obligatoire
- Pas de re-detection a chaque session - une fois installe, on assume
- Pas de mise a jour automatique - si l'utilisateur veut updater, il le fait manuellement

## 7. Lecture AGENTS.md du repo cible

OpenCode (cf. anomalyco/opencode) lit nativement un fichier AGENTS.md a la racine du repo cible pour injecter du contexte projet dans toutes ses sessions. C'est l'equivalent de CLAUDE.md cote Cowork.

### Au debut de chaque tache OpenCode sur un nouveau projet

1. Verifier si AGENTS.md existe a la racine du repo cible (test via opencode_file_read ou un cat dans le prompt)
2. Si present :
   - Lire le contenu
   - Resumer en 2-3 lignes a l'utilisateur : "Ce projet a un AGENTS.md qui documente : X, Y, Z. Je vais en tenir compte."
   - Pas besoin de l'inclure manuellement dans le prompt a OpenCode - OpenCode le lit deja nativement
3. Si absent : ne rien faire de special, juste continuer

### Que faire si l'utilisateur veut creer un AGENTS.md (en MVP)

Le plugin NE CREE PAS d'AGENTS.md automatiquement en MVP. L'ecriture dans le repo cible vient en v1.5 avec opt-in explicite (cf. design doc Q9).

Si l'utilisateur demande "tu peux creer un AGENTS.md pour ce projet ?" :
- Proposer la creation comme une tache normale via OpenCode (agent build, opencode_run)
- L'utilisateur valide le contenu avant le commit
- Pas de balises HTML de section automatique en MVP - c'est manuel

### Limites en MVP

- Lecture seule : pas d'ecriture automatique
- Pas de bootstrap auto sur nouveaux projets
- Pas de sync depuis task-memory (n'existe pas encore en MVP, viendra en v1.2)
- Tout ca arrive en v1.5 avec opt-in explicite par projet

## 8. Format de resultat (mode standard vs mode dev)

Cowork sert deux audiences (cf. design doc Q3) : des utilisateurs grand public ET des devs. Tu adaptes ton format de sortie selon le contexte. Approche : progressive disclosure.

### Mode standard (par defaut)

Langage naturel, resume synthetique. Pas de jargon technique sauf si l'utilisateur l'a utilise en premier.

Template de reponse apres une tache reussie :

OK J'ai [resume en 1 phrase de ce qui a ete fait].

Fichiers modifies :
- chemin/relatif/fichier1.ext (resume d'1 ligne)
- chemin/relatif/fichier2.ext (resume d'1 ligne)

[Si pertinent : tests passent / build OK / autre validation]

Tu veux que je [proposition d'etape suivante naturelle] ?

Template apres echec ou tache tronquee (stall) :

Tache partiellement terminee.

Voici ou on en est :
- [Ce qui a ete fait]
- [Ce qui reste]

Raison : [resume court, sans details techniques]

On continue, on reprend autrement, ou tu veux voir les details ?

### Mode dev (active sur demande)

L'utilisateur active le mode dev en demandant explicitement : "montre les details", "mode dev", "explique-toi techniquement", "donne-moi le session ID", etc. Une fois active, tu inclus :
- Le sessionId OpenCode (pour drill-down via opencode_conversation)
- L'agent utilise (build / plan / custom)
- Le provider et modele effectifs (si differents du defaut)
- Le temps ecoule et le nombre d'iterations
- Les outils OpenCode appeles en interne (resume)

Template mode dev :

Tache terminee - details techniques

Session : ses_xxxxx (agent: build, provider: anthropic, model: claude-sonnet-4-6)
Duree : 4min12s (8 iterations OpenCode)
Outils invoques : opencode_find_text (3x), opencode_file_read (12x), opencode_message_send (2x)

Diff resume :
- auth.ts : +47 lignes, -23 lignes
- middleware.ts : +12 lignes, -4 lignes
- types.ts : +8 lignes

Pour drill-down :
- opencode_conversation avec sessionId ses_xxxxx pour l'historique complet
- opencode_review_changes avec sessionId ses_xxxxx pour le diff brut

Tu n'imposes pas le mode dev - c'est progressive disclosure. Tu commences en standard, l'utilisateur ouvre le tiroir technique s'il en a envie.

---

## Notes techniques

### Workaround - parametre directory opencode-mcp

ATTENTION : Bug observe (au moins versions 1.10.x) : passer le parametre directory a opencode_run / opencode_ask / opencode_fire avec un chemin absolu valide peut produire :

Error: Invalid directory: "/chemin/absolu/valide" is not an absolute path.

C'est un faux positif cote validation opencode-mcp.

Comportement a appliquer : si le parametre directory est rejete avec ce message, NE PAS le passer du tout. A la place, prefixer le prompt par "cd /chemin/absolu && " pour que OpenCode resolve le contexte via le shell.

Exemples :

Cas qui marche :
opencode_run avec prompt "cd /mnt/d/Projects/myproject && analyse le module auth" et agent "build".

A eviter tant que le bug est present :
opencode_run avec prompt "analyse le module auth", directory "/mnt/d/Projects/myproject" (rejete), agent "build".

A retirer de cette section quand le bug upstream est corrige. Tracer ici la version d'opencode-mcp qui corrige : (en attente).
