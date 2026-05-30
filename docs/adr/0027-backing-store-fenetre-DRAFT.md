# ADR-27 (DRAFT) — Modèle de backing store fenêtre

- **Statut** : **DRAFT — option (b) retenue, ratification 2026-05-30q
  RÉTRACTÉE le 2026-05-30t** après validation interactive `--compact`.
  Constats : la plomberie compact (Étapes A + B1 + B2) leak `bpl` vers
  des chemins kernel direct non-instrumentés (`kernel_menu_draw`,
  `_wm_draw_one` chrome, etc.) → rendu corrompu en interaction réelle
  (carrés noirs aux positions des fenêtres système ; menu déroulé
  multiplié sur l'écran après clic). Le test unitaire
  `test_oricos_compact_backing_store` (12/12) couvrait un cas trop
  restreint (compose seul, sans interaction).
- **État actuel du code** : plomberie A/B1/B2 + hardening M=8 conservée
  comme **dormante** (`WM_COMPACT_FLAGS` reste à 0 partout, comportement
  runtime identique au pré-ADR-27). 24/24 suites Phosphoric vertes.
  task_compact, flag CLI `--compact`, test unitaire = revertés.
- **À instruire avant ré-ratification** : audit exhaustif des chemins
  de dessin kernel direct, plus tests d'intégration ciblés par chemin
  (menu, taskbar, chrome, dialogues), pas uniquement le compose-loop.
- **Date d'ouverture** : 2026-05-27
- **Date de ratification (rétractée)** : 2026-05-30q (rétractée 2026-05-30t)
- **Décideurs** : bmarty (choix option b + dé-ratification), Claude Code
- **Origine** : audit GPU toolbox senior (2026-05-27), Finding B. Découvert
  à l'occasion du fix BLIT v0.2 (byte_w/byte_h 16-bit, Phosphoric 1.22.87).

## 0. Décision et état d'avancement (2026-05-27)

L'humain a retenu l'**option (b)** : stride GPU (BPL) configurable + backing
store compact. Avancement :

- ✅ **Référence GPU implémentée** (Phosphoric 1.22.88) : registre `bpl`
  persistant (défaut 512) + opcode `GPU_OP_SET_BPL` ($08) ; BLIT utilise
  `bpl` pour la SOURCE et 512 (XVGA) pour la destination ; FILL_RECT*/LINE/
  TEXT* honorent `bpl`. Helper kernel `kernel_gfx_set_bpl`. 2 tests unitaires
  (`test_set_bpl_changes_fill_stride`, `test_blit_compact_source_stride`).
  **Rétro-compatible** : défaut 512 → comportement identique (594 tests verts).
- ✅ **Concurrence option 2 implémentée** (OS-gpu-race, 2026-05-27) : les 7
  helpers GPU sont bracketés `php;sei … plp` → chaque commande GPU est atomique
  vis-à-vis des IRQ (plus de clobber ARG/CMD par un mouse IRQ). Corrige aussi la
  race ARG préexistante. Transparent (594 verts). C'est le **pré-requis** au flip
  compact.
- ⏳ **Reste (flip compact + option 1, NON fait — risque/bénéfice à arbitrer)** :
  cf. §0ter pour le plan précis des points de gestion `bpl`. Bénéfice immédiat
  **limité** (les apps actuelles tiennent déjà en 1 banque ; les fenêtres
  hautes *pleine largeur* nécessitent en plus l'allocation multi-banques —
  indépendante de la stride). Recommandation : **bundler le flip compact avec
  l'allocation multi-banques** en un sprint délibéré, plutôt que de modifier le
  compositeur (cœur testé) pour un gain partiel.
- ✅ **Étape A livrée (2026-05-30n)** : plan en 3 étapes incrémentales retenu
  (vs sprint atomique du DRAFT). Étape A = **plomberie passive du shadow `bpl`**.
  Posé : constante `GFX_BPL_SHADOW = $016900` (2B bank 1), maintien dans
  `kernel_gfx_set_bpl`, helper `kernel_gfx_get_bpl_shadow`, init = 0 dans
  `kernel_wm_init`. Aucun appelant ne pose encore `set_bpl` → shadow = 0,
  comportement runtime **identique** (24/24 suites vertes). Leçon `.smart`
  enregistrée : `.a16` non refermée pollue les helpers voisins → entrée
  arbitraire des helpers = `php/sep #$20/.../plp` systématique.
- ✅ **Étape B1 livrée (2026-05-30o)** : garde IRQ posée sur
  `kernel_wm_mouse_step` (point §0ter 5). Refactor en wrapper + body :
  fast-path shadow == 0 (cas par défaut, ~10 cyc) ; sinon push shadow,
  force `bpl=0`, exécute body, restore. Effet runtime nul tant que B2
  inactif (24/24 verts). Sécurité posée avant le flip.
- ✅ **Étape B2 livrée — plomberie (2026-05-30p)** : tout le chemin GPU
  peut basculer slot par slot via `WM_COMPACT_FLAGS[slot]`. Posé :
  - Table 8B + magic `$A5` + scratch `WCMP_SLOT_ID`.
  - `kernel_gfx_window_base` lit le flag et pose `bpl=byte_w` ou `bpl=0`
    selon (point §0ter 1).
  - `kernel_gfx_finish` (nouveau) : confine `byte_w` au syscall
    (point §0ter 2). Inséré dans 5 wrappers `sys_gfx_*`.
  - `kernel_wm_compose` : `bpl=byte_w` per-slot compact avant BLIT,
    `bpl=0` en `wcmp_done` (point §0ter 3).
  - `kernel_wm_redraw` : `bpl=0` à l'entrée (point §0ter 4).
  - Flag inactif sur tous les slots → no-op runtime, 24/24 verts.
- ✅ **Étape B2.c livrée (2026-05-30q)** : activation effective + test
  de transparence. Task de test `task_compact_entry` (alloc.s) crée
  fenêtre 64×64 à (50,50), écrit `$A5` à `WM_COMPACT_FLAGS[handle]`,
  dessine bg bleu + rect rouge à (10,10,20,20) en compact stride 32,
  compose. Test C `test_oricos_compact_backing_store` lit `vram_peek`
  framebuffer XVGA à (61,61) → 7 (rouge), (52,52) → 1 (bleu).
  12/12 tests `helloc` verts, 24/24 globales. **Flip compact validé
  fonctionnellement** — la plomberie n'est plus juste dormante, elle
  produit le bon rendu. Bug fix capturé : ABI `SYS_WIN_CREATE` exige
  16-bit LO+HI séparés (`$D0-$D7`), pas double-`sta` qui clobbe HI.
- ⏳ **Étape C (à instruire à la demande)** : 2 chantiers indépendants.
  - **C1 — Clip surface compacte** (point §0ter 7) : `gpu_fill_rect_impl`
    et `gpu_set_pixel` clippent aujourd'hui à XVGA 1024×768 ; en compact
    étroit, un dessin out-of-bounds débordera dans les banques voisines.
    Robustesse à ajouter (option : registres GPU `clip_w/clip_h`, ou
    clip kernel-side dans les wrappers `sys_gfx_*`). Pas de cas
    déclencheur réel actuel — différer jusqu'à premier dépassement.
  - **C2 — Allocation multi-banques contiguës** (point §0ter 6) :
    requis pour fenêtres > 128 px de haut en pleine largeur (cf. §3
    de ce dossier). Modifie `sys_win_create` + allocateur LIFO actuel
    (ADR-2.h v0.1) qui ne supporte pas le contigu. Sprint dédié à
    instruire au critère §6.1 (app cible réelle).

## Ratification rétractée (2026-05-30t)

La ratification 2026-05-30q a été **rétractée** suite à la validation
interactive `--compact` : la plomberie compact leak `bpl` vers des
chemins kernel direct non-instrumentés (cf. §0quater). Le test
`test_oricos_compact_backing_store` (12/12) couvrait un cas trop
restreint (compose-loop seul, sans interaction). Le moratoire §10 exige
maintenant de COMPLÉTER §0quater avant ré-instruction.

## 0quater. Audit chemins kernel direct framebuffer XVGA (2026-05-30t)

Le compose-loop n'est PAS le seul écrivain du framebuffer XVGA. Le
kernel dessine son chrome, ses widgets, son menu, sa taskbar
**directement** dans le framebuffer via `kernel_gfx_fill_rect16` /
`text16` / `line` / `clear` SANS passer par `sys_gfx_*` (donc sans
`window_base` ni `finish`). Quand le flip compact est actif, ces appels
voient potentiellement un `bpl` résiduel ≠ 512 → dessin à mauvaise
stride → corruption visuelle (rect noirs, motifs étalés).

### Inventaire (grep `jsr kernel_gfx_*` post-revert B2.c)

**Catégorie A — sys_gfx_* wrappers (5, déjà OK)** :
- `wm.s:5068-5102` : 5 wrappers clear/fill_rect/blit/line/text avec
  `window_base` + `finish`.

**Catégorie B — compose BLIT (instrumenté par B2)** :
- `wm.s:1247-1268` : set_bpl per-slot compact + restore.

**Catégorie C — chemins kernel direct framebuffer XVGA (LEAK)** :
- `wm.s` `_wm_draw_one` chrome titlebar/close/maximize/minimize : ~10
  sites (lignes 1371, 1379, 1504, 1544, 1591, 1630, 1656, 1707, 1721,
  1769, 1848).
- `wm.s` taskbar `_taskbar_draw` : 2956 (et autres dans la fonction).
- `wm.s` icônes (`kernel_icon_draw_all`) : 807, 840.
- `wm.s` early/init (`wm_init` etc.) : 43, 56, 69, 81, 92, 103.
- `tk.s` toolkit (`kernel_tk_button`, `frame`, `label`, `view`,
  `scrollbar`, `radio`, `check`, `text_field`, `list`, `spin`,
  `field`) : ~20 sites (150-1390).
- `boot.s` démos GPU init : ~10 sites (122-416, peu critique hors
  démarrage).

**Total Catégorie C ≈ 36 sites kernel direct non-instrumentés.**

### Stratégies d'instrumentation possibles

| # | Stratégie | Sites à modifier | Coût/appel | Risque |
|---|---|---|---|---|
| **C-1** | Patcher chaque appel : `bpl=0` set avant `jsr kernel_gfx_*` | 36 sites | ~30 cyc + set_bpl trigger | invasif, oublis possibles |
| **C-2** | Modifier `kernel_gfx_fill_rect16` / `text16` à la source : si `GFX_BASE_HI ∈ [$10..$16]` (framebuffer XVGA), force `bpl=0` automatiquement | 2-3 helpers GPU | ~10 cyc/appel (compare + skip-if-shadow=0) | élégant, 1 changement couvre tous les chemins |
| **C-3** | Wrapping high-level : chaque entry point WM (redraw, draw_one, menu_draw, taskbar_draw, icon_draw, draw_widget) ouvre/ferme par `bpl=0`/restore. | ~10 entry points | ~30 cyc/entry | identification des entry points = mini-audit |
| **C-4** | Pivoter vers option (a) ADR-27 (multi-banques stride 512 partout) | 1 sprint allocateur | 0 leak possible | pas de gain SDRAM pour fenêtres étroites |

**Recommandation senior tracée** : **C-2** — modification à la source.
Justification : les 36 sites Catégorie C sont par construction des
dessins XVGA (`GFX_BASE_HI=$10..$15` framebuffer). Une heuristique
unique au point d'entrée GPU intercepte tous les cas, présents et
futurs (apps qui dessineraient en direct futurement). Coût négligeable
(skip si shadow=0 = cas usuel). Découple le kernel de la politique
backing-store. C-3 demande aussi une décision sur les entry points.
C-1 est trop fragile.

### Plan d'instruction C-2 (à exécuter pour ré-ratification)

1. Modifier `kernel_gfx_fill_rect16` (gfx.s) : ajouter en tête une
   garde « si `GFX_BASE_HI ≥ $10` ET shadow ≠ 0 alors `set_bpl(0)` ».
2. Idem `kernel_gfx_text16`.
3. Idem `kernel_gfx_line` et `kernel_gfx_fill_rect` (8-bit variants).
4. Pas besoin de toucher `kernel_gfx_clear` (impl C ne consomme pas
   `bpl`, c'est un memset linéaire).
5. Pas besoin de toucher `kernel_gfx_blit` (utilisé uniquement par
   compose qui pose son `bpl` explicitement).
6. Tests d'intégration ciblés (nouveaux) :
   - Test menu dropdown avec slot compact actif (= reproduire le bug
     interactif vu en validation).
   - Test taskbar/chrome draw avec slot compact actif.
   - Test widgets (button/list) avec slot compact actif.
7. Validation interactive : `--compact` doit produire un rendu propre.

### Statut post-audit

DRAFT (option (b) toujours retenue ; ratification après §0quater
complète avec C-2 implémentée et tests verts + validation interactive
positive).



## 0ter. Plan précis du flip compact (Étape 2, à exécuter en un bloc testé)

Points de gestion `bpl` identifiés (invariant cible : `bpl=512` hors section ;
`byte_w` uniquement pendant un dessin backing-store) :

1. `kernel_gfx_window_base` : calcule `byte_w = WM_TABLE[slot].W>>1` (via
   `kernel_wm_offset`), `GFX_BPL=byte_w`, `jsr kernel_gfx_set_bpl`.
2. Wrappers `sys_gfx_*` (clear/fill/blit/line/text) : après le dessin,
   `GFX_BPL=0 ; set_bpl` (restaure 512 — confine `byte_w` au syscall).
3. `kernel_wm_compose` : par fenêtre, `GFX_BPL=byte_w ; set_bpl` avant le BLIT ;
   `GFX_BPL=0 ; set_bpl` en fin de boucle (`wcmp_done`).
4. `kernel_wm_redraw` / `kernel_wm_redraw_drag` (peintres framebuffer directs) :
   `GFX_BPL=0 ; set_bpl` à l'entrée (512).
5. `kernel_wm_mouse_step` (IRQ) : **save/restore** `GFX_BPL` autour du traitement
   (push `GFX_BPL`, set 512, …redraws…, pull, set_bpl) → IRQ transparent au
   `bpl` du syscall interrompu.
6. Allocation : backing store compact tient en 1 banque si `byte_w·h ≤ 64 KiB` ;
   sinon **multi-banques contiguës** (à ajouter à `sys_win_create` / l'allocateur).
7. Clip : `gpu_fill_rect_impl`/`gpu_set_pixel` clippent à XVGA (1024×768) ; pour
   une surface compacte étroite, clipper à `(w,h)` réels (raffinement).

Validation : propriété de **transparence** (stride dessin == stride lecture
compose ⇒ framebuffer identique) ⇒ la suite (`win_draw`/`win_app`/`clock`/
`gui_demo`/`ctl_demo` + mouse/drag) doit rester verte. Tout écart = bug à
corriger ou revert.

## 0bis. Analyse de concurrence (bloquante pour la migration kernel)

Investigation 2026-05-27 (handlers.s, wm.s). Le registre `bpl` est un **état GPU
global persistant**. Or :

- Les syscalls tournent avec **IRQs activées** (`cli` dans le COP handler,
  handlers.s §34) ; `kernel_forbid` ne masque que la préemption *tâche*, pas
  l'IRQ matériel.
- Le handler IRQ appelle **`kernel_wm_mouse_step` inconditionnellement** sur
  event souris (handlers.s §125-129), AVANT la garde `FORBID_COUNT` (§199 — qui
  ne protège que le context-switch). `mouse_step` peut déclencher
  `kernel_wm_redraw` → dessin framebuffer GPU (`kernel_gfx_fill_rect16`).

**Conséquence pour la migration** : si un mouse IRQ survient pendant un syscall
gfx qui a posé `bpl = byte_w`, le redraw IRQ dessinerait le framebuffer à la
mauvaise stride (`byte_w` au lieu de 512) → corruption framebuffer ; et/ou la
reprise du syscall verrait `bpl` modifié.

**Finding préexistant (hors ADR-27, à tracer séparément)** : ce même chemin
clobbe DÉJÀ les registres `ARG1-4` du GPU si un mouse IRQ tombe entre le
setup des ARG et le `TRIGGER` d'un helper gfx en cours. Race latente tolérée
aujourd'hui (rare ; non couverte par les tests). La migration `bpl` ne crée pas
une classe nouvelle, mais ajoute un état **persistant** (pire que les ARG
transitoires que l'IRQ réécrit de toute façon).

**Options de résolution (à trancher avant la migration)** :
1. **Reset à l'entrée IRQ** : `mouse_step` pose `bpl = 512` à son entrée +
   syscalls/compose restaurent `bpl = 512` avant de rendre la main → état stable
   toujours 512 hors section. Résiduel : fenêtre étroite syscall (set→trigger)
   reste exposée (même tolérance que la race ARG existante).
2. **Section critique `sei/cli`** autour de (set `bpl` … trigger) dans les
   helpers gfx et compose → bulletproof, mais touche le modèle d'interruption
   (latence ; interaction Forbid/scheduler à valider).
3. **Stride par-commande** (pas d'état global) : encoder `src_bpl` dans les
   octets hauts libres d'ARG3/ARG4 pour le BLIT (compose) — hazard-free pour
   le compositor. Mais FILL_RECT/LINE ont de la place, **TEXT/TEXT16 non**
   (ARG4 plein) → dessin texte dans backing store compact non couvert.
4. **Shadow ZP + save/restore** dans le handler IRQ (sauve/rétablit `bpl` autour
   du dessin IRQ) — nécessite un registre de lecture `bpl` GPU (absent) → shadow
   kernel obligatoire.

**Recommandation senior** : option 1 (reset IRQ + restauration syscall),
cohérente avec la tolérance de risque déjà acceptée pour les ARG, à coupler
avec un **fix séparé de la race ARG préexistante** (idéalement option 2 ciblée :
`sei/cli` court autour de chaque séquence setup+trigger des helpers gfx). À
instruire/valider avant de toucher au compositeur.

### Contrainte dure tranchée : pas de port I/O libre

Les 16 ports GPU `$0340-$034F` sont **tous** assignés (ARG1-4, STATUS, TRIGGER,
INT_CTRL) et `$0350` est le contrôleur KBD2 (ADR-22). Un registre BPL **dédié
par port** imposerait d'étendre l'allocation I/O GPU → révision ADR-21 + ADR-22
+ MEMORY_MAP. **Écarté.** Le BPL est donc exposé via un **opcode `SET_BPL`**
(mécanisme CMD_OP + ARG1 + TRIGGER existant) qui fixe un état `bpl` persistant.
Zéro nouveau port, zéro impact memory map.

### Hazard identifié : état global `bpl`

`bpl` est un état GPU **global** partagé par toutes les surfaces. Le kernel doit
le discipliner : `SET_BPL byte_w` avant de composer une fenêtre, `SET_BPL 0`
(→512) avant tout dessin direct dans le framebuffer XVGA. Un oubli corrompt la
stride. Documenté dans `kernel_gfx_set_bpl`. (Une alternative v0.3 = src_bpl/
dst_bpl par-commande encodés dans les octets hauts libres d'ARG3/ARG4.)

### Hazard identifié : famine de la main loop pendant un drag souris rapide

Investigation 2026-05-28 (handlers.s, wm.s, alloc.s ; reproduction déterministe
`oricrobot`). **Distinct des races `bpl`/ARG** (qui sont des *data races*) : il
s'agit ici d'un **déséquilibre de budget CPU** entre l'IRQ et la tâche app.

**Symptôme observé (live, interactif)** : en baissant l'ascenseur au max par un
drag rapide, le thumb se fige (value bloquée à une valeur intermédiaire, ex. 39
sur 44) **alors que le curseur continue de bouger**. La value **rattrape dès que
le mouvement s'arrête**. Donc pas un gel logique : un retard qui ne se résorbe
jamais tant que la souris bouge vite.

**Cause racine — asymétrie de traitement** :
- Le **curseur** (`kernel_wm_cursor_blit`) et le **drag de fenêtre** (déplacement
  des coords) sont mis à jour **dans l'IRQ souris** (`kernel_wm_mouse_step`,
  handlers.s §129), à chaque event MOU2.
- La **value de l'ascenseur** est mise à jour par la **tâche app**, hors IRQ :
  chaque `MOUSE_MOVED` est dépilé par `SYS_MAIN_LOOP`, classifié
  (`_ml_classify` → `mlc_moved` → `_wm_scroll_update`, wm.s §3211/§2296), ce qui
  inclut un **redraw GPU ciblé** (`_wm_redraw_ctl`) puis un retour `MSG_CONTROL`
  à l'app (aller-retour syscall par event).
- En drag rapide, les IRQ souris s'enchaînent et consomment la quasi-totalité du
  CPU (lecture MOU2 + `mouse_step` + cursor blit + push event + wake). La tâche
  app n'obtient plus de tranche suffisante pour terminer une itération de main
  loop → la value (et le thumb) gèlent ; le curseur, piloté par l'IRQ, continue.

**Reproduction déterministe** (`oricrobot`, TC_SCR_FLAG, kernel `1.22.x`) : un
flux de `moverel 0 ±2` à cadence serrée (~2500 cycles/event) fige la value (ex.
6/44) ; un `run 200000` à souris immobile la fait rattraper d'un coup (6→44).
Le coalescing `MOUSE_MOVED` (`0144b0b`) **ne couvre pas** ce cas : il limite la
profondeur du ring (donc pas de perte de UP / pas de gel par ring saturé — bug
distinct déjà corrigé), mais ne réduit pas le coût CPU par itération.

**Direction retenue par l'humain (2026-05-28) : « découpler / alléger l'IRQ »**
— rendre du budget CPU à la main loop (p.ex. throttle/découplage du cursor blit
par event, ou allègement du chemin `mouse_step`). À **instruire avant tout
code** : cette direction touche frontalement le partage de travail IRQ↔syscall
de ce §0bis et **n'est pas tranchée** (moratoire ADR, CLAUDE.md §10). Alternatives
écartées par l'humain à ce stade : (i) déplacer le calcul de value dans l'IRQ
(symétrie avec le drag fenêtre, clamp arithmétique sans GPU) ; (ii) alléger la
main loop (supprimer l'aller-retour app par MOVED / coalescer les redraws). À
chiffrer dans l'instruction de cette option.

#### Instruction de l'option « découpler / alléger l'IRQ » (en cours, non tranchée)

**Budget mesuré (oricrobot, indicatif — voir caveat)** :
- *Coût d'une itération de scroll en main loop* : ~**1500 cycles** CPU. Mesure :
  depuis un état settle (value à jour, rien en attente), un `moverel` unique n'est
  traité (value mise à jour) qu'à partir de `run ≥ 1500` ; à `run ≤ 1000` la value
  ne bouge pas. Couvre : pop event + `_ml_classify` + `_wm_scroll_update` (clamp +
  store) + `_wm_redraw_ctl` (FILL_RECT gouttière + thumb + redraw curseur, sous
  `sei`) + retour `MSG_CONTROL` + ré-entrée `SYS_MAIN_LOOP` de l'app.
- *Coût du blit curseur par event* (`kernel_wm_cursor_blit`, par inspection) :
  **3 copies 8×8 px** par event (restore ancien fond + save nouveau fond + dessin
  glyph), chacune via port I/O VRAM ligne par ligne (`_cursor_copy_to_save` /
  `_cursor_copy_from_save` / `_cursor_draw_glyph`). C'est le **poste dominant** du
  travail IRQ par event souris, devant `mouse_step` (tests focus/drag) et le
  push/wake event.

**Caveat de mesure (à lever avant ratification)** : `oricrobot` exécute des
tranches `run N` déterministes et **ne reproduit pas la cadence temps-réel** du
build SDL `--machine oric2`. En timing SDL nominal, le device MOU2 coalesce
plusieurs `move_rel` d'une frame en **un seul event** (flag `event` unique →
**1 IRQ souris/frame**, ~19968 cycles de budget) — ce qui, face à ~1500+~2000
cycles de travail/event, **ne devrait pas** affamer la main loop. Or le trace
live (`MDIAG`) montre `MOUSE_Y` qui suit (IRQ actif chaque event) mais `val` figée
sur des centaines d'events → la main loop est **réellement** privée de tranche en
conditions réelles. **Écart à instruire** : soit le build réel voit > 1 IRQ
souris/frame (à vérifier : nb d'appels handler/frame sous drag), soit un effet
scheduler/`FORBID` prive la tâche app. **Mesure on-target requise** : instrumenter
le build SDL (compteur d'IRQ souris/frame + cycles passés en handler vs en tâche
app pendant un drag) pour fixer le vrai seuil avant tout code.

##### Instrumentation livrée (Phosphoric 1.22.90-alpha, 2026-05-28)

Le compteur d'IRQ MOU2/frame est **implémenté et vérifié** (build SDL,
`make tests` verts, 0 front parasite au repos sur 11314 frames headless). Activation :

```bash
PHOSPHORIC_MOU2_IRQTRACE=1 ./oric1-emu --machine oric2 --kernel ../OricOS/build/kernel.bin
# puis, fenêtre focus + capture souris (clic), faire un drag d'ascenseur rapide.
# Log par frame active : "MOU2 IRQ: frame N -> M edge(s) [max=.. total=.. multi=..]"
```

Compte les **fronts montants** de `IRQF_MOU2` (clear→assert = un IRQ présentable
au guest), pas les appels `irq_set` (idempotents tant que la ligne reste haute).
Champs `mou2_irq_*` dans `emulator_t` ; détection de front dans
`mou2_cpu_irq_set`/`_clr` ; résumé en fin de boucle frame (`src/main.c`). Coût
hors-trace nul (branche gardée, défaut off). C'est de l'**outillage de mesure**,
pas une correction : aucune des options D1-D4 n'est touchée (moratoire CLAUDE.md §10).

**Finding de lecture de code (à confirmer par run interactif on-target)** : dans le
build SDL, `SDL_PollEvent` draine **tous** les events souris d'une frame en **une
seule passe, APRÈS** les 19968 cycles CPU (boucle interne `main.c` §923-1001, poll
§~1196). Le device MOU2 n'est donc mis à jour qu'**une fois par frame** ; sa ligne
IRQ ne peut connaître qu'**≤ 1 front montant par frame** → le compteur `multi`
(frames à > 1 front) est **structurellement 0**. **Conséquence** : l'hypothèse
« le build réel voit > 1 IRQ souris/frame » est **réfutée par le chemin de code**.
Avec ≤ 1 IRQ/frame (~2000-4000 cyc de handler) sur 19968 cyc/frame, la *fréquence*
des IRQ n'affame pas la main loop. La value figée vient donc de l'**autre** branche
du caveat : effet scheduler/`FORBID`, **ou** coût du redraw/aller-retour app **par
event** dans la main loop elle-même (qui, lui, peut accumuler du retard si plusieurs
`MOUSE_MOVED` coalescés produisent un event lourd par frame mais que l'app ne draine
qu'un event par itération). **Prochaine mesure** : compteur d'events `MOUSE_MOVED`
consommés/frame côté main loop + cycles tâche-app vs handler (le 2e volet du caveat),
à instrumenter avant de chiffrer D1-D4.

**Sous-options chiffrables de « découpler / alléger l'IRQ »** :

| Sous-option | Principe | Bénéfice attendu | Coût / risque |
|---|---|---|---|
| **D1 — Throttle du cursor blit** | Ne redessiner le curseur qu'au plus 1×/tick (ou 1×/N events), pas à chaque event MOU2 ; accumuler la dernière position et blitter au tick T1 | Rend ~2000 cyc/event à la main loop ; le curseur reste fluide à la fréquence tick | Curseur potentiellement 1 tick en retard ; état « position curseur en attente » à gérer ; interaction avec le backing-store (CURSOR_SAVE) à valider |
| **D2 — Découpler le blit du chemin event** | `mouse_step` ne fait que maj coords + drag (léger) ; le blit curseur est délégué au tick T1 (ou à une tâche dédiée basse priorité) | Sépare proprement « tracking » (IRQ léger) et « rendu » (différé) | Refactor du point d'appel ; latence curseur ; éventuelle nouvelle tâche |
| **D3 — Blit curseur incrémental** | Ne re-save/restore que si la position a bougé d'≥1 px effectif ; sauter restore+save quand delta nul | Élimine le blit sur events « immobiles » (deltas saturés au bord/au max) | Gain nul si la souris bouge vraiment ; faible mais sûr |
| **D4 — Réduire le coût des copies 8×8** | Remplacer les copies px-par-px par un BLIT GPU (opcode existant) pour save/restore/glyph | Divise le coût des 3 copies | Réintroduit du travail GPU dans l'IRQ → **réveille la race ARG/`bpl`** du §0bis ; à coupler avec sa résolution |

**Note de cohérence (moratoire, CLAUDE.md §10)** : D1/D2/D3 **réduisent** le travail
GPU/IRQ et vont donc dans le sens de la résolution du §0bis (moins d'exposition
de la race ARG/`bpl`). D4 va à l'inverse (plus de GPU dans l'IRQ) → à n'envisager
qu'après résolution de la race. Aucune de ces sous-options ne révise une ADR
ratifiée ; elles affinent le partage IRQ↔syscall, sujet **ouvert** de ce §0bis.

**Recommandation senior tracée (non décisionnelle)** : commencer par **D3**
(sûr, sans latence, élimine les events au bord/max — précisément le cas du
« drag au max » signalé), **mesurer on-target**, puis si insuffisant ajouter
**D1** (throttle au tick). Garder **D4** lié au plan de résolution de la race
ARG/`bpl`. Écarter pour l'instant le déplacement du calcul de value dans l'IRQ
(alternative (i) supra) : il ré-introduirait le partage des scratch `WG_*`
IRQ↔main-loop (race documentée, cf. hardening `6c2d1f3`).

---

## 1. Contexte

Le modèle de composition GUI (« GrafPort-like », convention G.2 du Sprint 3.m)
attribue à chaque fenêtre un **backing store** : une zone SDRAM où l'app dessine
en coordonnées **locales** via les syscalls `SYS_GFX_*`. Le window manager
composite ensuite ces backing stores vers le framebuffer XVGA via
`kernel_wm_compose` (BLIT par fenêtre, respect du Z-order).

Convention d'implémentation actuelle (`wm.s`, `sys_win_create` §3005) :

> Backing store SDRAM implicite par slot : base = `($06+slot):$0000` — **1 banque
> de 64 KiB par slot**.

Le GPU utilise une **stride (BPL) figée à 512 octets/ligne** (= XVGA 1024 px / 2,
ADR-20), identique pour source et destination du BLIT.

## 2. Problème (Finding B)

64 KiB / 512 = **128 lignes maximum** par backing store.

`kernel_wm_compose` exécute `byte_h = h` (hauteur fenêtre en lignes). Pour une
fenêtre de hauteur `h > 128 px`, le BLIT source lit `h × 512` octets > 65536,
donc **déborde de la banque du slot dans les banques voisines** (`$06+slot+1`…),
qui appartiennent à d'autres backing stores :

| Hauteur fenêtre | Octets source (stride 512) | Banques sur-lues |
|---|---|---|
| 128 px | 65 536 | 1 (exactement la banque) |
| 200 px | 102 400 | 2 |
| 744 px (maximisée) | 380 928 | ~6 |

**Historique** : avant le fix v0.2, `byte_h` était tronqué à 8 bits
(`h=768 & 0xFF = 0` → 0 ligne → fenêtre invisible). La troncature **masquait**
le débordement. Le fix v0.2 (correct) **expose** cette limite d'architecture.

**Périmètre d'impact** : uniquement le chemin **compose** (apps dessinant en
backing store + `SYS_WIN_FLUSH`). Le chemin `kernel_wm_redraw` (FILL_RECT16
direct vers framebuffer, sans backing store) n'est **pas** concerné. Les apps
actuelles (clock, win_hello) créent de petites fenêtres → le bug n'est pas
encore déclenché en pratique, mais c'est un **piège latent** dès qu'une app
ouvre une fenêtre haute.

## 3. Contrainte de dimensionnement

Une fenêtre exploitable plein écran fait ~1024×744 (XVGA moins chrome). À
stride 512, le backing store correspondant pèse `744 × 512 = 380 928 octets`
≈ **6 banques de 64 KiB**. Pour 8 slots plein écran : jusqu'à **48 banques
(3 MiB)**. Le pool actuel « banks 4-15 » (768 KiB, MEMORY_MAP §6) est trop
petit pour ce cas ; les banks 16-127 (7 MiB) ou 128-191 (4 MiB) sont
candidates.

## 4. Options instruites (NON tranchées)

| Option | Principe | Coût SDRAM | Coût HDL/GPU | Complexité kernel | Risque |
|---|---|---|---|---|---|
| **(a) Backing store multi-banques contigu** | Allouer `ceil(h×512/65536)` banques contiguës par slot ; base recalculée au create/resize | jusqu'à 6 banques/slot | nul (stride 512 conservée) | allocateur contigu (vs LIFO actuel) ; `kernel_gfx_window_base` calcule base par slot→banque | fragmentation SDRAM ; resize = realloc |
| **(b) Stride backing store = largeur fenêtre** | Registre BPL GPU configurable (déjà annoncé ADR-21 v0.2 / gpu_device.h) ; backing store compact `ceil(w/2)×h` | idem ~6 banques si pleine largeur, moins si étroite | **registre BPL src/dst GPU** + recalcul offsets dans exec_blit/fill/text | GPU passe BPL au BLIT ; compose fournit 2 strides | change l'API GPU (ADR-21 v0.2) ; tests à étendre |
| **(c) Plafond 128 px + clamp** | `byte_h ≤ 128` dans compose ; fenêtres limitées 128 px | inchangé | nul | trivial | **UX cassée** : pas de grande fenêtre sur desktop 768 → non-viable |
| **(d) Dessin direct framebuffer + clipping** | Abandon backing store ; apps dessinent dans le FB, WM clippe au rect fenêtre | nul (pas de backing) | clipping rect GPU | gros refactor compose/redraw ; perte du redraw sans repaint app | rupture du modèle GrafPort ; scope élevé |

### Recommandation senior tracée (à valider, non décisionnelle)

L'option **(b)** s'aligne avec une intention **déjà documentée** :
- ADR-21 : « v0.1 sync ; **v0.2 : BPL et résolution configurables via registres
  dédiés** » ;
- `gpu_device.h` §73-75 : « v0.2 : BPL et résolution configurables ».

Elle résout Finding B *et* débloque les backing stores compacts (économie SDRAM
pour fenêtres étroites). Coût principal = un registre BPL GPU + propagation dans
les `exec_*`. (a) est plus simple côté GPU mais déplace la complexité vers
l'allocateur contigu et gaspille la SDRAM (stride pleine largeur toujours).
(c) est non-viable, (d) est hors-scope v1.

**Ce dossier ne ratifie pas (b).** Décision soumise aux 3 conditions du
moratoire (CLAUDE.md §10) : dossier chiffré (ce document, à compléter avec
mesures), ≥ 50 % d'implémentation de référence, cohérence ADR-19/20/21.

## 5. Garde défensive intermédiaire (optionnelle, sans décision d'archi)

En attendant l'instruction, une **garde défensive** dans `kernel_wm_compose`
(`byte_h = min(h, 128)`) éliminerait la sur-lecture (sûr) au prix d'une
troncature visuelle des fenêtres hautes. À arbitrer séparément — non inclus
dans ce DRAFT car c'est un changement de comportement.

## 6. Critères de réouverture / d'instruction

Instruire (passer DRAFT → dossier ratifiable) dès que **l'un** est atteint :
1. Une app cible nécessite une fenêtre > 128 px de haut composée via backing
   store (déclencheur fonctionnel réel).
2. ADR-21 v0.2 (BPL configurable) est mise en chantier → l'option (b) devient
   chiffrable et à moitié implémentée.
3. Date plancher : revue au prochain jalon GUI.

## 7. Conséquences du statut « ouverte »

- **Positif** : pas de ratification compulsive (moratoire respecté) ; le fix
  BLIT v0.2 reste correct et mergé ; les apps petites fenêtres fonctionnent.
- **Négatif** : piège latent non corrigé — toute app ouvrant une fenêtre haute
  + `SYS_WIN_FLUSH` produira une corruption visuelle (sur-lecture de banques
  voisines). Mitigation : documenter la limite « fenêtre ≤ 128 px en compose »
  pour les apps v1 ; garde défensive §5 disponible si un cas réel survient.

## 8. Références

- CLAUDE.md §3 (décisions ouvertes), §10 (moratoire ADR)
- ADR-19 (SDRAM unifiée), ADR-20 (framebuffer XVGA 1024×768×4bpp), ADR-21 (GPU
  blitter, BPL configurable v0.2 annoncée)
- docs/MEMORY_MAP.md §6-8 (pools de banques)
- `OricOS/kernel/modules/wm.s` : `kernel_wm_compose` (§1136), `sys_win_create`
  (§3001), `kernel_gfx_window_base` (gfx.s §13)
- Audit GPU toolbox 2026-05-27, Finding B (Phosphoric CHANGELOG 1.22.87)
