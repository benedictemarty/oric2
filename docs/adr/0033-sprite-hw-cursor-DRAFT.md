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
