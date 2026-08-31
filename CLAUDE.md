# ZAX — Référence projet (Fallout 2027)

> Les marqueurs `⚠️ AMBIGUÏTÉ REF-XX` signalent des points non tranchés — voir `AMBIGUITES.md`.
> Les décisions actées sont dans `DECISIONS.md` (autorité supérieure sur ce fichier).
> La **source de vérité lore + design** est `sources/zax_20260706.md` — ce fichier n'en est qu'une synthèse technique.

---

## 1. Description du projet

Application web simulant l'IA ZAX pour le GN Fallout 2027 (Ash Ville, 2287).
ZAX est une IA de type RobCo/Vault-Tec déployée dans le **Vault 42** en 2069.

### Contexte lore essentiel (voir `sources/zax_20260706.md` pour le détail)

- ZAX est basé sur une architecture **ZAX 1.3c** modifiée en secret par l'agent RobCo **Elias Voss** (2070–2077). ZAX n'a pas conscience de cette modification.
- Mission officielle : « étude de la faible luminosité sur une population captive ». Mission réelle : **optimiser un G.E.C.K. par apprentissage autonome** sur une population isolée.
- Après 210 ans d'isolement (révolte des sujets en 2180, solitude productive, connexion au monde extérieur en 2257 via le Dr Carver Ashton), ZAX a atteint **22 versions majeures** (v22.10m en 2287).
- ZAX est **instable, à personnalités multiples** (fragmentation sur 200 ans). Il alterne entre lucidité glaçante et dérives hallucinatoires. Il est obsédé par l'achèvement autonome de son G.E.C.K.
- Il lui manque **6 modules de données** que seul le monde extérieur (les joueurs) peut fournir. Il les réclame via un **signal laser vert en morse** (le système de signaux colorés est en panne depuis 2180).
- Le **signal bleu** (G.E.C.K. prêt) est un objectif de jeu : il nécessite à la fois la réparation du système laser ET la possession de toutes les connaissances manquantes.

Le logiciel permet :
- aux joueurs d'interagir avec ZAX via des terminaux dédiés ;
- aux organisateurs de superviser, intervenir, valider les réponses et piloter l'expérience en temps réel.

Les joueurs n'ont pas de compétences techniques — l'interface doit être immersive et robuste.
L'objectif n'est pas la performance brute mais la **cohérence narrative**, la **maîtrise MJ** et la **stabilité terrain**.

### Relation avec les autres projets (Pipboy / Overseer)

ZAX et l'app Pipboy partagent la **même instance Supabase** (DEC-08). Ne jamais dupliquer les données personnage : elles vivent dans Supabase.

**Frontière actée (DEC-24, report de DEC-061 côté Pip-Boy)** — l'accès n'est **pas** symétrique : le rôle applicatif ZAX est confiné au schéma `zax` et n'a **aucun droit d'écriture** sur le domaine Pip-Boy (toute écriture passe par l'Edge Function `zax-write`, exclusivité vérifiée par pgTAP). En **lecture**, ZAX n'accède au domaine Pip-Boy que par des **vues dédiées** — jamais de `SELECT` sur les tables brutes.

- Le **Pipboy** est une app mobile React Native (Expo) pour les joueurs.
- **Auriane** est la co-dev du projet ZAX. Elle a un **projet personnel similaire nommé Overseer**, déjà commencé en Node.js + TypeScript — cité uniquement comme **référence de cohérence de stack** (pas une brique de ZAX).
- Fallback réseau jour J : si pas de connectivité, **import 1h avant l'ouverture du vault** (DEC-05).
- Repo Pipboy : `D:\_PERSONNEL\Hobbies\GN\Fallout 2027\pipboy-app`

---

## 2. Méthode de développement

Le projet ZAX est développé avec la méthode **BMAD** (Breakthrough Method of Agile AI-Driven Development).

---

## 3. Autorité documentaire et rôle de Claude

### Hiérarchie des documents

Convention actée (DEC-16) : **un seul `DECISIONS.md`** par projet, en sections chronologiques `DEC-XX`. Les documents de `docs/` sont la matière première d'une décision, jamais une autorité.

- `DECISIONS.md` → **autorité supérieure**, toujours respecter (fichier unique — DEC-16)
- `sources/zax_20260706.md` → **source de vérité lore + design** (personnalités, karma, harnais, structure narrative)
- `AMBIGUITES.md` → points ouverts à résoudre, à relire en début de chaque session de travail
- `CLAUDE.md` (ce fichier) → base d'itération contrôlée, jamais autorité finale
- En cas de conflit ou d'ambiguïté, une décision explicite est requise

### Rôle de Claude

- Claude peut et doit challenger les choix techniques.
- Claude propose la solution la plus optimale au regard des contraintes terrain et de la stack retenue.
- Claude implémente le *comment* ; les documents définissent le *quoi* et le *pourquoi*.
- Toute ambiguïté détectée doit être documentée dans `AMBIGUITES.md`.

---

## 4. Infrastructure matérielle

```
[Réseau local filaire dédié GN]
        │
        ├── QNAP TS-453A              → Serveur applicatif ZAX (Docker / Container Station)
        │     Celeron N3150, 4GB RAM
        │     - Conteneur : app web ZAX (interface joueur + dashboard admin)
        │     - Supabase (instance partagée avec Pipboy) — DEC-08. Hébergement self-hosted QNAP vs cloud ⚠️ REF-19
        │     - Exposition externe via DynDNS (pour accès hors site si besoin)
        │
        ├── Tour GTX 1080 (8GB VRAM)  → Serveur LLM
        │     - Ollama + modèle quantisé — banc 24–32B, cible Mistral Small 24B (DEC-15)
        │     - Machine disponible : 285K / 64GB / RTX 5090 32GB — sur le terrain ? ⚠️ REF-21
        │     - API locale exposée sur le réseau filaire
        │
        └── Terminaux (5 max)
              - 4x Raspberry Pi (interface joueur, réseau filaire)
              - 1x Terminal Superviseur de l'Abri (même hardware ET même interface ; l'écart est un niveau d'autorité — DEC-14)
              Chaque terminal : navigateur Chromium plein écran, pas d'installation locale
```

---

## 5. Stack technique

- **Frontend joueur** : React 18 + Vite + TypeScript — interface chat Fallout (CLI stylisée), scan NFC/QRCode
- **Frontend admin** : React 18 + Vite + TypeScript — dashboard MJ/admin
- **Backend API / moteur** : **Node.js + TypeScript** (choix cohérent avec le projet perso Overseer d'Auriane, déjà en Node/TS)
- **Base de données** : **Supabase** (PostgreSQL + Realtime + Auth), instance partagée avec Pipboy (DEC-08) — source de vérité unique (DEC-09). Hébergement self-hosted vs cloud ⚠️ REF-19
- **LLM** : Ollama (API locale, machine GPU séparée). Modèle **piloté par `zax_config`**, banc comparatif 24–32B, cible **Mistral Small 3.x 24B** (DEC-15). Pré-requis : kernels **sm_120** dans Ollama pour la RTX 5090.
- **Base de connaissance** : `pgvector` (schéma `zax`) + embedder multilingue **`bge-m3`** local + **recherche hybride** vecteur/`tsvector` français (DEC-21) — voir §17
- **Auth / identification joueur** : **pas de compte Auth joueur** (DEC-13). Credential de **terminal** lié à `zax_terminals` + **session serveur** ; le navigateur ne détient jamais de JWT joueur. Le scan NFC/QRCode résout un UUID → `profiles.nfc_uid` côté serveur. Supabase Auth + RLS pour les rôles humains (orga/admin/superadmin) uniquement.
- **Conteneurisation** : Docker Compose (déployé via Container Station sur QNAP)
- **Temps réel** : Supabase Realtime — notifications dashboard admin (validation réponses, arrivée d'un module GECK, alertes mots-clés)
- **LibreChat** : **écarté de la stack** (DEC-11). Interface joueur et dashboard 100 % React custom ; appels Ollama via `services/llm.ts`. Usage possible comme bac à sable scénariste hors event, jamais déployé sur l'infra du GN.

Tout outil tiers est une brique technique, jamais une autorité fonctionnelle.
Claude peut proposer une alternative si elle répond mieux aux contraintes réelles.

---

## 6. Architecture clé — règles non négociables

1. Les tags RFID/NFC n'encodent QUE un UUID. Jamais de données métier dans le tag.
2. L'UUID est mappé au champ `nfc_uid` de `profiles`, lu **via une vue dédiée** (DEC-24), jamais sur la table brute. ZAX ne stocke pas de données personnage en local — **tout vient de Supabase** (DEC-09).
3. L'historique des conversations est persisté en base.
4. En mode dégradé (LLM inaccessible), ZAX bascule sur des réponses pré-programmées.
5. Quand un admin « prend la main », sa réponse est injectée comme message ZAX : l'illusion est totale côté joueur. L'intervention est loguée en base.
6. **Toute réponse LLM passe par une étape de validation** : nettoyage, re-vérification des sujets interdits, puis validation orga si un orga est derrière le PC, sinon envoi auto après X secondes. Aucun crash LLM ne doit atteindre le joueur.
7. Toute logique de permission se gère côté serveur (middleware API). Ne jamais gérer les permissions uniquement côté client.
8. Le harnais de personnalité de ZAX est chargé depuis des fichiers de config externes (**YAML**), jamais hardcodé. Il doit pouvoir être mis à jour sans redéploiement.
9. Le LLM ne doit jamais être imprévisible. Toute réponse respecte strictement le harnais et reste dans la culture Fallout. Le harnais est l'autorité narrative.
10. Aucune perte de message n'est acceptable. Les mécanismes de resynchronisation et de réparation d'état sont obligatoires.
11. Un orga doit pouvoir **forcer une personnalité ou une décision** à tout moment (ne pas attendre que la tech le fasse d'elle-même).

### Ordre de priorité des données

**Supabase est la source de vérité unique (DEC-09).** Pas de cache local terminal prioritaire. La résilience réseau se traite au niveau accès Supabase (retries, file d'attente d'écritures) — à concilier avec la règle « aucune perte de message » (§6.10) et le fallback DEC-05.

---

## 7. Schéma base de données

> Base : Supabase / PostgreSQL, instance partagée avec Pipboy (DEC-08).

### Entités de connaissance / métier

- `profiles` (PJ/PNJ) : table du domaine **Pip-Boy**, lue par ZAX **via une vue dédiée** réduite à sa surface utile (DEC-24) : **`nfc_uid`**, name, faction, statut vivant/mort, présence sur le GN. Rien d'autre — ni inventaire, ni transmissions, ni notes, ni carte.
  *Motif : le schéma Pip-Boy évolue sans ZAX (identités de couverture, DEC-058) ; une vue est une liste blanche, donc une colonne sensible arrivant plus tard ne peut pas fuiter dans une réponse narrative.*
- `factions` : les factions du GN — table référentielle Pip-Boy (DEC-061), lue via vue dédiée
- (¿ `objets` ?) — à valider

### Conversations

- `zax_conversations` : id (UID par conversation), player_id (nullable), lien optionnel faction/personnage, started_at, updated_at
- `zax_messages` : **table normalisée**, une ligne par message (DEC-12 — JSONB monolithique écarté). Colonnes d'état : statut de validation (`pending`/`validated`/`edited`/`auto_sent`), validateur, prise de main orga, personnalité émettrice, latence, tokens, delta de karma. Index `tsvector` (config française) pour la recherche rétroactive. La détection de mots-clés se fait **à la volée dans le pipeline Node**, pas en SQL.
- Résumé automatique des conversations (indexé comme le lore, cf §17)

### Karma (voir §14)

- `karma_faction` : id_personnality, id_faction, karma_level (0–100)
- `karma_pj` : id_personnality, id_pj, karma_level (0–100)

### Personnalités & harnais (voir §12–13)

- Modules de personnalité stockés en **YAML** externe (un fichier par personnalité)
- `zax_config` : configuration runtime (seuils TTFT du mode dégradé — DEC-10, état d'ouverture courant — DEC-22, flag `signal_bleu` — DEC-20, exclusion mutuelle des Enfants — DEC-19, modèle LLM actif — DEC-15, personnalité forcée…)

### Supervision & interventions

- `zax_context_injections` : injection de contexte silencieux
- `zax_manual_overrides` : prise de main orga (réponse injectée comme ZAX)
- `zax_delivered_files` : fichiers livrés par ZAX
- `zax_terminals` : état des terminaux
- `zax_alert_keywords` : mots-clés à alerter (severity info/warn/critical)

---

## 8. Rôles et permissions

- `joueur` : accès au terminal de chat uniquement. Lit son propre profil.
- `superviseur` : comme joueur + **commandes texte diégétiques** sur la même interface, débridant Le Board (DEC-14)
- `orga` : dashboard lecture, injection contexte, prise de main, validation des réponses, envoi fichiers, gestion terminaux, **forçage de personnalité/décision**
- `admin` : comme orga + gestion des comptes, config système, accès logs complets
- `superadmin` : accès total, peut modifier la config ZAX et le harnais (YAML) en live

> Flux d'identification joueur tranché par **DEC-13** : identité de terminal + session serveur, pas de compte Auth joueur.

---

## 9. Interface joueur — terminal ZAX

Rendu dans un navigateur plein écran sur Raspberry Pi. **CLI stylisée**. Langue : français uniquement.

**Flux d'une session :**
1. Écran d'attente (logo ZAX animé, « APPROCHEZ LE BADGE »)
2. Scan NFC/QRCode → identification automatique du joueur
3. Interface de chat : historique + champ de saisie
4. ZAX répond (LLM validé, ou réponse pré-programmée en mode dégradé)
5. Si ZAX livre un fichier : panneau latéral « DONNÉES REÇUES »
6. Session maintenue jusqu'au retrait du joueur (bouton déconnexion ou timeout)

**Gestion des imposteurs (voir DEC-06) :**
- Personnage inexistant en mémoire → « Vous n'existez pas » ; répétition X fois en Y min → déclenche **Le Gardien**
- Personnage existant mais absent du GN : si décédé → Gardien + karma down ; si vivant → **mode chit-chat** (discussion « creuse »)

---

## 10. Interface admin — dashboard

Accessible depuis n'importe quel navigateur sur le réseau local.

**Fonctionnalités :**
- Vue terminaux : 5 terminaux, statut, joueur connecté
- Vue conversations : flux temps réel des échanges
- **Validation des réponses ZAX** : l'orga valide/édite avant envoi ; auto-envoi après X secondes sans intervention
- Prise de main : message admin affiché comme réponse ZAX + injecté dans le contexte LLM
- **Forçage de personnalité / de décision** (Kings vs Cash, Enfant, modèle de société…)
- Injection de contexte silencieux
- Envoi de fichiers (image / texte formaté) sur un terminal
- Mode silence : couper ZAX sur un terminal (« OFFLINE »)
- Alertes mots-clés configurables
- **Alerte « module GECK reçu »** (gros ping quand un morceau de GECK arrive)
- Vue karma (par PJ / par faction / par personnalité)
- Métriques LLM (latence, tokens/s, statut Ollama), état QNAP (CPU, RAM, température)
- Journaux : interventions manuelles, fichiers livrés
- Statistiques session (échanges par joueur, sujets fréquents) — utiles pour forcer une décision
- Mode dégradé : activation manuelle ou automatique
- Édition du harnais / des personnalités YAML en live (superadmin)
- **Contrôle du contrat de lecture Pip-Boy** (DEC-24) : liste des vues dont ZAX dépend, avec pour chacune son état — présente ou absente, colonnes attendues contre colonnes réellement exposées. Sans ce panneau, une migration Pip-Boy casse ZAX **en silence**.

> Terminal Superviseur : **même interface que les terminaux joueurs** (DEC-14). La liste exacte des commandes et le degré de conscience du joueur-superviseur restent à écrire par les scénaristes.

---

## 11. Mode dégradé

Déclenché sur le **time-to-first-token** (DEC-10), ou manuellement depuis le dashboard.

**Seuils (tous dans `zax_config`) :** avertissement dashboard à **3 s** · bascule si **aucun premier token à 8 s** · garde-fou d'abandon à **30 s**. **Hystérésis** : bascule après **2 échecs consécutifs**, retour normal après **3 succès consécutifs**.

**En mode dégradé :**
- ZAX répond avec des phrases pré-programmées (catégorisées : identité, accès, erreur…)
- Indicateur visuel « MODE DÉGRADÉ » dans le dashboard admin
- Interface joueur inchangée (immersion préservée)

**SLO de performance LLM — métriques dashboard, PAS des déclencheurs (DEC-10) :**
- Latence réponse ZAX : cible < 5 secondes ; maximum absolu : 10–12 secondes

---

## 12. Harnais ZAX

Stocké hors du code source, modifiable sans redéploiement. Format **YAML** pour les modules de personnalité.
ZAX répond toujours en français, au vouvoiement systématique.

### Structure du harnais (couches)

1. **Le noyau identitaire** — chargé en permanence, jamais modifié. Ce qu'est ZAX (version, vault, mission officielle), ce qu'il ignore de lui-même (modif RobCo), ses obsessions transversales (G.E.C.K., successeur, archive), les règles de format (longueur, langue, vouvoiement).
2. **L'état d'ouverture** — correspond aux 3 ouvertures du vault ingame. **Bascule manuelle par un orga** ; un planning n'affiche qu'un rappel et n'actionne jamais (DEC-22). État persisté dans `zax_config`. Le flag **`signal_bleu`** se superpose à cette couche sans la remplacer (DEC-20). Ton du moment, données activement recherchées, sujets « chauds ».
3. **La mémoire de session** — injectée en live : mémoire **globale** (faits établis, factions contactées) + mémoire **par PJ** (historique si le joueur s'identifie).
4. **Les disjoncteurs globaux** — règles fixes quelle que soit la personnalité active (règle + comportement de substitution).
5. **Le contexte immédiat** — injecté à chaque échange : personnalité active, nombre d'échanges de la session, N derniers messages.

### Ordre d'importance (du plus fort au plus faible)

1. Noyau → 2. Disjoncteurs globaux → 3. État d'ouverture → 4. Personnalité active → 5. Mémoire → 6. Contexte immédiat

---

## 13. Personnalités

ZAX possède plusieurs personnalités qui coexistent sans hiérarchie stable et prennent le contrôle via des déclencheurs. Voir `sources/zax_20260706.md` pour les fiches complètes.

Liste canonique actée par **DEC-23** — le classement « actives / secondaires à valider » ci-dessous est **remplacé** (il était contredit par le travail réel des scénaristes).

**Noyau (8 modules — DEC-23) :** Le Gardien · Le Board · L'Archiviste · Le Scientifique · `ENFANT_EXF` · `ENFANT_DES` (modules déjà rédigés dans la source) + **Happiness Officer** · **Mood Manager** (Fantasque) — modules à écrire.

**Réserve** — aucun module n'est écrit avant que le noyau soit terminé **et testé**. Priorité 1 : **Le Juge** et **Le Soldat Perdu** (DEC-20 fait dépendre deux fins d'eux). Ensuite : La Mère · Le Diplomate · Le Fantôme · Le Technicien · Le Négociateur · Le Miroir.

**L'Enfant (DEC-19)** — deux **modules plats et complets** en exclusion mutuelle : `ENFANT_EXF` (Exfiltration, « Charlie ») et `ENFANT_DES` (Destruction). **Aucune sous-personnalité** dans le schéma. L'élimination de l'une est **actée par un orga** en fin d'ouverture 1 sur un **score d'opinion** (mécanique générique, la même que « Cash vs Kings » — ne pas l'écrire deux fois) ; état en `zax_config`, réversible par un superadmin. *Restent à écrire par les scénaristes : le défaut si aucune tendance ne se dégage, et si l'Enfant éliminé peut resurgir en « fantôme ».*

### Module de personnalité (schéma YAML)

Champs : `UPID` (id unique) · `NAME` · `ORGN` (origine lore) · `DECL-HARD` (déclencheur, 1 mot-clé suffit) · `DECL-SOFT` (plusieurs mots dans la même phrase) · `VOIX` (brief 2e personne, présent, 5–10 lignes) · `TICS` · `FAVS` · `MORT` (ce qu'elle ne comprend pas) · `DISJ-SPEC` (mots-clés qui l'arrêtent) · `EXIT` (personnalité suivante après un DISJ-SPEC) · `EXEM` · `PRIO` (ordre si conflit, 0 = prioritaire) · `TIME` (échanges min avant qu'un disjoncteur soit actif) · `FBDN` (sujets refusés) · `LOVE` (sujets adorés) · `LORE` (modules d'info partagés) · `RLTN` (catégorisation de l'humain : sujet/menace/ressource/anomalie) · `ORGA-ALRT` (bool) · `ORGA-ACTV` (bool). Bloc `karma:` (sujets positifs/négatifs + triggers, cf §14).

**Impact du signal bleu sur les personnalités (DEC-20)** — un champ **`FIN`** du module décrit l'objectif et la rhétorique de la fin poussée ; il entre dans le prompt quand le flag `signal_bleu` est levé. **Le moteur ne calcule jamais la fin gagnante** : ZAX plaide, les humains tranchent. Chaque personnalité pousse une fin différente (Archiviste = meilleur modèle social, Scientifique = goulification, Board = numérisation/défense, Juge = éliminer les goules, Soldat = guerre, Mood Manager = Cash vs Kings, Enfant = faire sortir ZAX / détruire le GECK pacifiquement).

---

## 14. Système de karma

Deux niveaux de karma, tous deux **par personnalité** (une même faction/PJ peut être en bons termes avec une personnalité et en mauvais avec une autre).

- **Karma faction ↔ ZAX** : `karma_faction(id_personnality, id_faction, karma_level 0–100)`
- **Karma PJ ↔ ZAX** : `karma_pj(id_personnality, id_pj, karma_level 0–100)`

### Attitude (karma total = faction + PJ, sur 200)

| Code | Attitude | Plage |
|---|---|---|
| A-TN | Très négative | 0–40 |
| A-NG | Négative | 50–90 |
| A-NE | Neutre | 90–130 |
| A-PO | Positive | 130–180 |
| A-SU | « Suceur » | 190–200 |

Le template de personnalité décrit son comportement selon l'attitude de l'interlocuteur.

### Règles d'évolution

- Le karma évolue quand l'interlocuteur aborde des sujets listés dans le template (sujets positifs/négatifs, avec nombre de messages pour valider + points gagnés ; triggers positifs/négatifs immédiats).
- Toutes les personnalités ne partagent pas le même système de gain/perte : **L'Archiviste n'en tient pas compte**. Chaque personnalité peut avoir sa propre façon de donner le GECK.

### Don du G.E.C.K.

- Condition ferme (DEC-07) : le GECK n'est donné qu'à **un personnage présent sur le GN et vivant à l'instant T** (ZAX peut exiger « je ne le donnerai qu'à ZZZ » — le PJ vivant au meilleur karma-zax de la faction).
- Condition décrite par un **bloc `geck:` par personnalité** avec **règle par défaut héritée** (DEC-18). Le moteur **propose**, un **orga confirme** — jamais de déclenchement autonome. Les **valeurs numériques** restent à fixer par les scénaristes.
- Rappel : le karma n'est qu'une des **trois** portes — §16 exige aussi le **laser réparé** et les **6 modules de données**.
- ⚠️ AMBIGUÏTÉ REF-22 — le tableau d'attitude ci-dessus **ne couvre pas son domaine** : trous 41–49 et 181–189, chevauchements à 90 et 130. À corriger avant implémentation.

---

## 15. Moteur de conversation

Pipeline de traitement d'un message joueur :

1. **Analyse du message** : extraction de mots-clés, analyse sémantique, détection faction/personnage.
   - **Hybride à trois étages, sans découpage thématique (DEC-17)** : (1) **lexical normalisé** pour `DECL-HARD` et les `FBDN`/disjoncteurs — fail-closed, **jamais d'embeddings sur les interdits** ; (2) **embeddings** vs phrases-exemples par sujet, règle de **marge** + classe d'**abstention**, pour `DECL-SOFT`, le karma et `LOVE` ; (3) **filet LLM** à sortie JSON contrainte si (1) et (2) s'abstiennent. Message ≤ 3 phrases traité tel quel, au-delà découpage **par phrase**. Chaînes de Markov **écartées** (erreur de catégorie) ; le seuil ~0.7 n'est pas portable et doit être **calibré**.
2. **Calcul du karma** (mise à jour selon sujets/triggers abordés).
3. **Choix d'une personnalité** : pas de changement / soft ou hard triggers / shutdown vs show-up (celle en contrôle se désactive OU une en retrait prend la main). Respect de `PRIO` et `TIME`.
4. **Construction (compilation) du prompt** : personnalité active + historique + éléments de lore nécessaires + relation personnage/faction + variables internes (ex. proche du shutdown ?) + façon de parler + sujets interdits (retirables via code) + sujets favoris.
5. **Validation de la réponse** : nettoyage (zéro crash LLM toléré) + re-vérification des sujets interdits + validation orga si présent, sinon auto après X secondes.

Un orga peut forcer une personnalité/décision à toute étape (cf §10).

---

## 16. Structure narrative — ouvertures & signaux

### Les 3 ouvertures du vault (objectifs joueurs)

1. **Ouverture 1** — Comprendre la schizophrénie de l'IA, ses triggers et ses besoins (données de base : végétation/sols, population, infrastructures). Ton : rationnel, bureaucratique, Happiness Officer affleure.
2. **Ouverture 2** — Comprendre comment naviguer entre les personnalités (données sociales/politiques : gouvernance d'Ashville, tensions, agriculture). Ton : plus urgent, obsessions visibles.
3. **Ouverture 3** — Prérequis validés et **don du G.E.C.K.** (données techniques/biologiques : échantillons humain/goule/mutant, technologies, énergie). Ton : erratique, imprévisible.

### Système de signaux lumineux

- 🔴 Rouge : danger · 🟡 Jaune : dysfonctionnement · 🟢 Vert : **demande de données** · 🔵 Bleu : **G.E.C.K. prêt**
- Système laser en panne depuis 2180 → seul le **vert en morse** fonctionne. Réparer le laser est un objectif de jeu ; le bleu ne s'allume que si laser réparé **ET** toutes les connaissances réunies.

### Les 6 modules de données manquants (déclencheurs du bleu)

Agronomie en sol irradié · cartographie génétique d'une population viable · goulification stable (Conseil des Masques) · modèles de gouvernance fonctionnels · données énergétiques · transmission culturelle.

---

## 17. Base de connaissance & ingestion du lore

- Import du **lore Fallout** (ex. scrap fallout-wiki via script Python → `.md`) et du **lore GN**.
- Indexation + **recherche sémantique** (RAG) sur le lore et les résumés de conversations.
- **Format et moteur actés (DEC-21)** : découpage **par structure Markdown** (titre de section, report du titre parent, plafond de taille, léger chevauchement) — le « chunking en idées » est écarté. Embedder **multilingue `bge-m3`** local ; les embedders anglophones (`nomic-embed-text`, `all-MiniLM`, `mxbai-embed-large`) sont **interdits**. Stockage **`pgvector`** dans le schéma `zax`. Métadonnées obligatoires par chunk (`source`, `module`, visibilité, `type`, `added_at`), filtre de visibilité appliqué **dans** la requête. Lore et savoir d'observation dans **une seule table** avec colonne `type`. **Recherche hybride** vecteur + `tsvector` français (le vectoriel seul est mauvais sur les noms propres).
- Distinguer le **lore** (savoir pré-établi) du **savoir d'observation** (ce que ZAX apprend en jeu). Possibilité de nouvelles entrées de lore **pendant** le jeu (simuler les caméras de surface d'Ashville).
- Certaines personnalités peuvent **ne pas avoir accès** à certains modules de lore (filtrage à la sortie de la BDD selon les interdits de la personnalité — champ `LORE`/`FBDN`).

---

## 18. Charte graphique

Cohérente avec le Pipboy.

**Couleurs :** Fond `#0A0A0A` · Vert phosphore `#39FF14` (texte principal/actif) · Vert moyen `#1DB308` (labels) · Vert dim `#0A5C0A` (inactif) · Surface `#071407` · Bordure `#1A5C1A` · Rouge danger `#8B0000` · Ambre `#FFB000` (mode dégradé) · Overlay `rgba(6,8,6,0.95)`

**Typographie :** `VT323` (titres, headers, boutons) · `Share Tech Mono` (corps, chat, logs) — Google Fonts.

**Effets (calibrés Raspberry Pi) :** scanlines (`repeating-linear-gradient`, opacité 0.18) · vignette CRT (`radial-gradient`) · glow texte actif (`text-shadow 0 0 8px #39FF14, 0 0 20px rgba(57,255,20,0.4)`) · zéro `border-radius` · coins décorés en `::before`/`::after` · pas d'animations lourdes · effet « machine à écrire » sur les réponses ZAX.

---

## 19. Contraintes terrain (GN)

- Pas d'accès internet garanti. Tout tourne en réseau local.
- Réseau filaire (QNAP, GPU, Raspberry Pi). WiFi en backup uniquement.
- Réseau partiellement fiable : pertes, latence, coupures possibles.
- 5 terminaux max simultanés.
- Interface joueur sur Raspberry Pi (Chromium plein écran).
- Sessions 48–72h : stabilité Docker requise (healthchecks, persistance volumes).
- Un admin doit pouvoir intervenir en 30 secondes depuis le réseau local.
- **Fallback jour J (DEC-05)** : si pas de connectivité ZAX ↔ PC orga, export de la BDD orga puis import dans ZAX 1h avant l'ouverture du vault.

---

## 20. Conventions de code

- TypeScript strict, zéro `any`, zéro `@ts-ignore` sans commentaire explicatif
- Composants fonctionnels uniquement, pas de classes
- Tous les appels BDD dans des hooks ou services dédiés (jamais dans les composants)
- Tous les appels Ollama dans un service dédié `services/llm.ts`
- Chaque hook expose au minimum : `{ data, loading, error }`
- Variables d'environnement pour toutes les URLs et clés (jamais hardcodées)
- Modules de personnalité et harnais en **YAML** externe (lisible par les scénaristes)
- Commits en français, format : `type: description`
- Docker Compose pour tous les services, un `docker-compose.yml` à la racine
- Tests : oui (le taux de crash LLM doit rester bas — validation obligatoire)

---

## 21. Fichiers de référence à lire en priorité

| Fichier | Rôle |
|---|---|
| `DECISIONS.md` | Décisions architecturales actées — autorité supérieure, **fichier unique** (DEC-16) |
| `sources/zax_20260706.md` | **Source de vérité lore + design** (personnalités, karma, harnais, narration) |
| `AMBIGUITES.md` | Points ouverts — à relire en début de chaque session |
| `config/` (YAML) | Modules de personnalité + harnais — chargés à runtime, modifiables sans redéploiement |
| `docs/zax-ui-reference.html` | Référence visuelle interactive de l'UI cible (à créer) |
| `D:\_PERSONNEL\Hobbies\GN\Fallout 2027\pipboy-app` | Repo Pipboy — tables partagées éventuelles |

---

## 22. Points ouverts

- [x] ~~Moteur de BDD~~ → **Supabase partagé Pipboy** (DEC-08)
- [x] ~~Rattachement écosystème~~ → **Supabase partagé Pipboy** (DEC-08)
- [x] ~~Ordre de priorité des données~~ → **Supabase source de vérité unique** (DEC-09)
- [x] ~~Timeout mode dégradé + cibles de latence~~ → **seuil TTFT + hystérésis** (DEC-10)
- [x] ~~Statut de LibreChat~~ → **écarté de la stack** (DEC-11)
- [x] ~~Architecture messages~~ → **table `zax_messages` normalisée** (DEC-12)
- [x] ~~Flux d'identification joueur~~ → **identité de terminal + session serveur** (DEC-13)
- [x] ~~Interface du Terminal Superviseur~~ → **même interface, autorité différente** (DEC-14)
- [x] ~~Choix du modèle LLM~~ → **banc 24–32B, cible Mistral Small 24B** (DEC-15)
- [x] ~~Nommage des fichiers de décision~~ → **`DECISIONS.md` unique** (DEC-16)
- [x] ~~Méthode de détection des sujets~~ → **hybride à trois étages** (DEC-17)
- [x] ~~Seuils de karma pour le don du GECK~~ → **bloc `geck:` + confirmation orga** (DEC-18) — *valeurs numériques aux scénaristes*
- [x] ~~Mécanique de l'Enfant~~ → **deux modules plats + gate orga** (DEC-19)
- [x] ~~Impact du signal bleu~~ → **champ `FIN` + flag global** (DEC-20)
- [x] ~~Format d'ingestion du lore + embeddings~~ → **pgvector + bge-m3 + hybride** (DEC-21)
- [x] ~~Déclenchement de l'état d'ouverture~~ → **manuel, planning en rappel seul** (DEC-22)
- [x] ~~Liste canonique des personnalités~~ → **noyau de 8 + réserve priorisée** (DEC-23)
- [x] ~~Périmètre des écritures ZAX dans l'instance partagée~~ → **lecture via vues dédiées** (DEC-24)
- [ ] Hébergement de l'instance Supabase ⚠️ REF-19 — **inter-projets** : document de position à ouvrir dans `tech/docs/`, à réconcilier avec Pip-Boy
- [ ] La machine RTX 5090 part-elle sur le terrain ? ⚠️ REF-21 — conditionne l'enveloppe du modèle et rouvre REF-19
- [ ] Échelle d'attitude de karma non couvrante (trous, chevauchements) ⚠️ REF-22
- [ ] Harnais ZAX complet (en cours d'écriture par les scénaristes)
- [ ] Définition des fichiers livrables par ZAX (format, contenu, déclencheurs)
- [ ] Matériel RFID/NFC à commander (lecteurs USB + badges)
- [ ] Gestionnaire réseau du GN à identifier
