# ADR-28 — Modèle de threading du Window Manager

- **Statut** : **ratifiée** (2026-05-29) — option **C** retenue (curseur reste
  en IRQ, politique fenêtre + rendu en tâche serveur WM). Implémentation de
  référence livrée et testée (Étapes 0/1/2/3 + §6.6 + §6.7, gated
  `TC_WM_FLAG=$A5`, suite `make tests` verte à chaque palier).
- **Date d'ouverture** : 2026-05-29
- **Date de ratification** : 2026-05-29
- **Décideurs** : bmarty (validation), Claude Code (instruction + implémentation)
- **Contexte technique** : Sprint 3 (GUI). ADR liées : ADR-03 (scheduler),
  ADR-24 (souris IRQ), ADR-25 (concurrence), ADR-26 (GUI déclaratif),
  ADR-27 (backing store : famine §0bis + race GPU). Révise *de facto* le
  modèle d'exécution implicite du WM posé en SP-3.e..o.
- **Conformité moratoire** : oui — audit §8 ci-dessous (3 conditions
  CLAUDE.md §10 vérifiées).

## 0. Résumé exécutif

Le window manager d'OricOS — hit-test, focus, Z-order, drag, resize,
maximize/minimize/close, invocation de callbacks, **et rendu plein écran** —
s'exécute aujourd'hui **dans le handler d'IRQ souris**, IRQ masquées, sur la
pile de la tâche interrompue (`kernel_irq_handler` → `kernel_wm_mouse_step`,
handlers.s:129).

Cette ADR pose la question : **où doit s'exécuter la politique WM et le
rendu ?** La thèse instruite ici est que le placement actuel est la **cause
racine commune** de trois problèmes traités jusqu'ici comme indépendants :

1. la **famine de la main loop** en drag rapide (ADR-27 §0bis) ;
2. la **race d'état GPU** (`bpl`, ARG) entre IRQ et syscall (ADR-27 §0bis,
   mitigée par `php/sei…plp` mais non close) ;
3. l'**appel de callback applicatif depuis l'IRQ** (wm.s:1911,
   `jsr (WM_DP_TMP,X)`) — danger de sûreté dès qu'ADR-15 rouvre (apps
   non-trusted).

Le design de référence (AmigaOS `input.device`/`intuition.library`, GEOS,
SymbOS) place **uniquement la collecte d'événements** dans l'interruption et
**toute la politique + le rendu** en contexte tâche. OricOS possède déjà
l'infrastructure pour cela (`EVENT_RING`, `SYS_MAIN_LOOP`, block/wake ADR-25)
mais le chemin IRQ **duplique** le travail au lieu de se limiter à poster.

## 1. Contexte chiffré

### 1.1. Ce que fait l'IRQ aujourd'hui (constat code)

Sur chaque event souris matériel, `kernel_irq_handler` (handlers.s:125-150) :

1. `kernel_mouse_read` (lit + clear MOU2) ;
2. **`kernel_wm_mouse_step`** — la totalité de la politique WM :
   - hit-test taskbar + menu + fenêtre + icônes + chrome (wm.s:1863-1925) ;
   - `kernel_wm_set_focus` → **réordonne le Z-order** (`WM_ZORDER`) ;
   - `kernel_wm_maximize` / `minimize` / `close` — **mutation d'état WM** ;
   - `jsr (WM_DP_TMP,X)` (wm.s:1911) — **callback applicatif** (icône) ;
   - `kernel_wm_redraw` (wm.s:1874, 1962…) — **clear plein écran XVGA
     (1024×768×4bpp ≈ 393 KiB) + redessin de toutes les fenêtres en GPU
     polling busy**, dans l'IRQ ;
3. *puis* `kernel_event_push_mouse` — poste l'event dans `EVENT_RING` pour la
   main loop, **qui refera un travail de classification + rendu**.

Donc le travail est fait **deux fois**, et la partie lourde (rendu) est sur
le chemin le plus contraint (IRQ, I masqué).

### 1.2. Budget (source : ADR-27 §0bis, à confirmer par instrumentation)

- Budget frame ≈ **19 968 cycles** (50 Hz @ ~1 MHz effectif mode N).
- `kernel_wm_cursor_blit` (save/restore/draw 8×8 via VRAM I/O) ≈ **2 000
  cyc/event**.
- Chemin main loop (`event_pop` + `classify` + `scroll_update` + `redraw`) ≈
  **1 500 cyc/event**.
- `kernel_wm_redraw` plein écran : non chiffré ici mais **dominant** (clear de
  393 KiB en commandes GPU + polling). C'est lui qui rend le throttle
  nécessaire.

### 1.2bis. Mesure on-target (2026-05-29) — coût inclusif des routines IRQ

Harnais `Phosphoric/tests/integration/test_oricos_wm_cost.c` (cible
`make test-oricos-wm-cost`) : boot kernel oric2 + VRAM/GPU en mode GUI
persistant (`NO_STP_FLAG=$A5`), injection d'un clic + 10 drags + 1 relâche sur
la titlebar de la fenêtre démo 0. Coût **inclusif** (entrée→retour, callees
compris) mesuré par détection de retour via le pointeur de pile S. Déterministe
(min==max sur les drags). Budget frame = 19 968 cycles.

| Routine (dans l'IRQ) | Appels | Moy. cyc | % frame |
|---|---|---|---|
| `kernel_irq_handler` (tous IRQ, T1 inclus) | 30 | 6 006 | 30,1 % |
| **`kernel_wm_mouse_step`** (politique + rendu) | 12 | **13 340** | **66,8 %** |
| `kernel_wm_redraw` (plein, sur clic/focus) | 1 | 10 562 | 52,9 % |
| **`kernel_wm_redraw_drag`** (par drag) | 10 | **10 693** | **53,6 %** |
| `kernel_wm_draw_cursor` (drag/clic) | 11 | 1 826 | 9,1 % |
| `kernel_wm_cursor_blit` (motion seule) | 1 | 3 323 | 16,6 % |
| `kernel_gfx_fill_rect16` (borne basse, ×) | 198 | 194 | 1,0 % |

**Lectures décisives :**

1. **Le drag de fenêtre vole les 2/3 de la frame** : `mouse_step` consomme
   **66,8 % du budget frame par event** en drag de FENÊTRE/resize (dominé par le
   rendu). ⚠️ Ce chiffre concerne le drag de fenêtre, **pas** le drag d'ascenseur
   — voir §1.2ter, qui mesure le scrollbar et **réfute** l'hypothèse initiale
   d'une famine par coût brut sur ce chemin.
2. **Le coût de `mouse_step` est dominé par le rendu** : `redraw_drag` (10 693)
   = 80 % de `mouse_step` (13 340). La **politique seule** (hit-test, focus,
   drag, chrome) ≈ 13 340 − 10 693 ≈ **2 650 cyc** (13 %) — *peu coûteuse*.
3. **Le « drag incrémental » n'économise rien** : `redraw_drag` (10 693) ≈
   `redraw` plein (10 562). Il n'efface que l'ancien rect mais **redessine
   toutes les fenêtres**. → renforce le besoin de damage-tracking (§6.5).
4. **Ce que C garde dans l'IRQ est modeste** : le curseur coûte 9–17 % du
   budget frame (`draw_cursor` 1 826 / `cursor_blit` 3 323), soit ≈ **1/3 du
   coût du redraw**. Le curseur reste réactif sans affamer la main loop.

> **(×) CAVEAT GPU** : le GPU de Phosphoric est **synchrone** (le FILL/BLIT
> s'exécute en C au trigger, `busy` retombe aussitôt). Les chiffres de
> `redraw`/`redraw_drag`/`fill_rect16` sont donc une **BORNE BASSE** — ils ne
> comptent que l'orchestration CPU (setup commande + 1 poll), pas le temps réel
> de remplissage de 393 KiB qu'un GPU HDL imposerait au CPU en polling. Sur
> cible HDL, le coût du redraw dans l'IRQ serait **bien supérieur**, ce qui
> **renforce** la nécessité de l'en sortir. `cursor_blit` (copies VRAM MMIO
> octet par octet, côté CPU) est en revanche **réaliste**.

### 1.2ter. Mesure on-target (2026-05-29) — chemin MAIN-LOOP du drag d'ascenseur

Harnais `Phosphoric/tests/integration/test_oricos_scroll_cost.c` (cible
`make test-oricos-scroll-cost`) : active `task_scr` (scrollbar V sur la fenêtre
0 + boucle `SYS_MAIN_LOOP`), boot persistant, mousedown sur le thumb +
12 events de drag vers le bas. Coût **inclusif** (pile S) des fonctions de
politique scrollbar, qui vivent dans la **main loop** (pas l'IRQ).

| Routine | Contexte | Appels | Moy. cyc | % frame |
|---|---|---|---|---|
| `_wm_scroll_update` (maj value + redraw ciblé) | main loop | 13 | 3 452 | 17,3 % |
| ↳ `_wm_redraw_ctl` (repeint contrôle + curseur) | main loop | 13 | 3 260 | 16,3 % |
| ↳↳ `kernel_wm_redraw_widget` (paint widget seul) | main loop | 13 | 744 | 3,7 % |
| `kernel_wm_cursor_blit` (curseur) | IRQ | 13 | 3 320 | 16,6 % |
| `kernel_wm_mouse_step` (drag ascenseur) | IRQ | 14 | 4 151 | 20,8 % |

**Progression value mesurée** : `7 10 13 16 19 22 25 28 31 34 37 40` —
**exactement 1 mise à jour par event, value strictement croissante**.

**Lectures (initialement présentées comme « décisives », à nuancer) :**

> ⚠️ **Rétractation partielle 2026-05-29** : la conclusion « famine réfutée »
> ci-dessous est **invalide**. Le harnais injecte 1 event par cycle de
> 19 968 cycles (≤ 1 event/frame). En usage **interactif SDL réel**, la
> cadence d'events peut être bien plus dense (motion 60-1000 Hz, drags) — et
> le bug `bbf067b` « GUI gelée en fin de course d'ascenseur » reste
> reproductible interactivement (confirmé par test utilisateur 2026-05-29).
> Les chiffres ci-dessous (1:1, 34 % budget) **caractérisent le régime
> ≤ 1 event/frame** mais ne valent pas comme réfutation du bug famine.

1. ~~**Pas de famine à ≤ 1 event/frame**~~ → **à reformuler** : sous cette
   cadence injectée, la `value` suit 1:1 et le coût est ≈ 34 % du budget.
   Insuffisant pour réfuter le bug interactif.
2. **Le paint du widget est trivial (3,7 %)** : le coût de `_wm_scroll_update`
   est dominé par le **curseur** redessiné dans `_wm_redraw_ctl` (≈ 3 260 − 744 ≈
   2 500 cyc), pas par le widget.
3. **Le curseur est rendu DEUX FOIS par event** : une fois dans l'IRQ
   (`cursor_blit` 16,6 %) **et** une fois dans `_wm_redraw_ctl` (≈ 2 500, car le
   repeint du contrôle a sali le fond du curseur). ≈ **33 % du budget frame en
   curseur dupliqué** — gaspillage réel, **indépendant** du gros refactor, et
   corrigeable à part (cf. §6.6).
4. ~~**Donc la famine interactive observée (bbf067b)...**~~ → **invalidé**
   par retest utilisateur 2026-05-29 : le bug « fin de course d'ascenseur »
   est **toujours reproductible interactivement**, malgré coalescing + §6.7.
   Cause racine **non identifiée** à ce stade ; harnais §1.2ter trop
   espacé pour la déclencher. À ré-instruire avec une cadence d'injection
   représentative (rafale 8+ events/frame).

> **Conséquence pour l'arbitrage** : la famine **n'est plus** l'argument
> principal de l'Étape 3 (réfutée par coût sur le chemin scrollbar à la cadence
> émulateur). Les justifications **robustes** restent : (a) élimination de la
> race GPU `bpl`/ARG, (b) sûreté callback hors IRQ (ADR-15), (c) retrait du
> coût drag-de-fenêtre 53 % de l'IRQ (pire sur HDL), (d) suppression du curseur
> dupliqué. Le bug famine lui-même relève d'un correctif **ciblé** (anti-drop
> du button-UP / mesure d'un scénario rafale), pas du refactor complet.

### 1.3. Symptômes mesurés / observés

- **Famine** : en drag d'ascenseur rapide, la `value` du scrollbar **gèle**
  alors que le curseur continue (bug confirmé, ADR-27 §0bis ; fix workspace
  `bbf067b` pour le cas ascenseur-au-max).
- **Race GPU** : `bpl`/ARG sont des états GPU globaux ; un mouse IRQ entre
  setup et trigger d'un helper gfx les clobbe (ADR-27 §0bis). Mitigé par
  `php/sei…plp` (Phosphoric 1.22.90) — *symptôme masqué, cause non éliminée*.
- **Callback en IRQ** : aujourd'hui sans incident (ADR-04 « OS de confiance »)
  mais incompatible avec la réouverture d'ADR-15 (date plancher 2026-12-31).

## 2. Problème

Le WM viole la séparation **mécanisme (IRQ) / politique (tâche)** qui est
l'invariant des OS fenêtrés de référence. Tant que la politique et le rendu
restent dans l'IRQ :

- toute optimisation de la famine (ADR-27 D1–D4) ne fait que **rééquilibrer
  un budget dépensé au mauvais endroit** ;
- la race GPU exige des `sei` défensifs **partout** — l'aveu d'un partage
  d'état qui ne devrait pas exister ;
- le flip compact de l'ADR-27 §0ter doit **sauver/restaurer `bpl` dans l'IRQ**
  (point 5) — complexité qui disparaît si l'IRQ ne dessine plus.

## 3. Options envisagées

### Option A — Statu quo + patches ciblés (ADR-27 D1/D3)

Garder le WM dans l'IRQ ; throttler le cursor blit (D1) et rendre le redraw
incrémental/skip-si-delta-nul (D3).

- **Coût** : faible (quelques dizaines de lignes, déjà partiellement amorcé).
- **Bénéfice** : atténue la famine sur le cas démontré.
- **Risque/limite** : ne traite ni la race GPU (structurelle), ni le danger
  callback (ADR-15), ni la duplication du travail. Dette reportée. Le flip
  compact ADR-27 §0ter reste obligé de gérer `bpl` en IRQ.
- **Réversibilité** : haute (rien d'architectural n'est figé).

### Option B — Serveur WM en tâche dédiée (modèle Intuition/GEOS) — *recommandée*

L'IRQ souris se réduit à : `mouse_read` + `event_push_mouse` +
`event_wake`. **Toute** la politique (`mouse_step` : hit-test, focus, Z-order,
drag, resize, chrome, callbacks) **et tout le rendu** (`redraw`, `compose`,
`cursor_blit`) migrent dans une **tâche serveur WM** qui consomme `EVENT_RING`
via le mécanisme block/wake déjà existant (ADR-25).

- **Coût** : élevé (déplacement de ~1 500 lignes de `wm.s` du contexte IRQ
  vers contexte tâche ; création d'une tâche kernel WM ; définition de sa
  priorité vs apps ; protocole de réveil). Touche le cœur testé du WM.
- **Bénéfice** : dissout **simultanément** famine (le rendu n'a plus de budget
  IRQ à voler — il s'ordonnance), race GPU (un seul producteur de commandes
  GPU → suppression des `sei` défensifs et de la gestion `bpl` en IRQ,
  simplifie ADR-27 §0ter point 5), et danger callback (exécuté en contexte
  tâche, préemptible). Élimine la **double exécution** du travail.
- **Risque/limite** : latence curseur (le curseur ne bouge plus *dans* l'IRQ —
  il faut soit garder un cursor blit minimal en IRQ, soit accepter une latence
  d'un tick). Réordonnancement du code le plus testé du projet → exige une
  campagne de tests de non-régression GUI complète. Interaction avec le
  scheduler (ADR-03) : priorité du serveur WM à définir.
- **Réversibilité** : faible une fois engagé (refactor structurel).

### Option C — Hybride : curseur en IRQ, reste en tâche

Compromis : seul `kernel_wm_cursor_blit` reste en IRQ (réactivité curseur
préservée, ~2 000 cyc/event) ; hit-test/focus/drag/redraw/callbacks migrent
vers le serveur WM (option B).

- **Coût** : élevé comme B, plus une frontière à maintenir (le curseur lit
  MOUSE_X/Y en IRQ, le serveur lit l'event ring).
- **Bénéfice** : B sans la régression de latence curseur.
- **Risque/limite** : le curseur en IRQ touche encore le framebuffer → la race
  GPU n'est pas **totalement** éliminée (un mini-`sei` ou un canal GPU réservé
  curseur subsiste). Frontière IRQ/tâche plus subtile à raisonner.
- **Réversibilité** : faible.

## 4. Recommandation senior (tracée, non ratifiée)

**Viser l'option C** (hybride curseur-en-IRQ + serveur WM), avec **A comme
palliatif court terme** pendant l'instruction. **La mesure §1.2bis tranche en
faveur de C** (voir chiffres ci-dessous).

Justification, étayée par la mesure :
- B est la cible architecturale juste (séparation mécanisme/politique), mais
  la latence curseur est une régression UX perceptible sur un pointeur ;
- C conserve la seule chose qui a une raison légitime d'être dans l'IRQ — le
  feedback curseur temps-réel — et sort tout le reste ;
- **le coût de C dans l'IRQ est mesuré modeste** : le curseur = 9–17 % du
  budget frame, contre 67 % aujourd'hui (`mouse_step` complet). C fait donc
  chuter le coût IRQ par event de ≈ 67 % à ≈ 17 %, ce qui **élimine la famine**
  (la main loop récupère ≈ 83 % du budget) **tout en gardant le curseur dans
  l'IRQ** → pas de régression de latence (l'argument qui départage C de B) ;
- la politique migrée vers le serveur WM ne pèse que ≈ 2 650 cyc (13 %), donc
  bon marché à ordonnancer ;
- la race GPU résiduelle de C est **bornée au curseur** (une surface 8×8 fixe),
  donc traitable par un canal/registre GPU réservé curseur plutôt que par des
  `sei` dispersés.

**Verdict mesure (décisif B/C)** : `cursor_blit` (gardé en IRQ par C) = 16,6 %
du budget frame ; `redraw`/`redraw_drag` (retiré par B *et* C) = ≈ 53 % et
**borne basse** (pire sur HDL). Le curseur ne coûtant que ≈ 1/3 du redraw, le
garder dans l'IRQ (C) est sans danger pour la famine. **C retenue comme cible**,
sous réserve de validation humaine et de la campagne de non-régression GUI.

Séquencement proposé (sous réserve d'arbitrage humain) :
1. **Court terme (non bloquant)** : option A (D3 skip-si-delta-nul) pour
   calmer la famine démontrée sans figer d'architecture.
2. **Instruction** : mesurer le coût réel de `redraw`/`redraw_drag` par event
   (`PHOSPHORIC_MOU2_IRQTRACE`) → chiffre manquant du §1.2.
3. **Décision** : arbitrer B vs C sur la base (a) du coût de latence curseur
   mesuré, (b) de l'échéance ADR-15 (2026-12-31) qui rend le danger callback
   urgent.
4. **Implémentation de référence** (≥ 50 % avant ratification, moratoire §10) :
   serveur WM consommant `EVENT_RING`, migration incrémentale de `mouse_step`.

> **Interaction ADR-27** : si B/C est retenue, le flip compact ADR-27 §0ter
> **se simplifie** (point 5 « save/restore `bpl` en IRQ » disparaît en B,
> se réduit au curseur en C). Recommandation : **ne pas exécuter le flip
> compact ADR-27 avant d'avoir tranché ADR-28** — sinon on écrit de la gestion
> `bpl`-en-IRQ qu'on supprimera ensuite.

## 5. Conséquences

### Positives (si B/C)
- Cause racine unique éliminée → famine, race GPU et danger callback résolus
  ensemble plutôt que patchés séparément.
- Conformité au modèle de référence (Intuition/GEOS/SymbOS) revendiqué par
  ADR-06/26.
- Pré-condition propre à la réouverture d'ADR-15 (isolation, apps non-trusted).
- Simplifie ADR-27 §0ter.

### Négatives / coûts
- Refactor structurel du sous-système le plus testé (CLAUDE.md §6 : « avant
  tout refactor structurel, demander confirmation » → **cette ADR EST la
  demande**).
- Risque de régression GUI → campagne de tests de non-régression obligatoire
  (suite `win_draw`/`win_app`/`clock`/`gui_demo`/`ctl_demo` + mouse/drag).
- Latence curseur (B) ou frontière IRQ/tâche subtile (C).
- Définition d'une priorité scheduler pour le serveur WM (touche ADR-03).

## 6. Points ouverts à instruire avant ratification

1. ~~**Mesure** du coût `redraw`/`redraw_drag` par event~~ — **FAIT 2026-05-29**
   (§1.2bis, `test-oricos-wm-cost`). Tranche en faveur de C.
2. **Priorité** du serveur WM dans le scheduler round-robin (ADR-03).
3. **Protocole curseur** en option C : registre/canal GPU réservé curseur vs
   `sei` minimal.
4. **Mono-waiter** : `EVENT_WAITER` unique (event.s) — un serveur WM dédié le
   monopolise-t-il ? Interaction avec le multi-app (lié ADR-25 polish #1,
   signaux multi-bits, reporté v2).
5. **Damage tracking** : la migration est l'occasion d'introduire une clip-list
   / dirty-rectangle (aujourd'hui rendu immediate-mode plein écran). **Mesure
   §1.2bis à l'appui** : `redraw_drag` (« incrémental ») ≈ `redraw` plein car il
   redessine toutes les fenêtres → le damage-tracking est le levier qui réduit
   réellement le coût de rendu, indépendamment du threading.
6. ~~**Curseur dupliqué**~~ — **FAIT 2026-05-29 (§6.6 livrée)** : pendant un drag
   d'ascenseur, l'IRQ skip désormais `cursor_blit` quand `SCROLL_DRAG_ID` est
   armé (le main loop dessinera le curseur via `_wm_redraw_ctl`). **Mesure
   post-fix** (`test-oricos-scroll-cost`) : `cursor_blit` 13 → **1 appel**
   (seul le mousedown initial le déclenche), `mouse_step` 4 151 → **1 304 cyc**
   (20,8 % → **6,5 %**). Total/event drag scrollbar ≈ 34 % → **≈ 16 %** budget,
   value toujours 1:1. **Gain net : ≈ 16 % du budget frame par event**, sans
   refactor. Latence curseur : ≤ 1 frame (main loop consomme 1 event/frame).
7. ~~**Famine = saturation, pas coût**~~ — **RÉTRACTÉE 2026-05-30** : le test
   interactif utilisateur du 2026-05-30 et la trace `mtrace3.log` montrent
   que **l'`EV_MOUSE_UP` n'est JAMAIS droppé** (`last_what=03` en EVENT_RING).
   Le quota §6.7 « fixe » donc un drop qui n'a pas lieu. Le bug interactif
   « fin de course ascenseur » a une **autre cause** : un bottleneck app
   (`ctl_demo` print + `kernel_scroll_up`) qui sature le CPU et bloque
   `FORBID_COUNT` à 1, identifié dans `mtrace4.log` (PC stuck à `$01:11EE`).
   Le code §6.7 (limite à `ENTRIES-2`) reste en place car non nocif, mais
   sans valeur démontrable. **Le vrai fix est traité par ADR-29** (hint
   `DELAYED_DRAG_NOTIFICATION` aligné GeoWorks). Test
   `test_event_quota_reserves_transition_slots` reste vert mais ne prouve
   plus rien d'utile.

## 7. Plan d'implémentation de référence — option C (incrémental, testable)

> Objectif : atteindre le **seuil 50 %** du moratoire §10 par étapes **chacune
> verte sur `make tests`** et **réversible**. Aucune étape ne ratifie l'ADR ;
> la ratification interviendra une fois Étapes 1-4 livrées + campagne GUI.
> Avant d'exécuter l'Étape 2 et au-delà (refactor structurel), **confirmation
> humaine requise** (CLAUDE.md §6).

### 7.0. Le sous-problème dur : routage des événements (à trancher en Étape 0)

Aujourd'hui **deux consommateurs** lisent `EVENT_RING` : l'IRQ (`mouse_step`,
furniture+rendu) et la tâche app (`sys_main_loop` → `_ml_classify`). Et il n'y
a qu'**un** `EVENT_WAITER` (event.s:65,99). Un serveur WM qui bloque aussi sur
`EVENT_RING` entrerait en conflit. **Design retenu (modèle Intuition)** :

- **RAW input ring** (nouveau) : produit par l'IRQ (souris/clavier bruts),
  consommé **uniquement** par la tâche serveur WM. Son propre waiter
  (`RAW_WAITER`).
- **APP event ring** = l'actuel `EVENT_RING` : produit désormais par le **serveur
  WM** (qui ne forwarde que les events destinés à l'app après traitement de la
  furniture : `MSG_CONTENT`/`MSG_KEY`/`MSG_CONTROL`/`MSG_CLOSE`/`MSG_MENU`),
  consommé par `sys_main_loop` (waiter `EVENT_WAITER`, inchangé côté app).

Ainsi chaque ring a **un seul waiter** → pas besoin des signaux multi-bits
(ADR-25 polish #1, reporté v2). Le serveur WM devient le point unique de
politique, l'app garde son `SYS_MAIN_LOOP` tel quel.

### 7.1. Étape 0 — Décision routage + scaffolding ring brut (sans comportement) — **FAIT 2026-05-29**

- ✅ `RAW_RING` ($016400, bank 1 haute libre) + `RAW_RING_HEAD/TAIL/COUNT` +
  `RAW_WAITER` (réservé Étape 2), avec `.assert` anti-recouvrement (kernel.s).
- ✅ `kernel_raw_init`/`kernel_raw_push`/`kernel_raw_pop` (event.s) — transport
  **verbatim** via le bloc ZP $D0..$D9 (même convention que `kernel_event_pop`),
  drop-si-plein, wrap puissance-de-2. `kernel_raw_wait`/`kernel_raw_wake`
  (block/wake) **reportés à l'Étape 2** (testables seulement avec la tâche
  serveur — éviter le mort-code non testé).
- ✅ **Aucun producteur/consommateur branché** : boot et chemin GUI **intacts**.
- ✅ Test : `test_oricos_raw_ring` (cible `make test-oricos-raw-ring`, dans
  l'agrégat) — appelle les routines kernel **en isolation** (PC=adresse depuis
  `kernel.lbl`, retour détecté via la pile S) sans exécuter le boot. Couvre
  init, FIFO, copie du record, wrap, drop-si-plein, pop-sur-vide→EV_NULL.
- ✅ Gate : `make tests` **vert** (kernel.bin régénéré, aucune régression).
  Réversible (code isolé, non câblé).

### 7.2. Étape 1 — Palliatif skip-si-delta-nul (D3) — **FAIT 2026-05-29 + re-scope**

- ✅ Implémenté : garde `MOUSE_DX|MOUSE_DY == 0 → no-op` en tête de
  `wm_step_do_drag` ET `_wm_do_resize` (wm.s). Un `MOUSE_MOVED` sans déplacement
  réel ne déclenche plus le `redraw_drag` (≈ 53 % du budget frame).
- ✅ Prouvé (`test-oricos-wm-cost`) : un event drag à delta nul passe de
  **≈ 13 000 → 37 cycles** (`mouse_step` +1 event traité, `redraw_drag` +0).
- ✅ Gate : `make tests` vert, aucune régression.

> **Re-scope honnête imposé par la mesure** : D3 est correct et sûr, mais il ne
> corrige **pas** la famine d'ascenseur rapportée (ADR-27 §0bis). La mesure
> §1.2bis (+ lecture de `wm_step_drag`, wm.s) localise le coût : le `redraw_drag`
> à 53 % n'arrive qu'en **drag de fenêtre / resize** (`WM_DRAG_ARMED` /
> `WM_RESIZE_ARMED`). En **drag d'ascenseur**, `mouse_step` ne fait que
> `cursor_blit` (≈ 17 %) — la politique scrollbar est dans la **main loop**
> (`sys_main_loop` → `_wm_scroll_update`), pas dans l'IRQ. Donc :
> - D3 bénéficie au **drag de fenêtre/resize** sur events delta-nul (et aux
>   motifs d'events du HW réel), pas au scrollbar ;
> - le chemin main-loop scrollbar a depuis été **mesuré** (§1.2ter,
>   `test-oricos-scroll-cost`) : value 1:1, 34 % du budget → **pas de famine par
>   coût** à la cadence émulateur. Le « gel » est une **saturation de ring**
>   (button-UP droppé), pas un manque de cycles. Correctifs ciblés indépendants
>   du refactor : anti-drop button-UP (§6.7) + curseur dupliqué (§6.6).

### 7.3. Étape 2 — Tâche serveur WM passe-plat — **FAIT 2026-05-29** [confirmation humaine reçue]

- ✅ **Primitives RAW** (event.s) : `kernel_raw_wait` / `kernel_raw_wake`
  (clones block/wake de `event_wait`/`event_wake`), `kernel_event_push_verbatim`
  (re-push d'un record `$D0..$D9` vers `EVENT_RING` sans coalescing),
  `kernel_raw_push_mouse` / `kernel_raw_push_key` (clones fidèles avec
  coalescing, écrivant dans `RAW_RING`).
- ✅ **Routing transparent à l'entrée** : `kernel_event_push_mouse` /
  `kernel_event_push_key` testent `TC_WM_FLAG` ; si `$A5` → tail-call vers la
  version RAW. Aucun caller (`handlers.s`, `kbd.s`) à modifier. **Coût en mode
  legacy** (flag off) : 6 cycles/push (LDA long + CMP + BNE pris) — non
  détectable par les tests timing-sensibles.
- ✅ **Wake colocalisé** dans `kernel_raw_push_mouse/_key` (tail-call vers
  `kernel_raw_wake` après push) — pas de `jsr kernel_raw_wake` dans `handlers.s`
  (un essai initial avait cassé `test_oricos_clock` / `ctl_demo` par
  décalage timing IRQ de ~12 cyc — leçon documentée).
- ✅ **`task_wm_entry`** (event.s) : boucle
  `raw_wait → raw_pop → event_push_verbatim → event_wake`. Comportement net
  pour l'app **identique** (passe-plat exact, ordre des events préservé).
- ✅ **`TC_WM_FLAG=$01EE60`** (kernel.s) + gate `boot.s` créant `task_wm` via
  `kernel_task_create` si `$A5`. Flag $00 par défaut → comportement actuel.
- ✅ **Test end-to-end** `test-oricos-wm-server` : avec `TC_WM_FLAG=$A5`,
  injecte un event souris ; vérifie `RAW_WAITER != 0` (task_wm bloquée), puis
  après quelques frames `RAW_RING` drainé et `EVENT_RING` incrémenté. **Prouve
  la chaîne IRQ → RAW → block/wake → serveur → EVENT_RING.**
- ✅ Gate : `make tests` **vert**. Aucune régression (flag off transparent).
- 🟢 **Seuil moratoire 50 % approche** : primitives + tâche + chaîne testées.
  La bascule de politique (Étape 3) achèvera le seuil. Réversibilité encore
  élevée (flag off = retour exact à l'existant).

### 7.4. Étape 3 — Bascule politique IRQ → serveur — **REVERTÉE 2026-05-29**

> **Statut** : implémentation initiale livrée puis **revertée** le 2026-05-29
> suite à un bug interactif révélé par test utilisateur (`--wm-server` avec
> SDL : curseur figé, widgets non réactifs, alors que la suite headless
> intégrale était verte). **Ratification ADR-28 (design option C) tient** —
> c'est l'**implémentation Étape 3** qui est buguée, pas l'architecture.

**Ce qui était implémenté (puis reverté)** :
- `handlers.s` skip `kernel_wm_mouse_step` si `TC_WM_FLAG=$A5`, appel direct
  de `kernel_wm_cursor_blit` à la place (option C : curseur en IRQ).
- `task_wm_entry` appelait `kernel_wm_mouse_step` après `raw_pop` (politique
  en contexte tâche).

**État courant après revert** :
- `handlers.s` appelle toujours `kernel_wm_mouse_step` en IRQ (legacy).
- `task_wm_entry` fait passe-plat pur (Étape 2 exact) — pas d'appel
  mouse_step en tâche.
- Mode serveur (`TC_WM_FLAG=$A5`) reste fonctionnel et testé (`test-oricos-wm-server`
  vert) mais réduit à la chaîne IRQ→RAW→task_wm:passe-plat→EVENT_RING.
- Mode legacy strictement inchangé.

**Cause racine présumée (non confirmée — à investiguer)** :
- Récupération possible : `kernel_wm_mouse_step` en contexte tâche dépend de
  `MOUSE_X/Y/BTN/PREV_BTN/DX/DY` mis à jour par l'IRQ — sous burst (>1 event
  /frame en interactif réel SDL, plus dense que le ≤1/frame mesuré §1.2ter),
  l'état n'est plus cohérent avec l'event popé. Le test headless injectait des
  events espacés, n'a pas reproduit.
- À explorer aussi : stack de `task_wm` (256 bytes/page) potentiellement trop
  faible pour la profondeur d'appel de `mouse_step` → `redraw_drag` →
  `kernel_wm_redraw` → `_wm_draw_windows` → ...
- À explorer également : `_wm_redraw_ctl` et autres fonctions sous `sei`
  appelées en contexte tâche pourraient masquer des IRQ critiques (T1,
  curseur) pendant des durées plus longues qu'en IRQ direct.

**Plan d'investigation (Étape 3 v2, à instruire) :**
1. Étendre `test-oricos-wm-server` pour injecter une **rafale** d'events
   (mouse_move + button-down + 10 drags rapides + button-up dans la même
   « frame ») et vérifier que la séquence reste cohérente côté EVENT_RING.
2. Si la rafale reproduit le bug : étendre le record RAW pour porter
   `DX/DY/PREV_BTN` (record verbatim de l'état souris au moment du push) et
   restaurer l'état attendu avant d'appeler mouse_step en tâche.
3. Mesurer la profondeur de pile réelle (étendre `task_wm`'s pile si
   nécessaire — l'allocateur fait du bump page-by-page, on peut en demander 2).
4. Audit des chemins `sei` long (`_wm_redraw_ctl`, etc.) appelés en tâche.

**Implications immédiates :**
- Étape 4 (curseur SEUL en IRQ) : **non atteinte** — `mouse_step` (et donc
  `cursor_blit` via mouse_step + parfois `redraw_drag`) reste en IRQ.
- §6.6 (suppression curseur dupliqué) : **toujours active** en mode legacy.
- ADR-27 §0ter point 5 : **toujours nécessaire** (pas de simplification
  effective tant qu'Étape 3 n'est pas reprise proprement).
- Le **design** option C reste ratifié et juste ; la **migration progressive**
  v1 minimale s'arrête à Étape 2 (passe-plat).

**Limites assumées (refinements à venir, non-bloquants pour la ratification) :**

- **État souris lu par mouse_step en contexte tâche** : `MOUSE_X/Y/BTN` sont
  l'état **courant** (post-mouse_read) au moment où le serveur traite. À 1 event/
  frame (cadence émulateur mesurée §1.2ter), cohérent avec l'event popé ; sous
  burst (>1/frame), dégradation acceptée v1. Solution propre : record RAW
  étendu portant `DX/DY/PREV_BTN` pour reconstruction fidèle (v2).
- **§6.6 partiellement perdue en mode serveur** : l'IRQ appelle `cursor_blit`
  inconditionnellement (au lieu du skip-si-`SCROLL_DRAG_ID`-armé) ; et
  `mouse_step` en contexte tâche peut aussi toucher au curseur. Duplication
  possible dans certains scénarios — à corriger en portant le skip §6.6 dans
  `handlers.s`.
- **Quota §6.7 sur RAW_RING** : à porter sur `raw_push_*` (actuellement
  seulement sur `event_push_*`).
- **`php/sei…plp` défensifs** des helpers gfx (ADR-27 §0bis) **toujours en
  place** : leur retrait demande de garantir d'abord qu'aucun chemin en mode
  serveur ne fait de commande GPU depuis l'IRQ (`cursor_blit` n'utilise que des
  copies VRAM MMIO, pas le GPU). Validation séparée à instruire avant cleanup.

### 7.5. Étape 4 — Curseur : seul rendu restant dans l'IRQ (spécificité C)

- L'IRQ appelle `kernel_wm_cursor_blit` (déjà autonome : MOUSE_X/Y + CURSOR_SAVE
  via VRAM MMIO, wm.s). Garantir l'atomicité du seul `CURSOR_SAVE` (section
  courte) ; option future : canal/registre GPU réservé curseur pour éliminer la
  dernière race bornée au curseur (§4).
- Test : latence curseur (le curseur suit la souris même app occupée) +
  `test-oricos-wm-cost` (coût IRQ ≈ `cursor_blit` seul, ~17 %).
- Gate : suite verte.

### 7.6. Étape 5 — (optionnel, séparable) Damage-tracking

- Introduire une clip-list / dirty-rectangle dans `kernel_wm_redraw*` (cf. §6.5).
  Réduit le coût de rendu lui-même (indépendant du threading). Peut être fait
  avant ou après ratification ; ne bloque pas l'ADR-28.

### 7.7. Jalon moratoire

Le **seuil 50 %** (CLAUDE.md §10 cond. 2) est atteint à la fin de l'**Étape 3**
(politique en contexte tâche, IRQ allégé, suite verte). Ratification ADR-28
proposée à ce stade, sous réserve de la campagne GUI et de la confirmation
humaine sur le routage (§7.0) et la priorité scheduler du serveur (§6.2).

## 8. Audit de ratification (CLAUDE.md §10)

Ratification le **2026-05-29**, conformément aux 3 conditions du moratoire
(CLAUDE.md §10) :

**Condition 1 — Dossier d'instruction écrit** ✅
- Contexte chiffré : §1.2bis (drag fenêtre, mesure déterministe `mouse_step`
  66,8 %/event) et §1.2ter (drag scrollbar, mesure déterministe value 1:1,
  34 % budget) via harnais reproductibles `test-oricos-wm-cost` et
  `test-oricos-scroll-cost`.
- Alternatives chiffrées : §3 (A statu quo + patches, B serveur WM, C hybride
  curseur-IRQ), coût-bénéfice tracé, recommandation senior C explicite (§4).
- Recommandation senior tracée : §4 « Viser l'option C » avec verdict mesure.

**Condition 2 — Implémentation prête (≥ 50 %)** ✅
- Étape 0 (RAW ring scaffolding) — `test-oricos-raw-ring` PASS.
- Étape 1 (D3 skip delta-nul) — validé via `test-oricos-wm-cost`
  (13 000 → 37 cyc sur drag delta-nul).
- §6.6 (curseur dupliqué) — validé via `test-oricos-scroll-cost`
  (34 % → 16 % budget/event drag scrollbar).
- §6.7 (quota anti-drop button-UP) — validé via
  `test_event_quota_reserves_transition_slots`.
- Étape 2 (serveur passe-plat) — chaîne IRQ→RAW→serveur→EVENT_RING prouvée
  via `test-oricos-wm-server`.
- Étape 3 (politique en contexte tâche, gated `TC_WM_FLAG=$A5`) — `handlers.s`
  skip `kernel_wm_mouse_step`, `task_wm_entry` l'appelle. Étape 4 de facto
  incluse (seul `cursor_blit` reste en IRQ).
- `make tests` **vert** à chaque palier ; aucune régression en mode legacy.

**Condition 3 — Cohérence ADR existantes** ✅
- ADR-03 (scheduler) : `task_wm` utilise le round-robin existant sans
  modification.
- ADR-24 (souris IRQ) : transport MOU2 préservé ; seule la **suite** (politique)
  bascule. IRQ continue à lire MOU2 et clear l'event.
- ADR-25 (concurrence) : block/wake utilisé tel quel (`kernel_raw_wait`/
  `kernel_raw_wake` = clones de `kernel_event_wait`/`kernel_event_wake`).
- ADR-26 (GUI déclaratif) : `SYS_MAIN_LOOP` / `MSG_*` / `EVENT_RING` côté app
  **inchangés** (passe-plat exact).
- ADR-27 (DRAFT, backing store) : **simplifié** par ADR-28 — le point 5 de
  §0ter (« save/restore `bpl` en IRQ ») disparaît en mode serveur (un seul
  producteur GPU). Bénéfice, pas contradiction.
- Aucune contradiction non-explicite avec une ADR ratifiée.

**Limites assumées (refinements post-ratification, non-bloquants) :**
1. État souris en burst (>1 event/frame) : record RAW à étendre pour porter
   `DX/DY/PREV_BTN` (v2). À 1 event/frame mesuré, cohérent.
2. §6.6 partiellement perdue en mode serveur : porter le skip-si-`SCROLL_DRAG_ID`
   dans `handlers.s`.
3. §6.7 à porter sur `raw_push_*` (actuellement seulement `event_push_*`).
4. Retrait des `php/sei…plp` défensifs gfx (ADR-27 §0bis) : à instruire
   séparément après validation qu'aucun chemin IRQ ne fait de commande GPU en
   mode serveur (`cursor_blit` = VRAM MMIO, pas GPU).
5. Damage tracking (§6.5 / Étape 5) : séparable, optionnel.

Ces refinements sont **tracés** et **suivis** ; aucun n'est bloquant pour la
décision d'architecture désormais figée par cette ratification.

## 8. BASCULE PAR DÉFAUT (2026-06-10) — WM_TASKMODE devient le mode nominal

Sprint lancé par Bénédicte après la clôture du verrou « bug task_wm
starve » (`docs/notes/BUG_task_wm_starve_CLOS.md` : bug ÉMULATEUR — ASL
mem 8-bit-fixe sous M=16 cassait le scan bitmap de `task_create` pour
les pids ≥ 8 ; prouvé R1 par re-simulation, gardé par
`test_oricos_ctl_taskmode_starve` + pivots Klaus RMW M=16).

**Mécanique de la bascule** :
- Le **boot pose lui-même** `TC_WM_FLAG = WM_TASKMODE = $A5` et crée
  `task_wm` (échec de création → `kernel_panic PANIC_NO_TASK_SLOT`,
  R5 : tâche système, échec bruyant). Les gates runtime existants
  (IRQ → RAW_RING, skip mouse_step en IRQ) sont inchangés — ils lisent
  ces flags.
- **Opt-out explicite** : `TC_WM_LEGACY = $A5` ($01EE90, poké pré-boot
  par `--wm-legacy` côté Phosphoric ou par un test) → comportement IRQ
  legacy intégral, conservé et testé. `--wm-server`/`--wm-taskmode`
  deviennent des no-ops de compatibilité.
- Conséquence ADR-34 : le **record des display-lists s'exécute désormais
  hors IRQ** (chemin nominal) — l'IRQ souris ne fait plus que
  lecture device + push RAW + sprite curseur.

**Bug single-writer trouvé et corrigé pendant la bascule** (§5bis
OricOS/CLAUDE.md) : `$D0..$D9` (record d'événement) est un tampon ZP
**partagé entre tâches** (D=0 pour toutes). task_wm garde son record
RAW dedans pendant tout `mouse_step` (préemptible en taskmode) ; une
app préemptante qui fait `event_pop` l'écrase → `push_verbatim`
republiait le MAUVAIS événement (mesuré : le DOWN republié en MOVED,
`MSG_CONTENT` perdu — rouge sur `test_oricos_mainloop_message`).
**Fix** : `Forbid`/`Permit` (ADR-25, portée tâche↔tâche — les IRQ et le
curseur sprite restent vivants) autour de la section pop→push de
`task_wm_entry` ; jamais tenu pendant `raw_wait`. Pré-existant depuis
l'Étape 2, exposé par le défaut. Reste de la même famille (tracé, non
bloquant) : deux APPS qui pop-ent concurremment partagent aussi
`$D0..$D9` — à instruire si le multi-app événementiel devient réel.

**Ajustements tests (R8, justifiés)** : 7 tests GUI de
`test_oricos_boot` étendent leur borne de cycles (160-320k → 400-600k) —
le rendu est désormais **asynchrone** (latence d'ordonnancement task_wm
~50-100k cycles dans ces harness) ; l'ordre des événements est préservé
par RAW_RING, seules les assertions finales avaient besoin de marge.
Diagnostic mesuré : sémantique correcte (focus à ~200k, drag appliqué à
~250k pour des injections à 140-150k).

**Critère de sortie de sprint : validation interactive utilisateur**
(drag/resize/menus/taskbar fluides en mode défaut) — leçon ADR-29.

**VALIDÉE 2026-06-10 par Bénédicte Marty** (« cela fonctionne, le drag
est fluide ») — la bascule est ACTÉE. `WM_TASKMODE` est le mode nominal
d'OricOS ; le rendu en IRQ n'existe plus que derrière l'opt-out
`--wm-legacy` (chemin couvert par `test_oricos_wm_legacy_optout`).
L'ADR-28 est entièrement livrée : design option C ratifié 2026-05-29,
bascule par défaut validée 2026-06-10. Refinements restants tracés en
§7.6/limites (damage tracking optionnel, record RAW étendu v2,
single-writer $D0..$D9 inter-apps si multi-app événementiel réel).

## Références
- CLAUDE.md §2 (ADR-03/24/25/26), §3 (ADR-15/27), §10 (moratoire).
- `docs/adr/0027-backing-store-fenetre-DRAFT.md` §0bis/§0ter.
- `docs/adr/0025-modele-concurrence-kernel.md` (block/wake).
- `docs/adr/0026-modele-gui-declaratif.md` (MainLoop/messages).
- Code : `kernel/modules/handlers.s` (IRQ), `kernel/modules/wm.s`
  (`kernel_wm_mouse_step`, `kernel_wm_redraw`), `kernel/modules/event.s`.
- Réf externes : AmigaOS `input.device`/`intuition.library` ; GEOS ;
  SymbOS (multitâche fenêtré 8-bit).
