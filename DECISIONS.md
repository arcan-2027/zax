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

---

## Règles d'utilisation

- Toute décision est numérotée séquentiellement (DEC-01, DEC-02…)
- Une décision ne se modifie pas : on en crée une nouvelle qui annule la précédente
- Référencer la décision dans les commits : `feat: implémentation DEC-03`
- Quand une décision résout une ambiguïté REF-XX, mettre son statut à ✅ Décidé dans `AMBIGUITES.md`
- Les décisions s'appliquent immédiatement — Claude les respecte sans demande de confirmation
