---
title: "Addendum — ZAX MVP mono-personnalité (Le Gardien)"
status: draft
created: 2026-09-21
updated: 2026-09-21
---

# Addendum : ZAX — MVP mono-personnalité (Le Gardien)

Contenu utile pour l'architecture/PRD, mais trop technique pour le brief lui-même.

## Référence au personnage depuis une table `zax_`

**Question posée par Auriane** : peut-on faire une table `zax_xxxxxxxx` dont la clé primaire serait une clé étrangère récupérée depuis le Pip-Boy ?

**Contrainte technique à respecter en architecture** : non, pas sous forme de contrainte `FOREIGN KEY` Postgres classique. Deux raisons :

1. **Une contrainte FK doit cibler une table avec un index unique/PK, jamais une vue.** ZAX ne lit le domaine Pip-Boy que via une vue dédiée (DEC-24) — il n'y a donc pas de cible valide pour une FK Postgres standard.
2. **Même si on contournait ça en pointant directement sur la table brute Pip-Boy (`profiles`), ce serait contraire à l'intention de DEC-24.** La vue existe précisément pour découpler les deux domaines : le schéma Pip-Boy peut évoluer (identités de couverture, DEC-058) sans casser ZAX, et une FK directe recréerait ce couplage.

**Décidé (Auriane, 21/09/2026)** : la table `zax_` stocke l'identifiant comme une **colonne simple** (pas de contrainte FK), clé = **`nfc_uid`** — c'est déjà l'identifiant que le badge résout côté serveur avant le début du chat (DEC-13, §9). La validité de la référence (le personnage existe-t-il bien côté Pip-Boy ?) se vérifie **côté API/service**, au moment où le flag est posé ou consulté — pas au niveau de la contrainte de base de données.

**Récupérer le nom du personnage à partir du `nfc_uid`** : pas besoin d'un mécanisme séparé. La vue dédiée Pip-Boy (DEC-24, CLAUDE.md §7) expose déjà `nfc_uid` **et** `name` (avec faction, statut vivant/mort, présence) côte à côte — une seule requête sur la vue suffit pour aller du `nfc_uid` stocké dans `zax_flags_personnage` jusqu'au nom affiché à l'orga. Pas de round-trip supplémentaire à concevoir.

**Reste à trancher en architecture** (pas dans ce brief) :
- Comportement si on veut poser un flag sur un personnage qui n'a pas encore de ligne côté Pip-Boy au moment de la saisie (le flag est défini « à la main » pour la v1 — voir brief, Must #1).
- Nom exact de la/les table(s) `zax_` (une table flag simple `zax_flags_personnage(nfc_uid, autorise, updated_at, updated_by)` suffit a priori pour le MVP, mais à confirmer en architecture).
