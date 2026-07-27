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
prototypes d'interface et l'outillage d'ingestion du lore. Le développement
(frontend joueur, dashboard admin, backend moteur) démarre une fois les
ambiguïtés bloquantes levées.

| | |
|---|---|
| Décisions actées | 9 (DEC-01 → DEC-09) |
| Points ouverts | 17 (sur 20 référencés) |
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
| LLM | Ollama, API locale sur machine GPU dédiée |
| Harnais & personnalités | **YAML** externe, chargé à runtime (DEC-03) |
| Conteneurisation | Docker Compose (Container Station sur QNAP) |

### Infrastructure terrain

```
[Réseau local filaire dédié GN]
        │
        ├── QNAP TS-453A              → Serveur applicatif ZAX (Docker)
        │     Celeron N3150, 4 GB RAM
        │
        ├── Tour GTX 1080 (8 GB VRAM) → Serveur LLM (Ollama)
        │
        └── Terminaux (5 max)
              4x Raspberry Pi (interface joueur)
              1x Terminal Superviseur de l'Abri
              Chromium plein écran, aucune installation locale
```

### Règles non négociables

1. Les tags RFID/NFC n'encodent **qu'un UUID** — jamais de données métier.
2. **Supabase est la source de vérité unique** (DEC-09) : pas de cache local
   prioritaire, aucune donnée personnage dupliquée côté ZAX.
3. **Toute réponse LLM passe par une étape de validation** — aucun crash LLM ne
   doit atteindre le joueur (DEC-04).
4. Le harnais de personnalité est chargé depuis des fichiers YAML externes,
   **jamais hardcodé**, modifiable sans redéploiement.
5. Le harnais est l'**autorité narrative** : le LLM ne doit jamais être imprévisible.
6. **Aucune perte de message n'est acceptable** — resynchronisation et réparation
   d'état obligatoires.
7. Toute logique de permission se gère **côté serveur**.
8. Un orga doit pouvoir **forcer une personnalité ou une décision** à tout moment.

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
│   └── position-zax-auth-pipboy.md   Frontière d'authentification ZAX ↔ Pip-Boy
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

Détail et justifications dans [`DECISIONS.md`](DECISIONS.md).

## Principaux points ouverts

Hébergement Supabase self-hosted QNAP vs cloud (REF-19) · choix du modèle LLM
(REF-08) · timeout du mode dégradé (REF-03/REF-10) · architecture des messages,
JSONB vs table normalisée (REF-05) · flux d'identification joueur (REF-06) ·
méthode de découpage et détection des sujets (REF-12) · seuils de karma pour le
don du G.E.C.K. (REF-13) · format d'ingestion du lore et moteur d'embeddings
(REF-16) · liste canonique des personnalités (REF-18).

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
- Latence de réponse ZAX : cible **< 5 s**, maximum absolu 10–12 s.

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

ZAX écrit dans le domaine Pip-Boy **exclusivement** via l'Edge Function
`zax-write` (secret partagé, header dédié, idempotence). Les tables propres à ZAX
vivent dans un schéma `zax` séparé, sur la même instance.

Voir [`docs/position-zax-auth-pipboy.md`](docs/position-zax-auth-pipboy.md).

---

## Équipe

**Boris** (`ex0nite` / `arcan-2027`) · **Auriane** — co-développeuse.
