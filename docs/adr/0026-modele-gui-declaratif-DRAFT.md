# ADR-26 — Modèle GUI déclaratif GenUI/SpecUI (GeoWorks-like) — DRAFT

- **Statut** : **DRAFT — à instruire** (NON ratifiée). Ouverte 2026-05-26.
  Ratification interdite tant que les 3 conditions du moratoire §10 ne sont pas
  réunies — en particulier **≥ 50 % d'implémentation de référence testée**
  (arc SP-3.n, étapes G.1→G.4 au minimum). Ce fichier pré-instruit la décision.
- **Date (draft)** : 2026-05-26
- **Décideurs (pressentis)** : bmarty (humain — a exprimé la préférence GeoWorks,
  2026-05-25/26), Claude Code.
- **Jalon déclencheur** : arc **SP-3.n** (Event Manager / Control / Dialog) —
  exposer une couche GUI applicative aux apps C userland.
- **ADR liées** : **révise/étend ADR-06** (GUI SymbOS-like → précise le *modèle
  d'API applicative*) ; **ADR-17** (ABI syscall — ajoute ~5-6 slots `$15-$3F`) ;
  **ADR-25** (concurrence — le MainLoop bloquant s'appuie sur block/wake) ;
  **ADR-16** (driver model event-driven — la file d'événements en est l'aval) ;
  s'assoit sur le « niveau QDF » (backing stores/coords locales, SP-3.m G.4).
- **Référence d'art** : `docs/REFERENCES_ART.md` (SymbOS / Apple IIgs / GeoWorks).

---

## Contexte

OricOS dispose déjà d'une GUI fonctionnelle (WM, focus, drag, chrome, modal,
menus table-driven `menu_defs`, icônes `ICON_TABLE`, widgets managés), mais son
modèle d'interaction app↔GUI est aujourd'hui **piloté par callbacks kernel** :
`_wm_invoke_active_cb` fait `jsr (vec,X)` vers une routine en bank 1. Ce modèle :

1. **Fragile en cross-bank** : un vecteur de callback userland appelé depuis le
   kernel mélange les espaces de banking et complique l'ABI (ADR-17).
2. **Peu adapté au C** : llvm-mos n'expose pas proprement les pointeurs de
   fonction cross-bank ; le pattern naturel en C est une boucle d'événements
   avec `switch`.
3. **Impératif** : chaque élément d'UI est construit par code, pas déclaré.

La question : **quel modèle d'API GUI exposer aux apps C userland ?**

## Décision (proposée)

Adopter le **modèle GeoWorks/GEOS** : **UI déclarative + MainLoop à messages +
séparation GenUI/SpecUI**, plutôt que le modèle impératif TaskMaster d'Apple IIgs.

Trois piliers :

1. **UI déclarative** — l'app décrit menus/dialogues/contrôles comme des *tables
   de données* (`const` arrays en C) ; le système les **rend et exécute**.
   Continu avec l'existant (`menu_defs`, `ICON_TABLE` déjà table-driven). Les
   dialogues suivent le modèle **command table** de GEOS C64 (`DB_POSITION`,
   `DB_TEXT`, `DB_CHECKBOX`, `DB_OK`, `DB_CANCEL`, `DB_END`), exécutés par un
   unique `SYS_DO_DLGBOX` modal.

2. **MainLoop → messages** — `SYS_MAIN_LOOP` (bloquant via block/wake ADR-25)
   rend des **messages sémantiques** (`MSG_MENU_ITEM`/`MSG_ICON`/`MSG_KEY`/
   `MSG_CONTENT`/`MSG_CLOSE`/`MSG_CONTROL`). Le MainLoop traite lui-même les
   interactions standard (drag fenêtre, déroulé menu, tracking contrôle) **un
   événement par appel** (état gardé entre appels → aucune boucle longue sous
   `Forbid`, préemption ADR-03 préservée). **Retrait des callbacks kernel.**

3. **GenUI / SpecUI** — l'UI **générique** déclarée par l'app est séparée du
   **rendu spécifique** (« look »). v1 = un seul SpecUI (look GEOS/Ensemble peint
   au niveau QDF). L'ABI app restant déclarative, le look pourra évoluer **sans
   recompiler les apps**.

**Invariant QDF** : tout rendu se fait dans le backing store de la fenêtre, en
coordonnées locales ; le compositor place à l'écran (jamais d'adresse XVGA
exposée à l'app). Cohérent avec SP-3.m (G.4/G.4bis).

## Alternatives écartées

- **(a) TaskMaster impératif (Apple IIgs)** — `NewControl` × N, `ModalDialog`
  avec contrôles construits, dispatch par *task codes*. Plus de surface syscall,
  modèle impératif moins naturel en C, et pas de séparation look/logique.
  L'humain a explicitement préféré GeoWorks après comparaison directe
  (cf. `docs/REFERENCES_ART.md`).
- **(b) Statu quo callbacks kernel** — conserver `jsr (vec,X)`. Rejeté :
  fragile cross-bank, mauvais fit C, non déclaratif.
- **(c) Moteur objet complet PC/GEOS (« Goc », géométrie auto-managée, fontes
  vectorielles)** — surdimensionné v1. On prend **l'esprit déclaratif**, pas le
  moteur objet. Cueillir brique par brique (cf. mémoire direction GUI).

## Conséquences

- **+** API C naturelle (boucle `switch`), surface syscall réduite (~5-6 slots),
  look découplé des apps (investissement durable), continuité avec l'existant
  table-driven.
- **−** Churn de tests (migration menus/widgets/modal vers MainLoop+déclaratif —
  **filet de tests obligatoire**) ; le MainLoop bloquant heurte la limite du
  **`KBD_WAITER` unique** → **motive les signaux multi-bits génériques**
  (polish #1 d'ADR-25) ; encodage des command tables à spécifier (format `DB_*`).
- **ADR-06** passe de « GUI SymbOS-like » (impératif WM) à « GUI SymbOS-like au
  noyau + modèle applicatif déclaratif GeoWorks-like » — révision à acter à la
  ratification.

## Découpage de l'implémentation de référence (arc SP-3.n)

`G.1` file d'événements · `G.2` MainLoop+messages · `G.3` modèle déclaratif ·
`G.4` contrôles génériques · `G.5` `SYS_DO_DLGBOX` · `G.6` `SYS_ALERT` ·
`G.7` démo C. (Détail : `BACKLOG.md`, arc SP-3.n.)

## Conformité moratoire §10 (à compléter avant ratification)

1. **Dossier d'instruction écrit** : partiel (ce draft + `REFERENCES_ART.md` +
   cadrage BACKLOG). À chiffrer : coût RAM file d'événements, surface de
   migration, format command table. **TODO**.
2. **≥ 50 % d'implémentation testée** : **non atteint** (arc non démarré). Bloque
   la ratification.
3. **Cohérence ADR** : compatible ADR-16/17/25 ; révise ADR-06 (à expliciter).

**Critère go (draft → ratifiée)** : étapes G.1→G.4 implémentées et testées
(file d'événements + MainLoop + déclaratif + contrôles), sans régression de la
suite GUI existante.

---

*Draft v0.1 — 2026-05-26. NON ratifiée. À instruire au fil de l'arc SP-3.n.*
