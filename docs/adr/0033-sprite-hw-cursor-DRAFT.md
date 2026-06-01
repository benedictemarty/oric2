# ADR-33 — Sprite hardware curseur (compositor) — DRAFT

> **Statut** : DRAFT (dossier d'instruction).
> **Date** : 2026-05-31.
> **Auteur** : bmarty + Claude Code.
> **Référence amont** : ADR-02 (compositor matériel), ADR-32 §9 (rétractation Étape 4 software).

---

## 1. Contexte et motivation

OricOS dessine le curseur souris en software : séquence atomique
`save backing 8×8 → draw 8×8 → restore au prochain mouvement`. En
contexte IRQ T1 c'était implicitement atomique (I=1). En contexte
tâche (`WM_TASKMODE=$A5`, ADR-32 Étape 4), le bug **curseur
invisible** apparaît : SEI/CLI étroits autour de la séquence ne le
résolvent pas (rétractation ADR-32 §9, 2026-05-31).

L'étude GEOS C64 (mist64/geos, 2026-05-31) révèle un fait
architectural : **GEOS n'a jamais résolu ce problème en software — il
l'évite via le sprite hardware VIC-II n°0**. Pas de backing store,
pas de save/draw/restore, pas de race possible. Le hardware compose
le sprite over le bitmap, ligne par ligne, automatiquement.

ADR-02 ratifie le compositor matériel pour Oric 2 (double ULA + mix).
Le sprite curseur s'inscrit naturellement dans ce compositor : un
overlay dédié, géré par registres I/O dédiés, indépendant de la
VRAM du framebuffer XVGA.

---

## 2. Décision proposée

Ajouter un **sprite hardware unique 16×16 4bpp avec transparence
par index $00** au compositor Oric 2.

### Caractéristiques

- **Taille** : 16×16 pixels (4× plus visible que cursor software actuel 8×8).
- **Format** : 4bpp indexé sur palette XVGA 16 couleurs (ADR-20).
- **Transparence** : pixel d'index $00 = transparent (background visible).
- **Position** : X 10-bit (0-1023), Y 10-bit (0-767), conforme MOU2.
- **Activation** : flag `$A5` (cohérent ADR-25 conventions).
- **Data** : 128 octets de bitmap sprite (16×16 × 4bpp).

### Composition

Pour chaque pixel `(px, py)` du framebuffer XVGA :
```
if SPR_ENABLE == $A5 and (px - SPR_X) in [0,16) and (py - SPR_Y) in [0,16):
    idx = (py - SPR_Y) * 16 + (px - SPR_X)
    byte = SPR_DATA[idx >> 1]
    nibble = (idx & 1) ? (byte & 0x0F) : (byte >> 4)
    if nibble != 0:
        output xvga_palette[nibble]
        continue
output xvga_palette[framebuffer_pixel]
```

### Registres I/O

Slot libre `$0370-$037F` (après MOU2 `$0360-$036F`).

| Offset | Reg              | R/W | Description                                |
|--------|------------------|-----|--------------------------------------------|
| $00    | SPR_X_LO         | R/W | Position X bits [7:0]                      |
| $01    | SPR_X_HI         | R/W | Position X bits [9:8] (bits 0-1)           |
| $02    | SPR_Y_LO         | R/W | Position Y bits [7:0]                      |
| $03    | SPR_Y_HI         | R/W | Position Y bits [9:8] (bits 0-1)           |
| $04    | SPR_ENABLE       | R/W | $A5 = on, autre = off                      |
| $05    | SPR_DATA_IDX_LO  | W   | Index data byte [7:0] (0-127)              |
| $06    | SPR_DATA_IDX_HI  | W   | Index data byte [8] (bit 0, unused)        |
| $07    | SPR_DATA         | R/W | Lit/écrit data[IDX], auto-incrémente IDX  |
| $08-$0F| réservés         | -   | retour $FF                                 |

Auto-increment évite 128 registres MMIO. Kernel upload bitmap en 128
écritures séquentielles à `$0377` après `STZ $0375 / STZ $0376`.

---

## 3. Alternatives écartées

### Alt-A : statu quo software + atomicité approfondie

Continuer en software, instruire les causes restantes (MOUSE_X non
updated en taskmode, DBR/DPR différent, etc.) et patcher. Coût :
itérations multiples sans garantie de stabilité ; complexité kernel
qui grandit. **Écartée** car GEOS prouve qu'il n'y a pas de bonne
solution software pure sur ce problème.

### Alt-B : sprite 8×8 mono (équivalent VIC-II strict)

Plus simple (8 oct data + 1 oct color), aligné GEOS. **Écartée** :
résolution XVGA 1024×768 rend un curseur 8×8 minuscule ; 16×16 est
le standard moderne (Windows, Mac, GEOS Wheels).

### Alt-C : sprite 16×16 mono + masque

8×8 × 2 plans = 16 oct + 1 byte color. Moins coûteux mais perd la
multi-couleur (flèche blanche bordée noire impossible). **Écartée**
au profit du 4bpp pour pouvoir rendre une flèche standard avec
contour.

### Alt-D : framebuffer overlay séparé (sprite multi-pixel à layer)

Allocation d'un 2e framebuffer XVGA dédié sprite, composition par
priorité. **Écartée** : surdimensionné pour un seul sprite curseur ;
coûteux en BRAM ECP5 (~96 KB pour XVGA dédié) ; pas réutilisable
sans repenser le pipeline complet.

---

## 4. Implications

### Phosphoric (logiciel)

- **Nouveau** : `src/io/sprite_device.c` + `include/io/sprite_device.h`
  (~200 lignes). État : 128 oct data + 6 registres + index courant.
- **Modifié** : `src/video/hires_oric2_xvga.c` — nouvelle fonction
  `hires_oric2_xvga_overlay_sprite()` appelée après render. ~30 lignes.
- **Modifié** : `src/main.c` — init device + I/O callbacks `$0370-$037F`
  + appel overlay après render. ~20 lignes.
- **Nouveau** : `tests/unit/test_sprite_device.c` + `tests/integration/
  test_oricos_sprite_cursor.c`.

### OricOS (kernel)

- **Modifié** : `kernel/modules/wm.s` — `kernel_wm_cursor_blit` et
  `kernel_wm_draw_cursor` deviennent : (1) écrire X/Y aux registres,
  (2) activer sprite. Suppression de `_cursor_save`, `_cursor_draw`,
  `_cursor_restore`, `_cursor_clamp` (clamp peut être fait avant
  d'écrire), `CURSOR_SAVE` (RAM backing 32 oct libérés).
- **Nouveau** : `kernel/data/cursor_sprite.s` — bitmap 128 oct flèche
  16×16 4bpp.
- **Init** : `boot.s` upload le bitmap au reset.

### ULX3S HDL (Phase HW)

- Logique compositor : comparateur position (`X - SPR_X`, `Y - SPR_Y`)
  + 8 entrées LUT × 16 (taille sprite), lookup 4bpp. ~50 LUT.
- BRAM : 128 oct sprite data = 1 demi-bloc BRAM (1 KiB minimum
  ECP5 dual-port) → trivial sur 416 KB disponibles.
- Total estimé : < 0.5% LUT / 1% BRAM ECP5 LFE5U-85F.

### Suppression code legacy

Une fois ADR-33 ratifié et migration kernel validée interactivement :
- Suppression des fonctions `_cursor_save_and_draw`, `_cursor_draw`,
  `_cursor_clamp`, `_cursor_next_row`, `_cursor_calc_addr`,
  `kernel_wm_cursor_save`, `kernel_wm_cursor_restore` dans wm.s.
- Suppression des ZP `CUR_DRAW_X`, `CUR_DRAW_Y`, `CUR_XB`, `CUR_MIDHI`,
  `CURSOR_VALID`, `CURSOR_OLD_X`, `CURSOR_OLD_Y`, `CURSOR_SAVE`.
- Mise à jour ADR-32 §9 : Étape 4 abandonnée définitivement au
  profit du sprite HW (ADR-33).

---

## 5. Plan d'implémentation incrémental

| # | Commit                                       | Validation                          |
|---|----------------------------------------------|-------------------------------------|
| 1 | ADR-33 DRAFT (ce dossier)                    | Lecture                             |
| 2 | sprite_device.c + tests unitaires            | `make tests` vert                   |
| 3 | I/O routing + overlay compositor + test pix  | Test PPM montre sprite              |
| 4 | Kernel : bitmap + refactor cursor → sprite   | Test interactif curseur visible     |
| 5 | Validation interactive --wm-taskmode         | Tous scenarios drag/resize/etc OK   |
| 6 | Cleanup code legacy + ratification ADR-33    | Suite verte + ADR §2                |

Validation interactive après chaque commit qui touche le rendu.

---

## 6. Critères de ratification

Conformément moratoire ADR (CLAUDE.md §10) :

1. ✅ **Dossier d'instruction écrit** (ce fichier).
2. ⏳ **Implémentation prête** : ≥ 50 % atteint après commit 4 (kernel
   refactor) + test interactif user OK.
3. ✅ **Cohérence ADR existantes** : extension naturelle ADR-02
   compositor, ferme ADR-32 §9 par construction. Aucune contradiction
   avec ADR-19 (VRAM unifiée — sprite est I/O séparé), ADR-20 (palette
   XVGA réutilisée), ADR-21 (GPU blitter indépendant — sprite ne passe
   pas par le blitter).

Ratification = à proposer après commit 5 (validation interactive
user). Si KO → rétractation ADR-33, retour software + investigation
hypothèses ADR-32 §9.

---

## 7. Références

- ADR-02 : Compositor matériel double ULA + mix.
- ADR-32 §9 : Rétractation Étape 4 (motivation initiale).
- `reference_geos_cursor_design.md` (mémoire) : étude GEOS qui guide
  ce design.
- mist64/geos `kernal/sprites/sprites.s`, `kernal/mouse/mouse2.s` :
  référence implémentation upstream.
- ULX3S LFE5U-85F : 416 KB BRAM, 84K LUT ECP5 (cible HDL).

---

*Rédigé 2026-05-31 — dossier d'instruction Claude Code suite
rétractation ADR-32 §9 et étude GEOS comparative.*

---

## 8. Implémentation livrée (2026-06-01)

Phases A-D livrées, plusieurs cycles d'instruction/debug nécessaires :

**Phase A** : ADR-33 DRAFT (ce dossier).
**Phase B** : `sprite_device.c` (sprite 16×16 4bpp transparent) + 8 tests
unitaires PASS. I/O `$0370-$037F`.
**Phase C** : Wiring compositor — `sprite_overlay()` appelé après
`hires_oric2_xvga_render` (rendu SDL + screenshot). Routing I/O
`$0370-$037F` → `sprite_read/write`. Champ `emu->sprite` ajouté à
`emulator_t`.
**Phase D** : Kernel — `kernel_wm_draw_cursor`/`cursor_blit` réécrits
en écriture sprite HW. Bitmap **PC/GEOS pBasic authentique** (porté
depuis `bluewaysw/pcgeos Library/SpecUI/CommonUI/CUtils/cutilsVariable.def`
lignes 567-608, identique aux 3 styles UI Motif/CUA/Open Look).

### Bugs résolus durant l'implémentation

1. **DBR ignoré par Phosphoric** — `cpu816_mem_read` n'utilise pas le
   DBR (commentaire « strict bus 16-bit B1.7 »). `lda f:label,X` avec
   label en bank 1 résout à bank 0 → sprite lit garbage. **Fix** :
   `lda [DP_PTR],y` (long indirect indexed, opcode `$B7`) avec bank
   byte explicite en ZP.
2. **Mode M=8/X=8 non garanti** — ca65 `.smart` suit l'état lexical
   mais le runtime peut différer. `lda #$01` en M=16 devient
   `lda #$4801` (3 bytes) → décalage. **Fix** : `sep #$30` + `.a8 .i8`
   explicites à l'entrée de `sprite_init` et `draw_cursor`.
3. **`bitmap` non propagé** — ca65/ld65 n'attache pas le bank byte au
   label `cursor_bitmap_data`. **Fix** : voir #1 (bank `$01` codé dur
   en ZP).
4. **Tests adaptés** : `test_oricos_cursor_backing_store` → SKIP
   (obsolète par construction). `test_oricos_wm_server`/
   `taskmode_full` : `CURSOR_OLD` → alias `MOUSE_X/Y`.

### Validation interactive utilisateur

- **Test 1 — mode legacy** (`--ctl-demo` sans `--wm-taskmode`) :
  flèche PC/GEOS visible, suit la souris. ✅
- **Test 2 — mode taskmode** (`--ctl-demo --wm-taskmode`) :
  flèche visible mais **ne suit pas la souris**. ❌

---

## 9. Diagnostic test 2 (validation interactive KO)

Bug isolé : sprite ne suit que **quand `ctl_demo` est actif**. Sans
app (juste kernel + WM en taskmode), le sprite suit normalement
(`test_oricos_taskmode_full` PASS).

Reproduit en headless : `test_oricos_ctl_taskmode_starve` (ADR-33 §10
nouveau target). 10 mouse moves rapides avec
`TC_CTLAPP_FLAG + TC_WM_FLAG + WM_TASKMODE = $A5` → **0/10 moves
suivent**.

### Symptômes observés (instrumentés)

- `RAW_RING_COUNT = 1` stable (event pendant non drainé)
- `RAW_WAITER = 0` (task_wm pas en wait, ou wake l'a clear)
- `TCB[task_wm].state = 03 BLOCKED` (jamais réveillée)
- task_wm tourne ≥ 255 fois pendant le boot (settle 3M cycles), puis
  **0 itération** pendant les moves (compteur reset à 0)
- `MOUSE_BTN = 00` (pas de bouton, donc cursor_blit DEVRAIT être appelé
  via la branche `motion-only`)
- `FORBID_COUNT = 0` (Forbid n'est pas tenu)
- Scheduler alterne entre pid 1 (task_a), 2 (task_b), 3 (task_c) ;
  task_wm (pid 8) jamais picked

### Tentatives de fix sans succès

- `php / sep #$30 / plp` autour de `kernel_wm_draw_cursor` (M-mode
  garanti)
- `save/restore SCHED_PTR` dans `kernel_raw_wake` (race ZP ADR-32 §3.3a
  hypothèse) — pas d'effet observable

### Hypothèses restantes (non instruites)

1. **TCB corruption** : `TCB_TABLE_BASE = CHARSET_XVGA_SRC = $015C00`.
   Le charset (1024 oct) couvre TCB[1..16]. Si le charset est uploadé
   au boot et la zone repurposée pour TCB, les écritures de state TCB
   se font dans la même mémoire. Une écriture mal placée pourrait
   corrompre task_wm.STATE. **À vérifier** : kernel_install_charset
   est-il vraiment terminé avant les premiers kernel_task_create ?
2. **kernel_raw_wake clear RAW_WAITER prématuré** : peut-être WAITER
   est clear avant que le scheduler ne consulte. Race scheduler.
3. **ctl_demo SYS_GFX_FILL_RECT massif** : si ctl_demo tient
   suffisamment longtemps le CPU en sys_gfx, l'IRQ MOU2 reste
   masquée. Mais T1 scheduler IRQ devrait toujours préempter.
4. **kernel_app_spawn** crée le task ctl_demo avec un pid > 8 mais
   sa TCB est dans la zone charset (overlap §10.1).

### Prochaine instruction

Session dédiée (1-2h+). Plan d'attaque :
1. **Vérifier l'overlap charset/TCB** : dump charset bytes initial à
   $015C00 vs TCB après boot — comparer. Si TCB écrite par tâche après
   `kernel_install_charset` → OK. Sinon, BUG critique.
2. **Tracer kernel_sched_find_next** : à chaque IRQ T1, log pid choisi.
   Si task_wm jamais choisi malgré state=READY momentané → bug sched.
3. **Tracer raw_wake** : à chaque appel, log RAW_WAITER avant/après et
   TCB[waiter].STATE avant/après. Identifie si raw_wake EST appelée et
   ce qu'elle fait.

---

## 9bis. Bug RÉSOLU : course atomicité GPU_BPL (2026-06-01)

**Diagnostic externe** : `BUG_curseur_fige_gpu_bpl.md` (analyse indépendante).

Mon §9 ci-dessus pointait sur task_wm starve — c'était la **mauvaise piste**
pour le scenario interactif default. Le vrai bug en mode legacy (qui est le
default `WM_TASKMODE=$00`) :

**Cause** : `kernel_gfx_fill_rect16` faisait `jsr _gfx_xvga_bpl_guard`
AVANT son propre `php;sei`. Le guard a sa propre section critique (envoie
une commande SET_BPL GPU complète). Trou I=0 entre le `plp` du guard et le
`sei` de fill_rect16. IRQ MOU2 fire dans le trou → `mouse_step` →
`kernel_gfx_set_bpl` (son propre SET_BPL) entrelacé avec le FILL en cours.
Le GPU reçoit `SET_BPL(stride X) → SET_BPL(stride 0) → FILL` → framebuffer
peint au mauvais stride → sprite composé par-dessus invisible/hors écran.

**Fix** (commit `eddf8ac`) : déplacer `php;sei` AVANT `jsr _gfx_xvga_bpl_guard`
dans `kernel_gfx_fill_rect16`. Section critique élargie englobant guard+fill.
Le `sei` interne du guard reste correct (nesting propre, `plp` restaure I=1
du sei externe).

**Validation interactive utilisateur (2026-06-01)** : SDL legacy mode +
`--ctl-demo` + stress mouse → curseur reste fluide. ✅

**Reste à faire (§4.4 du dossier)** : étendre le même pattern aux autres
commandes composites (`kernel_wm_redraw`, `wcmp`, autres helpers GPU qui
appellent `set_bpl` puis enchaînent). Audit + fix systématique à instruire.

**Pourquoi mon §9 était à côté** : j'avais focalisé sur le test 2
(`--wm-taskmode`) qui a effectivement un bug différent (task_wm starve §10
ci-dessous). Le diagnostic externe a recadré : le bug curseur figé du test 1
(legacy default) est antérieur, lié à ADR-27 patches BPL, orthogonal au
sprite ADR-33.

---

## 10. Bug ouvert : `task_wm` starve avec `ctl_demo` + `WM_TASKMODE` (2026-06-01)

**Statut** : ouvert, instruction reportée à une session dédiée.

**Repro** : `test_oricos_ctl_taskmode_starve` (Phosphoric, hors
aggrégateur `tests` car FAIL intentionnel).

**Impact** : `--wm-taskmode` (ADR-32 Étape 4 v2 via sprite HW) ne
peut pas être ratifié comme default tant que ce bug n'est pas résolu,
parce que tout scenario réel inclut au moins une app utilisateur.

**Workaround actuel** : ne PAS poser `WM_TASKMODE=$A5` par défaut. Le
mode legacy (`WM_TASKMODE=$00`) marche parfaitement avec le sprite HW
PC/GEOS. C'est suffisant pour le sprint courant : sprite HW remplace
le backing store software, ferme ADR-32 §9 par construction pour le
chemin legacy.

**Ratification ADR-33 partielle** : ratifier sur le scenario legacy.
ADR-32 §9 reste partiellement ouverte (Étape 4 v2 via sprite + taskmode
demande la résolution du §10 ci-dessus).
