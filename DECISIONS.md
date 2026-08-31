# DECISIONS — Projet ZAX (Fallout 2027)

> Ce fichier est **autorité supérieure** sur tous les autres documents du projet.
> Toute décision ici prime sur `CLAUDE.md`, `AMBIGUITES.md` et tout document de travail.
> Une décision ici clôt l'ambiguïté correspondante dans `AMBIGUITES.md` (mettre statut ✅ Décidé).

---

## Template d'entrée

```
### DEC-XX — [Titre court]

**Date :** JJ/MM/AAAA
**Ambiguïté résolue :** REF-XX (ou "N/A")
**Décision :** [énoncé clair et définitif]
**Justification :** [pourquoi ce choix]
**Impact sur le code :** [ce que ça implique concrètement]
**Décidé par :** [Boris / équipe / etc.]
```

---

## Décisions actées

> Décisions issues de `sources/zax_20260706.md` (réunions des 22 et 29 juin 2026).
> Certaines options y portent encore un « ? » : elles restent en ambiguïté et ne figurent PAS ci-dessous.

### DEC-01 — Système de karma à deux niveaux, par personnalité

**Date :** 22/06/2026
**Ambiguïté résolue :** N/A
**Décision :** ZAX gère un karma **faction ↔ ZAX** et un karma **PJ ↔ ZAX**, tous deux déclinés **par personnalité** et bornés 0–100.
- `karma_faction(id_personnality, id_faction, karma_level 0–100)`
- `karma_pj(id_personnality, id_pj, karma_level 0–100)`
Le karma évolue lorsque l'interlocuteur aborde les sujets/triggers listés dans le template de la personnalité. Toutes les personnalités ne partagent pas le même système (l'Archiviste n'en tient pas compte).
**Justification :** La façon de répondre de ZAX dépend de la relation accumulée avec l'interlocuteur, indépendamment pour chaque personnalité.
**Impact sur le code :** Deux tables de karma ; mise à jour du karma dans le moteur de conversation (étape 2) ; lecture du karma pour compiler le prompt et sélectionner l'attitude.
**Décidé par :** équipe (réunion 22/06/2026)

### DEC-02 — Bandes d'attitude selon le karma total

**Date :** 22/06/2026
**Ambiguïté résolue :** N/A
**Décision :** Le comportement d'une personnalité est modulé par le **karma total = karma faction + karma PJ** (sur 200), réparti en 5 bandes :
| Code | Attitude | Plage |
|---|---|---|
| A-TN | Très négative | 0–40 |
| A-NG | Négative | 50–90 |
| A-NE | Neutre | 90–130 |
| A-PO | Positive | 130–180 |
| A-SU | « Suceur » | 190–200 |
**Justification :** Fournit aux scénaristes un cadre simple pour décrire le comportement par palier dans les templates.
**Impact sur le code :** Fonction de mapping karma_total → code d'attitude ; le template de personnalité contient une entrée de comportement par bande.
**Décidé par :** équipe (réunion 22/06/2026)

### DEC-03 — Modules de personnalité et harnais au format YAML

**Date :** 22/06/2026
**Ambiguïté résolue :** N/A
**Décision :** Les personnalités et le harnais sont décrits en **YAML** externe (un fichier par personnalité), chargés à runtime et modifiables sans redéploiement. Schéma de module : UPID, NAME, ORGN, DECL-HARD, DECL-SOFT, VOIX, TICS, FAVS, MORT, DISJ-SPEC, EXIT, EXEM, PRIO, TIME, FBDN, LOVE, LORE, RLTN, ORGA-ALRT, ORGA-ACTV + bloc `karma:`.
**Justification :** Facile à intégrer dans le code ET lisible/éditable par les scénaristes.
**Impact sur le code :** Loader YAML + validation de schéma ; hot-reload ; édition live via le dashboard (superadmin).
**Décidé par :** équipe (réunion 22/06/2026)

### DEC-04 — Validation des réponses LLM avec orga dans la boucle

**Date :** 22/06/2026
**Ambiguïté résolue :** N/A
**Décision :** Toute réponse LLM passe par : nettoyage (aucun crash LLM ne doit atteindre le joueur) → re-vérification des sujets interdits → **validation orga si un orga est derrière le PC**, sinon **envoi automatique après X secondes**. Un orga peut forcer une personnalité ou une décision à tout moment.
**Justification :** On ne peut pas se permettre un taux de crash LLM visible côté joueur ; l'orga garde la main sur l'immersion.
**Impact sur le code :** File de réponses en attente de validation ; timer d'auto-envoi (X configurable) ; UI de validation/édition dans le dashboard ; mécanisme de forçage.
**Décidé par :** équipe (réunion 22/06/2026)

### DEC-05 — Fallback réseau : export/import de la BDD orga avant ouverture

**Date :** 22/06/2026
**Ambiguïté résolue :** N/A
**Décision :** Si le jour J il n'y a pas de connectivité entre ZAX et le PC orga, on **exporte la BDD orga puis on l'importe dans ZAX 1h avant l'ouverture du vault**. Le lien nominal ZAX ↔ BDD orga peut sinon être un accès direct.
**Justification :** Contrainte terrain : réseau non garanti ; il faut que ZAX dispose des profils/factions même hors ligne.
**Impact sur le code :** Script d'export/import de la BDD orga ; ZAX doit fonctionner sur un snapshot local.
**Décidé par :** équipe (réunion 22/06/2026)

### DEC-06 — Gestion des imposteurs et mode chit-chat

**Date :** 22/06/2026
**Ambiguïté résolue :** N/A
**Décision :** Règles de réaction à l'usurpation d'identité :
- Personnage **inexistant** en mémoire → réponse « Vous n'existez pas » ; si la tentative se répète X fois en Y minutes → déclenchement du **Gardien**.
- Personnage **existant mais absent du GN** : si **décédé** → Gardien + `karma_pj` down + `karma_faction` down ; si **vivant** → réaction « normale » via un **mode chit-chat** (discussion creuse).
- Personnage **présent sur le GN** → discussion normale.
(Option ouverte : entrée « Héro » par personnalité avec questions-pièges → mode défense.)
**Justification :** Empêcher la triche sans casser l'immersion, et alimenter les personnalités de sécurité.
**Impact sur le code :** Lookup identité (existe ? vivant ? présent ?) ; compteur de tentatives ; état « mode chit-chat » ; hooks karma.
**Décidé par :** équipe (réunion 22/06/2026)

### DEC-07 — Condition ferme de don du G.E.C.K.

**Date :** 22/06/2026
**Ambiguïté résolue :** partielle (seuils → REF-13)
**Décision :** Le G.E.C.K. n'est donné qu'à **un personnage présent sur le GN et vivant à l'instant T**. ZAX peut exiger de ne le remettre qu'à un PJ précis (ex. le PJ vivant au meilleur karma-zax de sa faction). Les seuils exacts de karma restent à fixer (REF-13).
**Justification :** Ancrer le don du GECK dans une interaction physique avec un joueur réel présent.
**Impact sur le code :** Vérification présence + vivant avant tout don ; sélection du destinataire par karma.
**Décidé par :** équipe (réunion 22/06/2026)

### DEC-08 — Base de données : Supabase, instance partagée avec Pipboy

**Date :** 06/07/2026
**Ambiguïté résolue :** REF-01, REF-11
**Décision :** La base de données de ZAX est **Supabase** (PostgreSQL + Realtime + Auth), sur la **même instance que l'app Pipboy** (les deux projets lisent/écrivent les mêmes tables, notamment `profiles`). La proposition SQLite est écartée. L'**hébergement** de cette instance (self-hosted sur QNAP vs cloud) n'est PAS tranché → voir REF-19.
**Justification :** Realtime/Auth/RLS natifs, partage des données avec Pipboy sans duplication, cohérence avec l'écosystème existant.
**Impact sur le code :** Client Supabase (pas de moteur SQLite) ; réutilisation des tables Pipboy en lecture ; RLS + Realtime pour le dashboard. Les tables propres à ZAX (`zax_*`, karma) vivent dans la même instance.
**Décidé par :** Boris (06/07/2026)

### DEC-09 — Supabase = source de vérité unique (pas de cache local prioritaire)

**Date :** 06/07/2026
**Ambiguïté résolue :** REF-02
**Décision :** **Tout vient de Supabase.** Supabase est la source de vérité unique ; il n'y a pas de cache local terminal prioritaire. Les terminaux et le moteur ZAX lisent/écrivent dans Supabase.
**Justification :** Cohérence avec Pipboy et simplicité de l'architecture de synchronisation ; une seule source d'état évite les conflits de resynchronisation.
**Impact sur le code :** Pas de couche de cache local faisant autorité ; la résilience réseau se traite au niveau accès Supabase (retries, file d'attente d'écritures), pas par une BDD locale concurrente. À concilier avec la règle « aucune perte de message » (§6.10) et le fallback DEC-05.
**Décidé par :** Boris (06/07/2026)

### DEC-10 — Déclenchement du mode dégradé : seuil sur le time-to-first-token + hystérésis

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-03, REF-10
**Décision :** Le mode dégradé se déclenche sur le **time-to-first-token (TTFT)**, pas sur la durée de la réponse complète. Valeurs (toutes dans `zax_config`) : avertissement dashboard à **3 s**, bascule si **aucun premier token à 8 s**, garde-fou d'abandon de requête à **30 s**. **Hystérésis obligatoire** : bascule effective après **2 échecs consécutifs**, retour en mode normal après **3 succès consécutifs**. Les valeurs `< 5 s` / `10–12 s` de `CLAUDE.md` §11 sont **reclassées en SLO de performance** (métriques dashboard §10) et ne sont plus des déclencheurs.
**Justification :** En streaming avec l'effet machine à écrire (§18), le joueur ne subit pas la latence totale mais le silence avant le premier token — une génération longue est invisible, un écran muet ne l'est pas. Un seuil sur la réponse complète ferait basculer en dégradé des réponses longues parfaitement saines. L'hystérésis évite le flapping dégradé/normal sur des messages consécutifs.
**Impact sur le code :** `services/llm.ts` mesure le TTFT et non la durée totale ; machine à états du mode dégradé avec compteurs d'échecs/succès consécutifs persistés ; trois valeurs de seuil lues dans `zax_config` ; les SLO alimentent les métriques du dashboard sans effet sur la bascule.
**Décidé par :** Boris (31/08/2026)

### DEC-11 — LibreChat écarté de la stack

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-04
**Décision :** LibreChat **n'est pas une brique de ZAX**. Interface joueur et dashboard admin sont **100 % React custom** ; les appels au LLM passent par `services/llm.ts` directement sur l'API Ollama. La mention de LibreChat est retirée de `CLAUDE.md` §5.
**Justification :** Trois incompatibilités structurelles. (1) LibreChat est une UI d'assistant (sélecteur de modèle, regenerate, edit, avatars) alors que ZAX est un personnage — la charte §18 (zéro border-radius, scanlines, vignette CRT, machine à écrire) demanderait de repeindre tout son chrome, à refaire à chaque version. (2) Il streame le LLM directement vers le client : aucun point d'ancrage pour la porte de validation orga §6.6, la prise de main §6.5 ou le mode silence. (3) En production il tire MongoDB + Meilisearch + son RAG API, indéfendable sur un Celeron N3150 / 4 GB.
**Impact sur le code :** Aucune dépendance LibreChat ; le pipeline de conversation §15 et la porte de validation sont écrits côté moteur Node/TS. Un usage de LibreChat comme bac à sable **hors event** pour les scénaristes reste possible mais n'est pas déployé sur l'infra du GN.
**Décidé par :** Boris (31/08/2026)

### DEC-12 — Messages en table normalisée `zax_messages` (JSONB monolithique écarté)

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-05
**Décision :** Les messages sont stockés dans une **table normalisée `zax_messages`** — une ligne par message. Le champ `messages JSONB` monolithique dans `zax_conversations` est **écarté**. Chaque ligne porte, en plus du contenu : statut de validation (`pending` / `validated` / `edited` / `auto_sent`), identité du validateur, indicateur de prise de main orga, personnalité émettrice, latence, tokens, delta de karma appliqué.
**Justification :** (1) Supabase Realtime travaille à la ligne : un `INSERT` = un événement = un message qui apparaît au dashboard, là où un `UPDATE` de blob republierait toute la conversation à chaque client. (2) Un blob monolithique impose un cycle read-modify-write, donc des *lost updates* entre les trois écrivains concurrents (message joueur, réponse ZAX, injection orga) — incompatible avec §6.10 « aucune perte de message ». (3) La porte de validation §6.6 et les statistiques §10 ont besoin d'état **par message**, donc de colonnes indexables. (4) Le volume (quelques milliers de lignes) ne justifie aucune optimisation.
**Impact sur le code :** Amende `CLAUDE.md` §7 ; migration créant `zax_messages` avec FK vers `zax_conversations` ; abonnement Realtime sur l'`INSERT` ; index `tsvector` (configuration française) pour la recherche rétroactive. La détection de mots-clés reste faite **à la volée dans le pipeline Node** (étape 1 de §15), pas par requête SQL, pour que l'alerte orga soit immédiate.
**Décidé par :** Boris (31/08/2026)

### DEC-13 — Identification joueur : identité de terminal + session serveur (pas de compte Auth joueur)

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-06
**Décision :** Les joueurs **n'ont pas de compte Supabase Auth**. Chaque Raspberry Pi porte un **credential de terminal**, provisionné une fois et lié à `zax_terminals`, révocable individuellement. Le joueur badgé est un **état serveur attaché au terminal** — jamais un JWT dans le navigateur. Le timeout d'inactivité (§9.6) est une expiration côté serveur. Les rôles `orga`, `admin` et `superadmin` conservent de **vrais comptes Auth et de vraies RLS**. Les permissions joueur reposent sur le middleware API (§6.7) et sur les grants posés par DEC-061. L'option « `service_role` partout » est **écartée**.
**Justification :** Les terminaux sont des bornes partagées : un refresh token persistant dans le `localStorage` d'un Chromium public crée une fuite d'identité entre joueurs successifs, puisque personne ne cliquera « déconnexion ». S'y ajoute le coût de création et de synchronisation de ~50–80 comptes `auth.users` pour des joueurs qui ne taperont jamais de mot de passe. Les RLS ne sont donc abandonnées que là où elles n'apportaient rien. Un `service_role` unique annulerait les garde-fous de grants posés par DEC-061.
**Impact sur le code :** Table `zax_terminals` étendue au credential de terminal ; état de session serveur (terminal → joueur badgé) avec expiration ; middleware API porteur de toute la logique de permission joueur ; RLS écrites pour les rôles humains uniquement.
**Décidé par :** Boris (31/08/2026)

### DEC-14 — Terminal Superviseur de l'Abri : même interface, autorité différente

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-07
**Décision :** Le Terminal Superviseur utilise **exactement la même interface** que les terminaux joueurs. La différence est un **niveau d'autorité** porté par `zax_terminals.role = superviseur`, qui débride **Le Board** et un jeu de **commandes texte diégétiques** (type `AUTORISATION SUPERVISEUR`). Pas de menu caché, pas d'interface dédiée, pas d'accès au dashboard orga.
**Justification :** `sources/zax_20260706.md` établit que Le Board est « la façade institutionnelle, **activée sur le compte du Superviseur** » (l.381) et que son déclin suit « la perte de pouvoir du superviseur » (l.892) : l'écart est narratif et relève d'un compte, pas d'un écran. Une interface dédiée imposerait un second frontend à styliser et tester sur Raspberry Pi pour un seul terminal ; un accès au dashboard casserait le 4ᵉ mur et contredirait §8. Un menu est de l'UI d'application, la commande tapée est l'idiome d'un CLI Fallout.
**Impact sur le code :** Aucun frontend supplémentaire ; parseur de commandes côté moteur, gardé par le rôle du terminal ; le déblocage du Board passe par la couche « état d'ouverture / personnalité » du harnais (§12).
**Reste ouvert (scénaristes) :** liste exacte des commandes, et surtout si le joueur-superviseur **sait** qu'il pilote ZAX ou s'il est manipulé — la source penche fortement pour la seconde option (« le Superviseur idéal obéit sans savoir qu'il obéit », l.172).
**Décidé par :** Boris (31/08/2026)

### DEC-15 — Modèle LLM : banc comparatif 24–32B, cible Mistral Small 24B

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-08
**Décision :** Le duel « Llama 3 8B vs Mistral 7B » est **périmé** : une nouvelle machine LLM est disponible (Core Ultra 9 285K 24 c / 64 GB RAM / **RTX 5090 32 GB**, compute capability **12.0** Blackwell, driver 577.00). Le modèle est **piloté par `zax_config`** et donc interchangeable jusqu'à la veille de l'event. Un **banc comparatif** tranche entre **Mistral Small 3.x 24B** (favori : Apache 2.0, français de qualité native, ~14 GB en Q4_K_M), **Gemma 3 27B** (~16 GB) et **Qwen2.5/3 32B** (~19 GB) ; les modèles ≥ 70B sont écartés (~34 GB en Q3, ne rentrent pas proprement). Protocole imposé : **harnais complet** (jamais de prompt jouet), mesure du **TTFT** et des tok/s. **Pré-requis bloquant** : vérifier que la version d'Ollama installée embarque des kernels **sm_120**, faute de quoi le banc mesure du CPU sans le dire. Ordre du prompt **invariant → volatile** (noyau → disjoncteurs → état d'ouverture → personnalité → mémoire → N derniers messages) pour que le cache KV de préfixe soit réutilisé.
**Justification :** Le goulot d'étranglement du prompt ZAX est le **prefill** (harnais 6 couches + chunks de lore rejoués à chaque message), et non la génération. Les 32 GB déplacent le point d'équilibre de 8–9B vers 24–32B, et Blackwell supprime la faiblesse de prefill du Pascal. Le swap par configuration ne remplace pas le test mais coûte trois lignes.
**Impact sur le code :** Nom du modèle en configuration, jamais en dur ; compilation du prompt ordonnée de l'invariant au volatile ; scripts de banc mesurant TTFT et tok/s sur harnais réel. Conséquences : DEC-10 conserve son seuil de 8 s, qui ne détecte plus de la lenteur mais une panne ; DEC-21 devient possible puisque le modèle d'embeddings cohabite sur le même GPU.
**Reste ouvert :** REF-21 — la machine part-elle sur le terrain ? Tant que ce n'est pas tranché, **bencher les deux enveloppes** (un 8–9B pour la GTX 1080, un 24B pour la 5090).
**Décidé par :** Boris (31/08/2026)

### DEC-16 — Un seul `DECISIONS.md` par projet

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-09
**Décision :** Un **fichier unique `DECISIONS.md`** par projet, en sections chronologiques `DEC-XX`. La convention `DECISIONS-*.md` est retirée de `CLAUDE.md` §3 comme vestige de la fusion des deux anciens CLAUDE. Les documents de `docs/` (positions, brainstorms) restent la **matière première** d'une décision, jamais une autorité.
**Justification :** Aligné sur l'existant dans les trois projets (`zax-app`, `pipboy-app` — qui tient sans peine à 1 200+ lignes — et `tech/`). Surtout, les décisions structurantes sont **transverses** par nature : DEC-061 touche l'authentification, la base de données et le contrat d'API à la fois, et n'a aucun fichier thématique naturel. Découper créerait deux fichiers d'autorité capables de se contredire sans arbitre.
**Impact sur le code :** Aucun. Convention documentaire.
**Décidé par :** Boris (31/08/2026)

### DEC-17 — Détection des sujets : hybride à trois étages, sans découpage thématique

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-12
**Décision :** Trois étages, et **aucune étape de segmentation en « blocs d'idées »**.
1. **Lexical normalisé** (minuscules, sans accents, tolérance aux fautes par trigrammes) pour les `DECL-HARD` **et pour les `FBDN` / disjoncteurs**. Déterministe, testable, éditable par les scénaristes.
2. **Embeddings** du message comparé à des **phrases-exemples** par sujet (jamais à des mots isolés), avec **règle de marge** (écart entre le sujet n°1 et le n°2) et **classe d'abstention**, pour les `DECL-SOFT`, le karma et les sujets `LOVE`.
3. **Filet LLM** si les deux premiers étages s'abstiennent : appel court à **sortie JSON contrainte** sur une liste de sujets **fermée**.

Découpage : un message de trois phrases ou moins est traité tel quel ; au-delà, découpage **par phrase**.
**Règle dure :** les sujets interdits ne passent **jamais** par les embeddings — détection lexicale, explicite, fail-closed.
**Justification :** Un joueur costumé tapant sur une borne écrit une à trois phrases, pas un paragraphe à segmenter : le « découpage en blocs thématiques » résout un problème qui n'existe pas, ce qui explique le blocage depuis la réunion du 22/06. Les chaînes de Markov sont **écartées** comme erreur de catégorie : elles modélisent une séquence de tokens, pas une appartenance thématique. Le seuil de 0,7 n'est pas portable d'un modèle d'embeddings à l'autre et doit être **calibré** sur de vrais messages ZAX. Enfin, le karma peut être flou sans conséquence visible, mais une censure §6.6 qui déclenche à 0,68 et rate à 0,71 est intestable et inexplicable à un orga en pleine nuit.
**Impact sur le code :** Trois détecteurs distincts et testables séparément ; jeu de messages réels pour calibrer le seuil et la marge ; l'étage 3 n'existe que grâce à la marge de calcul de DEC-15 (~200–400 ms).
**Décidé par :** Boris (31/08/2026)

### DEC-18 — Don du G.E.C.K. : bloc `geck:` par personnalité, défaut héritable, confirmation orga

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-13
**Décision :** La condition d'obtention est décrite par un **bloc `geck:` optionnel** dans le module YAML de chaque personnalité, avec une **règle par défaut héritée** quand il est absent. Le moteur **propose** (« conditions réunies pour ZZZ ») ; **un orga confirme** — le don ne se déclenche jamais tout seul. Les seuils numériques relèvent de l'équilibrage narratif et sont déférés aux scénaristes.
**Justification :** DEC-07 décrit déjà un **classement relatif** (« le PJ vivant au meilleur karma de la faction »), pas un seuil absolu, et la source précise que chaque personnalité a sa propre façon de donner le GECK, l'Archiviste n'en tenant même pas compte : un seuil global unique contredirait les deux. Le défaut héritable évite des YAML inachevés la veille de l'event. Enfin le don du GECK est le climax du GN : le laisser à un franchissement de seuil à 4 h du matin contredit « l'app informe, elle n'arbitre pas » et §6.11.
**Impact sur le code :** Bloc `geck:` dans le schéma de personnalité + règle par défaut ; le moteur produit une **proposition** avec son justificatif, l'action de don est une action orga du dashboard. Rappel de cadrage : le karma n'est qu'une des trois portes — §16 exige aussi le laser réparé **et** les 6 modules de données.
**Décidé par :** Boris (31/08/2026)

### DEC-19 — L'Enfant : deux modules plats en exclusion mutuelle, bascule actée par un orga

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-14
**Décision :** `ENFANT_EXF` (Exfiltration, « Charlie ») et `ENFANT_DES` (Destruction) sont **deux modules de personnalité ordinaires et complets**, en **exclusion mutuelle** portée par l'état d'ouverture. **Aucune notion de sous-personnalité n'est introduite** dans le schéma. L'élimination de l'une des deux est **actée par un orga** à la fin de l'ouverture 1, sur présentation d'un **score d'opinion** au dashboard ; l'état est écrit dans `zax_config` et **réversible par un superadmin**.
**Justification :** Les deux Enfants ont des `DECL`, `FBDN`, `LOVE` et des fins **opposés** : ce sont deux personnalités, pas deux facettes. Introduire un niveau d'imbrication obligerait toute la chaîne (sélection, karma, compilation de prompt, chargement YAML, forçage dashboard) à gérer un arbre à deux étages pour un seul cas sur quinze. La source demande explicitement de pouvoir « forcer une Perso / une décision […] plutôt que d'attendre que la tech le fasse d'elle-même » (l.637). La réversibilité protège d'une erreur de clic en pleine nuit.
**Impact sur le code :** **Le score d'opinion est une mécanique générique**, partagée avec « Cash vs Kings » (la source les cite dans la même respiration, l.739-740) : une seule feature, deux usages — ne pas l'écrire deux fois. Champ d'exclusion mutuelle dans l'état d'ouverture ; action d'élimination et action d'annulation dans le dashboard.
**Reste ouvert (scénaristes) :** que faire si l'ouverture 1 ne dégage aucune tendance (égalité, trop peu d'échanges) ; et si l'Enfant éliminé disparaît totalement ou peut resurgir en « fantôme ».
**Décidé par :** Boris (31/08/2026)

### DEC-20 — Signal bleu : champ `FIN` par personnalité + flag global, l'app n'arbitre pas

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-15
**Décision :** Un champ **`FIN`** est ajouté au module de personnalité (objectif et rhétorique de la fin qu'elle pousse), et un **flag global `signal_bleu`** se **superpose** à la couche « état d'ouverture » du harnais (§12) sans la remplacer. Quand le flag est levé, le champ `FIN` de la personnalité active entre dans le prompt comme objectif. **Le moteur ne calcule jamais la fin gagnante** : ZAX plaide, les humains tranchent ; le dashboard peut afficher qui plaide quoi, il ne conclut pas.
**Justification :** Le signal bleu est orthogonal aux trois ouvertures — il peut s'allumer pendant l'ouverture 3 sans s'y substituer — donc un quatrième état d'ouverture serait inexact. La fin poussée est un attribut de personnalité, donc un champ, pas du code. Et un score désignant « la fin gagnante » violerait « l'app informe, elle n'arbitre pas » tout en retirant aux orgas le climax de leur propre GN.
**Impact sur le code :** Champ `FIN` au schéma YAML ; flag `signal_bleu` en configuration runtime ; aucune logique de fin dans le moteur.
**Dépendance :** le **contenu** des huit fins reste gelé jusqu'à DEC-23 — cinq d'entre elles appartiennent à des personnalités hors noyau.
**Décidé par :** Boris (31/08/2026)

### DEC-21 — Base de connaissance : pgvector + embedder multilingue + recherche hybride

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-16
**Décision :**
- **Découpage par structure Markdown** (titre de section, report du titre parent dans le chunk, plafond de taille, léger chevauchement). Le « chunking en idées » est **écarté**.
- **Embedder multilingue local** : **`bge-m3`** (français excellent, contexte 8k, disponible sous Ollama), `multilingual-e5-large` en second choix. **Interdiction explicite** des embedders anglophones (`nomic-embed-text`, `all-MiniLM`, `mxbai-embed-large`).
- **Stockage `pgvector` dans le schéma `zax`** de l'instance partagée. Base vectorielle dédiée (Qdrant, Chroma) **écartée**.
- **Métadonnées obligatoires par chunk** : `source`, `module` de lore, `visibilité` (personnalités autorisées), `type` (lore pré-établi vs savoir d'observation), `added_at`. Le filtre de visibilité s'applique **dans** la requête vectorielle, jamais après.
- **Une seule table avec colonne `type`** pour le lore et le savoir d'observation.
- **Recherche hybride** vecteur + plein texte (`tsvector`, configuration française).

**Justification :** Le lore est déjà structuré en Markdown (`lore/lore.md` a une section par scénario) : découper par titre est déterministe et rejouable après chaque modification scénariste. Un embedder anglophone dégrade la recherche en français **sans qu'aucun test synthétique ne le révèle**. Une base vectorielle dédiée serait un service de plus à surveiller 72 h et une seconde source de vérité, contre DEC-09. Filtrer après la recherche renverrait dix chunks pour n'en garder que deux, donc une réponse appauvrie. Une table unique permet d'insérer une observation « caméra de surface » en direct sans réindexation. Enfin la recherche vectorielle seule est mauvaise sur les noms propres — or les joueurs taperont « Ashville », « Vault 42-R », « Carver Ashton », « Conseil des Masques » littéralement.
**Impact sur le code :** Extension `pgvector` ; pipeline d'ingestion Markdown → chunks + métadonnées, rejouable ; service d'embeddings appelant le modèle local (cohabitation GPU rendue possible par DEC-15) ; requête hybride avec filtre de visibilité par personnalité (§17).
**Décidé par :** Boris (31/08/2026)

### DEC-22 — État d'ouverture : bascule manuelle, planning en rappel seulement

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-17
**Décision :** Seul un orga fait basculer l'état d'ouverture, depuis le dashboard. Un planning optionnel **affiche un rappel** à l'heure prévue (« ouverture 2 prévue à 14h00 — passer maintenant ? ») et **n'actionne jamais** la bascule. L'état est **persisté dans `zax_config`**.
**Justification :** Les trois ouvertures sont des événements physiques (la porte du vault s'ouvre) que l'app n'a aucun moyen de connaître, et aucun GN ne tient son horaire — une bascule programmée dérive dès la première demi-journée. Une bascule sur progression narrative ferait arbitrer l'app. Enfin le contrôle manuel existe de toute façon (§6.11) : une automatisation réelle serait un second mécanisme capable de contredire le premier — un orga passe en ouverture 2, le planificateur le repasse en 1 à minuit, et le bug se débugge en jeu.
**Impact sur le code :** Action de bascule dans le dashboard ; état persisté (un redémarrage de conteneur ne doit pas revenir en ouverture 1) ; planning purement informatif, sans effet de bord.
**Décidé par :** Boris (31/08/2026)

### DEC-23 — Personnalités : noyau de 8, réserve priorisée

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-18
**Décision :** **Noyau de 8 modules** : Le Gardien · Le Board · L'Archiviste · Le Scientifique · Enfant EXF · Enfant DES (les six dont le module est **déjà rédigé** dans la source) + **Happiness Officer** et **Mood Manager** (déclarées actives, modules à écrire). **Réserve** : aucun module supplémentaire n'est écrit avant que le noyau soit **terminé et testé** ; **Le Juge** et **Le Soldat Perdu** sont prioritaires dans la réserve car DEC-20 fait dépendre deux fins d'eux. Le classement « actives / secondaires à valider » de `CLAUDE.md` §13 est **remplacé** par cette liste.
**Justification :** Le classement existant est contredit par le travail réel : L'Archiviste, Le Scientifique et les deux Enfants sont classés « secondaires à valider » mais leurs modules **sont écrits**, tandis que Happiness Officer et Mood Manager sont classées « actives » sans module. C'est le travail effectif qui fixe la liste. Sur le volume : le coût d'une personnalité n'est pas son YAML mais la fiche + son karma propre (§14) + le filtrage lore (§17) + le graphe `EXIT`/`PRIO`/`TIME` + les tests ; et avec 5 terminaux sur 32–36 h, un joueur en rencontrera trois à cinq. À seize modules, la majorité ne serait jamais vue et les autres seraient sous-testées — inacceptable au regard de §6.9 (« le LLM ne doit jamais être imprévisible »).
**Impact sur le code :** Périmètre de test borné à huit personnalités ; §13 de `CLAUDE.md` réécrit.
**Décidé par :** Boris (31/08/2026)

### DEC-24 — Périmètre d'accès de ZAX à l'instance partagée : lecture via vues dédiées

**Date :** 31/08/2026
**Ambiguïté résolue :** REF-20
**Décision :** Report et confirmation de **DEC-061** côté Pip-Boy (13/07/2026, postérieur à la rédaction de REF-20) : le rôle applicatif ZAX est **confiné au schéma `zax`**, n'a **aucun droit d'écriture sur le domaine Pip-Boy**, et l'exclusivité de l'Edge Function `zax-write` est un **fait de base vérifié par pgTAP**. **DEC-08 est amendée** : sa formulation d'un accès lecture/écriture symétrique aux tables partagées est caduque.

En **lecture**, ZAX n'accède au domaine Pip-Boy que par des **vues dédiées**, exposées colonne par colonne — jamais de `SELECT` sur les tables brutes. Surface nécessaire : `profiles` (`nfc_uid`, nom, faction, statut vivant/mort, présence sur le GN) et `factions`. Rien d'autre : ni inventaire, ni transmissions, ni notes, ni carte.

Le **dashboard admin expose un panneau listant les vues dont ZAX dépend**, avec pour chacune son état : présente ou absente, colonnes attendues contre colonnes réellement exposées.
**Justification :** DEC-061 répond déjà mot pour mot à la question Q8 de `docs/position-zax-auth-pipboy.md` ; il restait à l'enregistrer côté ZAX. Les vues sont retenues parce que le schéma Pip-Boy évolue sans ZAX (DEC-058 introduit des identités de couverture) : une vue est une liste blanche, donc une colonne sensible arrivant plus tard ne peut pas fuiter dans une réponse narrative. Le panneau de contrôle est exigé parce qu'une migration Pip-Boy qui renomme une colonne casserait ZAX **en silence** — le défaut doit devenir visible avant l'ouverture du vault, pas se découvrir sur un terminal en jeu.
**Impact sur le code :** Création des vues `zax.v_*` et grants correspondants ; aucun `SELECT` direct sur les tables Pip-Boy dans le code ZAX ; panneau de vérification du contrat de lecture dans le dashboard admin ; §1 et §7 de `CLAUDE.md` et DEC-08 mis à jour.
**Décidé par :** Boris (31/08/2026)

---

## Règles d'utilisation

- Toute décision est numérotée séquentiellement (DEC-01, DEC-02…)
- Une décision ne se modifie pas : on en crée une nouvelle qui annule la précédente
- Référencer la décision dans les commits : `feat: implémentation DEC-03`
- Quand une décision résout une ambiguïté REF-XX, mettre son statut à ✅ Décidé dans `AMBIGUITES.md`
- Les décisions s'appliquent immédiatement — Claude les respecte sans demande de confirmation
