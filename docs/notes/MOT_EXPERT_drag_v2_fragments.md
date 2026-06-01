# Mot pour l'expert — v2 livrée, étirement résolu mais fragments restants

> HEAD : OricOS `ed3f9a9`, Phosphoric `36f1383`. v2 Option A implémentée
> (delta dérivé + bouton edge-detect) suite à `BUG_drag_glitch_taskmode(2).md`.

---

## 1. Ce qui est résolu

### Étage 1 (resize_hit armé à tort) : ✅ FIXÉ
Validation interactive : plus aucune fenêtre étirée verticalement après le fix.
`_wm_resize_hit` → `_point_in_rect16` lit maintenant `MOUSE_X/Y` qui contient
la position de l'event poppé (et non plus device read-clear). Resize armé
correctement selon la position du clic.

### v2 livrée (commit OricOS `ed3f9a9`)
`task_wm_install_event_state` copie event → MOUSE_* AVANT mouse_step :
- `MOUSE_X/Y` ← `$D4/$D6` (EVT_WHERE_X/Y)
- `MOUSE_BTN` ← `$D3` (EVT_MODS)
- `MOUSE_PREV_BTN` ← `WM_LAST_BTN` (edge-detect)
- `MOUSE_DX/DY` ← WHERE - WM_LAST (delta dérivé, low byte / sat8 implicite)
- `WM_LAST_X/Y/BTN` mis à jour pour iteration suivante

CODE budget : `TICK_COUNTER` pushed `$5400→$5430` pour +48 bytes (5 tests
mis à jour pour pointer la nouvelle adresse).

Validation headless : `test_oricos_ctl_taskmode_starve` + scenario drag →
**W/H préservés** après burst de moves + drag titlebar.

---

## 2. Symptôme restant — fragments multiples (cf. image)

Image jointe : après drag rapide de OricOS_0 d'une position vers la
gauche/haut, plein de **fragments visibles** :
- Plusieurs petits "x" rouges (close buttons fantômes) à différentes positions
- Texte "OricOS" en fragment à (~100, 397)
- Lignes/rectangles bleus à droite
- Barres grises hautes (>= 600px) en bas de l'écran (chrome ou fond mal placé ?)

Les fenêtres elles-mêmes (OricOS_0, Editor_0, Ctrl) sont à leurs nouvelles
positions, **taille correcte (W/H préservés)**, mais les ANCIENNES positions
ont laissé des fragments.

---

## 3. Hypothèse principale

**Coalescing MOVED dans `RAW_RING` rend le delta dérivé "grand saut"** au lieu
d'incrémental :

1. User drag rapide → SDL pousse 20 events MOVED en quelques ms
2. IRQ MOU2 fire 20× → 20 events poussés dans RAW_RING. Coalescing en place
   (`kernel_raw_push_mouse` line 451+) → tous coalescés en 1 SEUL entry,
   `EVT_WHERE_X/Y` = position FINALE (dernière)
3. task_wm pop 1 event seulement, computes `DX = WHERE_final - WM_LAST = 50px`
4. `mouse_step` :
   - `_wm_capture` rect at current pos
   - apply `DX=50` → window jumps 50px direct
   - `redraw_drag` erase OLD rect (largeur fenêtre, pas trajectoire)
   - redraw at NEW pos
5. **Si la fenêtre était à T-1 position P1 (pré-drag), à T position P2 (50px
   plus loin), mais le chrome a aussi été dessiné aux positions
   intermédiaires** par un mécanisme que je ne vois pas → fragments
   persistants

OU plus simple :
- L'image montre fragments à des positions très différentes (haut, milieu,
  bas) → drag PUIS drag inverse ?
- Chaque iteration laisse l'ancienne position blue-erase → si la fenêtre
  passait BEFORE par cette position, le chrome ETAIT dessiné mais pas erased
  parce que `WM_DRAG_OLD_*` ne capturait pas cette ancienne position

Bref : **`WM_DRAG_OLD_*` n'enregistre que LE rect précédant l'iteration
courante**, pas toute la trajectoire visitée depuis le DOWN. Si delta grand
(coalescing), trajectoire = saute → positions intermédiaires non couvertes
par les erases.

---

## 4. Test différentiel — RÉSULTAT

bmarty a testé drag en legacy mode (`WM_TASKMODE=$00`, sans `--wm-taskmode`)
avec scenario identique : **"aucun glitch"**.

→ **Hypothèse coalescing CONFIRMÉE**. taskmode produit des sauts trop grands
par iteration (delta dérivé = somme nette de N events coalescés), et
`kernel_wm_redraw_drag` n'erase que le rect précédent (pas la trajectoire
intermédiaire) → fragments à chaque position où la fenêtre est passée
DURANT l'iteration mais n'a pas été effacée.

Legacy n'a pas le problème car chaque IRQ = 1 mouse_step immédiat = 1 petit
delta = erase couvre l'incrément.

---

## 5. Si coalescing est confirmé, options de fix

### Option C (proposée) — task_wm loop sur events restants
Au lieu d'appeler `mouse_step` 1× par iteration `task_wm_entry`, **drainer
RAW_RING par chunks de N events**, appeler `mouse_step` N fois (chacune avec
le record poppé). Petit delta par appel → erase couvre.

Coût : task_wm tourne plus longtemps par activation. Mais elle est destinée
à drainer, donc cohérent.

### Option D — annuler le coalescing MOVED dans `raw_push_mouse`
Pousser TOUS les events MOVED individuellement → RAW_RING fill rapidement
(16 entries max). task_wm process chaque event séparément → petits deltas.
Risque : RAW_RING overflow sous events flood.

### Option E — fix architectural redraw_drag : couvrir tout le saut
`_wm_capture_focused_rect` capture POSITION+W/H. Si on capture aussi la
position pré-DOWN (sauvegardée au DOWN event) et la position courante,
construire un rect ENGLOBANT toute la trajectoire pour erase. Plus invasif.

Recommandation : **Option C** (le plus naturel, aligné avec l'invariant
"event = source de vérité").

---

## 6. Demande à l'expert

1. **Coalescing est-il bien le coupable des fragments restants ?** (analyse
   `_raw_fill_ww` + `kernel_raw_push_mouse` coalescing path).
2. **Option C (loop drain) est-elle la bonne réponse, ou Option E (rect
   englobant) plus robuste ?**
3. Y a-t-il d'autres mécanismes (icon redraw, taskbar) qui dépendraient du
   dirty-rect précis et qu'on casserait avec une cure naïve ?

Merci.

---

*Rédigé 2026-06-02. Étirement étage 1 RÉSOLU, fragments étage 2 EN COURS.*
