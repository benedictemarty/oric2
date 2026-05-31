# CLAUDE.md — Phosphoric / Projet Oric 2

> Document directeur pour Claude Code. Ce fichier est la **source de vérité** pour les décisions d'architecture, les contraintes, et le périmètre courant. À lire en début de chaque session avant toute modification de code.

---

## 1. Identité du projet

**Phosphoric** est un émulateur Oric 1 multiplateforme écrit en C. Son rôle évolue : il devient le **modèle exécutable de référence** (golden model) du projet **Oric 2**, une machine "chimère" rétrocompatible Oric 1, destinée à terme à une implémentation HDL sur FPGA ULX3S.

**Oric 2** est défini par trois couches indissociables :
- Le **hardware Oric 2** : 65C816, ULA étendue, compositor matériel, banking mémoire, sortie HDMI.
- **OricOS** : système d'exploitation multitâche graphique, natif Oric 2.
- Le **guest Oric 1** : machine Oric 1 virtualisée tournant dans une fenêtre d'OricOS, exécutée nativement par le 65C816 en mode émulation (paravirtualisation matérielle via XCE).

Phosphoric doit modéliser ces trois couches en logiciel, validation comportementale avant le port HDL.

---

## 2. Décisions d'architecture figées (ADR ratifiées)

> **Résumé complet dans [`docs/adr/ADR_SUMMARY.md`](docs/adr/ADR_SUMMARY.md).** Ce tableau liste les décisions clés — lire le fichier complet pour les alternatives écartées, justifications et implications projet.

Ces décisions sont **non-négociables** sans nouvelle discussion explicite avec l'humain.

| ADR | Sujet | Décision clé |
|-----|-------|-------------|
| ADR-01 | CPU | WDC 65C816 — mode E (6502 compat) + mode N (16-bit natif) + XCE |
| ADR-02 | Vidéo | Double ULA (host XVGA + guest Oric 1) + compositor matériel |
| ADR-03 | Scheduler | Préemptif strict timer-driven, round-robin, réf. SymbOS |
| ADR-04 | Mémoire | Bank-based v1, pas de MMU, OS de confiance |
| ADR-05 | Langage | Kernel=asm 65C816, userland=C llvm-mos mono-bank 8-bit native |
| ADR-06 | GUI | SymbOS-like multifenêtré (noyau/WM) + GenUI/SpecUI déclaratif (ADR-26) |
| ADR-07 | FS | FAT32 SD sur HW, `--hostfs DIR` sur émulateur |
| ADR-08 | Apps | Bundle léger : header magique + sections (inspiré AmigaOS Hunk) |
| ADR-09 | Audio | AY-3-8912 + extension SID-like (3 voies + PCM) |
| ADR-10 | Compat | Oric 1 stricte en mode E — aucune régression tolérée |
| ADR-11 | Mode E | Hybride pragmatique : bug JMP($xxFF) reproduit, illegaux→NOP |
| ADR-12 | HIRES | ULA guest : 240×200×3bpp (compat Oric 1) |
| ADR-13 | Syscall | `COP #$AA` + table dispatch bank 1 `$5750` |
| ADR-14 | TCB | Table fixe 16 TCBs à `$01:5C00`, bitmap `$01:5B00` |
| ADR-16 | Drivers | Hybride : classe 1 IRQ-driven, classe 2 sync, pas de struct ops v1 |
| ADR-17 | ABI | 18 syscalls, `A`=num, `A=$FF`=err, `cop #$AA` |
| ADR-18 | 6502 | Retiré post-validation — 65C816 mode E seul |
| ADR-19 | VRAM | SDRAM unifiée 16 MiB, accès GPU direct, CPU via MMIO `$0330` |
| ADR-20 | Desktop | 1024×768×4bpp XVGA, 512 oct/ligne, palette VGA-IBM 16 couleurs |
| ADR-21 | GPU | Blitter HW command-based, `$0340-$034F`, 7 opcodes, v0.1 sync |
| ADR-22 | Clavier | IRQ-driven `$0350-$035F`, matrice guest virtuelle |
| ADR-23 | Console | Flux de caractères, backend interchangeable (v1=Oric1, cible=GPU) |
| ADR-24 | Souris | Position absolue 10-bit + deltas, IRQ `$0360-$036F` |
| ADR-25 | Concurrence | Exec-classique : Forbid/Permit + block/wake, pas de spin |
| ADR-26 | GUI API | GenUI/SpecUI déclaratif, SYS_MAIN_LOOP, messages sémantiques |
| ADR-27 | Backing store fenêtre | Option (b) : stride GPU (BPL) configurable + backing store compact slot par slot (`WM_COMPACT_FLAGS[slot]=$A5`) ; shadow kernel `bpl` + garde IRQ ; §0quater C-2 garde `_gfx_xvga_bpl_guard` dans 5 helpers GPU (heuristique `GFX_BASE_HI ≥ $10` ET shadow ≠ 0 → `bpl=0`) couvre les ~36 sites kernel direct |
| ADR-28 | Threading WM | Option C : politique fenêtre + rendu en tâche serveur WM, curseur seul en IRQ (`TC_WM_FLAG=$A5`) ; réf Intuition/GEOS |
| ADR-29 | Drag notification hint | Hint déclaratif aligné GeoWorks : default DELAYED (app notifiée 1× à la release), override IMMEDIATE via `WM_DRAG_NOTIFY_HINT=$A5` ; réf PC/GEOS `gValueC.def` |
| ADR-31 | Clip widget hors rect parent | Option A : skip widget si `rel.x+w > win.w` OR `rel.y+h > win.h` (test dans `_wm_draw_widget_body`) ; rendue redondante à terme par ADR-27 (backing store contraint le rendu par construction), mais conservée v1 — pas de migration coûteuse pour un cas couvert |

**Détail, alternatives écartées, implications** → [`docs/adr/ADR_SUMMARY.md`](docs/adr/ADR_SUMMARY.md) et fichiers individuels `docs/adr/00XX-*.md`.

---

## 3. Décisions ouvertes (ADR à instruire — NE PAS trancher unilatéralement)

Si une tâche force la main sur l'une de ces décisions, **arrête et demande**. Ne choisis pas par défaut.

Ces ADR ont été ouvertes le **2026-05-08** suite au point critique architecte
senior (cf. `BACKLOG.md` §annexe). Elles correspondent à des décisions
prises tacitement dans les premiers sprints OricOS et qu'il faut
expliciter avant d'avancer.

### ~~ADR-13~~ → ratifiée 2026-05-08, déplacée vers §2 (option a : COP + table)

### ~~ADR-14~~ → ratifiée 2026-05-08, déplacée vers §2 (table fixe 16 + bitmap free)

### ~~ADR-16~~ → ratifiée 2026-05-09, déplacée vers §2 (modèle hybride event-driven + sync, sans struct ops v1)

### ~~ADR-17~~ → ratifiée 2026-05-09, déplacée vers §2 (18 syscalls, COP `$AA`, sentinelle `A=$FF`, table dispatch)

### ~~ADR-18~~ → ratifiée 2026-05-09, déplacée vers §2 (retrait net 6502 post-validation, DEC-1 actée)

### ADR-15 — Isolation mémoire post-v1 (parquée v2 — 2026-05-09)

**Statut** : décision **parquée** au programme état-de-l'art Phase 0. Pas tranchée maintenant car aucun des 3 instruments d'instruction n'est disponible : pas de HDL ULX3S existant pour mesurer le budget BRAM/LUT, pas d'apps non-trusted en v1 (ADR-04 « OS de confiance »), pas de prototype des 3 alternatives.

**Question initiale** : à quoi ressemble la v2 d'ADR-04 ?
- (a) MMU custom HDL ECP5 (translation table par bank, BRAM).
- (b) MPU à segments avec privilege bits (kernel/user).
- (c) Banking matériel étendu avec tags d'accès.

**Critères de réouverture** : ADR-15 doit être réouverte dès qu'**au moins l'un** des 3 jalons suivants est atteint :

1. **Apps non-trusted ratifiées** (ouverture marketplace OricOS, code tiers exécuté, ou guest Oric 1 enrichi) — bascule modèle « OS de confiance » vers « OS multi-tenant ».
2. **HDL ULX3S à maturité HW-2** (port 65C816 ECP5 fonctionnel + contrat HW-1 figé) — budget BRAM/LUT restant connu, MMU custom vs MPU chiffrable.
3. **Date plancher 2026-12-31** — sécurité temporelle, force la réouverture même si jalons 1/2 pas atteints.

**Préparation préalable** : draft `docs/adr/0015-isolation-v2-DRAFT.md` à initier en Phase 4 du programme (Q3 2026), pré-instruisant les 3 options avec données budgétaires ECP5 LFE5U-85F (416 KB BRAM dont caches GPU/compositor déjà alloués), références projets retro avec MMU custom, coûts HDL typiques.

**Impact** : multitâche robuste, exécution apps non-trusted.

### ~~ADR-25~~ → ratifiée 2026-05-25, déplacée vers §2 (modèle Exec-classique : Forbid/Permit + block/wake)

### ~~ADR-27~~ → ratifiée 2026-05-30v, déplacée vers §2 (option (b) + §0quater C-2 : stride GPU BPL configurable + backing store compact + helper `_gfx_xvga_bpl_guard` dans 5 helpers GPU couvrant les ~36 sites kernel direct ; validation oricrobot : peek $107A1E=$77 stable après clic menu + mouvements souris ; 12/12 helloc, 24/24 globales ; dossier : `docs/adr/0027-backing-store-fenetre.md`)

### ~~ADR-28~~ → ratifiée 2026-05-29, déplacée vers §2 (option C : politique fenêtre + rendu en tâche serveur WM, curseur seul en IRQ ; gated `TC_WM_FLAG=$A5` ; implémentation Étapes 0/1/2/3 + §6.6 + §6.7 livrée, suite `make tests` verte ; simplifie ADR-27 §0ter point 5 ; dossier : `docs/adr/0028-threading-window-manager.md`). **Rétractations** (2026-05-30 suite test interactif utilisateur) : §1.2ter « famine réfutée » + §6.7 « quota anti-drop button-UP » étaient mal-ciblés (le bug fin-de-course n'est ni un drop ni un coût IRQ — c'est un bottleneck app, cf. ADR-29). Étape 3 (skip mouse_step IRQ) déjà revertée pour bug interactif. Le design option C tient ; les rétractations sont tracées dans le fichier ADR-28.

### ~~ADR-29~~ → ratifiée 2026-05-30, déplacée vers §2 (hint déclaratif aligné GeoWorks, default DELAYED, override IMMEDIATE via `WM_DRAG_NOTIFY_HINT=$A5` ; implémentation Étape 1 livrée + validation interactive utilisateur positive ; révèle et corrige bug pré-existant `_wm_redraw_ctl` ; dossier : `docs/adr/0029-drag-notification-hint.md`)

### ADR-32 — Course ZP IRQ↔tâche : propriétaire unique WM (dossier d'instruction, **DRAFT 2026-05-31**)

**Question** : comment fermer la course ZP entre contexte IRQ
(`kernel_wm_mouse_step`, `_cursor_draw`) et contexte tâche
(`sys_win_create`, `sys_gfx_*`, MainLoop) identifiée par l'audit
`AUDIT_65C816_REMEDIATION.md` §3.3a comme **suspect n°1 du revert
ADR-28 Étape 3** ? `forbid` ne masque PAS les IRQ (ADR-25). 206 sites
d'écriture `sta WM_ARG_*`/`sta GFX_*`/`sta WM_DP_*` dans `wm.s`,
patchwork `sei` ad-hoc épars sur certains sites tâche (`_ml_classify`,
`_wm_widget_hit`, `_wm_redraw_ctl`) — laisse `sys_win_create` non
gardé. 3 options chiffrées : (A) statu quo + `sei` épars partout ;
(B) migration complète `mouse_step` (focus, drag, resize, redraw,
compose, **callbacks**, **curseur**) hors IRQ vers tâche serveur WM —
finalise option C ADR-28 ; (C) partition stricte ZP IRQ-only.
**Recommandation senior tracée : (B)**, avec **plan d'atomicité**
critique (flag `WM_TASKMODE` de bascule, anti-revert ADR-28 Étape 3)
et **harnais d'injection event-async préalable** (axe 8.5,
`cpu816_set_pc_hook` côté Phosphoric). Dossier :
`docs/adr/0032-zp-race-irq-task-DRAFT.md`.

### ADR-30 — Roadmap toolbox (alignement GeoWorks) (dossier d'instruction, **Étapes 1+3 ratifiées 2026-05-30**)

### ~~ADR-31~~ → ratifiée 2026-05-30, déplacée vers §2 (option A : skip widget si `rel.x+w > win.w` OR `rel.y+h > win.h` dans `_wm_draw_widget_body` ; validation interactive utilisateur positive ; deviendra obsolète à la ratification ADR-27 backing store ; dossier : `docs/adr/0031-clip-widget-rect-parent.md`)

**Question** : quelle couverture cible OricOS sur la hiérarchie de classes
`Gen*` de PC/GEOS (40 classes) ? Audit factuel WebFetch 2026-05-30 du
[dossier Include/Objects](https://github.com/bluewaysw/pcgeos/tree/master/Include/Objects)
révèle 64 `.def` au total (40 Gen + 7 Vis + ~18 subsystems). Couverture
OricOS actuelle = **22 % des classes Gen, 57 % des widgets d'interaction
directe** (audit ADR-29). 5 widgets prioritaires manquants : `GU_LIST`
(impl interne existante, juste exposer), `GU_MENU`+`GU_MENU_ITEM` (refactor
`menu_defs` déclaratif), `GU_RANGE` (slider borné, hérite de `gValueC`
donc ADR-29 applicable), `GU_SPIN` (incrémenteur), `GU_FIELD` (champ
formaté). 3 options chiffrées : (A) statu quo ; (B) big-bang ;
(C) quick-wins-only ; (D) roadmap incrémentale par priorité.
**Recommandation senior tracée : (D)**, 5 étapes indépendantes ratifiables
chacune après validation interactive utilisateur (leçon ADR-29). Couverture
cible post-Étape 5 = ~85 % des widgets d'interaction. Au-delà (Vis*,
subsystems, Document/Application framework) hors scope ADR-30, à instruire
séparément si pertinence. Dossier : `docs/adr/0030-roadmap-toolbox-DRAFT.md`.

---

## 4. Roadmap et jalon courant

### Track B — Prototype Phosphoric (en cours)

- **B1 — Cœur 65C816 dans Phosphoric** ← **JALON COURANT**
- B2 — Modèle mémoire bank-based 256K minimum
- B3 — Démonstrateur bascule mode + virtualisation guest (stub kernel)
- B4 — Compositor logiciel + double ULA

### Track A — Document d'architecture (DAT)
À rédiger en parallèle, format IEEE 42010. Pas le sujet de Claude Code pour l'instant — Claude Code se concentre sur l'implémentation B1.

### Critères d'acceptation B1

- Le CPU core supporte l'intégralité du jeu d'instructions 65C816 (modes E et N).
- En mode émulation, la suite de tests Oric 1 existante de Phosphoric passe **à 100 %** sans régression.
- Cycle-accurate vérifié contre la suite Klaus Dormann 6502 (ou équivalent à valider) en mode émulation.
- Les nouveaux opcodes 65C816 sont couverts par des tests unitaires dédiés.
- Le flag E et la bascule XCE sont implémentés et testés.
- Aucun changement de comportement pour la ROM Oric 1 originale.

---

## 5. Contraintes dures

À respecter sans exception. Une PR qui les viole doit être refusée.

1. **Compat Oric 1 stricte** en mode émulation. Le ROM original boote et fonctionne identiquement. Aucune régression sur la suite de tests existante.
2. **Ne jamais modifier l'image ROM Oric 1**. Elle est immuable.
3. **Multiplateforme préservé**. Pas de dépendance OS-spécifique non portable. Le code doit compiler sur Linux, macOS, Windows comme actuellement.
4. **Cycle accuracy en mode émulation**. Pas de relaxation pour gagner en vitesse d'implémentation.
5. **Toute décision d'architecture trace à une ADR**. Si une décision n'a pas d'ADR, elle est ouverte (cf. §3) et doit être posée à l'humain.
6. **Nouvelles fonctionnalités gated derrière flags explicites**. Aucun changement de comportement caché. L'Oric 2 ne s'active jamais en mode legacy.
7. **Pas de gros refactor sans demande explicite**. Modifications minimales et chirurgicales.
8. **Aucune dépendance externe nouvelle sans validation**. Phosphoric doit rester léger et auto-contenu.

---

## 5bis. Layout du repo et commandes

Le code source de Phosphoric est importé sous `Phosphoric/` (cloné depuis `https://git.nagominosato.fr:6775/chipinette/Phosphoric`). Un `CLAUDE.md` détaillé existe dans ce sous-répertoire — **le lire en complément** pour l'architecture interne (sous-systèmes 6502, VIA, ULA, AY, Microdisc, etc.), le framework de tests, et le détail de tous les targets Make.

Toutes les commandes ci-dessous s'exécutent depuis `Phosphoric/` :

```bash
make SDL2=1              # Build standard
make DEBUG=1 SDL2=1      # Build debug (-g -O0)
make tests               # Lance toutes les suites de tests (doit passer à 100 % avant commit)
make test-cpu            # Suite CPU isolée (cf. CLAUDE.md de Phosphoric pour la liste complète)
make valgrind            # Détection de fuites mémoire
make clean
```

Lancer un binaire :

```bash
./oric1-emu -r roms/basic10.rom        # ORIC-1 BASIC 1.0
./oric1-emu -r roms/basic11b.rom       # Atmos BASIC 1.1 (auto-détecté)
```

Le compteur de tests autoritatif est dans `Phosphoric/VERSION_TRACKING`. Le critère B1 (§4) « 100 % de la suite Oric 1 sans régression » s'évalue contre `make tests`.

## 6. Conventions de code et workflow

### Style
- Le code est en **C**. Suivre rigoureusement les conventions déjà en place dans Phosphoric (indentation, nommage, organisation des fichiers, headers). Lire les fichiers existants avant d'écrire.
- Les noms de symboles 65C816 suivent la nomenclature WDC officielle (registres A, X, Y, S, D, DBR, PBR, P, PC ; flags N, V, M, X, D, I, Z, C, E ; instructions canoniques).
- Commentaires en français acceptés, mais les noms de symboles, fonctions et types restent en anglais (cohérence avec le code existant).

### Organisation
- Le cœur 65C816 est isolé dans son propre module. Le 6502 existant reste fonctionnel et indépendant tant que la transition n'est pas complète, ou est remplacé proprement si plus simple — **à valider avant exécution**.
- Les nouveaux modes Oric 2 (ULA étendue, banking, registres I/O Oric 2) sont gated derrière un flag de mode machine explicite.

### Workflow
- **Petits commits atomiques**. Un changement logique = un commit. Message clair, format `[B1] Implement XCE instruction and E flag handling`.
- **Tests systématiquement**. Tout nouveau opcode, tout nouveau comportement, a un test unitaire.
- **Référencer les ADR en commentaire** quand une décision n'est pas évidente : `/* ADR-01: emulation mode preserves 6502 cycle counts */`
- **Avant tout refactor structurel**, demander confirmation à l'humain.

---

## 7. Critères de validation continue

À chaque modification, vérifier dans cet ordre :

1. La compilation passe sur la plateforme de dev.
2. La suite de tests Oric 1 existante passe à 100 %.
3. Les nouveaux tests ajoutés passent.
4. (Si pertinent) Bench comparatif de performance — pas de régression > 5 % sans justification.
5. La ROM Oric 1 boote visuellement et exécute un programme BASIC simple.

---

## 8. Glossaire

- **ULA** — Uncommitted Logic Array. Sur Oric 1, gère le timing vidéo, la lecture VRAM, la génération couleur attribute-based.
- **ULA host** — Instance ULA générant le framebuffer principal d'OricOS (modes Oric 2 étendus).
- **ULA guest** — Instance ULA dédiée au rendu de la machine Oric 1 virtualisée.
- **Mode E (émulation)** — État du 65C816 où il se comporte en 6502 strict.
- **Mode N (natif)** — État 16 bits du 65C816 utilisé par OricOS.
- **XCE** — Exchange Carry/Emulation : instruction 65C816 qui bascule entre mode E et N.
- **Bank** — Page de 64K dans l'espace d'adressage 65C816. PBR pour les instructions, DBR pour les données.
- **PBR / DBR / DPR** — Program Bank Register, Data Bank Register, Direct Page Register.
- **Compositor** — Logique (logicielle dans Phosphoric, matérielle en HDL) qui combine framebuffers host et guest à la sortie vidéo.
- **OricOS** — Système d'exploitation natif Oric 2.
- **Guest** — Instance Oric 1 virtualisée dans une fenêtre OricOS.
- **Host** — La plateforme Oric 2 / OricOS qui héberge le guest.
- **Golden model** — Phosphoric sert de référence comportementale pour le futur HDL.
- **Paravirtualisation** — Stratégie de virtualisation où le guest s'exécute nativement sur le CPU (mode E ici), avec coopération minimale du kernel host.

---

## 9. Références techniques

- **WDC W65C816S Datasheet** — référence opcodes et timings.
- **The Western Design Center 65C816 Programming Manual** — sémantique détaillée des modes E/N, vecteurs, banking.
- **The Oric Hardware Manual** (community) — comportement ULA, VIA 6522, timing PAL.
- **Klaus Dormann's 6502 functional test** — suite de validation cycle-exact en mode émulation.
- **ULX3S documentation** — cible HDL finale (ECP5, BRAM, HDMI tx, SDRAM).
- **IEEE 42010** — standard d'architecture appliqué au DAT (track A).
- **SymbOS sources** — référence multitâche préemptif fenêtré sur 8 bits.
- **GS/OS** — référence OS sur 65C816 avec mode legacy.

---

## 10. Méta — Évolution de ce document

- Ce document est versionné dans le repo et fait partie intégrante du projet.
- Toute modification d'une ADR ratifiée passe par discussion explicite avec l'humain et mise à jour de §2.
- Toute ADR ouverte (§3) qui se ferme rejoint §2 avec sa date de ratification.
- Le jalon courant (§4) est mis à jour à chaque transition B1→B2→B3→B4.
- Si Claude Code identifie une décision implicite non documentée, il l'ajoute en §3 comme ADR ouverte plutôt que de trancher.

### Moratoire de ratification ADR (instauré 2026-05-09)

Aucune nouvelle ADR ne peut être ratifiée tant que **toutes** les conditions suivantes ne sont pas réunies :

1. **Dossier d'instruction écrit** : contexte chiffré, ≥ 2 alternatives chiffrées (coût/bénéfice), recommandation senior tracée. Pas de ratification verbale.
2. **Implémentation prête** : ≥ 50 % de l'implémentation de référence existe en code testé, OU jalon dur ≤ 4 semaines justifie la ratification anticipée.
3. **Cohérence ADR existantes** : pas de contradiction non-explicite avec les ADR §2 ratifiées. Toute révision rétroactive (vN→vN+1) suit la même règle.

**Liste blanche** :
- **Révisions mineures** d'une ADR ratifiée (clarifications, typos, mises à jour de chiffres factuels) : permises sans dossier complet, mention au CHANGELOG.
- **Parking explicite** d'une ADR ouverte vers v2/v3 avec critères de réouverture : permis (n'engage pas d'implémentation).
- **Révisions techniques bloquantes** suite à découverte d'incompatibilité tooling (e.g. ADR-05 v2 suite TC-llvmmos) : permises avec dossier raccourci documenté.

**Audit** : toute nouvelle ratification doit citer ce paragraphe et confirmer la conformité aux 3 conditions. Une ratification non-conforme est traitée comme bug d'architecture à corriger immédiatement.

Origine : critique architecte 2026-05-08 (cf. `BACKLOG.md` annexe) recommandait de geler les nouvelles ratifications jusqu'à 50 % d'implémentation. Le 2026-05-09, 3 ADR (19, 20, 21) ont été ratifiées en une journée, dont 2 révisées la même journée — pattern préoccupant. Le moratoire formalise le frein.

---

*Dernière révision : v0.2 — Phase 0 du programme état-de-l'art (2026-05-09) : ratification ADR-16 (driver model), ADR-17 (ABI syscall), ADR-18 (retrait 6502), parking ADR-15 v2, instauration du moratoire ADR.*
