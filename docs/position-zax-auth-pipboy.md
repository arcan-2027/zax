# Position côté ZAX — Authentification & intégration vers le backend Pip-Boy

> Brainstorm du 12/07/2026. Contrepartie du document équivalent côté Pip-Boy (format identique, pour réconciliation).
> Contexte autoritaire côté Pip-Boy (non rediscuté ici) : écritures ZAX → Pip-Boy **exclusivement** via l'Edge Function HTTPS `zax-write` (transmissions RADIO + notes ciblées uniquement, interdiction appliquée au niveau BDD par rôle Postgres restreint) ; secret partagé en header, en variable d'environnement des deux côtés ; contrat d'API documenté et versionné côté Pip-Boy.

---

## 1. Modèle de menace retenu

**Contexte d'exécution côté ZAX.** L'appelant est exclusivement le **backend Node.js/TS de ZAX**, conteneurisé sur le QNAP (Container Station / Docker Compose). Les terminaux joueurs (Raspberry Pi, Chromium) ne parlent jamais à `zax-write` : toute la logique sortante est serveur (cohérent avec la règle §6.7 « permissions côté serveur »). Le secret ne transite donc **jamais** par un poste accessible aux joueurs.

**Acteurs considérés :**
- **Joueurs curieux ou tricheurs** (~180 personnes, 32–36h, WiFi terrain partiel). Non techniques en majorité, mais il suffit d'un smartphone et d'une personne compétente. Vecteurs réalistes : découverte de l'URL de l'Edge Function, écoute du WiFi terrain, manipulation d'un terminal Raspberry Pi.
- **Erreur interne** : doublon d'envoi lors d'un retry, secret qui fuite dans un log ou un commit.
- **Hors périmètre** : adversaire externe motivé (l'event n'a aucune valeur d'attaque), compromission physique du QNAP (local orga), rupture de TLS, menace interne orga (équipe de confiance).

**Actif à protéger.** La capacité d'écrire **en tant que ZAX** — surtout le **broadcast radio** vers 180 joueurs : une injection forgée est l'attaque à plus fort impact narratif. Les dégâts sont toutefois **bornés par construction côté Pip-Boy** : le rôle Postgres de `zax-write` ne peut qu'insérer des transmissions et des notes ; inventaires, profils et quêtes sont hors d'atteinte même avec le secret.

**Chemins réseau.** Deux topologies selon REF-19 (hébergement Supabase non tranché) :
- **Self-hosted QNAP** : l'appel ZAX → `zax-write` reste sur la machine (réseau Docker interne / LAN filaire). Le secret ne traverse jamais les airs.
- **Cloud** : l'appel sort en HTTPS/TLS via la connexion internet du site. Le secret est protégé en transit par TLS ; le risque devient la **disponibilité** (coupure internet pendant le GN), pas la confidentialité.

**Conséquence maximale si le secret fuite** : injection de fausses radios/notes. Désagréable mais réparable — rotation du secret + rattrapage narratif orga (« interférences sur la fréquence »). Aucune donnée personnelle ni mécanique de jeu critique n'est exposée. Le niveau de défense proportionné est donc : **secret robuste + canal chiffré ou local + blast radius borné**, pas de la cryptographie de requête.

---

## 2. Option d'authentification recommandée + options écartées

### Recommandée : secret statique partagé, en header dédié, sur canal TLS ou réseau local

Un secret aléatoire fort (32 octets, base64) présenté à chaque requête dans un **header dédié** (ex. `X-ZAX-Secret` — voir question ouverte Q2 : sur Supabase cloud, `Authorization` est consommé par la plateforme si `verify_jwt` est actif). Comparaison en temps constant côté Pip-Boy.

C'est suffisant parce que : le canal est soit TLS (cloud) soit interne à la machine (self-hosted) ; l'appelant est unique et serveur ; le blast radius est borné en base ; et la fenêtre d'exposition est un event de 36h.

**Le contrat doit rester extensible** : champ/header de schéma d'auth identifiable, pour pouvoir monter vers HMAC plus tard sans casser l'existant si le besoin apparaissait.

### Écartées

- **HMAC par requête + timestamp/nonce anti-rejeu** — écarté. Le rejeu n'apporte rien à un attaquant qui ne peut pas lire le trafic (TLS ou réseau interne) ; et surtout, argument terrain fort : **le QNAP peut passer 36h sans internet donc sans NTP** — une fenêtre temporelle anti-rejeu avec horloges qui dérivent, c'est des rejets légitimes à 3h du matin, indébogables sur site. Coût réel, menace absente.
- **mTLS** — écarté. Gestion de certificats sur QNAP + les Edge Functions Supabase cloud ne terminent pas de mTLS entrant. Surdimensionné.
- **JWT de service signé (clé asymétrique)** — écarté. Même analyse que HMAC : infrastructure de clés et d'horloge (`exp`) pour un gain nul contre les acteurs retenus.
- **Secret en query param** — écarté absolument : fuite dans les logs (Kong, proxys, historique).
- **Réutiliser la `service_role` key Supabase comme credential** — écarté absolument : blast radius total sur l'instance partagée, exactement ce que l'architecture `zax-write` cherche à éviter.

---

## 3. Cycle de vie du secret proposé (vu de ZAX)

1. **Génération** : 32 octets aléatoires (`openssl rand -base64 32`). Échange entre les deux équipes via gestionnaire de mots de passe partagé — jamais par mail/Discord, jamais dans un repo.
2. **Stockage** : fichier `.env` à la racine du déploiement sur le QNAP (dans `.gitignore`), variables `PIPBOY_ZAX_WRITE_URL` + `PIPBOY_ZAX_WRITE_SECRET`, injectées **uniquement dans le conteneur backend Node** via Docker Compose. Jamais dans le frontend, jamais exposé par une API ou le dashboard, jamais loggé (les logs d'envoi tracent l'URL et le statut, pas les headers). Conforme §20 : toutes les clés en variables d'environnement.
3. **Séparation dev/prod** : deux **paires** indissociables (URL + secret) — `.env.development` pointe le projet Supabase dev du Pip-Boy avec son secret dev, `.env` de prod pointe la prod. On ne mélange jamais un secret d'un environnement avec l'URL de l'autre (le secret dev ne doit rien pouvoir écrire en prod).
4. **Accès** : Boris + Auriane uniquement. Les orgas n'en ont pas besoin (ils passent par le dashboard, qui passe par le backend).
5. **Consommation** : lu au démarrage par le service dédié `services/pipboy.ts` (même pattern que `services/llm.ts`, §20).
6. **Rotation sans interruption** : côté ZAX, un seul secret actif à la fois — la fenêtre de recouvrement est portée par le **Pip-Boy qui accepte {ancien, nouveau} pendant la rotation** (exigence, cf. §4). Procédure : Pip-Boy ajoute le nouveau secret → ZAX met à jour `.env` → redémarrage du conteneur backend (< 30 s ; la **file d'attente persistée** garantit zéro perte pendant la coupure, cf. Annexe A) → Pip-Boy retire l'ancien. Pas besoin de hot-reload de secret côté ZAX : le redémarrage est déjà sans perte.
7. **Politique événementielle** : aucune rotation planifiée pendant les 36h. Rotation immédiate (même procédure) uniquement sur suspicion de fuite ; les messages forgés déjà en base se traitent narrativement par les orgas.

---

## 4. Exigences sur le contrat d'API

**Authentification & transport**
- Nom du header porteur du secret **explicite dans le contrat** (recommandation : header dédié, pas `Authorization` — cf. Q2).
- Comparaison en temps constant, et **support de deux secrets valides pendant une fenêtre de rotation**.
- Endpoint joignable depuis le QNAP **dans la topologie du jour J** (dépend de REF-19 — cf. Q1).

**Idempotence (non négociable côté ZAX)**
- Chaque requête porte une clé d'idempotence (`idempotency_key`, UUID v4 générée par ZAX à la **création** du message, pas à l'envoi).
- Le Pip-Boy déduplique dessus (contrainte unique en base) et répond au rejeu par un succès idempotent (200/201 avec le même résultat), pas par une erreur ambiguë. C'est ce qui rend les retries de ZAX sûrs (règle §6.10 « aucune perte de message » ⇒ ZAX renverra systématiquement en cas de doute).

**Payload — transmissions RADIO**
- `alias` d'affichage **libre** (l'identité réelle `sender="ZAX"` reste en base côté Pip-Boy) : ZAX a plusieurs personnalités et des usages narratifs (signature du Gardien, fréquence fantôme, signal en morse textuel).
- Ciblage : `player` / `faction` / `broadcast`, avec identifiants = `profiles.id` et `factions.id` de l'instance Supabase partagée (à confirmer, Q4). Jamais `nfc_uid` comme identifiant de ciblage.
- Corps en texte brut UTF-8 ; **longueur max documentée** (ZAX tronque/segmente en amont plutôt que de découvrir un 400).
- Horodatage d'émission côté ZAX (`sent_at`) accepté ou à défaut ignoré proprement — utile pour l'ordre d'affichage si la file a retardé l'envoi.

**Payload — notes ciblées**
- `player_id` (même référentiel que ci-dessus), `title`, `body` ; format du corps précisé (texte brut vs markdown) et longueurs max.

**Erreurs exploitables par la logique de ZAX**
- Corps d'erreur JSON stable `{ error: { code, message } }` avec codes machine-lisibles énumérés dans le contrat (`UNAUTHORIZED`, `UNKNOWN_PLAYER`, `UNKNOWN_FACTION`, `INVALID_PAYLOAD`, `PAYLOAD_TOO_LARGE`, `RATE_LIMITED`…).
- Distinction claire **permanent vs réessayable** : ZAX classera 400/401/403/404/422 en échec permanent (alerte opérateur, pas de retry) et 408/429/5xx/timeout en réessayable (backoff). Si un code sort de cette convention, le contrat doit le dire.
- Sémantique du succès explicite : 200/201 = **persisté côté Pip-Boy** (pas « délivré au joueur »).

**Latence & limites**
- Ack de persistance attendu < 10 s (timeout client ZAX). Latence de bout en bout (décision ZAX → visibilité écran joueur) : cible < 30 s, acceptable narrativement jusqu'à ~2 min pour une radio — ZAX a besoin de connaître la **fenêtre de propagation PWA** (Q6) pour calibrer la mise en scène (ex. l'orga annonce une transmission).
- Rate limit éventuel documenté : la file de ZAX le respectera en débit de drainage (le moteur ne doit jamais se faire jeter pour du spam qu'il peut lisser lui-même).

**Versionnement**
- Version du contrat identifiable par requête (header ou champ) ; les changements cassants passent par une nouvelle version annoncée — ZAX épingle une version et les deux applis se déploient indépendamment tant que le contrat tient (principe déjà acté côté Pip-Boy).

---

## 5. Questions ouvertes à poser au côté Pip-Boy

1. **Topologie jour J (dépend de REF-19)** : où tourne `zax-write` pendant le GN — Supabase cloud ou self-hosted sur le QNAP ? Si cloud : quel comportement attendu de ZAX pendant une coupure internet (la file de ZAX bufferise, mais combien de temps la PWA tolère-t-elle un silence radio) ? L'endpoint est-il joignable en LAN pur ?
2. **Header du secret** : `verify_jwt` est-il désactivé sur l'Edge Function ? Sinon, `Authorization` est réservé à la plateforme Supabase — confirmer le header dédié retenu.
3. **Double secret en rotation** : la validation accepte-t-elle {ancien, nouveau} pendant une fenêtre de rotation ? (Exigence §3.6.)
4. **Référentiel de ciblage** : confirmez que `player_id` = `profiles.id` et `faction_id` = `factions.id` de l'instance partagée (DEC-08), et le comportement si la cible n'existe pas (`UNKNOWN_PLAYER` ?).
5. **Idempotence** : OK pour `idempotency_key` unique en base + réponse idempotente au rejeu ? Durée de rétention des clés (au moins la durée de l'event) ?
6. **Propagation PWA** : par quel mécanisme (Realtime, polling, à l'ouverture ?) et sous quel délai une transmission persistée devient visible sur le Pip-Boy d'un joueur, WiFi terrain partiel compris ?
7. **Limites** : taille max des corps (radio et note), rate limit, quota ou garde-fou spécifique au broadcast ?
8. **Symétrie du garde-fou BDD** : le rôle Postgres restreint s'applique à `zax-write` — la même logique de grants sera-t-elle appliquée à la connexion Supabase **directe** de ZAX sur l'instance partagée (ZAX sans INSERT sur les tables du domaine Pip-Boy), pour que l'exclusivité de l'Edge Function soit appliquée *contre ZAX aussi* et pas seulement documentée ? (Lié à REF-20, côté ZAX : périmètre exact lecture/écriture de ZAX sur les tables partagées.)
9. **Environnement dev** : existe-t-il un projet Supabase dev Pip-Boy avec `zax-write` déployée et un secret dev distinct, pour les tests d'intégration avant le jour J ?
10. **Catalogue d'erreurs** : demande de la liste versionnée des codes d'erreur avec leur classification permanent/réessayable.

---

## Annexe A — Consommation de l'API côté ZAX (interne ZAX, hors format commun)

**File d'attente sortante persistée (outbox).** Table `zax_pipboy_outbox` dans Supabase (source de vérité unique, DEC-09) : `id`, `idempotency_key`, `kind` (radio/note), `payload` (JSONB), `status` (`pending` / `sent` / `retrying` / `failed_permanent`), `attempts`, `last_error_code`, `next_attempt_at`, `created_at`, `sent_at`. Le moteur de conversation **écrit dans l'outbox et rend la main** ; un worker du backend draine la file. Aucun envoi n'est fait « en ligne » dans le pipeline de réponse au joueur.

- **Retries** : backoff exponentiel avec jitter — 1 s, 5 s, 30 s, 2 min, puis plafond 5 min, **sans abandon automatique** pour les erreurs réessayables (règle §6.10 : aucune perte de message ; un message narratif finit toujours par partir ou par être annulé explicitement par un orga depuis le dashboard).
- **Timeout requête** : 10 s.
- **Erreurs permanentes** (401/403 = secret invalide, 400/422 = payload rejeté) : statut `failed_permanent` + **alerte orga immédiate** — pas de retry aveugle qui masquerait un problème de contrat ou de secret.
- **Doublons** : impossibles par construction — la clé d'idempotence est générée à la création de la ligne d'outbox et réutilisée à chaque tentative ; c'est le Pip-Boy qui déduplique (exigence §4).
- **Cohérence des pannes** : l'outbox vit dans la même instance Supabase que le reste de ZAX. Si Supabase est indisponible, ZAX entier est déjà en mode dégradé — il n'existe pas de mode « ZAX vivant mais outbox morte » ; le cas « Edge Function en panne mais base vivante » est exactement ce que la file couvre.
- **Discipline de contrat** : ZAX ayant un accès direct à l'instance partagée (DEC-08), il serait *techniquement* possible d'insérer directement dans les tables du Pip-Boy. **Interdit** : tout passage vers le domaine Pip-Boy emprunte `zax-write`, et on demande au côté Pip-Boy de le rendre inviolable par grants (Q8).

## Annexe B — Observabilité côté ZAX (interne ZAX, hors format commun)

- **Dashboard admin, panneau « Liaison Pip-Boy »** (s'ajoute aux métriques §10) : statut de la liaison (dernier succès, dernier échec + code), profondeur de la file (`pending`/`retrying`), compteur `failed_permanent`.
- **Alertes** : échec permanent ou file qui gonfle au-delà d'un seuil → alerte orga au même titre que les mots-clés `severity: warn` ; secret rejeté (401/403) → `critical`.
- **Journal** : chaque envoi (tentatives, issue) est traçable via l'outbox — même philosophie que `zax_manual_overrides` et `zax_delivered_files`.
- **Dégradation narrative** : **aucune automatique.** Un échec d'envoi ne doit jamais fuiter vers le joueur (immersion, §11) ; ZAX ne « commente » pas la panne. C'est l'orga, informé par l'alerte, qui décide : attendre le retry silencieux, annuler le message, ou compenser en jeu (prise de main, annonce d'interférences).
