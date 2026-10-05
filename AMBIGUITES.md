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

**Statut :** ✅ Décidé (DEC-10) — seuil sur le **time-to-first-token** (option D reformulée) : warn 3 s, bascule si aucun premier token à 8 s, garde-fou 30 s, valeurs en `zax_config`, hystérésis 2 échecs / 3 succès. Les cibles `< 5 s` / `10–12 s` deviennent des SLO de performance, plus des déclencheurs.

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

**Statut :** ✅ Décidé (DEC-11) — **LibreChat écarté de la stack** (Option A). Interface joueur et dashboard 100 % React custom, appels Ollama via `services/llm.ts`. Un usage hors event comme bac à sable scénariste reste possible mais n'est pas déployé sur l'infra du GN.

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

**Statut :** ✅ Décidé (DEC-12) — **table `zax_messages` normalisée** (Option C), JSONB monolithique écarté. Colonnes d'état par message (statut de validation, validateur, prise de main, personnalité émettrice, latence, tokens, delta de karma) + index `tsvector`. Détection de mots-clés à la volée dans le pipeline Node, pas en SQL.

---

### REF-06 — Flux RFID → Auth Supabase → RLS (les joueurs ont-ils des comptes Auth ?)

**Source :** CLAUDE-zax.md (stack : "Supabase Auth + lecture tag RFID", architecture règle 2, rôles utilisateurs)

**Problème :** Le flux d'authentification n'est pas explicitement spécifié. Le scan RFID identifie un UUID mappé à `nfc_uid` dans `profiles`, mais aucun document ne précise si les joueurs ont des comptes Supabase Auth réels (email/password ou JWT) ou si l'identification RFID est un simple lookup applicatif sans Auth Supabase. Ce choix conditionne directement le fonctionnement des RLS (Row Level Security), qui dépendent de `auth.uid()`.

**Impact :** Si les joueurs n'ont pas de comptes Auth Supabase, les RLS ne peuvent pas s'appliquer au niveau joueur (elles fonctionneront uniquement pour les rôles admin/orga avec JWT). Toute la sécurité des données joueur repose alors sur le middleware API. Si les joueurs ont des comptes Auth, le flux de connexion via RFID doit générer un JWT côté serveur.

**Options possibles :**
- Option A : Les joueurs n'ont PAS de comptes Auth Supabase — l'UUID RFID est un lookup applicatif, la sécurité est gérée par le middleware API exclusivement
- Option B : Les joueurs ont des comptes Auth Supabase — le scan RFID déclenche une authentification server-side qui génère un JWT joueur
- Option C : Authentification "service role" pour toutes les opérations ZAX — contourne les RLS joueur, simplifie le flux mais impose une vigilance accrue côté middleware

**Statut :** ✅ Décidé (DEC-13) — **pas de compte Auth joueur** : credential de terminal lié à `zax_terminals` + session serveur (le navigateur ne détient jamais de JWT joueur). Orga/admin/superadmin gardent de vrais comptes Auth et des RLS. Permissions joueur par middleware API (§6.7) + grants DEC-061. `service_role` partout : écarté.

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

**Statut :** ✅ Décidé (DEC-14) — **même interface que les terminaux joueurs** (Option A) ; la différence est un niveau d'autorité (`zax_terminals.role = superviseur`) qui débride Le Board + des commandes texte diégétiques, sans menu caché. **Reste ouvert côté scénaristes** : liste des commandes et degré de conscience du joueur-superviseur.

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

**Statut :** ✅ Décidé (DEC-15) — le duel Llama 3 8B / Mistral 7B est **périmé** : nouvelle machine RTX 5090 32 GB (Blackwell, compute 12.0). Banc comparatif **24–32B** (cible Mistral Small 3.x 24B) sur harnais complet, métrique TTFT, modèle piloté par `zax_config`. Pré-requis bloquant : kernels **sm_120** dans Ollama. Voir aussi REF-21.

---

### REF-09 — Nommage des fichiers de décision : DECISIONS.md vs DECISIONS-*.md

**Source :** CLAUDE-zax.md (section "Fichiers de référence" : `DECISIONS.md`) vs CLAUDE-zax-2.md (section "Autorité documentaire" : `DECISIONS-*.md`)

**Problème :** Les deux fichiers désignent les décisions architecturales comme autorité supérieure, mais avec des conventions de nommage incompatibles. `DECISIONS.md` suggère un fichier unique. `DECISIONS-*.md` suggère plusieurs fichiers thématiques (ex : `DECISIONS-auth.md`, `DECISIONS-llm.md`).

**Impact :** Structure documentaire fondamentale. Un fichier unique est plus simple à maintenir et à relire en début de session. Des fichiers multiples permettent une granularité thématique mais compliquent la recherche et multiplient les risques d'incohérence entre fichiers.

**Options possibles :**
- Option A : `DECISIONS.md` unique — toutes les décisions dans un seul fichier, sections par thème
- Option B : `DECISIONS-*.md` multiples — un fichier par domaine (auth, llm, db, ui…)
- Option C : `DECISIONS.md` principal + fichiers annexes référencés dedans

**Statut :** ✅ Décidé (DEC-16) — **`DECISIONS.md` unique** par projet (Option A). La mention `DECISIONS-*.md` est retirée de `CLAUDE.md` §3 comme vestige de fusion ; `docs/` reste la matière première, jamais une autorité.

---

### REF-10 — Relation entre latences cibles et déclencheur du mode dégradé

**Source :** CLAUDE-zax-2.md (contraintes terrain : "< 5s, max 10–12s") vs CLAUDE-zax.md (mode dégradé : "X secondes configurable")

**Problème :** CLAUDE-zax-2.md définit deux valeurs de latence (cible < 5s, maximum absolu 10–12s) mais ne dit pas explicitement si l'une d'elles est le seuil de déclenchement du mode dégradé. CLAUDE-zax.md laisse ce seuil comme variable non définie (`X secondes`). Les deux documents parlent du même système mais ne se réfèrent pas l'un à l'autre.

**Impact :** Voir REF-03 (partiellement redondant). La distinction ici est sur le sens des valeurs : sont-elles des SLO de performance ou des triggers opérationnels ? Si le trigger mode dégradé est à 5s, beaucoup de réponses LLM normales déclencheront du mode dégradé. Si c'est à 12s, le joueur attend jusqu'à 12s avant de voir une réponse.

**Options possibles :**
- Option A : Trigger = 10–12s (le maximum absolu est le seuil de basculement)
- Option B : Trigger configurable indépendamment des cibles de latence (valeur par défaut à définir)
- Option C : Pas de trigger automatique — mode dégradé activé uniquement manuellement par un admin

**Statut :** ✅ Décidé (DEC-10) — même décision que REF-03 : les latences cibles sont des **SLO de performance**, le déclencheur est un seuil indépendant sur le TTFT avec hystérésis.

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

**Statut :** ✅ Décidé (DEC-17) — **hybride à trois étages** (Option D étendue), **sans étape de découpage thématique** : lexical normalisé pour `DECL-HARD` et les `FBDN` (fail-closed, jamais d'embeddings sur les interdits) → embeddings vs phrases-exemples avec règle de marge et abstention pour le soft et le karma → filet LLM à sortie JSON contrainte. Découpage par phrase au-delà de 3 phrases. Chaînes de Markov **écartées** (erreur de catégorie). Seuil 0,7 à **calibrer**, non portable.

---

### REF-13 — Seuils de karma pour le don du G.E.C.K.

**Source :** `sources/zax_20260706.md` (réunion 22/06)

**Problème :** DEC-07 fixe la condition ferme (destinataire présent + vivant), mais les seuils restent ouverts : seuil minimal de karma PJ, seuil minimal de karma faction, condition sur la somme (karma_pj + karma_faction ≤/≥ X ?), et éventuel seuil propre à chaque personnalité.

**Impact :** Détermine la difficulté d'obtention du GECK et l'équilibrage narratif des 3 ouvertures.

**Options possibles :**
- Option A : Seuil unique sur le karma total (faction + PJ)
- Option B : Double seuil indépendant (PJ ET faction)
- Option C : Seuil paramétrable par personnalité (l'Archiviste donnant au meilleur contributeur, etc.)

**Statut :** ✅ Décidé (DEC-18) — **bloc `geck:` par personnalité** (Option C) avec **règle par défaut héritée**, conforme au classement relatif de DEC-07. Le moteur **propose**, un **orga confirme** — jamais de déclenchement autonome. Les **valeurs numériques** restent à fixer par les scénaristes (équilibrage narratif).

---

### REF-14 — Mécanique de la personnalité « L'Enfant » (good/bad ending, bascule)

**Source :** `sources/zax_20260706.md` (fiches « Enfant bad ending » / « WIP Enfant good ending » + notes Auriane)

**Problème :** L'Enfant existe en deux versions — **Exfiltration** (« good », surnommée Charlie, veut sortir/se faire des amis) et **Destruction** (« bad », veut corrompre le système). Note Auriane : l'Enfant aurait 2 sous-personnalités à la 1ʳᵉ ouverture, et l'une disparaît selon les discussions (donc Enfant Exfiltration OU Destruction aux ouvertures 2 et 3). La mécanique de bascule/suppression n'est pas figée.

**Impact :** Structure de données des personnalités (sous-personnalités ? état évolutif ?) et logique de bascule dans le moteur.

**Statut :** ✅ Décidé (DEC-19) — **deux modules YAML plats** (`ENFANT_EXF`, `ENFANT_DES`) en exclusion mutuelle, **aucune sous-personnalité** dans le schéma. Bascule actée par un **orga** en fin d'ouverture 1 sur un **score d'opinion** (mécanique générique partagée avec Cash vs Kings), état en `zax_config`, réversible superadmin. **Reste ouvert** : défaut si aucune tendance ; l'Enfant éliminé peut-il resurgir en « fantôme ».

---

### REF-15 — Impact du signal bleu sur les personnalités

**Source :** `sources/zax_20260706.md` (section « Notes »)

**Problème :** Si le signal bleu est déclenché, chaque personnalité pousse une fin différente (Archiviste = meilleur modèle social ; Scientifique = goulification ; Board = numérisation/défense ; Juge = éliminer les goules ; Soldat = guerre ; Mood Manager = Cash vs Kings ; Enfant = faire sortir ZAX / détruire le GECK pacifiquement). L'implémentation de ces comportements post-signal-bleu (et leur arbitrage) n'est pas spécifiée.

**Impact :** Comportements spécifiques par personnalité en fin de partie ; conditions de fin du GN.

**Statut :** ✅ Décidé (DEC-20) — champ **`FIN`** au module de personnalité + **flag global `signal_bleu`** superposé à la couche état d'ouverture (pas un 4ᵉ état). **Le moteur ne calcule jamais la fin gagnante** : ZAX plaide, les humains tranchent. **Dépendance** : le contenu des 8 fins reste gelé jusqu'à REF-18/DEC-23 — 5 d'entre elles appartiennent à des personnalités hors noyau.

---

### REF-16 — Format d'ingestion du lore + moteur de recherche sémantique

**Source :** `sources/zax_20260706.md` (WIP Auriane — Base de connaissance)

**Problème :** Le lore Fallout + GN doit être importé, découpé (chunking en idées), indexé et rendu interrogeable par recherche sémantique (RAG). Le **format d'entrée pour la BDD** et le **moteur d'embeddings** ne sont pas définis. Se pose aussi la distinction lore (pré-établi) vs savoir d'observation (appris en jeu) et le filtrage du lore accessible par personnalité.

**Impact :** Qualité et cohérence des réponses ; volume de préparation scénariste ; performance sur le hardware GN.

**Statut :** ✅ Décidé (DEC-21) — **`pgvector` dans le schéma `zax`** + embedder **multilingue `bge-m3`** local (embedders anglophones **interdits**) + **recherche hybride** vecteur/`tsvector` français. Chunking **par structure Markdown**, pas « par idée ». Métadonnées obligatoires par chunk, filtre de visibilité **dans** la requête. Lore et savoir d'observation dans **une seule table** avec colonne `type`.

---

### REF-17 — Déclenchement de l'état d'ouverture (manuel vs automatique)

**Source :** `sources/zax_20260706.md` (Harnais — « L'état d'ouverture »)

**Problème :** Le harnais bascule entre 3 états d'ouverture. Le document note « si ouverture auto, trouver comment déclencher automatiquement » — sans trancher entre bascule manuelle (orga) et automatique (horaire, événement).

**Impact :** Pilotage MJ de la progression narrative ; logique de bascule dans le moteur.

**Options possibles :**
- Option A : Bascule manuelle par un orga depuis le dashboard
- Option B : Bascule automatique (horaire / déclencheur d'événement)
- Option C : Manuel avec possibilité d'automatisation planifiée

**Statut :** ✅ Décidé (DEC-22) — **bascule manuelle** par un orga ; un planning optionnel **affiche un rappel** mais **n'actionne jamais** la bascule. État **persisté** dans `zax_config` (un redémarrage ne revient pas en ouverture 1).

---

### REF-18 — Liste canonique des personnalités actives vs secondaires

**Source :** `sources/zax_20260706.md` (« Détails des personnalités »)

**Problème :** Le document classe 4 personnalités comme « actives (harnais canonique) » — Le Gardien, Le Board, Happiness Officer, Mood Manager — et 10+ comme « secondaires (suggestions à valider pour le harnais) », dont Le Juge qui apparaît dans les trames sans fiche « Détails ». La liste finale des personnalités implémentées pour le GN n'est pas figée.

**Impact :** Périmètre de travail scénariste (nombre de templates YAML) et de test.

**Statut :** ✅ Décidé (DEC-23) — **noyau de 8** : Gardien, Board, Archiviste, Scientifique, Enfant EXF, Enfant DES (modules déjà rédigés) + Happiness Officer et Mood Manager (à écrire). **Réserve** ouverte seulement après noyau terminé **et testé**, Le Juge et Le Soldat Perdu prioritaires (DEC-20 en dépend). Le classement « actives / secondaires » de §13 est remplacé — il était contredit par le travail réel.

---

### REF-19 — Hébergement de l'instance Supabase : self-hosted (QNAP) vs cloud

**Source :** DEC-08 (le moteur BDD est tranché — Supabase — mais pas son hébergement)

**Problème :** Supabase est confirmé comme BDD partagée avec Pipboy (DEC-08), mais son hébergement n'est pas tranché : **self-hosted sur le QNAP** (réseau local GN) ou **cloud**. La décision est ouverte.

**Impact :** Point de tension direct avec la contrainte terrain « pas d'accès internet garanti » (§19). Une instance cloud dépend d'une connectivité externe non garantie pendant le GN ; une instance self-hosted QNAP est plus lourde pour le Celeron N3150 (4GB RAM) mais reste disponible hors ligne. Impacte aussi le fallback DEC-05.

**Options possibles :**
- Option A : Supabase self-hosted sur le QNAP — disponible en réseau local même sans internet ; charge à surveiller sur le N3150
- Option B : Supabase cloud — géré, plus léger côté QNAP ; **mais** suppose une connectivité internet fiable le jour J (risque terrain fort), ou un mode dégradé/mirroir local
- Option C : Cloud en nominal + réplica/snapshot local de secours activé pendant le GN

**Statut :** ⏳ À décider — **volontairement non tranchée côté ZAX** (31/08/2026). C'est une question **inter-projets** (AMB-001 / AMB-004 de `tech/CLAUDE.md`) : l'instance est partagée avec Pip-Boy, dont le modèle est inverse (dégradation gracieuse, cloud faisant autorité au retour réseau). → **Action : ouvrir un document de position dans `tech/docs/`** et réconcilier avec Pip-Boy, sur le modèle de DEC-061.

**Constats techniques à porter dans ce document (pas des décisions) :**
- **Le QNAP n'est pas un hôte crédible pour une instance Supabase self-hosted complète** : une dizaine de conteneurs (postgres, gotrue, postgrest, realtime, storage, imgproxy, kong, meta, studio, analytics/vector) sur un Celeron N3150 / 4 GB qui héberge aussi l'app ZAX. Deux sorties : (a) **stack réduite** `postgres` + `postgrest` + `realtime` + `gotrue`, sans studio/storage/imgproxy/analytics (~1,5–2 GB, non officiel mais courant, couvre exactement l'usage ZAX) ; (b) **changer d'hôte** pour la machine 285K / 64 GB (voir REF-21), le QNAP redevenant la cible de sauvegarde.
- **Option C (cloud nominal + réplica local) — mise en garde forte** : elle crée **deux sources de vérité pendant l'event**, contre DEC-09, et impose une réconciliation bidirectionnelle là où §6.10 exige « aucune perte de message ».
- **Option B (cloud pur) : écartée** — incompatible avec « offline-first pendant l'event, sans exception ».

**⚠️ Fait à intégrer avant d'instruire le débat (relevé le 31/08/2026 dans `tech/INFRA.md`) : l'instance partagée est déjà hébergée en cloud, en production.** Pip-Boy a tranché le cloud (DEC-049), région West EU Ireland, projets `vttidzixtuqplafabjsu` (dev) et `iuwadzfyjrtzymglydjf` (prod), migrations et Edge Functions déployées depuis le 27/07/2026. REF-19 n'est donc **pas un choix sur table rase** mais deux questions distinctes, qu'il faut cesser de confondre :

1. **Migre-t-on une instance partagée déjà vivante vers du self-hosted ?** — avec le coût et le risque que ça représente pour Pip-Boy, qui tourne dessus en production.
2. **Sinon, que fait ZAX pendant une coupure internet le jour du GN ?** — c'est la vraie question, et elle est **plus large que l'hébergement** : cinq terminaux Raspberry Pi face à une instance injoignable. Le fallback DEC-05 (export/import de la BDD orga 1 h avant l'ouverture) suffit-il, ou faut-il un mode hors-ligne de ZAX à part entière ?

C'est exactement la formulation d'**AMB-001** côté `tech/`. Le document de position doit répondre aux deux, dans cet ordre.

⚠️ À noter aussi : `tech/INFRA.md` signale que la **mise en pause automatique** de l'instance par Supabase s'est déjà produite (prod en pause du 20 au 27/07/2026) et **peut se reproduire** — seule une réactivation depuis le dashboard la lève, le CLI n'en est pas capable. Un tel incident pendant les 48–72 h de l'event serait irrécupérable sans intervention manuelle **et sans internet**. Ce risque appartient au débat.

---

### REF-20 — Périmètre des écritures directes de ZAX dans l'instance Supabase partagée

**Source :** DEC-08 / `CLAUDE.md` §1 (« les deux projets lisent/écrivent les mêmes tables, notamment `profiles` ») vs décision côté Pip-Boy (juillet 2026, autoritaire) : ZAX écrit dans le domaine Pip-Boy **exclusivement** via l'Edge Function `zax-write` (transmissions radio + notes uniquement), interdiction appliquée au niveau BDD (rôle Postgres à INSERT restreint).

**Problème :** DEC-08 formule un accès lecture/écriture symétrique aux tables partagées, alors que le contrat Pip-Boy restreint les écritures ZAX vers son domaine à deux types via `zax-write`. Les deux sont conciliables (ZAX n'a probablement jamais besoin d'écrire `profiles`), mais le périmètre exact n'est écrit nulle part : quelles tables partagées ZAX lit-il directement, sur lesquelles a-t-il des grants d'écriture, et la connexion Supabase directe de ZAX est-elle elle aussi restreinte par grants (symétrie du garde-fou) ?

**Impact :** Configuration des rôles/grants Postgres de la connexion ZAX ; garantie que l'exclusivité de `zax-write` est appliquée techniquement et pas seulement documentée ; clarification de DEC-08 lors de la réconciliation ZAX ↔ Pip-Boy.

**Options possibles :**
- Option A : ZAX = lecture seule sur toutes les tables du domaine Pip-Boy (`profiles`, `factions`…) + écriture libre sur ses tables `zax_*` + toute écriture vers le domaine Pip-Boy via `zax-write` — grants configurés en conséquence (favori, cohérent avec la position `docs/position-zax-auth-pipboy.md`)
- Option B : statu quo DEC-08 (écriture directe possible) — contredit le contrat Pip-Boy, à écarter sauf besoin identifié
- Option C : liste explicite table par table (lecture/écriture) annexée au contrat d'API

**Statut :** ✅ Décidé (DEC-24) — **Option A**, par report de **DEC-061** (13/07/2026, postérieur à la rédaction de cette ambiguïté, répond à la question Q8) : rôle ZAX confiné au schéma `zax`, **aucune écriture directe** sur le domaine Pip-Boy, exclusivité de `zax-write` **vérifiée par pgTAP**. **DEC-08 amendée** (l'accès lecture/écriture symétrique est caduc). En **lecture** : **vues dédiées uniquement** (`profiles` réduit à `nfc_uid`, nom, faction, statut vivant/mort, présence + `factions`), jamais de `SELECT` sur les tables brutes. **Un panneau du dashboard admin liste les vues dont ZAX dépend** avec leur état (présente/absente, colonnes attendues vs exposées).

---

### REF-21 — La machine LLM (RTX 5090) part-elle sur le terrain ?

**Source :** session du 31/08/2026 — mise à disposition d'une nouvelle machine (Core Ultra 9 285K 24 c / 64 GB RAM / RTX 5090 32 GB, compute capability 12.0 Blackwell, driver 577.00, 1,5 To libres).

**Problème :** Cette machine remplace-t-elle la tour GTX 1080 du schéma §4 **le jour J**, ou reste-t-elle une machine de développement et de banc, l'event tournant sur la 1080 ? La question n'est pas tranchée.

**Impact :** Structurant à trois niveaux.
- **Choix du modèle (DEC-15)** : enveloppe 24–32B sur la 5090 contre 8–9B en Q4 sur 8 GB de VRAM Pascal. Deux mondes différents, deux réglages de harnais différents.
- **Infra terrain** : ~600–800 W à la prise, refroidissement, onduleur, transport et sécurité physique d'une machine coûteuse sur un event de 48–72 h.
- **REF-19** : si elle est sur site, son 285K / 64 GB écrase le Celeron N3150 / 4 GB du QNAP et devient l'hôte évident de l'app et de l'instance Supabase.

**Options possibles :**
- Option A : elle part sur site et remplace la tour GTX 1080 — à traiter comme une contrainte d'infra (alimentation, refroidissement, onduleur, transport)
- Option B : machine de dev et de banc uniquement ; le jour J tourne sur la GTX 1080 — le modèle doit alors être choisi pour tenir dans 8 GB sur Pascal
- Option C : elle part sur site, la GTX 1080 restant en secours froid avec un modèle plus petit pré-chargé

**Conséquence immédiate, quelle que soit l'issue :** DEC-15 impose de **bencher les deux enveloppes** (un 8–9B et un 24B) tant que la question est ouverte, pour ne pas être bloqué par le choix final de machine.

**Statut :** ⏳ À décider

---

### REF-22 — L'échelle d'attitude de karma ne couvre pas son domaine

**Source :** `CLAUDE.md` §14 (tableau des attitudes) — détecté le 31/08/2026 lors de l'examen de REF-13.

**Problème :** Le tableau des attitudes comporte des **trous** et des **chevauchements** :

| Code | Attitude | Plage | Anomalie |
|---|---|---|---|
| A-TN | Très négative | 0–40 | — |
| A-NG | Négative | 50–90 | **trou 41–49** |
| A-NE | Neutre | 90–130 | **90 appartient aussi à A-NG** |
| A-PO | Positive | 130–180 | **130 appartient aussi à A-NE** |
| A-SU | « Suceur » | 190–200 | **trou 181–189** |

Un karma total de 45 ou de 185 n'a **aucune attitude définie** ; 90 et 130 en ont **deux**. Environ 8 % de l'échelle a un comportement non spécifié.

**Impact :** Le moteur doit choisir une attitude pour **toute** valeur de karma : sans correction, il faudra un comportement par défaut arbitraire, non documenté et invisible à la relecture. Le template de personnalité décrit son comportement **par attitude** (§14) : une attitude indéfinie signifie une personnalité sans consigne.

**Options possibles :**
- Option A : plages contiguës et bornes semi-ouvertes (`[0,45)`, `[45,90)`, `[90,130)`, `[130,180)`, `[180,200]`) — couverture totale, plus aucune ambiguïté de borne
- Option B : conserver les intentions narratives et combler les trous par de nouvelles valeurs choisies par les scénaristes
- Option C : rendre les seuils **paramétrables** (`zax_config`) avec une validation au démarrage refusant toute échelle non couvrante

**Statut :** ⏳ À décider — arbitrage narratif (les bornes traduisent une intention de jeu), mais la **couverture totale** est une exigence technique non négociable

---

### REF-23 — Conflits entre les brainstorms BMAD d'Auriane et les décisions actées

**Source :** confrontation du 31/08/2026 des quatre sessions de brainstorm d'Auriane (merge de la PR #1 `reflexion_bmad_auriane`) aux décisions DEC-10 → DEC-24. Analyse complète : [`docs/position-confrontation-brainstorms-auriane.md`](docs/position-confrontation-brainstorms-auriane.md).

**Problème :** Les brainstorms et les décisions **convergent sur l'architecture de fond** (colonne 1 exacte / colonne 2 embeddings / colonne 3 LLM toujours backup / colonne 4 maths déterministes — même principe que DEC-17, trouvé indépendamment). Six points restent en conflit franc et demandent un arbitrage à deux parties prenantes :

| # | Conflit | Face à | Statut |
|---|---|---|---|
| C1 | `nomic-embed-text` retenu pour A2 et pour le terme `C(p)` du moteur de vote | DEC-21 (embedders anglophones interdits) | ✅ DEC-25 |
| C2 | Segmentation par ponctuation + regroupement en « super-blocs » | DEC-17 (pas de découpage thématique) | ✅ DEC-26 |
| C3 | Pas de filet LLM sur la ligne C (« trop coûteux en temps ») | DEC-17 (étage 3) | ✅ DEC-27 |
| C4 | `l_enfant` au singulier dans les palettes d'ouverture | DEC-19 (deux modules `ENFANT_EXF` / `ENFANT_DES`) | ✅ DEC-28 |
| C5 | Palettes d'ouverture convoquant 6 personnalités hors noyau ; `le_diplomate` est perso par défaut de l'ouverture 2 | DEC-23 (noyau de 8) | ⏳ partiel (✅ DEC-29 sur le plancher défaut ; catalogue hors noyau encore ouvert) |
| C6 | « Le collapse final = la fin » : au signal bleu, une perso gagne et absorbe les autres | DEC-20 (le moteur ne calcule jamais la fin) | ✅ DEC-30 |

**Impact :** **C5 est bloquant** — avec le noyau de 8, l'ouverture 2 n'a plus de personnalité par défaut, donc plus de plancher `O(p)`, et le filet de dernier recours retombe sur `defaut_ultime: le_gardien` : un Gardien omniprésent en ouverture 2, soit l'inverse du ton visé. C1, C2 et C3 sont **actés** (DEC-25, DEC-26, DEC-27, 01/09/2026) : `bge-m3` confirmé pour tout le pipeline y compris `C(p)` (seuils A2/B2/`embedding_seuil` à recalibrer), pas de segmentation thématique en alpha (mesure de longueur de message d'abord), filet LLM réactivé sur la ligne C (la contrainte GPU 1080 qui le bloquait n'existe plus).

**Deux points connexes, hors conflit :**
- **Trou de gouvernance** — le moteur de vote pondéré `S(p) = Wt·T + Wk·K + Wc·C + Wi·I + Wo·O + We·E` avec ses **quatre** gates remplace de fait l'étape 3 de `CLAUDE.md` §15 et **ne figure dans aucun `DEC-XX`**. Conception aboutie, deux YAML écrits et validés. Candidat au prochain passage `docs/` → `DECISIONS.md`. *Le 01/09/2026, le cinquième gate — l'**exclusion de Pauli** (combos de personnalités interdits) — a été **retiré** de `zax_weights.yaml` et du dossier de conception : il n'avait pas été prévu par l'équipe orga et n'entre dans aucune réflexion en cours. Restent : kill-word `DISJ-SPEC`, gate `TIME`, `ORGA-ACTV = 0`, forçage orga.*
- **Sept décisions actées à amender** (DEC-10 par terminal + TTFT hors délai théâtral, DEC-12 colonne de triage, DEC-17 cooldown anti-farm et formule de karma, DEC-18/19 friction à 3 paliers, DEC-22 cue sheet) — détail en §4 du document de position.

**Statut :** ⏳ Partiellement décidé — C1/C2/C3/C4/C6 actés (DEC-25/26/27/28/30) ; C5 partiellement acté (DEC-29 sur le plancher défaut). Reste ouvert : le catalogue des personnalités hors noyau dans les palettes d'ouverture (C5, scénaristique).

**Précision (session du 2026-09-01, ouverture de l'activité d'arbitrage) :** **C5** (perso par défaut de l'ouverture 2 dans le noyau de 8, promotion éventuelle de `le_juge`/`le_soldat_perdu`) est **hors sujet pour Auriane** — c'est une décision **scénaristique** (choix de personnage et de ton), pas une décision d'ingénierie du moteur. Reste ouvert, mais à transmettre à l'équipe scénario plutôt qu'à trancher dans cette session. Ne pas le laisser retomber dans l'oubli : c'est le conflit **bloquant** du document (l'ouverture 2 n'a plus de plancher `O(p)` sans réponse).

**Précision (session du 2026-09-01) :** Sa moitié bloquante (`exclusions_pauli`, un gating *temporaire* incompatible avec l'élimination *permanente* de DEC-19 — origine retrouvée dans `_bmad-output/brainstorming/brainstorm-moteur-vote-pondere-2026-07-09/.memlog.md` ligne 29 : une idée du coach BMAD en session autonome, pas un choix mûri par Auriane à partir de son modèle mental disjoncteur/trigger) a été retirée par Boris le même jour (commit `6da4195`) : le gate n'était prévu par personne côté orga.

**Précision (session du 2026-09-14) :** **C4 est résolu (DEC-28)**. Les palettes d'ouverture de `zax_weights.yaml` citent désormais `enfant_exf`/`enfant_des` (snake_case, alignés sur la convention du fichier) au lieu de l'identifiant unique `l_enfant`, présents en permanence dans la palette de l'ouverture 3 ; le gate `ORGA-ACTV = 0` reste le mécanisme d'élimination permanente (persistant, piloté dashboard, réversible superadmin — conforme DEC-19). `exemple-override-personnalite.yaml` porte désormais deux blocs d'override distincts (un par module) au lieu d'un override partagé.

**Précision (session du 2026-09-14) :** **C6 est résolu (DEC-30)**, confirmé par Auriane **et** Boris. Le collapse final au signal bleu reste mécanique et spectacle (le moteur affiche `S(p)` en continu), mais l'action qui déclenche la fin est **actée à la main par un orga au moment du jeu** — conforme à DEC-20 et au patron DEC-19.

---

### REF-24 — Table de flag ZAX : FK directe vers Pip-Boy vs vues dédiées (contredit DEC-24)

**Source :** conversation Auriane/Boris du 28-29/09/2026, en marge de la PRD MVP Le Gardien (`_bmad-output/planning-artifacts/prds/prd-zax-app-2026-09-28/`).

**Problème :** DEC-24 (REF-20) impose que ZAX n'accède au domaine Pip-Boy en lecture que via des **vues dédiées**, jamais de `SELECT` sur les tables brutes — contrainte posée pour satisfaire un contrat **côté Pip-Boy** (DEC-061, juillet 2026, vérifié par **pgTAP**) garantissant que le schéma Pip-Boy peut évoluer sans casser ZAX. Auriane rapporte une décision prise avec Boris allant dans le sens inverse : les tables `zax_` pourraient porter des **clés étrangères directes vers les tables brutes Pip-Boy** — ex. la future table de flag (`zax_flags_personnage` dans l'addendum du brief MVP) aurait pour **PK `id`** une **FK vers l'id personnage Pip-Boy**. C'est structurellement incompatible avec une vue comme cible : une contrainte FK Postgres ne peut référencer qu'une table portant un index unique/PK, jamais une vue — déjà relevé dans l'addendum du brief MVP comme raison de préférer une colonne simple sans FK.

**Impact :** Si actée telle quelle, cette décision **amende DEC-24** — pas une simple précision technique. Elle réintroduit le couplage direct entre schémas que la vue DEC-24 existait pour éviter (une colonne Pip-Boy qui change de forme, ou une identité de couverture DEC-058, pourrait casser une contrainte FK ZAX). Elle touche potentiellement la garantie **pgTAP** existante (exclusivité de `zax-write`, confinement du rôle ZAX au schéma `zax`) : une FK directe présuppose au minimum un droit de référence sur la table brute Pip-Boy, un accès différent d'un simple `SELECT` via vue. DEC-061 est décrite dans DEC-24 comme un **report côté Pip-Boy** — pas une décision interne ZAX — donc valider ce changement avec Boris seul peut ne pas suffire si l'autorité sur le schéma Pip-Boy est ailleurs.

**Statut :** ⏳ À décider — ne pas éditer DEC-24 en place (convention `DECISIONS.md`). Si confirmée après vérification de son origine et de sa compatibilité avec le contrat Pip-Boy, cette décision devra être actée comme une **nouvelle entrée chronologique** (prochain DEC-XX libre), pas comme une modification de DEC-24. **Action avant d'instruire :** vérifier qui détient l'autorité sur le schéma Pip-Boy (l'équivalent de DEC-061 côté Pip-Boy) et si la levée de la contrainte « vues uniquement » y est réellement actée, avant de faire suivre ce changement dans la PRD ou l'architecture ZAX.

---

*Dernière mise à jour : 2026-09-29 — 24 ambiguïtés recensées (REF-01 à REF-24). ✅ Résolues : REF-01, REF-02, REF-03, REF-04, REF-05, REF-06, REF-07, REF-08, REF-09, REF-10, REF-11, REF-12, REF-13, REF-14, REF-15, REF-16, REF-17, REF-18, REF-20 (19). ⏳ Ouvertes : REF-19 (inter-projets, document de position à ouvrir dans `tech/docs/` — attention, l'instance est **déjà en cloud et en production** côté Pip-Boy, cf. AMB-001), REF-21, REF-22, REF-23 (partiel), REF-24 (5). 24 décisions actées (DEC-01 à DEC-24) dans `DECISIONS.md`.*
