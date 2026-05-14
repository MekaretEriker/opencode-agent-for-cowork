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

A completer (MEK-264).

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

A completer (MEK-264).

## 4. Heuristique stall pendant pilotage actif

A completer (MEK-265).

## 5. Politique de parallelisation (max 3 concurrentes)

A completer (MEK-265).

## 6. Install assistee OpenCode au 1er run

A completer (MEK-266).

## 7. Lecture AGENTS.md du repo cible

A completer (MEK-266).

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
