# Mot pour l'expert — état session 2026-06-02

> Merci pour les 3 dossiers d'instruction successifs (lost-wakeup → instrumentation
> pid 8 → RMW mem M16). La méthode « pas de fix sans preuve » a payé : la cause
> racine était bien là où l'instrumentation a pointé. Petite synthèse + un
> nouveau symptôme à creuser.

---

## 1. Ce qui est résolu

### §10 task_wm starve → CLOS
Cause racine : **bug d'émulation Phosphoric** comme suggéré dans
`BUG_phosphoric_rmw_mem_M16.md`. Confirmé par l'instrumentation :
`asl SCHED_TMP` au 7ᵉ shift (mask `$0080`) donne `$0000` au lieu de `$0100`
en M=16 → bit 8 jamais set → task_wm pas allouée → ctl_demo prend pid 8.

**Commits livrés** :
- `08a9fbf` (Phosphoric) : ASL mem M-aware (4 opcodes, fix immédiat du symptôme)
- `748a8b7` (Phosphoric) : LSR/ROL/ROR/INC/DEC mem M-aware (20 opcodes restants)
  via 6 helpers `static inline` qui réutilisent `read_M`/`write_M`
- Tests : `test_oricos_ctl_taskmode_starve` → **10/10 OK** (était 0/10), suite
  globale 24/24 verte
- Notes : commentaire `raw_wake` corrigé pour ne plus pointer la fausse piste
  SCHED_PTR (§5 du dossier précédent)

### Bug GPU_BPL race → CLOS
Commit antérieur `eddf8ac` (audit §4.4 commit `e74d714` étend aux 5 commandes
composites). Validation interactive utilisateur OK.

### ADR-33 sprite HW → RATIFIÉ
Sprite PC/GEOS pBasic authentique. Bitmap porté depuis `bluewaysw/pcgeos`.
Visible interactif en mode legacy ET maintenant taskmode (depuis fix §10).

---

## 2. Nouveau symptôme — drag glitch en `--wm-taskmode`

**Reproduit dans l'image jointe** (Editor.png) : avec `--ctl-demo
--wm-taskmode`, quand on **drag une fenêtre par la titlebar** (Editor, Ctrl,
OricOS — tous concernés), des artéfacts apparaissent :

- La fenêtre déplacée se retrouve **étirée** verticalement OU réduite à son
  chrome seul
- Des **fragments** restent à l'écran (titlebar text "o", lignes horizontales,
  petits restes)
- Le curseur sprite est intact et suit la souris correctement

**Test différentiel** : aucun glitch en mode **legacy** (sans `--wm-taskmode`).
Avec ctl_demo + legacy : drag fluide, aucun artéfact.

→ Bug **spécifique au mode `WM_TASKMODE=$A5`** (mouse_step en task vs IRQ).

### Ce que j'ai exclu
- Pas mes fixes RMW M-aware : si c'était ça, legacy serait aussi cassé.
- Pas le sprite HW : sprite est overlay au compositor, pas dans VRAM des
  fenêtres.
- Grep `inc/dec/asl/lsr/ror/rol` mem dans tout le kernel : 6 occurrences
  totales, toutes connues et hors path drag (sched.s SCHED_TMP, gfx.s
  GFX_BPL, et 3 dans dead code ADR-33).

### Hypothèse principale (à instruire)
Le path `wm_step_pressed` → `wm_step_do_drag` → `kernel_wm_move_focused`
utilise `MOUSE_DX/DY` (deltas signés par event) + `MOUSE_BTN`/`MOUSE_PREV_BTN`
pour distinguer click/drag/release.

En **IRQ mode** : mouse_step run sous sei juste après kernel_mouse_read →
PREV_BTN et DX/DY cohérents avec l'événement courant.

En **task mode** : mouse_step retardé. Entre l'IRQ qui lit MOU2 et le moment
où task_wm processe, d'autres IRQ peuvent **overwrite MOUSE_DX/DY/BTN** (read-
clear sur device). task_wm voit donc les valeurs du **DERNIER** IRQ, pas
celles de l'event qu'il pop dans RAW_RING. Désynchro entre event-state et
device-state.

Conséquence plausible : la logique `cmp PREV_BTN` mal-identifie l'état drag
→ `_wm_do_resize` déclenché à la place de `_wm_do_drag` → modifie W/H au
lieu de X/Y → fenêtre étirée.

### Ce qui est livré pour avancer
Extension de `test_oricos_ctl_taskmode_starve` avec scenario drag :
- Capture W/H de la fenêtre Editor (slot 1) avant drag
- Click titlebar + 5 mouse moves + release
- Re-lecture W/H
- FAIL si W ou H changé

Si le test repro le bug headless, on a une boucle de fix-test rapide.
Sinon (drag glitch nécessite timing interactif), il faudra autre angle.

### Question / aide demandée

1. Une intuition sur les **vrais coupables** côté task_wm dans le path drag ?
   Notamment : `_wm_do_resize` pourrait-il se déclencher sans hit
   resize-margin valide si `WM_RESIZE_ARMED` est dans un état foireux ?
2. Si le repro headless ne fait pas apparaître le bug, **comment forcer la
   désynchro MOUSE_DX vs RAW_RING event** dans le test ? PC-hook pour
   différer task_wm wake après que plusieurs IRQ aient overwritten DX/DY ?
3. Y a-t-il un dossier d'instruction approprié à instaurer (style audit
   §3.3a ZP race mais centré sur l'invariant DX/DY/BTN ↔ RAW_RING event) ?

Merci, je continue le harness en parallèle.

---

*Rédigé 2026-06-02. HEAD : OricOS `d29de0b`, Phosphoric `748a8b7`,
workspace `50ce5b7`.*
