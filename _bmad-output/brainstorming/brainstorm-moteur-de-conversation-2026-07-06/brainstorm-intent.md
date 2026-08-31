# Intent — Moteur de conversation ZAX

> Sortie de brainstorming du 2026-07-06, complétée par une reprise de session ayant tranché tous les points restés ouverts. Contrat d'entrée pour le PRD. Les décisions `by user` du memlog font autorité.

## 1. Contexte & objectif

Point dur du pipeline ZAX : transformer un message brut joueur en décisions fiables et robustes terrain — découpage du texte, identification faction/personnage, détection de mots-clés/triggers, calcul de karma, choix de la personnalité active, détection d'imposteur. Exploré via une Solution Matrix à 6 lignes (A=découpage, B=faction/personnage, C=triggers, D=karma, E=choix de personnalité, F=imposteur) x 4 colonnes d'approches (1=détection exacte, 2=embeddings/similarité, 3=LLM, 4=logique/maths).

## 2. Principes directeurs actés

- **Pipeline hybride** : front de perception (A/B/C) porté par mots-clés + embeddings ; back de décision (D/E/F) porté par maths/règles/compteurs (col.4).
- **Pattern transversal par colonne** : col.1 = détection exacte/déterministe (premier passage systématique) ; col.2 = embeddings/similarité pour le fuzzy/groupement ; col.3 = LLM, unanimement **backup à risque de crash, jamais primaire** (décision actée sur toute la colonne A3/B3/C3/D3/E3/F3, y compris B3 après résolution de sa tension initiale — voir §3.B) ; col.4 = logique/maths déterministe en sortie.
- **Logging admin transversal sur toute la colonne 3** : chaque catch-up/détection LLM backup (A3/B3/C3/D3/E3/F3) émet un log qui remonte sur les écrans admin. Double usage : en alpha/tests = debug et tuning des seuils/paramètres ; en live = filet de rattrapage visible par l'orga sur ce qui a été raté à l'instant.
- Le **back du moteur (D/E/F) est entièrement déterministe** — jugé rassurant pour la fiabilité terrain.
- L'identification d'identité (faction/personnage) **ne se traite jamais en probabilités** — voir rejet B4.

## 3. Décisions actées, par ligne

### A — Découpage du texte
Pipeline en 4 rôles complémentaires : **A4** segmente le texte brut par mots de ponctuation (mais, donc…) en sous-blocs — le corpus de test représentatif est **reporté**, alimenté plus tard par un mini-projet séparé déjà en cours (source non détaillée dans le memlog) ; pas de génération synthétique décidée pour l'instant. **A2** groupe ces sous-blocs via embeddings **nomic-embed-text (Ollama, local)** par clustering non supervisé — comparaison au bloc précédent, fusion si similaire sinon nouveau super-bloc, ce sont ces super-blocs qui sont traités en aval ; seuil de similarité paramétré via une **variable de config dédiée, initialisée à 0,7** (point de départ, à affiner en tests/alpha via la config live). **A1** pose des tags (mots-clés perso + mots-clés génériques hors perso, dictionnaires pré-indexés au lancement, fixes sur le GN) au niveau bloc ET message entier, pour permettre des échos entre conversations sur les mêmes thèmes. **A3** est le backup LLM, sortie JSON structurée imposée et validée au même format que le pipeline déterministe.

### B — Identification faction/personnage
**B1** en premier passage systématique : comparaison de string exacte en dur contre une table des noms de factions/personnages. **B2** ne s'active que si B1 ne trouve rien : vectorisation avec seuil de similarité paramétré via une **variable de config dédiée** (distincte de celle d'A2 — usages différents), elle aussi **initialisée à 0,7**, peut aussi ne rien trouver (l'utilisateur peut ne parler de personne). **B3** : la tension initiale (« LLM lancé en premier, avant B1/B2 ») est **résolue** — B3 redevient un backup pur, aligné sur le pattern transversal des autres lignes : B1 reste le premier passage, B2 le passage secondaire, B3 tourne en parallèle/async en catch-up, sans jamais bloquer ni gater la réponse temps réel (log admin, cf. §2).

### C — Mots-clés / triggers
**C1** : champ mots-clés hard dédié dans la config de chaque personnalité, match exact insensible à la casse/accents/pluriel. **C2** : validation des soft triggers au niveau du bloc — le process A doit transmettre tous les soft triggers détectés. **C3** : pas de LLM backup pour l'instant (trop coûteux en temps), idée gardée de côté pour plus tard. **C4** : C se limite à la détection ; le comptage (nombre de messages abordant un sujet) est délégué à D.

### D — Karma
**D2** : comptage à l'échelle de la conversation, intégrant la valeur de karma déjà enregistrée en base des conversations précédentes ; nouvelle valeur réenregistrée en fin de conversation si le personnage a été identifié. **D3** : pas de LLM. **D4** : mécanisme anti-farm par **cooldown** — date `lastUsed` par triplet (personnage, personnalité active, sujet de karma) ; si l'écart est < 3h, pas de regain de karma sur ce triplet. **Formule karma** (tranchée en reprise de session) : additive simple, chaque delta validé (sujet ou trigger) s'ajoute/se soustrait tel quel, **clampée 0–100 par ligne** — `karma_pj` et `karma_faction` calculés et clampés séparément ; **pas de decay temporel**. **Encodage d'une personnalité « sans karma » (l'Archiviste)** (tranché en reprise de session, avec révision) : **pas de flag** dédié dans le YAML — la structure du bloc `karma:` reste strictement identique entre toutes les personas ; pour une persona sans karma, les points gagnés/perdus sont simplement mis à **0 sur tous les sujets**, le moteur D calcule normalement et le delta est toujours nul. Zéro branche spéciale dans le moteur, aucune perturbation pour les scénaristes/écrivains.

### E — Choix de la personnalité active
Ligne **supplantée** : non tranchée dans cette session, elle a été entièrement redésignée par la session dédiée **`brainstorm-moteur-vote-pondere-2026-07-09`** (statut complete) — cette session est la source de vérité pour la ligne E.

### F — Détection d'imposteur
Déjà ~90% couvert par **DEC-06** (lookup existe/vivant/présent + compteur X essais/Y min → déclenche Le Gardien) ; F1-F4 ne sont que des raffinements non urgents. **F1** — **confirmé** après validation avec Boris (co-organisateur) : tant que le joueur n'est pas identifié, ZAX passe en **mode chit-chat** et cherche activement à identifier la personne, au minimum en le demandant. **F2** : hors scope MVP. **F3** : pas de LLM, lookup standard en base suffisant. **F4** : confirmé hors urgence — priorité de ces raffinements à déterminer en sortie de MVP.

## 4. Rejets explicites

- **B4 — probabilités pour l'identité** : abandonnées. L'identification faction/personnage reste 100% exacte via B1 (comparaison de string en dur) ; on ne joue pas avec des probabilités sur une identité.
- **Flag `karma_exempt`** (encodage Archiviste) : décidé une première fois en reprise de session, puis **explicitement annulé** par une révision dans la même session au profit de l'encodage « points à 0 sur tous les sujets » — voir §3.D. Ne pas réintroduire ce flag.

## 5. Points ouverts pour le PRD

Aucun. La reprise de session en fin de memlog a explicitement tranché ou reporté (décision assumée) tous les points listés dans la version précédente de ce document : la tension B3 (résolue en §3.B), l'encodage karma de l'Archiviste (résolu en §3.D), la formule karma complète (résolue en §3.D), les seuils de similarité A2/B2 (fixés à 0,7 en config, §3.A/§3.B), le corpus de test A4 (reporté à un mini-projet séparé, §3.A) et F1 (confirmé avec Boris, §3.F). Le memlog le constate lui-même explicitement : « tous les points de la section 5 sont tranchés ou explicitement reportés ».
