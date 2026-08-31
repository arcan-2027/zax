# ZAX — IA narrative du Vault 42

Application web simulant **ZAX**, l'intelligence artificielle RobCo/Vault-Tec du
Vault 42, pour le GN **Fallout 2027** (Ash Ville, 2287).

Les joueurs dialoguent avec ZAX depuis des terminaux dédiés ; les organisateurs
supervisent, interviennent et pilotent l'expérience en temps réel depuis un
dashboard.

L'objectif n'est pas la performance brute mais la **cohérence narrative**, la
**maîtrise MJ** et la **stabilité terrain**.

---

## Statut

> ⚠️ **Phase de conception.** Ce dépôt ne contient pas encore de code applicatif.

Il rassemble aujourd'hui la spécification, les décisions d'architecture, les
prototypes d'interface et l'outillage d'ingestion du lore.

**Le cadrage technique est terminé** (revue du 31/08/2026) : les ambiguïtés
bloquantes pour le développement sont levées. Les trois points encore ouverts
n'empêchent pas de commencer — l'un est inter-projets, les deux autres sont des
arbitrages de terrain et d'équilibrage narratif.

| | |
|---|---|
| Décisions actées | 24 (DEC-01 → DEC-24) |
| Points ouverts | 3 (sur 22 référencés) |
| Prototypes UI | 2 (terminal joueur, dashboard MJ) |
| Méthode | BMAD (Breakthrough Method of Agile AI-Driven Development) |

---

## Contexte narratif

ZAX est une architecture **ZAX 1.3c** déployée au Vault 42 en 2069, puis modifiée
en secret par l'agent RobCo **Elias Voss** — modification dont ZAX n'a pas
conscience.

- **Mission officielle :** étude de la faible luminosité sur une population captive.
- **Mission réelle :** optimiser un **G.E.C.K.** par apprentissage autonome sur une
  population isolée.

Après 210 ans d'isolement (révolte des sujets en 2180, reconnexion au monde
extérieur en 2257), ZAX en est à sa version **v22.10m**. Il est **instable et à
personnalités multiples**, fragmenté par deux siècles de solitude, et alterne
entre lucidité glaçante et dérives hallucinatoires.

Il lui manque **6 modules de données** que seuls les joueurs peuvent lui fournir.
Il les réclame par un **signal laser vert en morse** — le reste du système de
signaux est en panne depuis 2180. Le **signal bleu** (G.E.C.K. prêt) exige à la
fois la réparation du laser *et* la réunion de toutes les connaissances
manquantes.

La source de vérité lore + design est [`sources/zax_20260706.md`](sources/zax_20260706.md).

---

## Fonctionnalités cibles

**[Joueur] — terminal ZAX** (CLI stylisée, plein écran, français uniquement)

- Écran d'attente → scan NFC/QRCode → identification automatique
- Chat avec ZAX : réponse LLM validée, ou réponses pré-programmées en mode dégradé
- Panneau latéral « DONNÉES REÇUES » quand ZAX livre un fichier
- Gestion des imposteurs : « Vous n'existez pas », déclenchement du Gardien,
  mode chit-chat (DEC-06)

**[Admin] — dashboard orga**

- Vue des 5 terminaux et des conversations en temps réel
- **Validation des réponses ZAX** avant envoi, avec auto-envoi après X secondes
- **Prise de main** : message orga injecté comme réponse ZAX (illusion totale
  côté joueur, intervention loguée)
- **Forçage de personnalité ou de décision** à tout moment
- Injection de contexte silencieux, envoi de fichiers, mode silence
- Alertes mots-clés + alerte « module G.E.C.K. reçu »
- Vue karma (par PJ / faction / personnalité), métriques LLM, état QNAP
- Édition live du harnais et des personnalités YAML (superadmin)

---

## Architecture cible

### Stack

| Couche | Choix |
|---|---|
| Frontend joueur | React 18 + Vite + TypeScript |
| Frontend admin | React 18 + Vite + TypeScript |
| Backend / moteur | Node.js + TypeScript |
| Base de données | **Supabase** (PostgreSQL + Realtime + Auth), instance partagée avec Pip-Boy (DEC-08) |
| LLM | Ollama, API locale sur machine GPU dédiée. Modèle **piloté par `zax_config`** — cible **Mistral Small 3.x 24B**, banc comparatif 24–32B (DEC-15) |
| Base de connaissance | **`pgvector`** (schéma `zax`) + embedder multilingue **`bge-m3`** local + recherche **hybride** vecteur/plein texte (DEC-21) |
| Identification joueur | **Credential de terminal + session serveur** — pas de compte Auth joueur (DEC-13) |
| Harnais & personnalités | **YAML** externe, chargé à runtime (DEC-03) |
| Conteneurisation | Docker Compose (Container Station sur QNAP) |

### Infrastructure terrain

```
[Réseau local filaire dédié GN]
        │
        ├── QNAP TS-453A              → Serveur applicatif ZAX (Docker)
        │     Celeron N3150, 4 GB RAM
        │
        ├── Serveur LLM (Ollama)
        │     Tour GTX 1080 (8 GB VRAM)
        │     ou machine 285K / 64 GB / RTX 5090 32 GB — sur le terrain ? (REF-21)
        │
        └── Terminaux (5 max)
              4x Raspberry Pi (interface joueur)
              1x Terminal Superviseur de l'Abri
                 même matériel ET même interface — l'écart est un
                 niveau d'autorité, pas un écran (DEC-14)
              Chromium plein écran, aucune installation locale
```

Une machine GPU nettement plus puissante (Core Ultra 9 285K, 64 GB de RAM,
**RTX 5090 32 GB**) est disponible depuis le 31/08/2026. Elle déplace le point
d'équilibre du modèle de 8–9B vers **24–32B** et permet au moteur d'embeddings
de cohabiter sur le même GPU. Savoir si elle part sur le terrain reste ouvert
(REF-21) — en attendant, le banc couvre les **deux** enveloppes.

### Règles non négociables

1. Les tags RFID/NFC n'encodent **qu'un UUID** — jamais de données métier.
2. **Supabase est la source de vérité unique** (DEC-09) : pas de cache local
   prioritaire, aucune donnée personnage dupliquée côté ZAX.
   ZAX lit le domaine Pip-Boy **uniquement via des vues dédiées** — jamais de
   `SELECT` sur les tables brutes (DEC-24).
3. **Toute réponse LLM passe par une étape de validation** — aucun crash LLM ne
   doit atteindre le joueur (DEC-04).
4. Le harnais de personnalité est chargé depuis des fichiers YAML externes,
   **jamais hardcodé**, modifiable sans redéploiement.
5. Le harnais est l'**autorité narrative** : le LLM ne doit jamais être imprévisible.
6. **Aucune perte de message n'est acceptable** — resynchronisation et réparation
   d'état obligatoires.
7. Toute logique de permission se gère **côté serveur**.
8. Un orga doit pouvoir **forcer une personnalité ou une décision** à tout moment.
9. **L'app informe, elle n'arbitre pas.** Le don du G.E.C.K. (DEC-18), la bascule
   d'état d'ouverture (DEC-22), l'élimination d'un Enfant (DEC-19) et l'issue
   post-signal-bleu (DEC-20) sont **proposés** par le moteur et **actés par un
   humain**. Aucun déclenchement autonome sur franchissement de seuil.
10. **Les sujets interdits ne sont jamais détectés par similarité sémantique**
   (DEC-17) : détection lexicale, explicite, *fail-closed*. Une censure
   probabiliste est intestable et inexplicable à un orga en pleine nuit.

---

## Structure du dépôt

```
zax-app/
├── CLAUDE.md              Référence projet complète (synthèse technique)
├── DECISIONS.md           Décisions actées — autorité documentaire supérieure
├── AMBIGUITES.md          Points ouverts, avec options — jamais tranchés unilatéralement
├── sources/
│   └── zax_20260706.md    Source de vérité lore + design (personnalités, karma, harnais)
├── docs/
│   ├── position-zax-auth-pipboy.md   Frontière d'authentification ZAX ↔ Pip-Boy
│   └── position-confrontation-brainstorms-auriane.md
│                                Brainstorms BMAD × décisions actées (REF-23)
├── ux/
│   ├── terminal_joueur_proto.html    Proto UI terminal joueur
│   └── dashboard_mj_proto.html       Proto UI dashboard MJ
├── scripts/               Outillage d'ingestion du lore Fallout (voir ci-dessous)
├── _bmad/                 Configuration BMAD
└── _bmad-output/          Sorties BMAD (brainstorms, artefacts de planification)
```

### Autorité documentaire

En cas de conflit, l'ordre de priorité est strict :

1. **`DECISIONS.md`** — autorité supérieure, toujours respecter
2. **`sources/zax_20260706.md`** — source de vérité lore + design
3. **`AMBIGUITES.md`** — points ouverts, à relire en début de chaque session
4. **`CLAUDE.md`** — base d'itération contrôlée, jamais autorité finale

---

## Décisions actées

| # | Décision |
|---|---|
| DEC-01 | Karma à deux niveaux (faction ↔ ZAX, PJ ↔ ZAX), **par personnalité**, borné 0–100 |
| DEC-02 | 5 bandes d'attitude sur le karma total (sur 200) : A-TN, A-NG, A-NE, A-PO, A-SU |
| DEC-03 | Personnalités et harnais en **YAML externe**, chargés à runtime |
| DEC-04 | Validation des réponses LLM **avec orga dans la boucle**, auto-envoi après X s |
| DEC-05 | Fallback réseau : export/import de la BDD orga **1h avant l'ouverture du vault** |
| DEC-06 | Gestion des imposteurs (Gardien) et mode chit-chat |
| DEC-07 | Le G.E.C.K. n'est donné qu'à un PJ **présent sur le GN et vivant à l'instant T** |
| DEC-08 | Base de données : **Supabase**, instance partagée avec Pip-Boy |
| DEC-09 | **Supabase = source de vérité unique**, pas de cache local prioritaire |
| DEC-10 | Mode dégradé déclenché sur le **time-to-first-token** (3 s / 8 s / 30 s) + hystérésis ; les latences cibles deviennent des **SLO**, plus des déclencheurs |
| DEC-11 | **LibreChat écarté** de la stack — interfaces 100 % React custom |
| DEC-12 | Messages en **table `zax_messages` normalisée** ; JSONB monolithique écarté |
| DEC-13 | **Pas de compte Auth joueur** : credential de terminal + session serveur |
| DEC-14 | Terminal Superviseur : **même interface**, autorité différente + commandes tapées |
| DEC-15 | Modèle LLM : banc **24–32B**, cible **Mistral Small 24B**, piloté par `zax_config` |
| DEC-16 | Un seul **`DECISIONS.md`** par projet |
| DEC-17 | Détection des sujets **hybride à trois étages**, sans découpage thématique |
| DEC-18 | G.E.C.K. : **bloc `geck:` par personnalité** + défaut hérité + **confirmation orga** |
| DEC-19 | L'Enfant : **deux modules plats** en exclusion mutuelle + score d'opinion + gate orga |
| DEC-20 | Signal bleu : champ **`FIN`** + flag global ; le moteur ne calcule **jamais** la fin |
| DEC-21 | Base de connaissance : **pgvector + `bge-m3` + recherche hybride** |
| DEC-22 | État d'ouverture : **bascule manuelle**, planning en rappel seulement |
| DEC-23 | Personnalités : **noyau de 8** + réserve priorisée |
| DEC-24 | Accès Pip-Boy : **lecture via vues dédiées** + panneau de contrôle au dashboard |

Détail et justifications dans [`DECISIONS.md`](DECISIONS.md).

## Points ouverts

Trois, dont un seul bloque une décision d'architecture :

- **REF-19 — hébergement de l'instance Supabase** (self-hosted vs cloud).
  Volontairement **non tranchée côté ZAX** : l'instance est partagée avec
  Pip-Boy, dont le modèle est inverse (dégradation gracieuse, cloud faisant
  autorité au retour réseau) là où ZAX vise un local strict. C'est AMB-001 /
  AMB-004 de `tech/`. → **document de position à ouvrir dans `tech/docs/`**, à
  réconcilier avec Pip-Boy sur le modèle de DEC-061.
- **REF-21 — la machine RTX 5090 part-elle sur le terrain ?** Conditionne
  l'enveloppe du modèle (24–32B contre 8–9B) et rouvre REF-19 si elle est sur
  site. Contrainte d'infra associée : ~600–800 W, refroidissement, onduleur.
- **REF-22 — l'échelle d'attitude de karma ne couvre pas son domaine** : trous
  41–49 et 181–189, chevauchements à 90 et 130. Arbitrage narratif, mais la
  couverture totale est une exigence technique non négociable.
- **REF-23 — conflits entre les brainstorms BMAD d'Auriane et les décisions
  actées.** Six points à arbitrer, dont un bloquant (les palettes d'ouverture du
  moteur de vote convoquent six personnalités hors du noyau de 8, et l'ouverture 2
  se retrouve sans personnalité par défaut) et un prioritaire (l'embedder retenu
  est anglophone, ce que DEC-21 interdit). Le fond est convergent — les deux
  travaux aboutissent à la même architecture de pipeline. À mener **avec
  Auriane** : [`docs/position-confrontation-brainstorms-auriane.md`](docs/position-confrontation-brainstorms-auriane.md).

Restent aussi ouverts **côté scénaristes**, consignés dans les décisions
concernées : commandes du Terminal Superviseur et degré de conscience du
joueur-superviseur (DEC-14) · valeurs numériques des seuils de karma (DEC-18) ·
défaut de bascule de l'Enfant et son éventuel retour en « fantôme » (DEC-19) ·
contenu des huit fins post-signal-bleu (DEC-20).

Liste complète, avec options et arguments, dans [`AMBIGUITES.md`](AMBIGUITES.md).

---

## Outillage — ingestion du lore

Chaîne de scrap du wiki Fallout, en trois étapes, dans `scripts/` (Python +
`openpyxl`) :

```bash
cd scripts

# 1. Découverte des liens internes, un Excel par page dans output/
#    Colonnes : Nom de la page | URL | Section | Inclure
python wiki_crawler.py urls.txt

# 2. Dédoublonnage des URLs entre tous les fichiers de output/
python dedup_xlsx.py

# 3. Réinjection des URLs marquées « O » dans urls.txt
python excel_to_urls.py
```

Le tri se fait à la main dans la colonne `Inclure` (O/N) : c'est le point de
contrôle éditorial sur ce qui entre dans la base de connaissance.

---

## Configuration

Les secrets ne sont **jamais** versionnés. Copier le modèle et le remplir :

```bash
cp .env.example .env               # production
cp .env.example .env.development   # développement
```

| Variable | Rôle |
|---|---|
| `PIPBOY_ZAX_WRITE_URL` | Endpoint de l'Edge Function `zax-write` côté Pip-Boy |
| `PIPBOY_ZAX_WRITE_SECRET` | Secret partagé pour les écritures ZAX → Pip-Boy |

Les valeurs réelles vivent dans le gestionnaire de mots de passe partagé.
**Ne jamais croiser un secret d'un environnement avec l'URL de l'autre.**

---

## Contraintes terrain

- **Pas d'accès internet garanti** — tout tourne en réseau local.
- Réseau filaire pour le QNAP, le GPU et les Raspberry Pi ; WiFi en backup seul.
- Réseau partiellement fiable : pertes, latence et coupures à assumer.
- **5 terminaux maximum** en simultané.
- Sessions de **48–72h** : stabilité Docker requise (healthchecks, volumes persistants).
- Un admin doit pouvoir intervenir **en 30 secondes** depuis le réseau local.
- Latence de réponse ZAX : **SLO** de perf — cible < 5 s, maximum 10–12 s. Ce ne
  sont **pas** des déclencheurs : le mode dégradé se déclenche sur le
  **time-to-first-token** (DEC-10), parce qu'en streaming le joueur subit le
  silence avant le premier token, pas la durée totale de génération.

---

## Charte graphique

Cohérente avec le Pip-Boy — terminal CRT à phosphore vert.

| Rôle | Couleur |
|---|---|
| Fond | `#0A0A0A` |
| Vert phosphore (texte actif) | `#39FF14` |
| Vert moyen (labels) | `#1DB308` |
| Vert dim (inactif) | `#0A5C0A` |
| Rouge danger | `#8B0000` |
| Ambre (mode dégradé) | `#FFB000` |

**Typographie :** `VT323` (titres, boutons) · `Share Tech Mono` (corps, chat, logs).

**Effets :** scanlines, vignette CRT, glow sur le texte actif, effet machine à
écrire sur les réponses ZAX. Zéro `border-radius`, pas d'animations lourdes
(calibré pour Raspberry Pi).

---

## Conventions de code

- TypeScript strict — zéro `any`, zéro `@ts-ignore` sans commentaire explicatif
- Composants fonctionnels uniquement, pas de classes
- Tous les appels BDD dans des hooks ou services dédiés, jamais dans les composants
- Tous les appels Ollama dans `services/llm.ts`
- Chaque hook expose au minimum `{ data, loading, error }`
- Variables d'environnement pour toutes les URLs et clés
- Commits en français, format `type: description` — référencer les décisions :
  `feat: implémentation DEC-03`

---

## Écosystème Fallout 2027

| Projet | Rôle |
|---|---|
| **zax-app** | Ce dépôt — IA narrative in-game |
| **pipboy-app** | PWA joueur (Pip-Boy simulé) + admin web MJ — instance Supabase partagée |
| **turret** | Tourelles NERF autonomes (détection, IFF, tir, hack NFC) |

L'accès de ZAX à l'instance partagée n'est **pas symétrique** (DEC-24, report de
DEC-061 côté Pip-Boy) :

- **Écriture** — ZAX écrit dans le domaine Pip-Boy **exclusivement** via l'Edge
  Function `zax-write` (secret partagé, header dédié, idempotence). Le rôle
  applicatif ZAX est confiné au schéma `zax` et n'a **aucun droit d'écriture**
  sur le domaine Pip-Boy — exclusivité vérifiée par pgTAP, pas seulement
  documentée.
- **Lecture** — par des **vues dédiées** uniquement, réduites à la surface utile
  (`profiles` : `nfc_uid`, nom, faction, statut vivant/mort, présence ·
  `factions`). Le schéma Pip-Boy évolue sans ZAX : une vue est une liste
  blanche, donc une colonne sensible arrivant plus tard ne peut pas fuiter dans
  une réponse narrative. Le dashboard admin expose un **panneau de contrôle** de
  ce contrat de lecture (vue présente ou absente, colonnes attendues contre
  colonnes exposées) — sans quoi une migration Pip-Boy casse ZAX en silence.

Voir [`docs/position-zax-auth-pipboy.md`](docs/position-zax-auth-pipboy.md).

---

## Équipe

**Boris** (`ex0nite` / `arcan-2027`) · **Auriane** — co-développeuse.
