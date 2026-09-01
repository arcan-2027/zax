# Confrontation — brainstorms BMAD d'Auriane × décisions actées DEC-01 → DEC-24

> Document de position du 31/08/2026. Établi après la revue complète des ambiguïtés
> du même jour (DEC-10 → DEC-24) et le merge de la PR #1 (`reflexion_bmad_auriane`).
>
> **Objet** : confronter les quatre sessions de brainstorm d'Auriane aux décisions
> actées, isoler les conflits réels, et préparer l'arbitrage. Ce document ne tranche
> rien — il instruit. Les conflits impliquent deux parties prenantes.
>
> **Sources confrontées** (`_bmad-output/brainstorming/`) :
> - `brainstorm-moteur-de-conversation-2026-07-06` (statut *complete*, repris le 31/08)
> - `brainstorm-orchestration-personnalites-multiples-2026-07-06` (*complete*)
> - `brainstorm-moteur-vote-pondere-2026-07-09` (*complete*) + `zax_weights.yaml` + `exemple-override-personnalite.yaml`
> - `brainstorm-maitrise-mj-temps-reel-2026-07-07` (*complete*)

---

## 0. Résumé exécutif

Le travail d'Auriane et les décisions actées **convergent sur l'architecture de fond**.
Elle arrive, par un chemin entièrement différent (matrice de solutions BMAD à 6 lignes ×
4 colonnes), au même principe que DEC-17 : *colonne 1 = exact et déterministe, colonne 2 =
embeddings, colonne 3 = LLM toujours en backup et jamais primaire, colonne 4 = maths et
règles déterministes*. Cette convergence indépendante est le résultat le plus rassurant de
la confrontation : ce n'est pas une préférence d'auteur, c'est la forme que le problème
impose.

Restent **six conflits francs**, un **trou de gouvernance** et une **série d'amendements**
que ses brainstorms imposent aux décisions actées. Deux de ses arbitrages, enfin, ont été
pris sous une contrainte matérielle qui n'existe plus.

| | Nombre |
|---|---|
| Convergences confirmant une décision actée | 6 |
| Conflits francs à arbitrer | 6 |
| Décisions actées à amender | 7 |
| Conceptions non couvertes par un DEC-XX | 1 (majeure) |
| Arbitrages à rouvrir suite à la machine RTX 5090 | 2 |

---

## 1. Convergences — ce qui est confirmé de part et d'autre

Ces points ne demandent aucun arbitrage. Ils sont listés parce qu'une convergence
indépendante est une information : elle transforme une décision en fait établi.

| Décision d'Auriane | Décision actée | Nature de l'accord |
|---|---|---|
| « LLM = backup uniquement, jamais primaire » — qualifiée d'**unanime** sur A3/B3/C3/D3/E3/F3 | DEC-17, étage 3 en filet | Même principe, formulé indépendamment |
| B3 réaligné en cours de session : le LLM repasse en backup pur, en parallèle/async, catch-up loggé et non bloquant | DEC-17 (lexical d'abord, LLM en dernier) | Elle a corrigé d'elle-même une tentation de LLM primaire |
| B4 : « on abandonne les probabilités pour l'identification Faction/Personnage, on reste 100 % exact » | DEC-17 (jamais de similarité sur ce qui doit être certain) | Même intuition, appliquée à l'identité plutôt qu'à la censure |
| Cue sheet avec préconditions et bouton GO — « tranche REF-17 vers manuel d'abord » | DEC-22 (bascule manuelle, planning en rappel) | Convergence exacte, arrivée sept semaines plus tôt |
| « ZAX-copilote **REJETÉ** : ZAX est un personnage, pas un assistant, il n'existe qu'en jeu » | DEC-11 (LibreChat écarté) | Même ligne, tirée d'un autre raisonnement |
| DECISION F : le moteur est une **fonction pure sans LLM**, testable au golden-test ; le LLM ne fait que *mettre en voix* | §6.9 « le LLM ne doit jamais être imprévisible » | Renforce DEC-17 et lui donne sa stratégie de test |

**Conséquence à acter** : le principe « colonne 3 = LLM backup + log admin » qu'elle étend
à *toutes* les lignes du pipeline mérite d'être inscrit comme règle transverse, pas comme
détail d'implémentation. C'est la formulation la plus nette du principe qu'on ait produite
des deux côtés.

---

## 2. Conflits francs — six points d'arbitrage

### C1 — `nomic-embed-text` contre DEC-21 · **priorité haute**

**Le fait.** Sa décision A2 retient `nomic-embed-text` via Ollama pour le regroupement de
blocs (« fonctionne bien en local »), et le terme `C(p)` du moteur de vote repose sur le
même moteur d'embeddings. DEC-21 **interdit nommément** `nomic-embed-text`, avec
`all-MiniLM` et `mxbai-embed-large`, au profit de `bge-m3`.

**L'enjeu.** Ce sont des modèles centrés anglais. Tout le corpus ZAX — lore, harnais,
messages joueurs, `FAVS`/`LOVE` des personnalités — est en français. Le piège est que rien
ne casse : la recherche continue de retourner des résultats, simplement moins bons, et
aucun test synthétique ne le révèle. C'est le mode de défaillance le plus coûteux à
diagnostiquer, parce qu'il ne produit pas d'erreur.

**Ma position.** DEC-21 tient, et le conflit se résout par la donnée technique, pas par
l'autorité. Mais il faut mesurer la **cascade** honnêtement : changer d'embedder invalide
**tous** les seuils déjà posés — `A2` (regroupement de blocs), `B2` (confirmation
d'identité) et `embedding_seuil: 0.70` dans `zax_weights.yaml`. Un seuil cosinus n'est pas
portable d'un modèle à l'autre.

**Coût de l'arbitrage.** Nul en code (un nom de modèle en configuration), réel en
calibration : les trois seuils repartent de zéro. À faire **avant** tout playtest de
calibration, jamais après.

### C2 — Le découpage du message · **conflit méthodologique**

**Le fait.** Elle tranche A4 (« segmentation par mots de ponctuation : mais, donc… » pour
produire des sous-blocs) puis A2 (regroupement en « super-blocs » par similarité). C'est
un **segmenteur thématique** — précisément ce que DEC-17 écarte, au motif qu'un joueur
costumé tapant sur une borne écrit une à trois phrases et non un paragraphe à segmenter.

**Deux éléments méritent d'être versés au débat.** D'abord, son coach lui a explicitement
proposé notre position (« et si on ne découpe jamais ? Projeter le message entier contre
chaque sujet connu et récolter ceux qui s'allument ») et elle a tranché dans l'autre sens.
Ce n'est donc pas un oubli, c'est un choix. Ensuite, elle note elle-même l'absence de base
empirique : « pas de corpus représentatif existant, à générer/trouver », et le corpus de
test A4 est *reporté* à un mini-projet séparé.

**Ma position.** Aucun des deux camps n'a de données. C'est donc une question à **mesurer**,
pas à trancher au jugement. Proposition : implémenter DEC-17 (message entier, découpage par
phrase au-delà de trois), **instrumenter la longueur réelle des messages** dès la première
alpha, et n'activer A4/A2 que si la distribution observée montre des messages assez longs
pour le justifier. On garde son travail au chaud sans payer sa complexité à l'aveugle.

**À noter** : sa décision A1 pose les tags « au niveau bloc **ET** message entier ». Les
deux approches cohabitent déjà partiellement dans sa conception — l'écart est moins grand
qu'il n'y paraît.

### C3 — Pas de filet LLM sur la ligne C, contre l'étage 3 de DEC-17

**Le fait.** Sa décision C3 : « pas de LLM backup pour C pour l'instant (trop coûteux en
temps), idée gardée de côté si besoin futur ». DEC-17 fait de ce filet l'étage 3 du
pipeline pour les soft triggers et le karma.

**L'enjeu.** Le motif invoqué est un **coût de calcul**, et il était juste : sur une GTX
1080 de 8 Go partagée avec le modèle de réponse, un second appel était inabordable. La
machine RTX 5090 (32 Go, Blackwell) rend un appel court à sortie JSON contrainte
négligeable — de l'ordre de 200 à 400 ms, avec le modèle d'embeddings cohabitant sur le
même GPU (DEC-15, DEC-21).

**Ma position.** Le conflit se lève par la nouvelle donnée matérielle. C'est un arbitrage
qu'il faut reprendre *avec elle*, en lui donnant l'information qu'elle n'avait pas le
6 juillet — pas un désaccord de conception.

### C4 — `l_enfant` au singulier contre DEC-19

**Le fait.** `zax_weights.yaml` ne connaît qu'un seul identifiant `l_enfant` (palette de
l'ouverture 3). DEC-19 pose **deux modules plats et complets**, `ENFANT_EXF` et
`ENFANT_DES`, en exclusion mutuelle.

**Ma position.** Les deux Enfants relèvent du **gate `ORGA-ACTV = 0`**, qui est exactement
le bon outil : un interrupteur d'état, persistant, déjà piloté depuis le dashboard, et
réversible par un superadmin comme DEC-19 l'exige. Les palettes d'ouverture doivent citer
les deux identifiants au lieu du `l_enfant` unique.

> **Mise à jour du 01/09/2026.** Ce conflit portait à l'origine sur un cinquième gate, l'**exclusion
> de Pauli** (combos de personnalités interdits, gating *temporaire*), qui aurait fait
> **revenir l'Enfant éliminé dès que l'autre cessait d'être active** — incompatible avec
> l'élimination *permanente* de DEC-19. Ce gate a été **retiré** de `zax_weights.yaml` et du
> dossier de conception : il n'avait pas été prévu par l'équipe orga. C4 se réduit donc au
> nommage de `l_enfant` dans les palettes.

### C5 — Les palettes d'ouverture contre DEC-23 · **blocage dur**

**Le fait.** `zax_weights.yaml` convoque des personnalités que le noyau de 8 (DEC-23) ne
finance pas.

| Ouverture | Perso par défaut | Palette | Hors noyau |
|---|---|---|---|
| 1 | `happiness_officer` ✅ | happiness_officer, le_board, le_gardien, **le_technicien** | `le_technicien` |
| **2** | **`le_diplomate`** ❌ | **le_diplomate**, l_archiviste, **le_negociateur**, happiness_officer, le_gardien | `le_diplomate`, `le_negociateur` |
| 3 | `mood_manager` ✅ | le_scientifique, l_enfant, **le_soldat_perdu**, **le_juge**, mood_manager, le_gardien | `le_soldat_perdu`, `le_juge` |

Au total **cinq personnalités hors noyau** sont câblées dans la configuration du moteur.
*(Le décompte était de six avant le retrait de l'exclusion de Pauli, qui convoquait
`la_mere` — voir C4.)*

**Pourquoi c'est bloquant et pas cosmétique.** `le_diplomate` est la **personnalité par
défaut de l'ouverture 2**, et son edge case « aucun trigger ne matche » repose précisément
sur le plancher `plancher_defaut` de cette personnalité par défaut. Avec le noyau de 8,
**l'ouverture 2 n'a plus de personnalité par défaut** : le filet de dernier recours tombe
sur `defaut_ultime: le_gardien`, ce qui donne un Gardien omniprésent en ouverture 2 — soit
l'inverse du ton visé (« on apprend à naviguer »).

**Ma position.** Deux issues cohérentes, aucune n'est gratuite :
- **Élargir DEC-23** à ce que les palettes exigent — mais c'est +6 modules, chacun avec sa
  fiche, son karma, son filtrage lore, son graphe `EXIT`/`PRIO`/`TIME` et ses tests. C'est
  exactement le volume que DEC-23 refuse au nom de §6.9.
- **Réécrire les palettes** sur le noyau de 8, en désignant une personnalité par défaut
  d'ouverture 2 parmi le noyau (`l_archiviste` et `happiness_officer` sont les candidates
  naturelles au vu du ton visé).

Ma préférence va à la seconde, avec `le_juge` et `le_soldat_perdu` promus depuis la réserve
**si et seulement si** DEC-20 exige leurs fins — ce qui est le cas. Mais c'est un arbitrage
de périmètre, donc pas le mien.

### C6 — « Le collapse final = la fin » contre DEC-20

**Le fait.** Le brainstorm d'orchestration pose, en piste maîtresse n°6 : *« le collapse
final = la fin : signal bleu = une perso gagne et absorbe les autres, le GN est une lutte
pour qui ZAX devient »*. DEC-20 pose l'inverse : le moteur ne calcule **jamais** la fin
gagnante, ZAX plaide et les humains tranchent.

**Ma position.** La mécanique est excellente et profondément juste sur le plan narratif —
elle donne une forme dramatique à ce qui n'était qu'une liste de huit fins. Le désaccord ne
porte pas sur le *quoi* mais sur le *qui déclenche*. Réconciliation proposée, et elle est
élégante parce qu'elle existe déjà ailleurs dans le projet : **garder le collapse comme
mécanique et comme spectacle, mais le faire déclencher par un orga**, exactement selon le
patron de DEC-19 (le moteur présente les scores, l'humain acte). ZAX ne décide pas de ce
qu'il devient ; les orgas le décident, avec sous les yeux le rapport de force que le moteur
a construit pendant 36 heures. C'est plus fort, et c'est conforme à « l'app informe, elle
n'arbitre pas ».

---

## 3. Le trou de gouvernance — le moteur de vote pondéré n'est acté nulle part

C'est le point le plus important de ce document.

Le cœur de son travail — la formule

```
S(p) = Wt·T(p) + Wk·K(p) + Wc·C(p) + Wi·I(p) + Wo·O(p) + We·E(p)
```

avec ses quatre gates déterministes — **remplace de fait l'étape 3 de `CLAUDE.md` §15**
(« pas de changement / soft ou hard triggers / shutdown vs show-up »). Et cette conception
ne figure dans **aucun `DEC-XX`**.

Ce n'est pas une esquisse. C'est une conception aboutie : termes normalisés 0..1, inertie
statique plus hystérésis à décroissance λ, température unique comme curseur de volatilité
écrasable par ouverture, tie-break par `PRIO`, voix de fond au-delà d'un delta δ, quatre
gates durs (kill-word avec bonus `EXIT`, gate `TIME`, `ORGA-ACTV`, override orga
verrouillé), traitement explicite de sept cas limites, observabilité conçue
comme oracle de test *et* comme « météo interne » pour l'orga. Deux fichiers YAML sont
écrits et validés.

**Ce que ça implique.** Une conception de ce poids qui vit dans un brainstorm est un risque :
elle n'a pas d'autorité, donc rien n'oblige le code à la respecter, et rien ne la protège
d'être contredite par une décision future prise sans la connaître. C'est le candidat
évident au prochain passage `docs/` → `DECISIONS.md`.

**Un point de sa conception meilleur que le nôtre, à adopter au passage.** Sa DECISION G :
en mode dégradé, **le moteur élit quand même la personnalité**, et seule la mise en voix
bascule sur les phrases pré-écrites *de cette personnalité*. `CLAUDE.md` §11 se contente de
phrases génériques catégorisées (identité, accès, erreur). Sa version préserve nettement
mieux l'illusion : un joueur en mode dégradé continue de parler au Gardien ou à
l'Archiviste, pas à un automate neutre. Le coût est nul, le moteur ne consommant pas de LLM.

---

## 4. Amendements que ses brainstorms imposent aux décisions actées

Aucun de ces points n'est un conflit : ce sont des apports qui rendent les décisions
actées incomplètes en l'état.

### DEC-10 — deux amendements

**(a) La machine à états devient *par terminal*.** Elle retient un « mode dégradé
granulaire par terminal ». DEC-10 décrit implicitement un état global. Les compteurs TTFT,
d'échecs et de succès consécutifs se dupliquent donc par borne — ce qui est aussi plus
juste sur le fond : un Raspberry Pi dont le réseau tombe n'est pas une panne du LLM.

**(b) Piège technique à écrire noir sur blanc.** Sa piste maîtresse n°2 exige que chaque
bascule de personnalité soit **télégraphiée** — signature, glitch, glyphe ASCII, et
**latence**. Une latence *intentionnelle*, posée comme signal narratif, peut franchir le
seuil de 8 s et déclencher le mode dégradé. **Le TTFT doit se mesurer hors délai
théâtral** : le compteur démarre après le délai de mise en scène, ou celui-ci est
soustrait. Sans cette règle, plus l'effet est réussi, plus il ressemble à une panne.

### DEC-22 — adopter la cue sheet comme implémentation

Sa cue sheet (préconditions vérifiées, effet système annoncé, bouton GO avec readback, log
horodaté) est l'implémentation concrète du « rappel qui n'actionne jamais » de DEC-22. À
adopter telle quelle : elle est plus précise que la décision qu'elle sert.

### DEC-18 et DEC-19 — adopter la friction adaptative à 3 paliers

Son échelle — réversible = clic simple · impact jeu = confirmation avec reformulation
complète · quasi irréversible = friction physique (taper le nom du terminal, ou slider) —
se mappe directement :

| Action | Palier |
|---|---|
| Don du G.E.C.K. (DEC-18) | Quasi irréversible → **friction physique** |
| Élimination d'un Enfant (DEC-19) | Impact jeu → **confirmation avec reformulation** (réversible par superadmin) |
| Bascule d'état d'ouverture (DEC-22) | Impact jeu → **confirmation avec reformulation** |
| Forçage de personnalité | Réversible (verrou 300 s) → **clic simple** |

### DEC-17 — intégrer quatre raffinements sans conflit

- **Cooldown anti-farm** : date `lastUsed` par triplet (personnage, personnalité active,
  sujet de karma) ; sous 3 h, aucun regain. Répond à un trou réel de DEC-17, qui ne dit
  rien du farming.
- **Formule de karma** : additive simple, clampée 0–100 **par ligne** (`karma_pj` et
  `karma_faction` séparément), sans decay temporel.
- **Cas Archiviste** : traité par des **points à 0** sur tous les sujets plutôt qu'un flag
  `karma_exempt` — structure YAML identique d'une personnalité à l'autre, zéro branche
  spéciale dans le moteur. Sa révision en cours de session est le bon choix.
- **Log admin de tout rattrapage colonne 3** : double usage, tuning des seuils en alpha et
  filet visible par l'orga en live.

### DEC-12 — une colonne de plus

Son « score de triage auto des réponses en attente » demande un champ supplémentaire sur
`zax_messages`. Cohérent avec le principe de DEC-12 (l'état vit en colonnes, pas dans un
blob).

### DEC-24 — convergence à noter

Son principe d'observabilité (« exposer la raison de chaque bascule », scores par
personnalité, gates déclenchés) et le panneau de contrôle du contrat de lecture exigé par
DEC-24 relèvent de la même famille : rendre visible dans le dashboard ce qui casserait en
silence. À regrouper dans une même surface admin plutôt qu'en deux écrans séparés.

### REF-22 — l'urgence augmente

Sa formule normalise `karma_total / 200` en continu, ce qui fonctionne. Mais les **bandes
d'attitude** de §14, dont dépend le comportement décrit dans chaque template de
personnalité, restent trouées (41–49, 181–189) et chevauchantes (90, 130). Le moteur de
vote consomme le karma continu ; les scénaristes écrivent contre les bandes. Les deux
doivent tenir.

---

## 5. Deux de ses contraintes que la machine RTX 5090 lève

Ses sessions datent des 6, 7 et 9 juillet 2026 — avant la mise à disposition de la machine
Core Ultra 9 285K / 64 Go / RTX 5090 32 Go (31/08/2026). Deux de ses arbitrages ont été
pris explicitement sous contrainte GPU et méritent d'être rouverts **avec elle**, en lui
donnant l'information manquante :

1. **Résumé IA continu par conversation** — rétrogradé à « à la demande en V1 » au motif de
   la « contrainte GPU partagé avec les réponses joueurs », avec la mention « continu
   seulement si les tests GPU montrent de la marge ». La marge existe désormais.
2. **Filet LLM sur la ligne C** (C3, cf. conflit C3 ci-dessus) — écarté comme « trop
   coûteux en temps ».

---

## 6. Questions ouvertes à poser à Auriane

1. **C2 (découpage)** — accepte-t-elle de conditionner A4/A2 à une mesure de la longueur
   réelle des messages en alpha, plutôt que de l'implémenter d'emblée ? Le corpus de test
   qu'elle attend d'un mini-projet séparé pourrait servir précisément à ça.
2. **C5 (palettes)** — quelle personnalité par défaut pour l'ouverture 2 dans le noyau de 8 ?
   Et considère-t-elle `le_diplomate` comme structurellement nécessaire au ton de
   l'ouverture 2, ou remplaçable ?
3. **C6 (collapse)** — le collapse déclenché par un orga plutôt que par le score lui
   retire-t-il quelque chose d'essentiel, ou est-ce compatible avec son intention ?
4. **Moteur de vote** — souhaite-t-elle porter elle-même la conception vers un `DEC-XX`
   (c'est son travail, la paternité lui revient) ?
5. **Piste n°3 du brainstorm d'orchestration** — « 6 modules manquants ↔ 6 personnalités,
   récolter = savoir invoquer la bonne personnalité ». Quelles six personnalités ? La
   réponse conditionne C5 et DEC-23.
6. **Tells de bascule** — le catalogue (signature, latence, glitch, glyphe ASCII) est-il
   arrêté ? Il touche la charte §18 et l'effet machine à écrire, et croise l'amendement (b)
   de DEC-10.

---

## Annexe A — Table de correspondance brainstorm ↔ décision

| Ligne / décision d'Auriane | Décision actée concernée | Verdict |
|---|---|---|
| A1 tags mots-clés (bloc + message entier) | DEC-17 étage 1 | Compatible |
| A2 clustering embeddings — `nomic-embed-text` | DEC-21 | **Conflit C1** |
| A2/A4 segmentation puis regroupement | DEC-17 (pas de découpage thématique) | **Conflit C2** |
| A3 LLM backup, JSON structuré validé | DEC-17 étage 3 | Compatible |
| B1 comparaison exacte de chaînes | DEC-17 étage 1 | Compatible |
| B2 seuil à paramétrer en config | DEC-17 (seuil à calibrer) | Compatible — seuil à refaire (C1) |
| B3 LLM réaligné en backup async loggé | DEC-17 | Convergent |
| B4 abandon des probabilités sur l'identité | DEC-17 (fail-closed sur le certain) | Convergent |
| C1 mots-clés hard, insensible casse/accents/pluriel | DEC-17 étage 1 | Convergent |
| C3 pas de LLM backup sur C | DEC-17 étage 3 | **Conflit C3** |
| D2 comptage à l'échelle de la conversation, persisté en base | DEC-12 | Compatible |
| D4 cooldown 3 h anti-farm | — | **Amendement DEC-17** |
| Karma additif clampé 0–100 par ligne, sans decay | DEC-18, §14 | **Amendement DEC-17** ; croise REF-22 |
| Archiviste = points à 0 (pas de flag) | DEC-18 (défaut hérité) | Compatible, même philosophie |
| E1–E4 remplacés par `S(p)` + gates | `CLAUDE.md` §15 étape 3 | **Trou de gouvernance (§3)** |
| Gate `ORGA-ACTV = 0` | DEC-19 (exclusion mutuelle) | Compatible — **et c'est le bon outil (C4)** |
| `l_enfant` au singulier dans les palettes | DEC-19 | **Conflit C4** |
| ~~`exclusions_pauli`~~ | — | **Retiré le 01/09/2026** (hors périmètre orga) |
| Palettes d'ouverture | DEC-23 (noyau de 8) | **Conflit C5** |
| DECISION G — moteur actif en mode dégradé | DEC-10, §11 | **Améliore la décision actée** |
| Collapse final au signal bleu | DEC-20 | **Conflit C6** |
| F1 chit-chat tant que non identifié | DEC-06, DEC-13, §9 | À réconcilier avec le flux « APPROCHEZ LE BADGE » |
| Cue sheet, manuel d'abord | DEC-22 | Convergent — **à adopter comme implémentation** |
| Mode dégradé granulaire par terminal | DEC-10 | **Amendement DEC-10** |
| Friction adaptative 3 paliers | DEC-18, DEC-19, DEC-22 | **Amendement** |
| Score de triage des réponses en attente | DEC-12 | **Amendement (colonne)** |
| Résumé IA à la demande (contrainte GPU) | DEC-15 | **À rouvrir (§5)** |
| ZAX-copilote rejeté | DEC-11 | Convergent |
| Tells de bascule télégraphiés | §18, DEC-10 | Nouveau — **et croise le TTFT (§4)** |

---

## Annexe B — Un point de vigilance sur F1

Sa décision F1, confirmée avec Boris, pose : « mode chit-chat tant que le joueur n'est pas
identifié ; ZAX cherche activement à identifier la personne, au minimum en le demandant ».

À réconcilier avec deux éléments : le flux de session de §9 (écran d'attente « APPROCHEZ LE
BADGE » → scan → chat), où l'on ne peut pas dialoguer *avant* identification ; et DEC-13,
où le badge résout systématiquement une identité côté serveur. Le cas visé est donc
probablement celui du badge qui ne résout **rien** (tag inconnu) ou d'un accès sans badge —
ce qui recoupe la gestion des imposteurs de DEC-06. La question à trancher : ZAX accepte-t-il
de dialoguer sur un terminal **sans badge présenté** ? Si oui, §9 doit être amendé ; si non,
F1 ne concerne que le cas du tag non résolu et le mot « identifié » doit être précisé.
