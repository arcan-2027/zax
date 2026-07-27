# AMBIGUITES — Projet ZAX (Fallout 2027)

> Ce fichier recense toutes les ambiguïtés et contradictions détectées lors de la fusion
> de `CLAUDE-zax.md` et `CLAUDE-zax-2.md`.
> **À relire en début de chaque session de travail.**
> Quand une ambiguïté est résolue, mettre son statut à ✅ Décidé et référencer la décision correspondante.

---

### REF-01 — Supabase self-hosted vs PostgreSQL standalone

**Source :** CLAUDE-zax.md (infrastructure matérielle + stack technique + points ouverts)

**Problème :** Le fichier mentionne à la fois "PostgreSQL ou Supabase self-hosted" dans le schéma infrastructure, "Supabase self-hosted" dans la stack, et liste ce choix comme point ouvert en fin de document. Les trois occurrences sont cohérentes entre elles (c'est bien un point ouvert), mais aucune décision n'a été prise. Le lien avec la charge du projet Pipboy est cité comme critère de décision.

**Impact :** Le choix conditionne directement l'utilisation de Supabase Realtime (subscriptions temps réel du dashboard), de Supabase Auth (authentification joueurs), et des RLS. Un PostgreSQL standalone supprime ces fonctionnalités natives et impose de les reconstruire.

**Mise à jour (source `zax_20260706.md`, propal Auriane) :** un nouveau candidat apparaît — **SQLite** (« supportera notre usage, pas d'install »). Le document ne mentionne plus Supabase du tout, mais l'option porte encore un « ? » (proposition, pas décision). SQLite supprime nativement Realtime/Auth/RLS et impose de reconstruire le temps réel dashboard.

**Options possibles :**
- Option A : Supabase self-hosted sur QNAP → Realtime + Auth + RLS natifs, même instance que Pipboy, mais plus lourd pour le Celeron N3150
- Option B : PostgreSQL standalone sur QNAP → plus léger, mais Realtime et Auth à implémenter manuellement (WebSockets, gestion sessions)
- Option C : Supabase cloud (hors ligne en GN) → incompatible avec la contrainte offline-first
- Option D : **SQLite (propal Auriane)** → zéro install, largement suffisant pour 5 terminaux ; mais pas de Realtime/Auth natifs (à gérer côté moteur Node/TS)

**Statut :** ✅ Décidé (DEC-08) — **Supabase** (Option A), SQLite écarté. L'hébergement (self-hosted vs cloud) reste ouvert → REF-19.

---

### REF-02 — Ordre de priorité des données (cache local vs source distante)

**Source :** CLAUDE-zax.md (règle 3 architecture) vs CLAUDE-zax-2.md (section "Données et responsabilité")

**Problème :** Contradiction directe entre les deux fichiers.
- CLAUDE-zax.md, règle 3 : "ZAX ne stocke pas de données personnages en local — tout vient de Supabase." → Supabase est la source de vérité unique, pas de cache local.
- CLAUDE-zax-2.md : "Ordre de priorité des états : 1. Cache local terminal / ZAX, 2. État ZAX serveur, 3. Supabase." → Supabase est la dernière priorité, le cache local prime.

**Impact :** Fondamental pour l'architecture de synchronisation. Choisir l'option A interdit tout cache local. Choisir l'option B impose un mécanisme de cache + resynchronisation. Les deux approches impliquent des designs très différents pour la gestion des messages, des profils joueur, et du mode offline.

**Mise à jour (source `zax_20260706.md`) :** le nouveau modèle décrit une **BDD ZAX propre** alimentée par **import de la BDD orga** (DEC-05, 1h avant ouverture) — ce qui tend nettement vers une **source locale prioritaire** côté ZAX. À reconfirmer explicitement.

**Options possibles :**
- Option A : Source distante de vérité unique — simple, mais vulnérable aux latences réseau (peu compatible avec le fallback offline)
- Option B : **BDD locale ZAX prioritaire** — alimentée par import de la BDD orga ; résiliente aux coupures ; cohérente avec DEC-05 (favori)
- Option C : Hybride selon le type de donnée — profils importés de la BDD orga, conversations/karma en local ZAX

**Statut :** ✅ Décidé (DEC-09) — **Supabase source de vérité unique** (Option A). Pas de cache local prioritaire. La résilience réseau se traite au niveau accès Supabase (retries/file d'écritures), pas par une BDD locale concurrente.

---

### REF-03 — Timeout LLM non défini

**Source :** CLAUDE-zax.md (section "Mode dégradé") + CLAUDE-zax-2.md (section "Contraintes terrain")

**Problème :** Le mode dégradé se déclenche "si Ollama ne répond pas sous X secondes (configurable dans `zax_config`)" — la valeur X n'est nulle part définie. CLAUDE-zax-2.md donne des cibles de latence (< 5s, max 10–12s) mais ne précise pas si ces valeurs sont le déclencheur du mode dégradé ou juste des cibles de performance normales.

**Impact :** Sans valeur de timeout, le code du mode dégradé ne peut pas être implémenté de façon définitive. La valeur a un impact direct sur l'expérience joueur (délai subi avant basculement) et sur la détection de pannes réelles vs lenteurs ponctuelles.

**Options possibles :**
- Option A : Timeout = 10–12s (maximum absolu de latence depuis CLAUDE-zax-2 → au-delà on bascule)
- Option B : Timeout = 5s (cible de latence → on bascule dès qu'on dépasse la cible)
- Option C : Timeout configurable via `zax_config`, avec une valeur par défaut à choisir parmi A ou B
- Option D : Double seuil — avertissement à 5s, basculement forcé à 12s

**Statut :** ⏳ À décider

---

### REF-04 — Statut de LibreChat dans la stack

**Source :** CLAUDE-zax-2.md (section "LibreChat / outils") — absent de CLAUDE-zax.md

**Problème :** CLAUDE-zax-2.md mentionne LibreChat comme exemple de "brique technique" possible, avec la nuance qu'il "n'est jamais une autorité fonctionnelle". Mais CLAUDE-zax.md ne le mentionne pas du tout dans la stack. Il est impossible de savoir si LibreChat est : en cours d'évaluation, utilisé quelque part, ou simplement cité comme exemple générique.

**Impact :** Si LibreChat est effectivement dans la stack, il faut définir son rôle (interface de chat joueur ? interface admin ? wrapper LLM ?). S'il ne l'est pas, sa mention crée de la confusion dans les documents.

**Options possibles :**
- Option A : LibreChat n'est pas utilisé — la mention est générique, à supprimer pour éviter la confusion
- Option B : LibreChat est évalué comme interface de chat joueur alternative à l'interface React custom
- Option C : LibreChat est évalué comme interface admin ou comme wrapper vers Ollama
- Option D : LibreChat est conservé comme option de fallback si l'interface React custom pose problème

**Statut :** ⏳ À décider

---

### REF-05 — Architecture messages : JSONB monolithique vs table normalisée pour la détection de mots-clés

**Source :** CLAUDE-zax.md (schéma `zax_conversations.messages JSONB` + table `zax_alert_keywords`)

**Problème :** Les messages de conversation sont stockés dans un champ `messages (JSONB)` monolithique dans `zax_conversations`. La table `zax_alert_keywords` implique une détection de mots-clés en temps réel sur le contenu des messages. Scanner du JSONB en temps réel pour la détection de mots-clés est soit inefficace (scan applicatif) soit complexe (index GIN PostgreSQL), par opposition à une table de messages normalisée (une ligne = un message) qui permet des requêtes SQL classiques.

**Impact :** Choix structurant pour le schéma de base de données. Le JSONB monolithique est plus simple à implémenter initialement mais rend la détection de mots-clés et les statistiques de session plus difficiles. La table normalisée est plus propre pour les requêtes mais alourdit le schéma.

**Options possibles :**
- Option A : JSONB monolithique — détection de mots-clés côté applicatif (scan des messages en mémoire)
- Option B : JSONB monolithique + index GIN PostgreSQL — détection SQL via `jsonb_array_elements`
- Option C : Table `zax_messages` normalisée (id, conversation_id, role, content, created_at) — requêtes simples, statistiques natives
- Option D : Hybride — JSONB pour le stockage + vue matérialisée pour la détection

**Statut :** ⏳ À décider

---

### REF-06 — Flux RFID → Auth Supabase → RLS (les joueurs ont-ils des comptes Auth ?)

**Source :** CLAUDE-zax.md (stack : "Supabase Auth + lecture tag RFID", architecture règle 2, rôles utilisateurs)

**Problème :** Le flux d'authentification n'est pas explicitement spécifié. Le scan RFID identifie un UUID mappé à `nfc_uid` dans `profiles`, mais aucun document ne précise si les joueurs ont des comptes Supabase Auth réels (email/password ou JWT) ou si l'identification RFID est un simple lookup applicatif sans Auth Supabase. Ce choix conditionne directement le fonctionnement des RLS (Row Level Security), qui dépendent de `auth.uid()`.

**Impact :** Si les joueurs n'ont pas de comptes Auth Supabase, les RLS ne peuvent pas s'appliquer au niveau joueur (elles fonctionneront uniquement pour les rôles admin/orga avec JWT). Toute la sécurité des données joueur repose alors sur le middleware API. Si les joueurs ont des comptes Auth, le flux de connexion via RFID doit générer un JWT côté serveur.

**Options possibles :**
- Option A : Les joueurs n'ont PAS de comptes Auth Supabase — l'UUID RFID est un lookup applicatif, la sécurité est gérée par le middleware API exclusivement
- Option B : Les joueurs ont des comptes Auth Supabase — le scan RFID déclenche une authentification server-side qui génère un JWT joueur
- Option C : Authentification "service role" pour toutes les opérations ZAX — contourne les RLS joueur, simplifie le flux mais impose une vigilance accrue côté middleware

**Statut :** ⏳ À décider

---

### REF-07 — Interface du Terminal Superviseur de l'Abri non spécifiée

**Source :** CLAUDE-zax.md (infrastructure : "1x Terminal Superviseur de l'Abri (rôle spécial)"; rôles : "superviseur")

**Problème :** Le Terminal Superviseur de l'Abri est mentionné comme un terminal distinct avec un rôle spécial (`superviseur`), différent des 4 terminaux joueurs. Mais aucune spec d'interface n'existe pour ce terminal. On ne sait pas : quelles commandes ZAX spéciales il expose, si son UI est distincte de l'interface joueur standard, comment il s'authentifie, ni quel est son rôle narratif dans le GN.

**Impact :** Sans spec, le développement de ce terminal ne peut pas commencer. Le rôle `superviseur` est défini dans les permissions mais son périmètre fonctionnel est vide. Risque de traiter ce terminal comme un terminal joueur standard et de rater des fonctionnalités importantes pour le scénario.

**Options possibles :**
- Option A : Interface joueur identique + commandes spéciales accessibles via un menu caché ou des commandes texte spécifiques
- Option B : Interface dédiée distincte de l'interface joueur standard
- Option C : Interface joueur + accès partiel au dashboard admin (vue lecture seule des terminaux)
- Option D : À définir par les scénaristes (scope narratif à préciser avant décision technique)

**Statut :** ⏳ À décider

---

### REF-08 — Modèle LLM final non choisi (Llama 3 8B vs Mistral 7B)

**Source :** CLAUDE-zax.md (infrastructure + points ouverts)

**Problème :** Le choix entre Llama 3 8B et Mistral 7B est listé comme point ouvert. Les deux sont des modèles quantisés compatibles Ollama sur GTX 1080 (8GB VRAM), mais leurs performances en français, leur respect du harnais narratif et leur latence sur cette machine spécifique n'ont pas été testés.

**Impact :** Le modèle conditionne directement la qualité des réponses ZAX, la latence (critique pour respecter les < 5s / 10–12s max), et la conformité au harnais narratif. Un changement de modèle après développement peut nécessiter des ajustements de prompt.

**Options possibles :**
- Option A : Llama 3 8B — meilleur en français selon benchmarks récents, plus récent
- Option B : Mistral 7B — plus léger, potentiellement plus rapide sur GPU limité
- Option C : Tests comparatifs sur la GTX 1080 avec le harnais ZAX avant décision
- Option D : Architecture permettant le swap de modèle via `zax_config` (décision non bloquante)

**Statut :** ⏳ À décider

---

### REF-09 — Nommage des fichiers de décision : DECISIONS.md vs DECISIONS-*.md

**Source :** CLAUDE-zax.md (section "Fichiers de référence" : `DECISIONS.md`) vs CLAUDE-zax-2.md (section "Autorité documentaire" : `DECISIONS-*.md`)

**Problème :** Les deux fichiers désignent les décisions architecturales comme autorité supérieure, mais avec des conventions de nommage incompatibles. `DECISIONS.md` suggère un fichier unique. `DECISIONS-*.md` suggère plusieurs fichiers thématiques (ex : `DECISIONS-auth.md`, `DECISIONS-llm.md`).

**Impact :** Structure documentaire fondamentale. Un fichier unique est plus simple à maintenir et à relire en début de session. Des fichiers multiples permettent une granularité thématique mais compliquent la recherche et multiplient les risques d'incohérence entre fichiers.

**Options possibles :**
- Option A : `DECISIONS.md` unique — toutes les décisions dans un seul fichier, sections par thème
- Option B : `DECISIONS-*.md` multiples — un fichier par domaine (auth, llm, db, ui…)
- Option C : `DECISIONS.md` principal + fichiers annexes référencés dedans

**Statut :** ⏳ À décider

---

### REF-10 — Relation entre latences cibles et déclencheur du mode dégradé

**Source :** CLAUDE-zax-2.md (contraintes terrain : "< 5s, max 10–12s") vs CLAUDE-zax.md (mode dégradé : "X secondes configurable")

**Problème :** CLAUDE-zax-2.md définit deux valeurs de latence (cible < 5s, maximum absolu 10–12s) mais ne dit pas explicitement si l'une d'elles est le seuil de déclenchement du mode dégradé. CLAUDE-zax.md laisse ce seuil comme variable non définie (`X secondes`). Les deux documents parlent du même système mais ne se réfèrent pas l'un à l'autre.

**Impact :** Voir REF-03 (partiellement redondant). La distinction ici est sur le sens des valeurs : sont-elles des SLO de performance ou des triggers opérationnels ? Si le trigger mode dégradé est à 5s, beaucoup de réponses LLM normales déclencheront du mode dégradé. Si c'est à 12s, le joueur attend jusqu'à 12s avant de voir une réponse.

**Options possibles :**
- Option A : Trigger = 10–12s (le maximum absolu est le seuil de basculement)
- Option B : Trigger configurable indépendamment des cibles de latence (valeur par défaut à définir)
- Option C : Pas de trigger automatique — mode dégradé activé uniquement manuellement par un admin

**Statut :** ⏳ À décider

---

### REF-11 — Rattachement écosystème : BDD ZAX propre + BDD orga vs Supabase partagé Pipboy

**Source :** `CLAUDE.md` (Supabase partagé avec Pipboy) vs `sources/zax_20260706.md` (BDD ZAX propre + « lien avec la BDD orga »)

**Problème :** L'ancienne architecture posait une **instance Supabase partagée avec Pipboy** comme socle. Le nouveau document ne mentionne jamais Supabase ni Pipboy : il décrit une **BDD propre à ZAX** et un **lien avec la BDD orga** (accès direct ou import). Le rattachement à l'écosystème est donc en révision. (Note : « Overseer » cité dans la source est le **projet perso d'Auriane**, co-dev ZAX, référence de stack Node/TS — ce n'est PAS l'outil orga de ZAX.)

**Impact :** Conditionne où vivent les profils PJ/PNJ et factions, comment ZAX les obtient, et si Pipboy et ZAX partagent encore quoi que ce soit. Impacte REF-01, REF-02, REF-06.

**Options possibles :**
- Option A : Supabase partagé Pipboy (ancienne cible) — profils dans `profiles`, RLS communes
- Option B : BDD ZAX propre (SQLite) + **import de la BDD orga** 1h avant ouverture (DEC-05) — découplé de Pipboy (favori)
- Option C : BDD ZAX propre + **accès direct** à la BDD orga quand le réseau est dispo, import en fallback

**Statut :** ✅ Décidé (DEC-08) — **Supabase partagé avec Pipboy** (Option A). Le modèle « BDD ZAX propre » est écarté ; l'hébergement de l'instance reste ouvert → REF-19.

---

### REF-12 — Méthode de découpage et de détection des sujets

**Source :** `sources/zax_20260706.md` (réunion 22/06 — « le problème est là »)

**Problème :** Le moteur doit découper l'input joueur en **blocs thématiques** puis déterminer, pour chacun, s'il correspond à un sujet de la personnalité (activation de triggers, karma). On ne peut pas faire confiance à l'utilisateur pour une écriture propre. La méthode n'est pas tranchée : chaînes de Markov évoquées, produit scalaire d'embeddings avec seuil ~0.7 évoqué. Le découpage en blocs d'idées est explicitement désigné comme le point dur.

**Impact :** Cœur du moteur de conversation (choix de personnalité + karma + sujets interdits/favoris). Détermine la robustesse de tout le système.

**Options possibles :**
- Option A : Embeddings + similarité cosinus (seuil ~0.7) sur des blocs découpés
- Option B : Chaînes de Markov / approche statistique légère
- Option C : Classification par mots-clés simple (DECL-HARD/SOFT) sans NLP lourd
- Option D : Hybride mots-clés (hard triggers) + embeddings (soft triggers/karma)

**Statut :** ⏳ À décider

---

### REF-13 — Seuils de karma pour le don du G.E.C.K.

**Source :** `sources/zax_20260706.md` (réunion 22/06)

**Problème :** DEC-07 fixe la condition ferme (destinataire présent + vivant), mais les seuils restent ouverts : seuil minimal de karma PJ, seuil minimal de karma faction, condition sur la somme (karma_pj + karma_faction ≤/≥ X ?), et éventuel seuil propre à chaque personnalité.

**Impact :** Détermine la difficulté d'obtention du GECK et l'équilibrage narratif des 3 ouvertures.

**Options possibles :**
- Option A : Seuil unique sur le karma total (faction + PJ)
- Option B : Double seuil indépendant (PJ ET faction)
- Option C : Seuil paramétrable par personnalité (l'Archiviste donnant au meilleur contributeur, etc.)

**Statut :** ⏳ À décider

---

### REF-14 — Mécanique de la personnalité « L'Enfant » (good/bad ending, bascule)

**Source :** `sources/zax_20260706.md` (fiches « Enfant bad ending » / « WIP Enfant good ending » + notes Auriane)

**Problème :** L'Enfant existe en deux versions — **Exfiltration** (« good », surnommée Charlie, veut sortir/se faire des amis) et **Destruction** (« bad », veut corrompre le système). Note Auriane : l'Enfant aurait 2 sous-personnalités à la 1ʳᵉ ouverture, et l'une disparaît selon les discussions (donc Enfant Exfiltration OU Destruction aux ouvertures 2 et 3). La mécanique de bascule/suppression n'est pas figée.

**Impact :** Structure de données des personnalités (sous-personnalités ? état évolutif ?) et logique de bascule dans le moteur.

**Statut :** ⏳ À décider

---

### REF-15 — Impact du signal bleu sur les personnalités

**Source :** `sources/zax_20260706.md` (section « Notes »)

**Problème :** Si le signal bleu est déclenché, chaque personnalité pousse une fin différente (Archiviste = meilleur modèle social ; Scientifique = goulification ; Board = numérisation/défense ; Juge = éliminer les goules ; Soldat = guerre ; Mood Manager = Cash vs Kings ; Enfant = faire sortir ZAX / détruire le GECK pacifiquement). L'implémentation de ces comportements post-signal-bleu (et leur arbitrage) n'est pas spécifiée.

**Impact :** Comportements spécifiques par personnalité en fin de partie ; conditions de fin du GN.

**Statut :** ⏳ À décider

---

### REF-16 — Format d'ingestion du lore + moteur de recherche sémantique

**Source :** `sources/zax_20260706.md` (WIP Auriane — Base de connaissance)

**Problème :** Le lore Fallout + GN doit être importé, découpé (chunking en idées), indexé et rendu interrogeable par recherche sémantique (RAG). Le **format d'entrée pour la BDD** et le **moteur d'embeddings** ne sont pas définis. Se pose aussi la distinction lore (pré-établi) vs savoir d'observation (appris en jeu) et le filtrage du lore accessible par personnalité.

**Impact :** Qualité et cohérence des réponses ; volume de préparation scénariste ; performance sur le hardware GN.

**Statut :** ⏳ À décider

---

### REF-17 — Déclenchement de l'état d'ouverture (manuel vs automatique)

**Source :** `sources/zax_20260706.md` (Harnais — « L'état d'ouverture »)

**Problème :** Le harnais bascule entre 3 états d'ouverture. Le document note « si ouverture auto, trouver comment déclencher automatiquement » — sans trancher entre bascule manuelle (orga) et automatique (horaire, événement).

**Impact :** Pilotage MJ de la progression narrative ; logique de bascule dans le moteur.

**Options possibles :**
- Option A : Bascule manuelle par un orga depuis le dashboard
- Option B : Bascule automatique (horaire / déclencheur d'événement)
- Option C : Manuel avec possibilité d'automatisation planifiée

**Statut :** ⏳ À décider

---

### REF-18 — Liste canonique des personnalités actives vs secondaires

**Source :** `sources/zax_20260706.md` (« Détails des personnalités »)

**Problème :** Le document classe 4 personnalités comme « actives (harnais canonique) » — Le Gardien, Le Board, Happiness Officer, Mood Manager — et 10+ comme « secondaires (suggestions à valider pour le harnais) », dont Le Juge qui apparaît dans les trames sans fiche « Détails ». La liste finale des personnalités implémentées pour le GN n'est pas figée.

**Impact :** Périmètre de travail scénariste (nombre de templates YAML) et de test.

**Statut :** ⏳ À décider

---

### REF-19 — Hébergement de l'instance Supabase : self-hosted (QNAP) vs cloud

**Source :** DEC-08 (le moteur BDD est tranché — Supabase — mais pas son hébergement)

**Problème :** Supabase est confirmé comme BDD partagée avec Pipboy (DEC-08), mais son hébergement n'est pas tranché : **self-hosted sur le QNAP** (réseau local GN) ou **cloud**. La décision est ouverte.

**Impact :** Point de tension direct avec la contrainte terrain « pas d'accès internet garanti » (§19). Une instance cloud dépend d'une connectivité externe non garantie pendant le GN ; une instance self-hosted QNAP est plus lourde pour le Celeron N3150 (4GB RAM) mais reste disponible hors ligne. Impacte aussi le fallback DEC-05.

**Options possibles :**
- Option A : Supabase self-hosted sur le QNAP — disponible en réseau local même sans internet ; charge à surveiller sur le N3150
- Option B : Supabase cloud — géré, plus léger côté QNAP ; **mais** suppose une connectivité internet fiable le jour J (risque terrain fort), ou un mode dégradé/mirroir local
- Option C : Cloud en nominal + réplica/snapshot local de secours activé pendant le GN

**Statut :** ⏳ À décider

---

### REF-20 — Périmètre des écritures directes de ZAX dans l'instance Supabase partagée

**Source :** DEC-08 / `CLAUDE.md` §1 (« les deux projets lisent/écrivent les mêmes tables, notamment `profiles` ») vs décision côté Pip-Boy (juillet 2026, autoritaire) : ZAX écrit dans le domaine Pip-Boy **exclusivement** via l'Edge Function `zax-write` (transmissions radio + notes uniquement), interdiction appliquée au niveau BDD (rôle Postgres à INSERT restreint).

**Problème :** DEC-08 formule un accès lecture/écriture symétrique aux tables partagées, alors que le contrat Pip-Boy restreint les écritures ZAX vers son domaine à deux types via `zax-write`. Les deux sont conciliables (ZAX n'a probablement jamais besoin d'écrire `profiles`), mais le périmètre exact n'est écrit nulle part : quelles tables partagées ZAX lit-il directement, sur lesquelles a-t-il des grants d'écriture, et la connexion Supabase directe de ZAX est-elle elle aussi restreinte par grants (symétrie du garde-fou) ?

**Impact :** Configuration des rôles/grants Postgres de la connexion ZAX ; garantie que l'exclusivité de `zax-write` est appliquée techniquement et pas seulement documentée ; clarification de DEC-08 lors de la réconciliation ZAX ↔ Pip-Boy.

**Options possibles :**
- Option A : ZAX = lecture seule sur toutes les tables du domaine Pip-Boy (`profiles`, `factions`…) + écriture libre sur ses tables `zax_*` + toute écriture vers le domaine Pip-Boy via `zax-write` — grants configurés en conséquence (favori, cohérent avec la position `docs/position-zax-auth-pipboy.md`)
- Option B : statu quo DEC-08 (écriture directe possible) — contredit le contrat Pip-Boy, à écarter sauf besoin identifié
- Option C : liste explicite table par table (lecture/écriture) annexée au contrat d'API

**Statut :** ⏳ À décider (à trancher lors de la réconciliation avec le côté Pip-Boy — cf. question Q8 de `docs/position-zax-auth-pipboy.md`)

---

*Dernière mise à jour : 2026-07-12 — 20 ambiguïtés recensées (REF-01 à REF-20). REF-01, REF-02, REF-11 ✅ résolues (DEC-08, DEC-09). 9 décisions actées (DEC-01 à DEC-09) dans `DECISIONS.md`.*
