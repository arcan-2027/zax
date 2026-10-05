---
title: "PRD — ZAX MVP mono-personnalité (Le Gardien)"
status: draft
created: 2026-09-28
updated: 2026-09-28
---

# PRD : ZAX — MVP mono-personnalité (Le Gardien)

## 0. Objet du document

Cette PRD cadre le développement du plancher MVP ZAX décrit dans `brief-zax-app-2026-09-21` (brief.md + addendum.md) : une seule personnalité (Le Gardien), un flag autorisé/interdit, un écran orga de supervision. Elle s'adresse à Auriane et Boris pour le cadrage produit, et sert d'entrée à l'architecture et aux epics/stories qui suivent. Vocabulaire ancré sur le Glossaire (§3) ; fonctionnalités groupées avec exigences fonctionnelles (FR) imbriquées et numérotées globalement ; hypothèses tagguées `[ASSUMPTION]` et indexées en §9.

Le brief et son addendum restent la référence pour le *pourquoi* (contexte, risques, JTBD) ; cette PRD ne les duplique pas, elle les traduit en capacités testables. Le détail technique (nommage de tables, mécanisme FK) reste dans l'addendum du brief et migrera vers l'architecture.

## 1. Vision

ZAX, à terme, est un système à 8 personnalités, karma à deux niveaux et RAG lore — pensé pour tenir un GN de 36 à 72h sans supervision constante. Ce plancher construit et éprouve, sur une seule personnalité choisie pour sa simplicité narrative (Le Gardien : binaire, rigide, peu de nuances), le mécanisme central que toutes les personnalités partageront : une réponse générée par LLM, conditionnée par un signal métier simple (le flag autorisé/interdit), supervisée en temps réel par un orga qui peut laisser faire ou intervenir.

Ce n'est pas un ZAX diminué — c'est la fondation sur laquelle les 7 autres personnalités, le karma et le RAG se poseront comme des ajouts de modules plutôt que comme une refonte. Si le Gardien répond juste et que l'orga garde le contrôle sans se noyer, le pari architectural est validé.

## 2. Utilisateurs cibles

### 2.1 Jobs To Be Done

- **L'orga en poste pendant le GN** (Boris et les autres). *Quand le GN tourne sur plusieurs terminaux en simultané, je veux surveiller les conversations et reprendre la main sans lire chaque échange en entier, pour garder le contrôle narratif sans devenir le goulot d'étranglement.* Succès : jamais surpris par un message parti tout seul qu'il n'aurait pas laissé passer.
- **Le joueur au terminal** (indirect — il ne voit jamais le MVP en tant que tel, seulement Le Gardien). *Quand je me présente à un terminal, je veux que ZAX me traite de façon cohérente avec qui je suis, pour que l'immersion tienne même en figurant.*
- **Auriane et Boris, côté produit**. *Avant d'industrialiser 8 personnalités + karma + RAG, je veux valider le plancher technique et narratif sur un seul module, pour ne pas construire une architecture qui ne tient pas au premier test réel.*

### 2.2 Non-utilisateurs (v1)

- Les autres rôles humains (admin, superadmin) ne sont pas des utilisateurs distincts de ce MVP : un seul rôle orga est modélisé (§5, non-goal).
- Les joueurs n'interagissent avec aucune interface de gestion — ils ne voient que Le Gardien.

### 2.3 Parcours utilisateurs clés

- **UJ-1. Boris surveille trois conversations en simultané sans se noyer.**
  - **Persona + contexte :** Boris, orga en poste devant le dashboard pendant une session de test, plusieurs joueurs se présentent à des terminaux différents à quelques minutes d'intervalle.
  - **État d'entrée :** connecté à l'écran orga conversations, aucune conversation active pour l'instant.
  - **Parcours :** un joueur se présente à un terminal → une conversation apparaît dans la liste avec la réponse que ZAX propose d'envoyer et les thèmes détectés au-dessus du seuil → Boris scanne rapidement la réponse et les thèmes plutôt que de lire tout l'échange → pour une conversation, rien à signaler, il laisse filer → pour une autre, un thème sensible apparaît, il ouvre la conversation en détail et intervient avant l'auto-envoi (réalise FR-6, FR-7).
  - **Climax :** le message est envoyé — soit automatiquement après le délai configuré (FR-6), soit après l'intervention de Boris (FR-7) — et Boris sait dans les deux cas ce qui a été envoyé et pourquoi.
  - **Résolution :** Boris reste concentré sur l'écran conversations sans naviguer ailleurs ; aucun message n'est parti sans qu'il ait eu la visibilité pour intervenir s'il l'avait voulu.
  - **Cas limite :** si le LLM ne répond pas à temps, le mode dégradé bascule et propose une réponse pré-programmée à la place — Boris voit l'indicateur mode dégradé plutôt qu'un plantage (FR-10).

- **UJ-2. Un joueur teste Le Gardien.** Un joueur (autorisé ou interdit selon son personnage) se présente à un terminal, écrit un message, et reçoit une réponse du Gardien cohérente avec son statut et le lore — sans jamais savoir qu'un orga a pu arbitrer ou qu'un flag existe en base.

## 3. Glossaire

- **ZAX** — l'IA du Vault 42 simulée par le système ; dans ce MVP, une seule de ses personnalités est active.
- **Le Gardien** — la personnalité active de ce MVP. Comportement rigide, binaire (menace/ressource), peu de nuance narrative. Fiche de référence : `sources/zax_20260706.md`.
- **Flag autorisé/interdit** — statut binaire posé à la main par personnage, qui conditionne le comportement du Gardien envers ce personnage. Positionné en base pour ce MVP (pas d'UI de gestion — voir §6.2).
- **Personnage** — entité du domaine Pip-Boy (PJ/PNJ), identifiée par son `nfc_uid` et lue par ZAX via une vue dédiée (jamais la table brute).
- **Conversation** — échange en cours entre un personnage identifié à un terminal et Le Gardien.
- **Réponse proposée** — réponse générée par le pipeline LLM pour une conversation, affichée à l'orga avant envoi effectif.
- **Auto-envoi** — envoi automatique de la réponse proposée après un délai configurable sans intervention orga.
- **Prise de main orga** — intervention d'un orga qui édite ou remplace la réponse proposée ; le résultat est envoyé comme si c'était Le Gardien qui répondait, sans que le joueur puisse le distinguer.
- **Thème détecté** — sujet ou mot-clé identifié dans la réponse proposée, affiché à l'orga quand son score dépasse un seuil configuré.
- **Mode dégradé** — bascule automatique (seuils TTFT) ou manuelle vers des réponses pré-programmées quand le pipeline LLM ne répond pas à temps.
- **Orga** — le rôle humain unique modélisé dans ce MVP (voir §5, pas de distinction admin/superadmin).

## 4. Fonctionnalités

### 4.1 Génération de réponse du Gardien

**Description :** Chaque message d'un personnage identifié déclenche une génération de réponse via le pipeline LLM (Ollama), construite à partir du harnais du Gardien. Si le LLM ne répond pas dans les seuils définis, le système bascule en mode dégradé et sert une réponse pré-programmée à la place — zéro crash LLM ne doit atteindre l'orga ou le joueur. Réalise UJ-1, UJ-2.

**Exigences fonctionnelles :**

#### FR-1 : Génération de réponse via LLM
Le système génère une réponse candidate pour chaque message reçu, en appliquant le harnais du Gardien (voix, tics, sujets interdits) chargé depuis sa configuration YAML.

**Conséquences (testables) :**
- La réponse générée respecte les sujets interdits (`FBDN`) du module Gardien — aucune réponse ne les traverse, y compris en cas de reformulation du joueur.
- Une nouvelle réponse est générée pour chaque message reçu dans une conversation active.

#### FR-2 : Bascule en mode dégradé
Le système surveille le temps avant premier token (TTFT) et bascule vers des réponses pré-programmées catégorisées (identité, accès, erreur…) selon les seuils de `zax_config` (avertissement 3s, bascule si aucun premier token à 8s, abandon à 30s, hystérésis 2 échecs / 3 succès — hérité de CLAUDE.md §11, DEC-10).

**Conséquences (testables) :**
- Aucune requête ne reste sans réponse au-delà du seuil d'abandon (30s) — une réponse pré-programmée est toujours servie.
- Le dashboard orga affiche un indicateur "MODE DÉGRADÉ" actif quand la bascule a eu lieu.
- Le retour en mode normal ne se fait qu'après 3 succès consécutifs (anti-oscillation).

**Hors périmètre :** la calibration fine des seuils pour d'autres personnalités — ces valeurs sont déjà actées (DEC-10) et communes à tout ZAX.

### 4.2 Comportement conditionné par le flag autorisé/interdit

**Description :** Le Gardien adapte sa réponse selon que le personnage qui lui parle est marqué autorisé ou interdit. Le flag est posé à la main en base pour ce MVP (pas d'UI de gestion, voir §6.2). Réalise UJ-2.

**Exigences fonctionnelles :**

#### FR-3 : Lecture du flag au moment de la génération
Avant de générer une réponse, le système récupère le flag autorisé/interdit associé au `nfc_uid` du personnage identifié et l'injecte dans le prompt du Gardien.

**Conséquences (testables) :**
- Un même message envoyé par un personnage autorisé et par un personnage interdit produit des réponses différentes et cohérentes avec le statut.
- Le flag est stocké dans une table dédiée du schéma `zax`, référencée par l'identifiant personnage du domaine Pip-Boy. **[NOTE FOR PM] Le mécanisme exact de cette référence (colonne simple sans contrainte, vue dédiée, ou FK directe vers la table Pip-Boy) n'est plus tranché** : une divergence est apparue entre l'addendum du brief (colonne simple, pas de FK, conforme à DEC-24) et une décision Auriane/Boris plus récente (FK directe). Documenté comme conflit à arbitrer avant l'architecture — voir `AMBIGUITES.md` REF-24.

#### FR-4 : Comportement par défaut si le flag est absent
Si le `nfc_uid` reçu n'a pas de flag posé en base (personnage jamais configuré, ou inconnu côté Pip-Boy au moment de l'identification), Le Gardien applique un défaut **interdit** — jamais un défaut neutre ou une absence de réponse.

**Conséquences (testables) :**
- Un personnage sans flag reçoit le même traitement qu'un personnage explicitement marqué interdit.
- Le dashboard orga affiche un avertissement visible et proéminent (pas une ligne discrète dans un log) chaque fois qu'une conversation tombe sur ce cas — l'orga doit savoir immédiatement qu'un flag manque, pas le découvrir a posteriori.

### 4.3 Supervision orga : conversations, validation, auto-envoi

**Description :** Un écran unique permet à l'orga de voir toutes les conversations en cours, la réponse que Le Gardien propose d'envoyer pour chacune, les thèmes détectés, et de laisser l'auto-envoi agir ou d'intervenir. Réalise UJ-1.

**Exigences fonctionnelles :**

#### FR-5 : Liste des conversations en cours
L'orga voit, sur un écran unique, la liste des conversations actives avec, pour chacune, le nom du personnage (résolu via la vue dédiée Pip-Boy, DEC-24) et la réponse proposée.

**Conséquences (testables) :**
- L'orga n'a besoin de naviguer sur aucun autre écran pour voir l'ensemble des conversations actives et leurs réponses proposées (critère de succès du brief).
- L'écran prévoit 5 emplacements de conversation simultanés, alignés sur les 5 terminaux prévus à terme — pas une liste à un seul emplacement, même si ce MVP tourne en environnement dev/test (§6.1).

#### FR-6 : Auto-envoi paramétrable
Si aucune intervention orga n'a lieu dans le délai configuré après la génération d'une réponse proposée, celle-ci est envoyée automatiquement.

**Conséquences (testables) :**
- Le délai est lu depuis un fichier de configuration du projet, jamais codé en dur ; valeur par défaut 5 secondes.
- Modifier le délai ne nécessite aucun redéploiement de l'application.

#### FR-7 : Intervention orga avant envoi
L'orga peut valider, éditer, ou remplacer (prise de main) la réponse proposée avant l'auto-envoi. Une prise de main est envoyée et affichée côté joueur comme si Le Gardien avait répondu directement.

**Conséquences (testables) :**
- Toute intervention orga (validation, édition, prise de main) annule le décompte d'auto-envoi en cours pour cette conversation.
- Une prise de main est loguée en base avec l'identité de l'orga et l'horodatage (CLAUDE.md §6.5).
- Côté interface joueur, rien ne distingue une prise de main d'une réponse générée par le Gardien.

**NFR spécifiques à cette fonctionnalité :**
- Aucune perte de message : une réponse en attente de validation ou d'auto-envoi doit survivre à un rafraîchissement de l'écran orga ou à une reconnexion (CLAUDE.md §6.10).

### 4.4 Détection de thèmes/mots-clés

**Description :** Chaque réponse proposée est comparée par embedding à une liste de thèmes définie à la main, pour aider l'orga à scanner rapidement plutôt qu'à tout lire. Réalise UJ-1.

**Exigences fonctionnelles :**

#### FR-8 : Score de thèmes sur la réponse proposée
Le système calcule un score d'embedding entre la réponse proposée et chaque thème d'une liste configurée, et affiche à l'orga les thèmes dont le score dépasse un seuil configuré.

**Conséquences (testables) :**
- La liste de thèmes et leurs seuils sont définis dans un fichier de configuration du projet, distinct du harnais YAML des personnalités — modifiable sans redéploiement.
- Les seuils ne sont pas fixés dans cette PRD : ils sont définis empiriquement pendant les sessions de test décrites en §7, comme confirmé par Auriane.
- Un thème affiché à l'orga est visuellement associé à la conversation et à la réponse concernées (pas un flux séparé à corréler manuellement).

## 5. Non-objectifs explicites

- **Pas de karma, pas de disjoncteurs élaborés, pas de RAG lore, pas des 7 autres personnalités du noyau** — hors scope MVP, prévus en itérations suivantes une fois ce plancher validé (brief §"Hors scope MVP").
- **Pas de mécanisme d'erratum/correction rétroactive** — une contradiction narrative après un message déjà envoyé est gérée nativement par le LLM en diégèse ; ce n'est pas un bug à corriger par du code (principe directeur du brief).
- **Pas de soupape de secours automatique** — même dans le pire scénario de blocage narratif, le moteur ne s'assouplit jamais de lui-même ; seul le forçage/la prise de main orga débloque (CLAUDE.md §6.11).
- **Pas de hiérarchie de rôles** — un seul rôle orga est modélisé ; admin/superadmin (gestion de comptes, édition live du harnais) restent hors MVP.
- **Pas de déploiement en conditions terrain pour cette itération** — Raspberry Pi, réseau filaire dédié, contraintes GN (§19 CLAUDE.md) sont un MVP suivant ; celui-ci tourne en environnement de dev/test.
- **Pas d'UI de gestion du flag** — le flag autorisé/interdit est posé directement en base pour ce MVP ; une mini-interface (checkbox par personnage) est un Should reporté (brief).
- **Pas de mémoire multi-session** — si un joueur se ré-identifie sur une session distincte, l'historique n'est pas repêché dans ce MVP (Should reporté).

## 6. Périmètre MVP

### 6.1 Dans le périmètre

- Le Gardien répond via pipeline LLM réel (Ollama), avec bascule mode dégradé (FR-1, FR-2).
- Flag autorisé/interdit par personnage, posé en base, conditionnant la réponse (FR-3, FR-4).
- Écran orga unique : conversations en cours, réponses proposées, thèmes détectés, auto-envoi, intervention (FR-5 à FR-8).
- Un seul rôle orga, pas de hiérarchie de permissions.
- Environnement de dev/test (navigateur standard) — pas encore les contraintes matérielles/réseau du GN.

### 6.2 Hors périmètre MVP

- Mini-interface orga de gestion du flag (checkbox par personnage) — Should, itération suivante.
- Mémoire multi-session (historique repêché à la ré-identification) — Should, itération suivante.
- Karma, disjoncteurs élaborés, RAG lore, 7 autres personnalités — itérations ultérieures (voir §5).
- Déploiement en conditions terrain (Raspberry Pi, réseau filaire GN) — itération suivante, une fois le plancher fonctionnel validé en dev.
- Rôles admin/superadmin, gestion de comptes, édition live du harnais.

## 7. Critères de succès

**Primaire**
- **SM-1** : Le Gardien répond de façon cohérente avec le lore et avec le flag autorisé/interdit du personnage, vérifié par une succession de sessions de test simulant des conditions réelles (personnage autorisé/interdit, thèmes variés). Valide FR-1, FR-3, FR-4.
- **SM-2** : Un orga peut surveiller les conversations en cours et arbitrer (laisser l'auto-envoi ou intervenir) sans naviguer en dehors de l'écran conversations. Valide FR-5, FR-6, FR-7, FR-8.

**Secondaire**
- **SM-3** : Aucun crash LLM n'atteint l'orga ou le joueur — toute défaillance du LLM se traduit par une bascule en mode dégradé visible, jamais par une erreur brute. Valide FR-2.

**Contre-métriques (à ne pas optimiser)**
- **SM-C1** : Le taux d'auto-envoi ne doit pas être optimisé à la hausse en réduisant le délai ou la sensibilité des thèmes détectés — le moteur expose des signaux, il ne doit jamais pousser vers moins d'intervention orga au nom de la fluidité (principe directeur du brief, contrebalance SM-2).

**Forme du test** : pas un test binaire go/no-go, mais des itérations de sessions de test jusqu'à ce que le comportement du Gardien et l'écran orga conviennent aux orgas (Boris en particulier).

## 8. Questions ouvertes

*Chaque question ci-dessous est reportée à l'étape suivante du flux BMAD (`bmad-architecture`), qui prend cette PRD en entrée.*

1. **Mécanisme de référence de la table de flag au domaine Pip-Boy** — colonne simple sans contrainte (addendum du brief, conforme DEC-24) vs FK directe vers la table brute Pip-Boy (décision Auriane/Boris plus récente, contredit DEC-24). **Bloquant avant architecture** : conflit documenté dans `AMBIGUITES.md` REF-24, à faire valider avec l'autorité côté Pip-Boy (DEC-061) avant de trancher — ne pas modifier DEC-24 sans cette validation.
2. **Authentification de l'écran orga** — ce MVP ne modélise qu'un rôle, mais l'accès au dashboard nécessite-t-il déjà Supabase Auth (CLAUDE.md §5, §8), ou un accès réseau local suffit-il pour cette itération dev ?
3. **Nom exact de la table de flag et son schéma précis** — dépend de l'issue de la question 1 (`zax_flags_personnage` proposé dans l'addendum du brief, à confirmer une fois REF-24 tranché).
4. **Format du fichier de configuration des thèmes/mots-clés** (FR-8) et du fichier de config du délai d'auto-envoi (FR-6) — structure exacte, à décider en architecture en cohérence avec le format YAML déjà retenu pour le harnais.

## 9. Index des hypothèses

Aucune hypothèse ouverte à ce stade : les deux hypothèses de la première version (conversations concurrentes en FR-5, seuils empiriques en FR-8) ont été confirmées par Auriane et intégrées directement dans les FR concernées (voir `.memlog.md`).
