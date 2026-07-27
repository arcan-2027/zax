# Intent — Maîtrise MJ en temps réel (dashboard orga ZAX)

> Sortie de brainstorming du 2026-07-07. Contrat d'entrée pour le PRD. Les décisions `by user` du memlog font autorité.

## 1. Contexte & objectif

Dashboard orga du GN Fallout 2027 : supervision temps réel de l'IA ZAX sur 5 terminaux joueurs, sessions de 48–72h, orgas fatigués et en effectif réduit. Le dashboard n'est pas un outil d'admin mais un **poste de pilotage narratif** (inspirations : contrôle aérien, urgences, régie TV).

## 2. Principes directeurs actés

- **ZAX est un personnage, pas un assistant** : il n'existe qu'en jeu. Le dashboard ne fait jamais parler ZAX hors fiction (fondement du rejet du ZAX-copilote).
- **Principe « 3h du matin »** : chaque feature est jugée sur « un orga épuisé, seul, à 3h du matin peut-il s'en servir sans réfléchir ».
- **Surface obligatoire** : chaque feature est taguée [Joueur] / [Admin] / [Les deux] — critère nécessaire pour trancher.
- **Le timer d'auto-envoi est LA pièce maîtresse UX** : il définit le rythme du jeu entier (lié à DEC-04 et aux cibles de latence REF-10). Investir dans son réglage par terminal et par personnalité.
- Optimiser pour **un orga fatigué**, pas pour la charge : 5 terminaux max, zéro scalabilité superflue.
- Toute action orga est loguée dès le jour 1 (journal de débrief gratuit).

## 3. Features priorisées — 3 cercles

### C1 — Vital MVP jour J

- **Cockpit 5 conversations** [Admin] : flux live des 5 terminaux. La forme UX (colonnes ou non) reste ouverte — à trancher plus tard.
- **File de validation unifiée avec timer** [Les deux] : toutes les réponses en attente dans une seule file (pas par terminal), triée par **score de triage auto** (enjeu narratif + risque — retenu). Timer d'auto-envoi visible, gelable ; bouton « ZAX réfléchit » côté joueur (animation immersive) pour absorber la latence de validation (résout la tension DEC-04).
- **Prise de main orga + injection de contexte** [Les deux] : nécessaire en V1 (V1 : champ texte brut par terminal).
- **Forçage de personnalité** [Admin] : V1 = update `zax_config` + 5 boutons bruts.
- **Mode dégradé manuel** [Les deux] : nécessaire au MVP. **Granulaire par terminal** (retenu) : ex. T2 en réponses pré-programmées, les autres en LLM.
- **Persistance des messages** [Les deux] : nécessaire au MVP, non fakable — file d'écritures Supabase = noyau dur jour 1 (§6.10, aucune perte de message).
- **Vue terminaux** [Admin] : V1 = 5 cartes statut (connecté, joueur, personnalité active).

### C2 — Confort de garde 48–72h

- **Alertes multi-niveaux** [Admin] : info=badge, warn=bannière, critical=plein écran + son. Toute alerte affiche d'abord terminal + joueur + personnalité. **Le son doit être désactivable.** V1 : bannière seule ; son + mobile (webhook ntfy/Telegram) en itération 2.
- **Handover / passation** [Admin] : mini-résumé de handover à mise à jour **événementielle** (événements majeurs) + rafraîchissement périodique **15 min** aligné sur le digest ; l'orga sortant relit, l'orga entrant signe.
- **Readback + friction adaptative à 3 paliers** [Admin] : réversible = clic simple · impact jeu = confirmation avec reformulation complète (« Vous forcez LE GARDIEN sur T3 — confirmer ») · quasi irréversible = friction physique (taper le nom du terminal ou slider). Objectif : protéger l'orga fatigué de ses propres erreurs.
- **Résumé IA par conversation** [Admin] : **à la demande en V1** (bouton « résumer ») — le GPU est partagé avec les réponses joueurs ; résumé continu seulement si les tests GPU montrent de la marge (REF-08/REF-10).
- **Digest 15 min** [Admin] : résumé de ce qui s'est passé sur les terminaux non surveillés.
- **Journal de bord orga partagé** [Admin] : notes horodatées libres (support du handover).
- **Bibliothèque de réponses pré-approuvées par personnalité** [Admin] : insertion en un clic puis édition.

### C3 — Excellence narrative

- **Cue sheet narrative** [Admin] : les déclencheurs narratifs (signal vert, morse, ouvertures) sont des **cues numérotées avec préconditions, effet système, bouton GO avec readback, log horodaté**. Tranche REF-17 : bascule d'état d'ouverture **manuelle d'abord**, planification optionnelle ensuite.
- **Page dédiée progression GECK** [Admin] : vue progression (6 modules) dans une page dédiée du dashboard — pas en header. L'alerte « module GECK reçu » (cérémonie plein écran + son distinctif) reste.
- **Replay / bookmarks karma** [Admin uniquement, jamais côté joueur] : replay des dernières minutes d'une conversation, bookmarks des forts deltas karma pour le débrief.
- **Kill switch narratif** [Les deux] : bouton « crash ZAX scénarisé » (panne immersive volontaire) qui retourne la contrainte technique en événement de jeu.
- Radar narratif des sujets chauds, jauges de temps d'antenne par personnalité, mode simulation/répétition pré-GN, presets de crise, raccourcis physiques (stream deck).

## 4. Rejets explicites

- **ZAX-copilote** (panneau de suggestions « ce joueur tourne autour du module agronomie… ») : REJETÉ — ZAX est un personnage, pas un assistant ; il n'existe qu'en jeu.
- **Compteur GECK en header du dashboard** : REJETÉ — remplacé par la page dédiée progression GECK (C3).

## 5. Points ouverts pour le PRD

- **Forme UX du cockpit** (colonnes, kanban, picture-in-picture…) : volontairement non tranchée — à instruire en phase UX.
- **Valeurs du timer d'auto-envoi** par terminal / personnalité (lié à REF-03 / REF-10 / DEC-04) : à calibrer, prévoir les métriques de charge orga (validations/h, temps moyen) pour l'ajuster.
- **Résumé IA continu** : conditionné aux tests GPU (marge sur la GTX 1080 partagée).
- **REF-17** : tranché vers manuel d'abord ; la planification automatique des ouvertures reste optionnelle/ultérieure.
- Rôles multi-orgas (main levée, expeditor, tableau de garde) : évoqués, non priorisés — à positionner selon l'effectif orga réel.
