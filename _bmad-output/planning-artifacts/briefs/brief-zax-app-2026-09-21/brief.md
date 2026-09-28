---
title: "Product Brief — ZAX MVP mono-personnalité (Le Gardien)"
status: draft
created: 2026-09-21
updated: 2026-09-21
---

# Product Brief : ZAX — MVP mono-personnalité (Le Gardien)

## Résumé exécutif

ZAX est le futur système complet pour le GN Fallout 2027 : 8 personnalités du noyau, karma à deux niveaux, RAG lore, disjoncteurs riches. C'est une architecture large, et elle n'a encore jamais tourné en vrai. Ce brief cadre un **plancher volontaire** — pas un prototype jetable — construit sur une seule personnalité, **Le Gardien** (la plus simple : rigide, binaire, peu de nuances narratives à modéliser).

L'objectif n'est pas de livrer un ZAX diminué, mais de faire tenir debout, sur un périmètre restreint, le cœur de ce que le système devra faire à grande échelle : une personnalité qui répond avec un lore cohérent selon qui lui parle, et une interface admin qui permet à un orga de surveiller et d'arbitrer en temps réel. Si ce plancher tient, l'extension aux 7 autres personnalités, au karma et au RAG est un problème d'ajout de modules, pas de refonte d'architecture.

Ce brief sert à cadrer la PRD qui suit.

## Le contexte et pourquoi ce plancher

Le CLAUDE.md du projet décrit un système à 8 personnalités du noyau (DEC-23), karma à deux niveaux, RAG lore hybride, mode dégradé avec hystérésis, 6 modules de données déclenchant un signal bleu — une architecture pensée pour tenir 36-72h de GN sans supervision constante. Construire tout ça d'un coup avant d'avoir testé la moindre brique en conditions réelles est le risque principal identifié : on industrialiserait une mécanique (validation orga, auto-envoi, détection de sujets) qui n'a jamais été confrontée à un vrai flux de conversation.

Le brainstorming du 14/09 (`brainstorm-mvp-gardien-2026-09-14`) a tranché ce risque en réduisant le scope à l'irréductible : une personnalité, un mécanisme d'autorisation binaire, un écran orga. Le choix du Gardien n'est pas arbitraire — c'est la personnalité au comportement le plus simple à spécifier (menace/ressource, protocoles, peu de nuance), donc celle qui isole le mieux les risques *techniques* (pipeline, validation, dashboard) des risques *narratifs* (richesse d'une personnalité complexe).

Le Gardien n'est pas à écrire from scratch : une fiche complète existe déjà (`sources/zax_20260706.md`) — origine, caractéristiques, déclencheurs, exemple de réponse, et le squelette du module de personnalité (`VOIX`, `DECL-HARD`, `FBDN`, etc.). C'est un template de mémoire à confirmer par les scénaristes, pas une fiche figée, mais le MVP part d'une base écrite, pas d'une page blanche.

## La solution

Le Gardien MVP répond à un personnage qui se présente à un terminal, avec un comportement conditionné par un flag **autorisé / interdit** défini à la main par personnage — pas par identification en jeu (l'identification est déjà réglée en amont par le badge, DEC-13). Un orga supervise le flux de conversations depuis un écran dédié, voit la réponse que ZAX propose d'envoyer, les thèmes qu'elle aborde, et peut laisser l'auto-envoi faire son travail ou intervenir.

**Périmètre du build (Must — plancher irréductible, sans eux ce n'est pas un ZAX) :**

1. Flag autorisé/interdit par personnage — le mécanisme central du Gardien.
2. Tables `zax_` dédiées, lecture/écriture complète pour ZAX — dans le schéma `zax`, donc sans conflit avec le périmètre lecture-seule sur le domaine Pip-Boy (DEC-24).
3. Écran orga conversations : liste des conversations en cours + réponse ZAX proposée pour chacune.
4. Auto-envoi après délai sans intervention orga.
5. Thèmes/mots-clés affichés à côté de la réponse proposée (score d'embedding de la réponse contre une liste de thèmes à la main, affichage au-dessus d'un seuil) — pour un scan rapide plutôt qu'une lecture complète.

**Should (peut attendre une itération) :**

- Mini interface orga : liste des personnages + checkbox pour modifier le statut autorisé/interdit (le MVP peut démarrer avec ce flag positionné directement en base).
- Mémoire multi-session : repêcher l'historique en BDD quand un joueur se ré-identifie sur une session distincte.

**Hors scope MVP (plus tard) :** karma, disjoncteurs élaborés, RAG lore, les 7 autres personnalités du noyau.

**Paramétrage (aucune valeur figée dans le code) :**

- Délai d'auto-envoi (Must #4) : variable d'environnement / fichier de config, valeur par défaut **5 secondes** pour tester sans attendre à chaque itération, ajustable pour le live.
- Thèmes/mots-clés et leurs seuils (Must #5) : liste définie à la main dans un fichier de config. En dev, elle est écrite par Auriane ; en conditions live (équivalent production), elle passe en config côté client — donc Boris/les orgas. Les seuils ne peuvent pas être devinés a priori — ils seront calibrés par les sessions de test décrites en Critères de succès.

## Principe directeur

Le moteur ne porte jamais de jugement narratif — ni sur la cohérence lore, ni sur la raison d'un changement de statut, ni sur une éventuelle exception au comportement strict du Gardien. Il **expose des signaux à un humain** (le flag, les thèmes détectés, la réponse proposée), et c'est l'orga qui tranche. C'est cohérent avec DEC-19/DEC-20 (l'app informe, n'arbitre pas), et ça structure directement les deux non-features actées ci-dessous.

### Deux non-features actées (rien à développer)

- **Erratum / contradiction** : si un message envoyé ne peut pas être corrigé rétroactivement mais qu'un correctif narratif arrive plus tard, le LLM gère nativement la contradiction en diégèse (un ZAX aux personnalités multiples peut se contredire sans que ce soit un bug). Pas de mécanisme de correction à construire.
- **Pas de soupape de secours automatique** : même dans le pire scénario (un point d'accès unique, par exemple un code trouvable à un seul endroit du GN, sans lequel le Gardien ne divulgue rien), le moteur ne s'assouplit pas de lui-même. C'est un comportement voulu — le Gardien est un outil du GN censé être strict. Le déblocage passe uniquement par le forçage / la prise de main orga déjà prévue (§6.11 du CLAUDE.md), pas par une nouvelle logique moteur.

## À qui ça sert

Les job-to-be-done ci-dessous suivent le travail déjà posé avec Boris — ils sont considérés validés, pas à revalider ; Boris relira le document mais ce n'est plus un objectif de ce brief.

- **L'orga en poste pendant le GN** (Boris et les autres). *Quand le GN tourne sur 36-72h avec 5 terminaux, je veux surveiller et reprendre la main sans lire chaque conversation en entier, pour garder le contrôle narratif sans devenir le goulot d'étranglement.* Succès pour lui : jamais surpris par un message parti tout seul qu'il n'aurait pas laissé passer.
- **Le joueur au terminal** (indirect — il ne voit jamais le MVP en tant que tel, seulement Le Gardien). *Quand je me présente à un terminal, je veux que ZAX me traite de façon cohérente avec qui je suis, pour que l'immersion tienne même en figurant.*
- **Auriane et Boris, côté produit**. *Avant d'industrialiser 8 personnalités + karma + RAG, on veut valider le plancher technique et narratif sur un seul module, pour ne pas construire une architecture qui ne tient pas au premier test réel.*

## Critères de succès

Le MVP fonctionne si, simultanément :

- **Le Gardien** est configuré et répond de façon cohérente avec le lore, en fonction du personnage qui se présente (flag autorisé/interdit appliqué correctement).
- **L'interface admin** est fonctionnelle : un orga peut surveiller les conversations en cours et orchestrer l'expérience (voir les réponses proposées, les thèmes détectés, laisser l'auto-envoi agir ou intervenir) sans naviguer ailleurs que dans cet écran.

**Forme du test** : une succession de simulations de conditions réelles envisageables — on teste la fonctionnalité de chat et ses variables (personnage autorisé/interdit, thèmes détectés, auto-envoi) pour observer le comportement du Gardien et recueillir le ressenti des orgas. Pas un unique test binaire go/no-go, mais des itérations jusqu'à ce que le comportement et l'interface admin conviennent.

## Ce qui reste ouvert

- **Provenance du personnage dans le flag `zax_`** : décidé — le flag référence le personnage par `nfc_uid` (déjà l'identifiant résolu par le badge, DEC-13), comme colonne simple et non comme contrainte `FOREIGN KEY` (une FK Postgres ne peut pas cibler une vue, et pointer sur la table brute recréerait le couplage que la vue DEC-24 est censée éviter). La même vue dédiée expose déjà le nom du personnage à côté du `nfc_uid`, donc pas de mécanisme supplémentaire à concevoir pour l'affichage côté orga. Détail complet en `addendum.md`.
- **Comportement si le personnage n'existe pas encore côté Pip-Boy** au moment où on pose le flag — non tranché, à lever en architecture.

## Vision

Si ce plancher tient — le Gardien répond juste, l'orga garde le contrôle sans se noyer — la suite est un travail d'extension plutôt que de refonte : ajouter les 7 autres personnalités du noyau une par une (chacune plus riche narrativement que le Gardien), brancher le karma à deux niveaux, ouvrir le RAG lore, activer les disjoncteurs élaborés et le mode dégradé complet. Le MVP n'est pas une version light de ZAX qu'on jettera : c'est la fondation sur laquelle le reste du système se pose.
