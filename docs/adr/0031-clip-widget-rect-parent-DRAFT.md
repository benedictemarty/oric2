# ADR-31 (DRAFT) — Clip des widgets hors rect window parent

- **Statut** : **ouverte — dossier d'instruction** (2026-05-30). DRAFT non
  ratifié.
- **Date d'ouverture** : 2026-05-30
- **Décideurs** : bmarty (arbitrage), Claude Code (instruction)
- **Contexte technique** : bug visuel observé interactivement par
  l'utilisateur le 2026-05-30 lors de la validation d'ADR-30 Étape 1
  (`GU_LIST`) : après resize-down d'une fenêtre, ses widgets dont
  `rel.y + h > window.h` restent visibles **en dehors** du rect window.
  Bug pré-existant à ADR-30 (touche tous les widgets), simplement plus
  visible avec `GU_LIST` qui prend ~48 px en hauteur. ADR liées : ADR-26
  (GUI déclaratif), ADR-28 §6.5 (damage tracking évoqué comme refinement
  non-bloquant), ADR-30 (révèle le bug en pratique).
- **Conformité moratoire** : oui — dossier d'instruction (CLAUDE.md §10
  cond. 1). Aucune ADR ratifiée modifiée.

## 0. Résumé exécutif

OricOS n'a actuellement **aucun mécanisme** de clipping des widgets par
rapport au rect de leur fenêtre parent. Quand un widget est dessiné par
`_wm_draw_widget_body`, ses coords absolues `(window.x + rel.x,
window.y + rel.y, w, h)` sont passées directement aux primitives GPU
(`kernel_gfx_fill_rect16`, `TEXT16`, etc.) sans test de containement.

**Manifestation** : si l'utilisateur réduit une fenêtre de sorte que
`rel.y + h > window.h` pour un widget, ce widget est peint à des
coordonnées absolues qui dépassent le bas du nouveau rect window. Comme
`kernel_wm_redraw_drag` efface l'ancien rect window (plus large) puis
redessine les fenêtres, le widget débordant est visible sur la zone "ex-
fenêtre" non recouverte par le nouveau rect.

3 options chiffrées :
- **A — Skip widget hors rect** (10-15 LOC) : si widget rect ne tient pas
  dans window rect, ne pas le dessiner. Simple, correct, mais retire
  visuellement le widget abruptement.
- **B — Clip partiel** (40-60 LOC) : clamp `WM_ARG_W/H` du widget au rect
  window avant fill. Le widget est dessiné jusqu'au bord puis coupé. Plus
  joli.
- **C — Clip-list / damage tracking complète** (refactor majeur, ~500+ LOC)
  : système de régions clip avec damage rectangles. Aligné avec ADR-28
  §6.5 (déjà tracé non-bloquant). Vraie solution architecturale GeoWorks-
  like.

**Recommandation senior** : **Option A** pour court terme (résout le bug
visible avec peu de code), Option C tracée comme refinement long terme
(ADR-28 §6.5). Option B = compromis non-recommandé (coût intermédiaire
sans la propreté architecturale de C).

## 1. Contexte chiffré (observation)

### 1.1. Scénario de reproduction

1. Lancer `./oric1-emu --kernel ../OricOS/build/kernel.bin --xvga
   --ctl-demo`.
2. Activer la capture souris (clic dans la fenêtre Phosphoric).
3. La fenêtre "Ctrl" affiche 5 widgets : check, scroll_v, text, list.
4. Drag le bord bas vers le haut pour réduire la fenêtre à h ≈ 60-80 px.
5. **Observation** : la liste `GU_LIST` (à rel.y=72, h=48) reste peinte
   visuellement en dehors du nouveau rect window, sur le bureau bleu.
6. Le scroll_v (rel.y=14, h=100) déborde aussi.

### 1.2. Analyse code

Chemin de redraw au resize :
- `wm.s wm_step_do_resize` → `_wm_do_resize` →
  `kernel_wm_redraw_drag` + `kernel_wm_draw_cursor`.
- `kernel_wm_redraw_drag` efface l'ancien rect window
  (`WM_DRAG_OLD_X/Y/W/H`, peint en bleu desktop) puis appelle
  `_wm_draw_windows`.
- `_wm_draw_windows` itère sur le Z-order et appelle `_wm_draw_one` par
  fenêtre.
- `_wm_draw_one` peint le frame/titre/body de la fenêtre **au rect courant**
  (post-resize) puis appelle `_wm_draw_widgets_for_slot`.
- `_wm_draw_widgets_for_slot` itère sur `WIDGET_TABLE`, pour chaque widget
  dont `WG_PARENT == slot`, calcule `abs.x = window.x + rel.x`,
  `abs.y = window.y + rel.y` et appelle `_wm_draw_widget_body`.
- `_wm_draw_widget_body` dispatch selon `WG_TYPE` vers les paint primitives
  (`kernel_tk_button`, `kernel_tk_scrollbar`, `kernel_tk_list`, etc.).
- **Aucun test** que `(abs.y + h) ≤ (window.y + window.h)` n'est effectué.

### 1.3. Impact

Le bug touche **tous les widgets** dont le rect peut sortir du rect window
après resize-down. Plus visible avec :
- Widgets occupant beaucoup d'espace (`GU_LIST`, `GU_VIEW`).
- Widgets en bas de fenêtre.
- Resize agressif (réduction de >50 % de la hauteur).

Aucun impact fonctionnel direct (l'app continue de fonctionner, le widget
reste cliquable même hors rect — autre bug séparé éventuellement, mais
le hit-test calcule aussi en coords abs sans clip). Impact purement
visuel mais perturbe l'UX.

## 2. Problème

OricOS hérite d'un modèle « peinture directe au framebuffer XVGA » : pas
de backing store par fenêtre (cf. ADR-27 backing store en cours), pas de
clip-list, pas de damage tracking. C'est cohérent avec la simplicité du
système mais inadapté aux opérations où la **taille du rendu** dépasse la
**taille du conteneur logique** (cas du resize-down).

Les OS de référence ont tous résolu ce problème :
- **GeoWorks** : `VisCompClass` / `VisContentClass` font du clipping
  hiérarchique automatique. Le clip est une property de chaque objet
  visible.
- **Intuition (Amiga)** : `Layer` system avec clip-lists par fenêtre,
  redraw automatique sur invalidation.
- **SymbOS** : redraw plein avec invalidation par fenêtre (modèle plus
  simple, plus proche d'OricOS).

OricOS doit choisir entre :
1. Pas de clip (statu quo, accepté).
2. Clip minimal (skip widget hors rect = A).
3. Clip riche (B = clipping partiel par widget).
4. Refactor complet (C = clip-list / damage tracking, vraie solution).

## 3. Options envisagées

### Option A — Skip widget si hors rect parent — **recommandée court terme**

Modifier `_wm_draw_widgets_for_slot` (ou `_wm_draw_widget_body`) pour
tester avant le dispatch :

```asm
; pseudo-code
if (rel_y + widget_h > window_h) || (rel_x + widget_w > window_w):
    skip ce widget (rts sans draw)
```

- **Coût** : ~15 LOC asm (tests cmp + bcs vers rts).
- **Bénéfice** : résout 100 % du symptôme visuel observé. Widgets non
  visibles dans le rect window sont simplement absents (rect vide).
- **Limites** : pas de clip partiel — un widget dont la moitié déborde
  disparaît entièrement. Acceptable v1 (l'utilisateur sait qu'il a réduit
  la fenêtre).
- **Réversibilité** : élevée (1 patch local).

### Option B — Clip partiel (clamp WM_ARG_W/H)

Calculer pour chaque widget la portion intersectée avec window rect, puis
passer `WM_ARG_W/H` clamped au GPU. Demande de calculer une intersection
de rectangles et de clamper.

- **Coût** : ~50 LOC asm (tests + 4 clamps + adjustments).
- **Bénéfice** : widget partiellement visible jusqu'au bord du rect window.
  Plus joli UX.
- **Limites** :
  - Le widget partiel peut être inutilisable (label coupé, scrollbar mi-
    thumb).
  - Le redraw widget n'est pas idempotent vis-à-vis du clamping (ex. `tk.s
    kernel_tk_scrollbar` calcule des positions thumb relatives à la course
    totale H, pas à H clipped — clip donne un thumb visuel mal placé).
  - Demande des tests pixel-précis ou validation visuelle systématique.
- **Réversibilité** : moyenne (touche plusieurs paints).

**Écartée** : coût intermédiaire sans qualité de l'option C, et l'option A
suffit pour le symptôme observé.

### Option C — Clip-list / damage tracking architectural

Vraie solution GeoWorks-like. Chaque fenêtre tient une clip-list (liste
de rectangles « visibles »). Les paints passent par un système de
dispatch qui clip chaque opération par la clip-list. Damage rectangles
tracent quoi repeindre après chaque modification.

- **Coût** : ~500-1000 LOC asm. Refactor majeur de tous les chemins de
  paint. Touche `_wm_draw_windows`, `_wm_redraw`, `_wm_redraw_drag`,
  `_wm_draw_widget_body`, et tous les `kernel_tk_*`.
- **Bénéfice** :
  - Vraie correction architecturale.
  - Performance gain potentiel (ne pas repeindre les régions non
    invalidées).
  - Aligne OricOS avec GeoWorks / Intuition / Layer systems.
  - Couplable avec ADR-27 (backing store par fenêtre).
- **Limites** :
  - Très gros chantier.
  - Risque de régressions multiples.
  - Hors scope ADR-31 isolée.
- **Réversibilité** : faible (refactor structurel).

**Tracé comme refinement long terme** : voir ADR-28 §6.5 (damage tracking
déjà mentionné comme non-bloquant). Une ADR future dédiée pourra
l'instruire en détail. Pas dans ADR-31.

## 4. Recommandation senior (tracée)

**Viser l'option A** maintenant (résout le bug avec ~15 LOC), tracer
l'option C comme refinement long terme dans une ADR future séparée (à
synchroniser avec ADR-27 backing store et ADR-28 §6.5 damage tracking).

Justification :
1. **Pragmatisme** : le bug observé est purement visuel et l'option A le
   résout. Pas la peine d'engager un refactor majeur pour un bug
   d'UX edge case.
2. **Cohérence avec l'esprit OricOS** : minimal viable d'abord, refactor
   structurel quand les conditions sont mûres (backing store ADR-27
   ratifié, damage tracking ADR-28 §6.5 décidé).
3. **Réversibilité élevée** : option A est un patch local trivialement
   reverté si on adopte C plus tard.
4. **Risque minimum** : pas de touche aux paints existants, juste un test
   préalable.

## 5. Conséquences

### 5.1. Positives (option A)

- Bug visuel observé éliminé pour 100 % des widgets.
- Comportement déterministe (widget visible ou absent, jamais partiel).
- Patch local, ne touche pas les paints individuels.

### 5.2. Négatives / coûts

- Widget partiellement coupé devient invisible abruptement (mauvaise UX
  edge case). Acceptable v1.
- Ne résout pas les autres symptômes architecturaux (pas de damage
  tracking, redraw full à chaque resize).

### 5.3. Sur ADR-27 (backing store fenêtre)

ADR-31 est **compatible** avec ADR-27 : quand le backing store par fenêtre
sera ratifié (option b), le rendu sera contraint à la taille du backing,
donc le clip sera implicite. ADR-31 deviendra alors **obsolète** (à
parquer ou retirer).

### 5.4. Sur ADR-28 §6.5 (damage tracking)

ADR-31 est **précurseur léger** d'un éventuel ADR damage-tracking complet
(option C). Si C est instruit plus tard, A devient un cas particulier
trivialement absorbé par C.

## 6. Plan d'implémentation (option A)

### 6.1. Étape 0 — Reproduction headless

- Étendre un test (e.g., `test_oricos_window` ou ajouter
  `test_oricos_widget_clip`) qui crée une fenêtre + widget en y dépassant
  la hauteur, et vérifie post-`kernel_wm_redraw` que le widget rect n'a
  pas écrit dans VRAM en dehors du window rect.
- Gate : `make tests` vert (test échoue avant fix → confirme reproduction).

### 6.2. Étape 1 — Implémentation skip-si-hors-rect

- Modifier `_wm_draw_widgets_for_slot` (`tk.s`) ou `_wm_draw_widget_body`
  (selon ce qui est plus net) pour ajouter le test de containement avant
  dispatch :
  - Lit `rel.x`, `rel.y`, `w`, `h` du widget.
  - Lit `window.w`, `window.h` du parent.
  - Test : `rel.x + w > window.w` OR `rel.y + h > window.h` → return
    sans draw.
- Test post-implémentation : le test d'étape 0 doit passer.
- Validation interactive : refaire le scénario user (resize-down `ctl_demo`)
  et vérifier que les widgets disparaissent proprement.

### 6.3. Étape 2 — Ratification

Si retour interactif positif :
- Marquer ADR-31 ratifiée.
- Renommer fichier (retirer DRAFT).
- Update CLAUDE.md §3 → §2.
- Update ADR_SUMMARY.

## 7. Points ouverts

1. **Lieu exact du test** : `_wm_draw_widgets_for_slot` (avant le call
   `_wm_draw_widget_body`) ou `_wm_draw_widget_body` (au début) ? Le
   premier est plus économe (test une fois par widget) mais accès aux
   champs widget via offset. Le second est plus simple à coder.
2. **Hit-test** : le hit-test (`_wm_widget_hit`) doit-il aussi être clip ?
   Sinon, un widget invisible reste cliquable (bug latent). Recommandation :
   oui, ajouter le même test au hit-test pour cohérence.
3. **Performance** : 2-4 cmp + bcs additionnels par widget par redraw. Coût
   marginal (~20-30 cyc/widget). Négligeable.
4. **Cohérence avec ADR-27** : quand backing store par fenêtre sera
   ratifié, ADR-31 devient redondante. Documenter cette obsolescence
   anticipée.

## Références

### Internes OricOS
- CLAUDE.md §2 (ADR-26 GUI déclaratif), §3 (ADR-27 backing store, ADR-28
  §6.5 damage tracking).
- `docs/adr/0027-backing-store-fenetre-DRAFT.md` (rendra ADR-31 obsolète
  à la ratification).
- `docs/adr/0028-threading-window-manager.md` §6.5 (damage tracking
  long-terme).
- `docs/adr/0030-roadmap-toolbox-DRAFT.md` §7.1 (Étape 1 GU_LIST a révélé
  le bug).
- `kernel/modules/tk.s` (`_wm_draw_widgets_for_slot`, `_wm_draw_widget_body`).
- `kernel/modules/wm.s` (`kernel_wm_redraw`, `kernel_wm_redraw_drag`).

### Externes (références conceptuelles)
- [PC/GEOS gListC.def](https://github.com/bluewaysw/pcgeos/blob/master/Include/Objects/gListC.def)
  (le widget qui a révélé le bug).
- GeoWorks `VisCompClass` / `VisContentClass` : hiérarchie de clip
  automatique (modèle long terme pour OricOS).
- Intuition `Layer` system : clip-list par fenêtre.
- SymbOS : redraw plein avec invalidation par fenêtre (modèle proche
  d'OricOS actuel).

## Méta — Particularité de cette ADR

ADR-31 est un **mini-ADR** sur un bug très spécifique. Elle est ouverte
parce que :
1. Le bug a été révélé en cours de validation d'ADR-30 Étape 1.
2. Il n'est pas spécifique à ADR-30 (touche tous les widgets).
3. La fix est triviale (option A) mais mérite trace pour cohérence.
4. ADR-27 et ADR-28 §6.5 traiteront la cause architecturale plus tard ;
   ADR-31 trace le patch minimum pour combler en attendant.

Une fois ratifiée et fix mergé, ADR-31 pourra être marquée **obsolète**
quand ADR-27 (backing store) ou un futur ADR damage-tracking absorbera
le besoin.
