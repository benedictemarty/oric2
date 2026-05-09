# ADR-15 — Isolation mémoire post-v1

- **Statut** : **parquée v2** 2026-05-09 (Phase 0 programme état-de-l'art) — décision *de ne pas décider tant que…*
- **Date d'ouverture** : 2026-05-08
- **Date de parking** : 2026-05-09
- **Décideurs** : bmarty, Claude Code
- **Contexte** : ADR-04 v1 (bank-based, OS de confiance) ratifiée. Question v2 : que devient l'isolation mémoire quand on ouvre OricOS aux apps non-trusted, ou quand le HDL ULX3S est mature ?

## Pourquoi parking et pas décision ?

Au moment de Phase 0 (2026-05-09), aucun des 3 instruments d'instruction n'est disponible :

1. **Pas de HDL ULX3S existant** pour mesurer le budget BRAM/LUT restant. Décider d'ajouter une MMU custom à un design qui n'existe pas serait du vaporware spécifié.
2. **Pas d'apps non-trusted en v1** (ADR-04 explicitement « OS de confiance »). Aucun cas d'usage immédiat.
3. **Pas de prototype** des 3 alternatives candidates. Coûts comparatifs inconnus.

**Trancher maintenant entre (a)/(b)/(c) reproduirait le pattern de ratification compulsive identifié par l'audit du 2026-05-09 et formalisé dans le moratoire ADR (CLAUDE.md §10).**

## Question initiale

À quoi ressemble la v2 d'ADR-04 ?

- (a) MMU custom HDL ECP5 (translation table par bank, BRAM).
- (b) MPU à segments avec privilege bits (kernel/user).
- (c) Banking matériel étendu avec tags d'accès.

## Critères de réouverture

ADR-15 doit être réouverte dès qu'**au moins l'un** des 3 jalons suivants est atteint :

1. **Apps non-trusted ratifiées** (ouverture marketplace OricOS, code tiers exécuté, ou guest Oric 1 enrichi) — bascule modèle « OS de confiance » vers « OS multi-tenant ».
2. **HDL ULX3S à maturité HW-2** (port 65C816 ECP5 fonctionnel + contrat HW-1 figé) — budget BRAM/LUT restant connu, MMU custom vs MPU chiffrable.
3. **Date plancher 2026-12-31** — sécurité temporelle.

## Préparation préalable

Phase 4 du programme état-de-l'art (S6-S8) prévoit la création d'un draft `docs/adr/0015-isolation-v2-DRAFT.md` qui pré-instruit les 3 options avec :

- Données budgétaires ECP5 LFE5U-85F (416 KB BRAM total dont caches GPU/compositor/scanout déjà alloués).
- Coûts HDL typiques d'une MMU custom (translation table walk, TLB, latence).
- Coûts MPU à segments comparés.
- Références projets retro avec MMU custom (à enquêter).
- État des questions ABI exposées par chaque option à userland.

Ce draft ne tranche rien. Il prépare la décision future.

## Conséquences du parking

### Positives

- Évite décision prématurée non-instruisable.
- Force discipline d'architecte conforme au moratoire ADR (CLAUDE.md §10).
- Phase 0 du programme n'est pas bloquée sur une décision lointaine.

### Négatives

- Aucune visibilité v2 immédiate sur l'isolation. Acceptable car ADR-04 v1 fournit modèle de confiance suffisant.

## Si le parking devait être levé pour décision immédiate

Ce qui suit est **non-recommandé** mais documenté pour information :

| Option | Coût HDL | Coût RAM | Risque |
|---|---|---|---|
| (a) MMU custom HDL ECP5 | élevé (translation + walk + TLB) | ~1-2 KiB BRAM | refactor SDRAM controller, retard B2 |
| (b) MPU à segments avec privilege bits | moyen | minimal | sans précédent 65C816, R&D |
| (c) Banking étendu tags d'accès | faible | minimal | extension propriétaire ADR-04 |

(c) serait le moins disruptif, (a) le plus moderne, (b) le plus « custom ». Mais aucun n'a de sens sans contexte HDL réel.

## Références

- CLAUDE.md §3 (ADR-15 parquée)
- CLAUDE.md §10 (moratoire ADR)
- ADR-04 (isolation v1 bank-based)
- BACKLOG.md (DEC-4 fusionnée)
