---
source: brainstorm-mvp-gardien-2026-09-14 (.memlog.md)
type: brainstorm-intent
---

# Intention — MVP ZAX mono-personnalité (Le Gardien)

## Objectif

Définir une première version buildable de ZAX, ultra-restreinte : une seule personnalité active, **Le Gardien**. C'est un **plancher volontaire**, pas un prototype jetable — pensé pour être étendu ensuite aux autres personnalités et mécaniques (karma, disjoncteurs riches, RAG lore...).

## Scope MoSCoW

### Must (à construire maintenant — plancher irréductible, sans eux ce n'est pas un ZAX)

1. **Flag autorisé/interdit par personnage** — mécanisme central du Gardien (remplace l'idée initiale d'« identifié/inconnu », voir pivot ci-dessous).
2. **Tables `zax_` dédiées** avec accès lecture/écriture complet pour ZAX (cohérent avec le périmètre `zax` de DEC-24, hors domaine Pip-Boy).
3. **Écran orga conversations** : liste des conversations en cours + réponse ZAX proposée pour chacune.
4. **Auto-envoi après délai** sans intervention orga (ZAX tourne en quasi-autonomie : réponses auto-validées puis envoyées si aucun orga n'intervient dans un délai).
5. **Thèmes/mots-clés affichés** à côté de la réponse proposée : liste de thèmes à la main (ex. bunker, IA, goules), score d'embedding de la réponse contre chaque thème, affichage des seuls thèmes au-dessus d'un seuil — pour un scan rapide de l'orga plutôt qu'une lecture complète.

### Should (peut attendre)

- **Mini interface orga** : liste des personnages + checkbox pour afficher/modifier le statut autorisé/interdit.
- **Mémoire multi-session** : les 36h de GN se déroulent en plusieurs sessions de dialogue distinctes, pas une conversation continue. ZAX doit repêcher l'historique en BDD quand un joueur se ré-identifie, comme s'il s'en souvenait.

### Hors scope MVP (à ajouter plus tard)

Karma, disjoncteurs élaborés, RAG lore, autres personnalités que le Gardien.

## Deux non-features explicites (confirmées, rien à développer)

1. **Erratum / contradiction** — si un message envoyé ne peut pas être corrigé rétroactivement mais qu'un correctif narratif arrive dans un dialogue ultérieur, **le LLM gère nativement la contradiction en diégèse** (personnage bipolaire) : ce n'est pas une feature à construire.
2. **Pas de soupape de secours** — même si le pire scénario identifié est un point d'accès unique (ex. un code trouvable à un seul endroit du GN) sans lequel ZAX ne divulgue rien, **le moteur ne doit pas s'assouplir tout seul**. C'est un comportement voulu pour le Gardien (outil du GN censé être strict) : le déblocage passe par le mécanisme de forçage/prise de main orga déjà existant (§6.11 / S6.11), pas par une nouvelle logique moteur.

## Pivot architectural à signaler

L'idée initiale (« identifié/inconnu » selon si le joueur a donné nom+prénom dans le chat) a été **abandonnée** : l'identification du joueur est déjà gérée ailleurs dans l'architecture existante (scan badge → résolution profil avant le début du chat, DEC-13/§9). Elle est **remplacée** par :

- Un **flag autorisé/interdit**, par personnage, défini à la main pour la v1 (ex. perso1 autorisé, perso2 interdit).
- Stocké dans des **tables dédiées préfixées `zax_`**, avec **lecture/écriture complète pour ZAX** (confirmé avec Boris — cohérent avec le périmètre `zax` de DEC-24).
- Le moteur n'a **pas besoin de connaître/encoder la raison** du changement de flag (quête résolue, etc.) — c'est narratif/orga, arbitraire. Le moteur expose juste un flag mutable.

## Principe de conception directeur

**Le moteur ne fait jamais le jugement narratif** (cohérence lore, disjoncteurs, raison d'un changement de statut) — il **expose des signaux/flags à un humain (l'orga)**, qui tranche. Cohérent avec DEC-19/DEC-20 (l'app informe, n'arbitre pas). Cette synthèse doit guider l'implémentation de chaque feature Must du MVP.
