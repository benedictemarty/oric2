# CHANGELOG — Workspace Oric 2

CHANGELOG commun aux 3 sous-projets (`Phosphoric`, `OricOS`, `oric2`
docs). Format Keep a Changelog v1.0.0.

Entrées détaillées par sous-projet :
- [Phosphoric/CHANGELOG](./Phosphoric/CHANGELOG)
- [OricOS/CHANGELOG.md](./OricOS/CHANGELOG.md)

## [2026-06-10l] — ADR-34 étape B LIVRÉE : GPU-ISA v2 async (FIFO + IRQ + CAPS)

Device Phosphoric : FIFO 16 (snapshot au TRIGGER), mode timé opt-in
(harness legacy inchangés, émulateur vivant timé), IRQ de complétion
(IRQF_GPU, ack W1C, enable INT_CTRL), CAPS/VERSION en lecture TRIGGER
($32, additif), QFULL/OVF sticky — 4 tests device avec rouge-check
démontré. Kernel : GPU_CAPS_KERNEL lu au boot, blit_post + drain
(GUICODE), compose post-and-continue routé par capacités, sans drain
final (audit callers). Gain mesuré : compose 22 534 → 10 591 cyc CPU
(−52 %) vs carte v1 simulée — test end-to-end dans make tests. Leçon
collatérale : saturation du chemin IRQ legacy mesurée à 400 Hz souris
(±30 octets de code basculaient task_wm en famine) — test atomique
recalibré à 125 Hz réaliste ; argument supplémentaire pour l'étape C.
Critères de ratification B remplis — arbitrage Bénédicte attendu.

## [2026-06-10k] — ADR-34 ouverte : GPU-ISA (primitives GUI, async, display-lists)

Dossier d'instruction à la demande de Bénédicte (« les fenêtres GUI sont
des primitives du GPU » ; « si je change de carte, je ne change pas les
primitives » ; « l'objectif est la réduction de la charge IRQ »).
Références : Apple IIgs (jumeau 65C816 — stratification QuickDraw II =
contrat, et leçon de perf SANS blitter), SymbOS (mêmes primitives sur CPC
soft et MSX V9938 command engine async/CE), Amiga (Blitter/Copper). 4
options chiffrées ; recommandation : B (v2 async — FIFO + IRQ complétion
+ vsync + GPU_CAPS) puis C (v3 display-lists EXEC_LIST : une fenêtre = sa
liste, l'IRQ ne rend plus rien) ; D (DRAW_WINDOW figée) écartée. Règles de
contrat : sémantique immuable, extensions additives, GPU_CAPS obligatoire,
Phosphoric = suite de conformance. CLAUDE.md §3 mis à jour. Dossier :
docs/adr/0034-gpu-isa-gui-primitives-DRAFT.md.

## [2026-06-10j] — P0-a : bug task_wm_starve CLOS (cause prouvée)

C'était un bug ÉMULATEUR fixé le 2026-06-01 (Phosphoric 08a9bf, ASL mem
M-aware) — dossier jamais refermé, verrou fantôme sur ADR-28 pendant 9
jours. Preuve apportée : kernel d'époque + émulateur actuel = PASS ;
kernel d'époque + bug CPU re-simulé = 10/10 STALE, sites loggés
(kernel_task_create scan bitmap + bitmap_clear). Mécanisme : ASL mem
8-bit-fixe en M=16 → masque du pid 8 (premier bit du HIGH byte) jamais
posé dans TCB_BITMAP → la création suivante (ctl_demo) écrasait le TCB de
task_wm — d'où la double condition exacte des flags. Le verrou ADR-28 est
levé ; la bascule WM_TASKMODE par défaut reste un sprint dédié avec
validation interactive. Clôture : docs/notes/BUG_task_wm_starve_CLOS.md.

## [2026-06-10i] — Revue critique senior du kernel OricOS

`docs/notes/REVIEW_senior_OricOS_2026-06-10.md` : audit 14 724 lignes.
3 problèmes structurels (WM dans l'IRQ — verrou = bug task_wm_starve ;
layout mémoire manuel — 4 collisions historiques de la même classe ;
wm.s monolithe 5 578 L), 6 problèmes de second rang (erreurs silencieuses,
boot pollué de tests, 3 rings en cohabitation, aliasing ABI $D0, GPU
sync sous sei, TCB_PRIO mort), risques HDL, et plan priorisé P0/P1/P2.
Recommandation transversale : P0-a (finir ADR-28) et P0-b (layout par le
linker) avant toute nouvelle feature GUI.

## [2026-06-10h] — SP-3.p M.1 : highlight du menu ouvert (rouge→vert)

Titre du menu ouvert en vidéo inversée (fond blanc mesuré au proportionnel,
texte noir) — feedback GeoWorks standard. Garde :
test_oricos_menu_open_highlight. Détails : OricOS/CHANGELOG.

## [2026-06-10g] — SP-3.p : suite GUI — fontes proportionnelles + textes de dialogues

Reprise des briques GUI parquées (réf GeoWorks/GEOS, ADR-26 déclaratif) :
**F.1 fontes proportionnelles** — table FONT_WIDTHS générée du charset XVGA
(gen-font-widths.py), kernel_tk_text_width (mesure) + kernel_tk_label_prop
(rendu char-à-char, avance variable), nouveau type widget LABELP, segment
GUICODE ($9200, CODE était plein). **D.1 dialogues** — tag DB_TEXT (GEOS
DBTXTSTR : relx16 rely16 ptr16) dans SYS_DO_DLGBOX + message optionnel
SYS_ALERT ($D0-$D2), staging DLG_TEXT_BUF, troncature par largeur pour
l'alerte. Au passage : fix bug UX préexistant (les dialogues ne peignaient
JAMAIS leurs widgets — aucun redraw avant la boucle modale) et fix far
addressing des labels GUICODE (widths lues en bank 0 → texte superposé, vu
ROUGE par l'assertion d'étendue). SDK : defines DB_*, oricos_alert zérote
$D0-$D2, oricos_alert_msg ajouté. Démos : --dlg-demo / --alert-demo
(Phosphoric). Tests dlgbox/alert durcis (staging + widget + pixels +
étendue proportionnelle). Suite complète verte. Screenshots validés.

## [2026-06-10f] — ADR-32 RATIFIÉE : invariant ZP/IRQ P1/P2/P3

Les 2 conditions §10.7 restantes sont traitées : (1) invariant ZP/IRQ acté
— P1 enveloppe IRQ_ZP_SAVE, P2 sections critiques rings côté tâche, P3
frame IRQ 16-bit — texte normatif §10.14(1) repris dans OricOS/CLAUDE.md
§5nonies (+ item 10 checklist) ; (2) course exempt↔focus actée option (γ)
telle qu'implémentée (kernel_kbd_waiter_eligible), critère de réouverture
tracé. Ratification §11 conforme moratoire (3 conditions citées et
vérifiées : dossier complet, implémentation 100 % livrée avec 4 gardes
rouge→vert, cohérence ADR-25/28/33). Périmètre honnête : la migration
mouse_step hors IRQ (option B originelle) reste explicitement NON ratifiée.
Fichier renommé docs/adr/0032-zp-race-irq-task.md, CLAUDE.md §3→§2,
ADR_SUMMARY mis à jour.

## [2026-06-10e] — ADR-32 §10.13 : TICK_COUNTER/NMI $5500 fixé, bitmap TCB = fausse alerte

(a) TICK_COUNTER ($015500) écrasait l'opcode rti du segment NMI_HANDLER à
chaque tick — tout NMI réel exécutait le compteur comme opcode. Rouge :
test-oricos-nmi-safe (byte handler = $0D au lieu de $40 ; NMI réels tuaient
le run). Fix : TICK_COUNTER relocalisé → $019093 + asserts non-overlap ;
5 lectures hardcodées migrées dans les tests. Vert : handler intact, 4 NMI
absorbés, app au bout. Garde dans make tests.
(c) TCB_BITMAP « slot 4 » : fausse alerte close par relecture + mesure —
convention bit=pid (bit 0 = sentinelle), les valeurs observées ($00EF run
cassé, $01EF run sain) sont correctes. Aucun changement de code.

## [2026-06-10d] — ADR-32 §10.12 : fenêtre EVT_TMP/T1 fermée (push EVENT_RING atomique)

La pré-cond « pushers EVENT_RING = IRQ-only, I=1 » était cassée depuis
push_menu (ADR-30) et push_verbatim (task_wm, ADR-28) qui tournent côté
tâche : section critique [COUNT → EVT_TMP → record → RMW TAIL/COUNT]
préemptible par les pushers IRQ → file corrompue. Rouge : nouvel invariant
test-oricos-evt-push-atomic (« _evt_advance_tail jamais avec I=0 ») — 10/10
violations sur scénario wm-server. Fix : php/sei…plp sur les 2 pushers
tâche (pattern event_pop/raw_pop). Vert : 0 violation, garde mécanique dans
make tests (couvre tout pusher futur). Pré-cond réécrite en tête d'event.s.

## [2026-06-10c] — ADR-32 §10.11 : Opt-A retiré, classe souris-drag validée rouge→vert

Sur demande de Bénédicte. La sentinelle test-position-shift v2.2 (multi-slots
$73-$78 vs args COP $D0-$D4) fait enfin ROUGIR la classe souris-drag : 20
corruptions sans Opt-A (l'IRQ drag écrase GFX_BASE/GFX_COLOR pendant un body
sys_gfx_fill_rect — v2.1 surveillait $73 que le chemin fill_rect16 n'écrit
jamais). Protection de classe : kernel_irq_handler sauvegarde/restaure les
scratch ZP $08-$93 (140 octets, IRQ_ZP_SAVE $019100) autour du bloc souris —
tous les syscalls protégés, coût payé uniquement sur IRQ souris. Opt-A
(sei/cli sur fill_rect/flush) retiré. VERT : 0/33 phases, irq_in_window>0,
suite complète verte sans rebasage des budgets cycles. Garde de classe dans
make tests.

## [2026-06-10b] — ADR-32 §10.10 : FIX frame IRQ 16-bit livré (option A, validée par Bénédicte)

`kernel_irq_handler` sauve/restaure désormais A/X/Y en 16-bit (`rep #$30`),
et les 7 forgeurs de resume frame (task_create, boot task B, yield, sleep,
read_char, get_next_event, main_loop, event_wait, raw_wait) sont au format
10 octets [Y16][X16][A16][P][PCL][PCH][PBR]. Validation : test verrouillage
irq-frame-m16 ROUGE→VERT (intégré à make tests), ground-truth pad+100
no-SEI ⇒ test_oricos_clock PASS, suite Phosphoric verte intégralement.
L'invariant « X=1 aux points préemptibles » n'est plus requis (couvert par
construction). Retrait Opt-A différé (protège aussi la classe souris-drag).

## [2026-06-10] — ADR-32 §10.9 : chasse au slot CLOSE — cause racine = frame IRQ 8-bit vs M=16

Le bug clock Opt-A est élucidé par la mesure (§10.3bis « mesurer, jamais
deviner ») : la collision n'est pas un slot ZP mais le **registre B (A.high)
à travers la frame IRQ 8-bit** de `kernel_irq_handler` (`sep #$30` +
pha/phx/phy). Preuve par trace registres : tâche clock interrompue en M=16
(C=$BC98) reprend avec C=$5C98 → CURSOR_ADDR corrompu → prints hors écran.
Test de verrouillage `test-oricos-irq-frame-m16` ROUGE démontré (3/5) sur
kernel courant, même avec Opt-A. Fix kernel à arbitrer (option A recommandée :
frame 16-bit). Détail : `docs/adr/0032-zp-race-irq-task.md` §10.9,
CHANGELOGs Phosphoric (instruments) et OricOS (analyse kernel).

## [2026-06-09e] — Fix Phosphoric DPR + Klaus 65C816 étendu (17 tests)

Bug Phosphoric latent fixé : `addr816_zp`/`zp_x`/`zp_y` ignoraient `cpu->D`
(Direct Page Register) — ~20 opcodes Direct Page simple accédaient à
`dp_byte` au lieu de `D + dp_byte`. Conforme désormais à WDC W65C816S §6.6.
Impact OricOS nul (DPR=0 partout) mais bug latent invisible critique pour
conformité spec et tout consommateur futur DPR != 0.

Klaus 65C816 étendu livré : 17 tests couvrant COP/IRQ/RTI/NMI stack frame
mode N, TCD/TDC, RMW M=16 pivots (ASL/INC/DEC), régression DPR. Réponse
au trou conformance documenté dans `BUG_phosphoric_rmw_mem_M16.md` §4.

Hypothèse (δ) ADR-32 §10.5 (bug Phosphoric cause bug clock historique)
**affaiblie** : suspects principaux écartés (T1, COP/IRQ frame, RMW M-aware,
push/pull P). Bug clock probablement côté kernel (α lectures, β timing,
γ stack — voir mémoire `project_adr32_chasse_slot_etat.md`).

### Fixed — Phosphoric
- `src/cpu/cpu65c816_opcodes.c` : `addr816_zp`/`zp_x`/`zp_y` ajoutent
  `cpu->D` (Phosphoric commit `8d06198`).

### Added — Phosphoric
- `tests/unit/test_cpu65c816_cop_frame.c` (17 tests) + target Makefile
  `test-cpu65c816-cop-frame` intégré dans `make tests` (24/24 PASS).

## [2026-06-09d] — `make test-position-shift` : v1 scaffolding (détection non démontrée)

⚠️ **Rectification** : annoncé hier comme « verrou CI de la classe »,
qualification fausse. Run contradictoire 2026-06-09 (sei/cli retirés
de sys_gfx_fill_rect — collision Opt-A *prouvée*) ⇒ test toujours vert
sur 18 runs. Cause : stimulus = move nu, jamais sur la branche clic/drag
de `_wm_mouse_step_body` qui écrit `WM_ARG_*`. v1 = scaffolding seul
(cible Makefile + structure + canary), aucune capacité de détection
démontrée. **Aucune pression CI sur §10 actuellement.**

v2 requis (consigne senior) : stimulus drag (button-down + moves), cible
validation = gfx_fill_rect-sans-SEI ⇒ ROUGE avant tout push, détecteur
mémoire direct (lecture `WM_DP_TMP`/rect) plutôt que canary `clock: done`.

### Added — Phosphoric
- `tests/integration/test_oricos_position_shift.c` + targets
  `test-oricos-position-shift` / `test-position-shift`. Conservé comme
  scaffolding v1, à enrichir en v2.

## [2026-06-09c] — Sign-off senior session Fix B v2 + Opt-A

CR de clôture session formalisant le sign-off externe : Fix B v2 + Opt-A
shippés, Opt-C correctement non shippé. Conditions de suivi tracées
(miroir ADR-32 §10.7) : §10 comme item réel, invariant ZP IRQ en ADR
ratifié, `test-position-shift` à lander, ligne ADR sur course
exempt↔focus, coalesce-on-overflow RAW_RING optionnel, retrait Opt-A
post-§10 bonus.

### Docs
- `docs/CR/CR_signoff_session_FixB_v2_OptA.md` — sign-off + 6 conditions
  de suivi non bloquantes pour le push.

## [2026-06-09b] — Réentrance IRQ↔syscall ZP scratch : sei ciblé (Opt-A)

Suite mot expert sur le hang `test_oricos_clock` ~50 octets : recadrage de
la cause racine. PAS positionnel, c'est la **réentrance IRQ top-half ↔
syscall sur les ZP scratch kernel** — FORBID ne couvre que tâche↔tâche.
Confirmation via test 2 senior (sei ciblé) en A/B clean.

### Fixed — OricOS
- `sys_gfx_fill_rect`, `sys_win_flush` : `sei`/`cli` en tête/sortie. Ferme
  la fenêtre de réentrance sur les 2 sites confirmés (commit `5017990`).
- Suite Phosphoric 24/24 verte. IRQ_CONFORMITE §5 inchangé (14987 cyc).

### Investigation — pivot Opt-C → Opt-A
Tentative Opt-C radicale (retirer cli de cop_handler, ajouter cli aux 6
bloquants) : régression IRQ_CONFORMITE (34k cyc) + test_oricos_win_app
fail. Cause non identifiée v1. Reverté. Opt-C complète ouverte pour
**ADR-32 §10** (audit systématique ZP partagés + scratch dédiée IRQ
top-half, sprint dédié post-livraison interactive).

### Référence
`docs/CR/CR_reentrance_irq_syscall_confirmed.md` (analyse complète,
A/B test, options, §9 pivot Opt-C→A).

## [2026-06-09] — Fix B (BUG_drag_v2_fragments) : désactiver coalescing MOVED en taskmode

Origine : `MOT_EXPERT_drag_v2_fragments.md`. Bug A (delta 16-bit) résolu
2026-06-02 ; Bug B (fragments visuels persistants sous drag rapide en
`WM_TASKMODE=$A5`) restait ouvert. Cause racine confirmée : le coalescing
MOUSE_MOVED dans `kernel_raw_push_mouse` fusionne N events IRQ en un seul
record RAW_RING avec la position FINALE, ce qui dérive un delta `WHERE -
WM_LAST` qui dépasse la largeur fenêtre. L'erase OLD seul dans
`kernel_wm_redraw_drag` ne couvre pas la trajectoire OLD → NEW → fragments.

### Fixed
- **OricOS** : gate `WM_TASKMODE=$A5` au début de `kernel_raw_push_mouse`
  qui saute le bloc de coalescing → chaque IRQ MOU2 pousse son propre event
  individuel. `task_wm` les drain un à un → delta petit (≤ MOU2 IRQ rate
  per event), erase OLD couvre, plus de fragments. EVENT_RING côté app
  conserve son coalescing intact dans `kernel_event_push_mouse` (UI app
  recevait des events groupés, pas de régression). Coût : +10 octets CODE
  ($5259→$5263), budget IRQ_CONFORMITE §5 inchangé (mouse_step lit toujours
  l'event poppé event-source).
- Risque accepté : RAW_RING (16 slots) peut overflow sous burst mouse —
  drop silencieux, pas de régression visible (task_wm verra moins d'events
  mais leur substance est dans le suivant).

### Investigation collatérale livrée
- **Fragilité position-dépendante test_oricos_clock** documentée : tout
  shift de CODE OricOS ≥ 50 octets fait crasher la tâche clock en milieu
  d'iteration 2 du loop print_string (TCB state → DEAD). Investigation via
  oricrobot (commande `cpu` ajoutée) : reproductible à +50 bytes padding
  pure, indépendant du contenu ajouté. Cause racine non encore tranchée
  (kernel vs émulateur §5septies). Fix B v1 alternatif (rect englobant
  `kernel_wm_redraw_drag`, +65 octets gaté taskmode) butait sur cette
  fragilité — d'où le pivot vers cette v2 (-coalescing, +10 octets).
  À investiguer en cycle dédié si une autre tâche le rejoue.
- **Phosphoric** : nouvelle commande `cpu` dans `tools/oricrobot.c` dumpe
  PBR/PC/DBR/D/S/C/X/Y/P/E — utilisée pour tracer le hang clock, conservée
  pour debug futur.

### Notes
- L'option E originale (rect englobant) reste valable conceptuellement
  mais cette v2 est plus simple, plus locale, et n'augmente pas la charge
  fill_rect16. Si une situation nécessite l'union OLD ∪ NEW (taskmode
  + saut > IRQ rate, théoriquement impossible avec ce fix), revisiter.
- Commit unique cross-repo (Phosphoric + OricOS).

## [2026-06-02c] — Palier 1 : relocation variables RAM kernel ($5432→$9032)

Origine : audit `BUG_code_ecrase_variables.md` (corruption silencieuse de CODE
par les écritures `TASK_*` runtime — l'assert linker `<= $5500` était mal
calibré, le vrai plancher était $5432). Remap `$0154xx` → `$0190xx` côté
OricOS + suivi des littéraux côté Phosphoric.

- **OricOS** : 28 décl dans `kernel/kernel.s` remappées vers zone libre $9000+.
- **Phosphoric** : ~80 littéraux `0x0154xx` dans 8 tests intégration +
  `src/main.c` remappés. Refs `0x0155xx` (TICK_COUNTER, NMI_HANDLER) intactes.
- **Résultat** : suite Phosphoric 100 % verte (gain net 7+ tests débloqués
  vs baseline — les fails pré-existants `event_syscall`/`mainloop_*`/`ui_define`/
  `dlgbox`/`alert` étaient eux aussi des effets de la corruption silencieuse).
- **Différé** : Fix B (rect englobant `kernel_wm_redraw_drag` pour fragments
  drag taskmode) stashé — sa taille (+111 octets) décale les symboles wm.s
  au point de casser `test_oricos_clock` (budget cyc serré). À reprendre avec
  optim taille (subroutine + factor X/Y, viser ≤ 50 octets) + Option D (gate
  `WM_TASKMODE=$A5`, validé par mot expert : rect englobant inutile en legacy).

## [2026-06-02b] — Gouvernance debug CLAUDE.md §5bis-§5octies + fix sat8 §5ter

### Added
- **OricOS/CLAUDE.md** : 7 sections de gouvernance (§5bis-§5octies) tirées
  des bugs résolus récemment. Invariant event-source taskmode, largeur
  16-bit grandeurs dérivées, discipline RMW M-aware, 9 règles process
  debug (R1-R9), frontière kernel/émulateur, checklist commit 9 points.

### Fixed
- **§5ter / audit grep proactif** : sat8 propre dans `install_event_state`
  (`event.s` helper `_install_sat8`). Remplace troncature 8-bit
  (commentaire trompeur « sat8 implicite ») par clamp signé [-128, +127].
  Deltas multiples de 256 (`+256`/`−256`) ne tronquent plus à `$00`,
  préservant le skip-zero test 8-bit dans `wm_step_do_drag`/`_wm_do_resize`.
- Détecté par grep proactif §5ter (consommateurs `MOUSE_DX/DY` +
  `_sext8_to16`). Alternative naïve (test 16-bit dans hot path IRQ)
  causait régression `test_oricos_wm_cost` +19000 cycles inexpliquée
  → préférence pour fix à la source.
- Note : `_sext8_to16` n'a plus d'appelants après Fix A → dead code
  à nettoyer dans une passe ultérieure.

## [2026-06-02] — BUG_drag_v2_fragments Fix A : delta 16-bit drag taskmode

### Fixed
- **Bug A (delta tronqué 8-bit) résolu** suite expert
  `BUG_drag_v2_fragments.md`. En `WM_TASKMODE=$A5`, le coalescing MOVED
  fusionne N events en 1 → delta dérivé `WHERE - WM_LAST` pouvait dépasser
  ±127 px. La v2 stockait ce delta tronqué 8-bit dans `MOUSE_DX/DY` puis
  `_sext8_to16` inversait le signe → fenêtre déplacée du mauvais côté
  (« faux drag inverse » dans capture utilisateur).
- **Fix A** : nouveaux slots `MOUSE_DX16/DY16` (16-bit signé) en parallèle
  des `MOUSE_DX/DY` 8-bit (legacy IRQ conservé). `task_wm_install_event_state`
  écrit le delta 16-bit complet ; `wm_step_do_drag` / `_wm_do_resize`
  lisent `MOUSE_DX16/DY16` directement (M=16, plus de `_sext8_to16`).
  `kernel_mouse_read` (legacy IRQ) sign-extend `MOU2_DX/DY` 8-bit →
  `MOUSE_DX16/DY16` pour cohérence.
- **Conséquence CODE budget** : `TICK_COUNTER` poussé `$5430→$5500`
  (gap `$54F6-$557F` libre). Assertion `__CODE_LOAD__ + __CODE_SIZE__
  <= $5500` mise à jour. 2 tests Phosphoric (boot/helloc) mis à jour.
- **Note Fix B** : rect englobant `kernel_wm_redraw_drag` initialement
  tenté → régression 29/48 tests headless → reporté en session de debug
  dédiée. Bug B (erase partiel sur sauts coalescés > largeur fenêtre)
  toujours présent visuellement, mais la fenêtre va au bon endroit
  (correction de Bug A).
- **Tests** : suite Phosphoric verte (incl. `test_oricos_taskmode_full`,
  `test_oricos_ctl_taskmode_starve` drag scenario W/H préservé).
- Commits OricOS `62f2690` + Phosphoric `8e5222b`.

## [2026-05-31aa] — ADR-32 Étape 4 NON ratifiée — rétractation interactive

### Changed
- **ADR-32 §9** : ajout rétractation Étape 4. Test interactif utilisateur
  avec `--wm-taskmode` (TC_WM_FLAG + WM_TASKMODE = $A5) → **curseur
  invisible**. Test contrôle `--wm-server` seul → curseur OK. Donc la
  bascule `WM_TASKMODE=$A5` (migration `cursor_blit` IRQ→task) ne tient
  pas sans plan d'atomicité supplémentaire.
- **Leçon méthodo** : `test_oricos_taskmode_full` mesurait
  `CURSOR_OLD_X/Y` (état logique post-blit) — pas le pixel. **Headless
  vert ≠ interactif vert**. Étape 4 v2 doit livrer un test pixel-level
  (inspection VRAM à offset curseur, ou snapshot PPM comparé).
- **Pas de revert kernel** : `WM_TASKMODE` reste défini, default `$00`,
  flag CLI `--wm-taskmode` conservé comme harness de debug pour
  instruire les hypothèses (préemption T1 mid-blit / race ZP §3.3a /
  re-save backing avant restore).

### Added
- **Phosphoric / main.c** : option `--wm-taskmode` (pose `TC_WM_FLAG +
  WM_TASKMODE = $A5` au boot). Conservée comme harness debug malgré
  rétractation.

## [2026-05-31z] — ADR-32 Étape 4 / §3.1 : validation headless mode complet

### Added
- **Phosphoric / test_oricos_taskmode_full** : valide en headless le
  mode complet ADR-32 Étape 4 (= IRQ_CONFORMITE §3.1) : `TC_WM_FLAG=$A5`
  + `WM_TASKMODE=$A5` ensemble → IRQ skip mouse_step, task_wm consomme
  RAW_RING et exécute mouse_step + cursor_blit en contexte tâche.
  Scénario : mouse_move + clic titlebar. Curseur bouge, focus change,
  pas de STP. Signal fort que l'infrastructure §3.1 fonctionne ;
  validation interactive utilisateur reste requise pour drag/resize.
  Cible `make test-oricos-taskmode-full` (+ aggrégateur `tests`).
  Réf : ADR-32 Étape 4, IRQ_CONFORMITE §3.1.

## [2026-05-31y] — IRQ_CONFORMITE §5.4 : test_oricos_zp_race scaffolding

### Added
- **Phosphoric / test_oricos_zp_race** : infrastructure du test
  verrouillant la course ZP IRQ↔task sur SYS_WIN_CREATE (audit §3.3a).
  Combine PC-hook + mouse2_move_abs POISON + flag WM_TASKMODE. Status
  v1 : 2/2 PASS mais bug pas reproduit en legacy (hook tire sur 1er
  kernel_wm_add ≠ task_win). Raffinement v2 attendu post-§3.1 :
  tâche test dédiée + sentinelle sync, OU compteur d'arming PC-hook.
  Cible `make test-oricos-zp-race` ajoutée à `make tests`. Réf :
  IRQ_CONFORMITE §5.4.

## [2026-05-31x] — IRQ_CONFORMITE §3.4 NMI doc + §5 assertion budget IRQ

### Added
- **OricOS / handlers.s** : commentaire NMI ajouté (aucune source NMI
  câblée v1). Réf : IRQ_CONFORMITE §3.4.
- **Phosphoric / test_oricos_wm_cost** : assertion fin-de-test
  `kernel_irq_handler.cmax ≤ IRQ_HANDLER_MAX_BASELINE` (baseline v1 =
  18000 cycles, marge ~3% sur observé 17355). Anti-régression
  validé. Post-§3.1 (ADR-32 Étape 4) baseline à descendre à 4096
  (= période T1, goal IRQ_CONFORMITE §5 point 1). Réf :
  IRQ_CONFORMITE §5.

### Investigated (reverté)
- **OricOS / handlers.s** : court-circuit conditionnel kbd_poll/wake
  testé (KBD2_STATUS bit7), gain ~24 cycles/IRQ. Cassait sentinelles
  cycle-précises ctl_demo (drag scrollbar). Reverté, note in-line à
  reconsidérer après §3.1.

## [2026-05-31w] — IRQ_CONFORMITE §3.3 A : invariant index 8-bit + audit CI

### Added
- **OricOS** : invariant « X=1 aux points préemptibles » documenté en
  tête de `handlers.s` (le `sep #$30` avant `pha/phx/phy` de
  `kernel_irq_handler` détruit X.hi/Y.hi si caller en X=0). Audit
  complet : 8 sites `rep #$10/#$30` kernel, 3 RÉELS (scroll_up,
  sd_copy, ae_copy bundle load) annotés in-line avec risque +
  mitigation actuelle (boucle bornée OU contexte rare). Nouvelle
  cible Makefile `audit-rep-x` (baseline 8 sites) câblée à `all:`
  bloque le build sur régression. v1 stable — suite Phosphoric verte,
  aucun bug observable. Plan v2 : sei/cli wrappers ou option B IRQ
  save 16-bit (multi-fichiers atomique). Réf : `IRQ_CONFORMITE.md`
  §3.3 A.

## [2026-05-31v] — GFX ABI livré (audit §3) + fix dette ZP via save/restore

### Added
- **OricOS** : `oricos_gfx_fill_rect` écrit désormais dans le bloc
  `$D0-$D4` (PAS `$73-$78`). Kernel `sys_gfx_fill_rect` save/restore
  `$90-$93` (`GFX_BPL/ARG4`) qui overlap user imag-regs `__rc7-__rc10`.
  Solution P2 option D — 8 instructions kernel (4 pha + 4 pla) plutôt
  que refactor invasif (options A/B/C). Ferme la course IRQ↔task §3.3a
  pour ce syscall. 2 sites kernel-internal (task_wdraw, task_compact)
  migrés aussi.
- **Suite Phosphoric COMPLET verte** — 12/12 test_oricos_helloc, 0 FAIL
  global. Régression chronique des 6 tests depuis 881c9f3 entièrement
  résolue (P0 + P1 + P2 GFX ABI). Réf : critique utilisateur
  2026-05-31 (revue toolbox §3).

## [2026-05-31u] — Investigation GFX ABI : dette ZP kernel↔user (P2)

### Investigation
- **OricOS** : tentative de relivrer GFX_FILL_RECT via $D0-$D4 (audit §3
  toolbox) échoue sur `test_oricos_clock`. Root cause : conflit
  pré-existant kernel ZP `$90-$93` (`GFX_BPL_LO/HI`, `GFX_ARG4_*`)
  OVERLAP user imag-regs `__rc7..__rc10`. La recompilation contre
  la nouvelle oricos.h change l'allocation rc du compilateur LLVM,
  réveillant le conflit qui était fortuitement évité avec l'ABI
  $73-$78. **GFX ABI reverté** ; documenté en §9 P2 (move kernel
  ZP hors zone imag-regs OU offset link.ld). Suite Phosphoric
  reste verte au HEAD.

## [2026-05-31t] — P1 : install.sh `rm crt0.o` → duplicate `jsr main`

### Fixed
- **OricOS / `install.sh`** : retrait du `rm crt0.o` après création de
  `libcrt0.a`. Cause des 6 tests Phosphoric en fail avec fresh build
  depuis 881c9f3 (= depuis le premier commit hello_c). Le driver clang
  auto-ajoute `-l:crt0.o` (literal filename) ; sans notre crt0.o en
  `oricos/lib/`, ld.lld retombait sur `common/lib/crt0.o` qui a SON
  propre `.call_main: jsr main`. Concaténé avec la nôtre (via -lcrt0
  → libcrt0.a) → **2 jsr main back-to-back** → main() s'exécute
  DEUX fois → 2e run bloque sur SYS_READ_CHAR (key déjà délivrée
  au 1er run) → CPU stuck.
- **Validation** : suite Phosphoric **COMPLET vert** pour la première
  fois de toute cette session de remédiation. test_oricos_helloc :
  **12/12 PASS** (vs 6/12 avant). Suite tests Phosphoric COMPLET vert.
- **Conséquence rétrospective** : tous mes fixes SDK de la session
  (audit-smart, %u, calloc, print_string, GFX ABI) étaient bien
  fondés — ils n'avaient simplement jamais pu être validés
  end-to-end à cause de ce bug d'install.sh. Le P0 (Makefile stamp)
  garantit que ce type de divergence ne pourra plus s'accumuler.

## [2026-05-31s] — `Makefile` P0 : install.sh câblé dans la chaîne make

### Added
- **OricOS / `Makefile`** : mécanisme stamp file (`tools/oricos-sdk/
  .installed-stamp`) qui force `install.sh` dès qu'un fichier SDK
  source change (oricos.h, liboricos.c, malloc.c, crt0.S, link.ld,
  mos-oricos.cfg, install.sh). Le stamp recompile aussi les apps
  (suppr `apps/*/build/*.{bin,oos,oosobj}`) pour qu'elles linkent
  contre la nouvelle `liboricos.a`. Sans ça, le SDK source et la
  `.a` installée pouvaient diverger silencieusement — cause racine
  du désastre 2026-05-31 (6 tests Phosphoric fail avec fresh build
  depuis 881c9f3 = commit initial hello_c). Nouvelle cible
  `make sdk-install` pour forcer. Variable `LLVM_MOS` overridable.
  Reste P1 : les 6 tests qui fail avec fresh build sont un bug
  pré-existant non résolu par P0 (qui empêche juste la prochaine
  divergence). Réf : critique utilisateur 2026-05-31 + bisect.

## [2026-05-31r] — `oricos_print_string` : N COP → 1 COP

### Changed
- **OricOS / `oricos.h`** : `oricos_print_string` bouclait sur
  `oricos_print_char` → N COP pour N caractères. Fix : wrapper
  inline asm sur SYS_PRINT_STRING ($02) qui existe déjà côté
  kernel — **UN SEUL COP** pour toute la chaîne. Bank = DBR app
  (posé par crt0 phk/plb). Bonus : `printf("%s", …)` aussi
  optimisé via `_vfmtcore` (1 COP + strlen pour le count).
  Sprintf path inchangé. Gain typique × 10-20 sur le débit
  console des apps textuelles. Comportement utilisateur
  identique, juste plus rapide. Réf : critique utilisateur
  2026-05-31 (revue toolbox).

## [2026-05-31q] — Fix `calloc` overflow 16-bit + corpus natif

### Fixed
- **OricOS / `malloc.c`** : `calloc(256, 256)` = 65536 wrappait à 0
  en size_t 16-bit (env OricOS) → `malloc(0)` → buffer 2 octets →
  caller écrit 65534 octets de trop → corruption silencieuse du
  bank. Fix : helper `_calloc_overflows(nmemb, size)` qui détecte
  le wrap AVANT le mul (via `nmemb > SIZE_MAX / size`), retourne
  NULL net si overflow.
- **Nouveau corpus natif** (`tools/oricos-sdk/lib/tests/test_liboricos_calloc.c`) :
  15 cas testant le helper directement (cas zéro, limites SIZE_MAX,
  overflows triviaux/non-triviaux) + simulation uint16_t qui
  verrouille le cas exact du bug OricOS (256×256). Cible
  `make test-libc-calloc`. Anti-régression : sans le fix, le test
  ne compile pas.
- Réf : critique utilisateur 2026-05-31 (revue toolbox).

## [2026-05-31p] — Fix `%u` cassé pour val ≥ 32768 + corpus natif

### Fixed
- **OricOS / `liboricos.c`** : `%u` faisait `itoa((int)v, …, 10)` →
  en env int=16-bit, `(int)40000` = -25536 → `itoa` ajoutait `-`
  et imprimait `"-25536"`. Toute app printf un compteur > 32767
  voyait des résultats faux silencieusement. Fix : primitive
  interne `_utoa(uint16_t, char*, int)` unsigned-only, utilisée
  par `%u`, `%x`, et wrappée par `itoa` (signed).
- **Nouveau corpus natif** `tools/oricos-sdk/lib/tests/` (22 cas,
  uint16_t critiques en bases 10/16 + sprintf). Stub minimal
  d'`oricos.h` pour compile gcc native. Cible `make test-libc-fmt`.
  Anti-régression : sans le fix, le test échoue à la COMPILE
  (signal plus fort qu'un FAIL runtime).
- Réf : critique utilisateur 2026-05-31 (revue toolbox).

## [2026-05-31o] — Fix audit-smart `#$30` ignoré + corpus de régression

### Fixed
- **OricOS / `tools/audit-smart.py`** : pré-filtre `"20" in operand`
  écartait `rep/sep #$30` du tracker M-state → verdicts faussement
  verts sur tous les fichiers utilisant `rep/sep #$30` (presque tout
  le kernel). Fix : suppression du pré-filtre, `parse_imm` + `val &
  0x20` suffit. Boucle morte du handler `.a16` aussi nettoyée.
- **Nouveau corpus `tools/tests/`** : 4 fixtures `.s` (2 bad : `#$20`
  + `#$30`, 2 good : `.a16` explicite + sans branche M=16) + runner
  `test_audit_smart.py`. **Régression vérifiée** : ancien code →
  fixture `#$30` raté ; nouveau code → 4/4 OK. `make audit-smart`
  lance le corpus AVANT le scan kernel.
- **Conséquence importante** : audit-smart relancé sur le kernel
  actuel → toujours propre. Les refactors §3.6 et changements
  ADR-32 livrés cette session étaient *vraiment* propres, pas
  faussement propres.
- Réf : critique utilisateur 2026-05-31 (revue toolbox) + audit
  axe 8.3.

## [2026-05-31n] — ADR-32 Étapes 2+3 : flag `WM_TASKMODE` (gates atomiques)

### Added
- **OricOS / `kernel.s` + `handlers.s` + `event.s`** : nouveau flag
  `WM_TASKMODE = $01EE68`. Gate atomique unique : default $00 →
  mouse_step en IRQ (legacy), $A5 → mouse_step dans `task_wm` après
  `raw_pop` (l'IRQ skip son appel). Atomicité par flag unique
  (anti-revert ADR-28 Étape 3) — soit IRQ, soit task, jamais les
  deux ni aucun.
- Default LEGACY → comportement utilisateur inchangé, suite tests
  Phosphoric verte. Activation requiert Étape 4 (migration
  `_cursor_draw` hors IRQ + validation interactive) AVANT flip du
  défaut. Le flag existe pour permettre aux tests d'injection async
  (avec `cpu816_set_pc_hook` Phosphoric, Étape 1 déjà livrée) de
  valider le nouveau chemin sans impact production. Réf :
  `docs/adr/0032-zp-race-irq-task.md` §3 + §5.

## [2026-05-31m] — ADR-32 Étape 1 : harnais PC-hook Phosphoric

### Added
- **Phosphoric** : `cpu816_set_pc_hook` / `cpu816_clear_pc_hook` —
  un test peut armer un callback qui fire à un (PBR, PC) précis,
  juste avant le fetch d'opcode. Pré-requis pour les tests
  d'injection event-async ADR-32 §4 (anti-revert ADR-28 Étape 3).
  3 cas testés (`test_pc_hook`), 0 régression suite tests
  Phosphoric. Réf : `docs/adr/0032-zp-race-irq-task.md`
  Étape 1 + axe 8.5 audit.

## [2026-05-31l] — Dossier d'instruction ADR-32 (course ZP IRQ↔tâche)

### Added — `docs/adr/0032-zp-race-irq-task.md`
- Dossier d'instruction (conforme moratoire CLAUDE.md §10 cond. 1)
  pour le fix de la course ZP IRQ↔tâche identifiée par
  `AUDIT_65C816_REMEDIATION.md` §3.3a — **suspect n°1 du revert
  ADR-28 Étape 3**.
- 3 options chiffrées : (A) sei épars partout, (B) migration
  complète `mouse_step` hors IRQ vers tâche serveur WM, (C)
  partition stricte ZP. Recommandation senior tracée : **(B)**,
  finalise option C d'ADR-28 sans rejouer le revert.
- Plan d'atomicité critique : flag `WM_TASKMODE` de bascule
  runtime, harnais d'injection event-async (axe 8.5,
  `cpu816_set_pc_hook` Phosphoric) PRÉALABLE à toute migration.
- Plan d'implémentation 6 étapes, rollback instantané, ratification
  graduelle après validation interactive utilisateur sur tous les
  scenarios qui avaient fait reverter ADR-28 Étape 3.
- Référencé dans `CLAUDE.md` §3 (ADR ouvertes).

## [2026-05-31k] — Audit 65C816 §3.6 COMPLET : couche géométrie isolée

### Refactored
- **OricOS** : primitive `_point_in_rect16` (UNE frontière `rep/sep`,
  AUCUNE branche ne la traverse) + refactor des 4 hit-testers WM
  (`_wm_chrome_hit`, `_wm_resize_hit`, `_wm_widget_hit`,
  `_wm_hotzone_hit`) pour l'appeler. Élimine la zone fragile pointée
  par l'audit §3.6.1 (bcc/bcs traversant rep/sep). 4 commits atomiques,
  tests verts entre chaque. Couverture identique avec ~1 implémentation
  géométrique au lieu de 4. Réf : `AUDIT_65C816_REMEDIATION.md` §3.6
  + axe 8.2.

## [2026-05-31j] — Audit 65C816 §3.6 partiel : macros ASSERT_A*/I* + scratch resize

### Added
- **OricOS / `kernel.s`** : 4 macros `ASSERT_A16/A8/I16/I8` (via
  `.asize`/`.isize` de ca65) → vérification à l'assemblage de
  l'invariant mode M/X en tête de routine. Pré-requis axe 8.3.
  Placées en entrée des 4 hit-testers WM (`_wm_chrome_hit`,
  `_wm_resize_hit`, `_wm_widget_hit`, `_wm_hotzone_hit`).
- **OricOS / `kernel.s` + `wm.s`** : nouveau scratch ZP `WM_RH_TMP`
  ($32, 2B) dédié `_wm_resize_hit`. Remplace l'overload de
  `WM_ARG_DX` (arg syscall partagé avec IRQ — conflit identifié
  audit §3.6.2). Suite tests Phosphoric verte. Réf :
  `AUDIT_65C816_REMEDIATION.md` §3.6 + axe 8.3.

## [2026-05-31i] — Audit 65C816 §5 : nettoyages cosmétiques

### Cleaned
- **OricOS / `kernel.s`** : commentaire ADR-20 corrigé `SVGA 800×600`
  → `XVGA 1024×768` (incohérence vs reste du kernel).
- **OricOS / `fat.s`** : `cmp #$00` redondant supprimé après
  `kernel_alloc_bank` (Z déjà positionné).
Réf : `AUDIT_65C816_REMEDIATION.md` §5.

## [2026-05-31h] — Audit 65C816 §3.2 : fuite page de pile

### Fixed
- **OricOS / `kernel_task_create` (sched.s)** : page de pile dérivée du
  pid (`page = pid + 1` pour pid ≥ 3, saute I/O $03) au lieu d'un bump
  `STACK_NEXT_PAGE` qui ne redescendait jamais. Pids 1/2 conservés en
  cas spécial (task A/B). Plus de fuite par construction. Suite tests
  Phosphoric verte sans régression. Réf : `AUDIT_65C816_REMEDIATION.md`
  §3.2.

## [2026-05-31g] — Audit 65C816 : test verrouillant fix §2.1

### Added
- **Phosphoric / `test_oricos_teardown_bank`** : nouveau test d'intégration
  qui spawn `bundle_clock` (TC_CLOCKAPP_FLAG=$A5), laisse clock sortir
  (~4 ticks, pas de clavier), asserte `BANK_FREE_TOP >= 1` et bank libéré
  ≥ BANK_POOL_BASE. Régression vérifiée : revert temporaire du fix §2.1 →
  test FAIL ; restauration → test PASS. Ajouté à l'aggregate `make tests`.
  Réf : `AUDIT_65C816_REMEDIATION.md` §2.1.

## [2026-05-31f] — Audit 65C816 §3.1 : garde linker CODE overflow

### Added
- **OricOS / `kernel.s` + `kernel.cfg`** : assertion linker
  `(__CODE_LOAD__ + __CODE_SIZE__) <= $5400` (via `define = yes` sur
  segment CODE). Tout débordement CODE vers la zone data runtime
  ($5400-$54FF) est désormais une **erreur de build** au lieu d'une
  corruption silencieuse. Réf audit : `AUDIT_65C816_REMEDIATION.md`
  §3.1.

## [2026-05-31e] — Audit 65C816 §2.1 : fuite bank teardown (P0)

### Fixed
- **OricOS / `se_teardown` (wm.s)** : libération du bank de code de
  l'app au `sys_exit` (était fuit). Garde `PB ≥ BANK_POOL_BASE` pour
  ne jamais pousser un bank kernel (PB=1) sur la free-list. Élimine
  l'épuisement `ERR_BANK_EXHAUSTED` après ~124 spawn/exit. Réf audit :
  `AUDIT_65C816_REMEDIATION.md` §2.1. Suite tests Phosphoric verte.

## [2026-05-31c] — App file_select (GenFileSelectorClass MVP) + finding labels

### Added — file_select dialog modal (pattern GeoWorks)
- Nouvelle app C `apps/file_select/fileselect.c` : fenêtre modale
  avec GU_LIST (5 fichiers hardcoded) + OK/Cancel. ~60 LOC.
- Flag CLI `--fileselect` (Phosphoric main.c) + TC_FILESELECT_FLAG
  (kernel.s) + spawn boot.s + incbin console.s.
- MVP : items hardcoded ≤ 7 chars (limitation GU_LIST stride).
  Intégration vraie FAT32 SD différée.
- 24/24 suites globales.

### Finding — Labels boutons partagés (bug pré-existant)
- Révélé par validation visuelle file_select : 2 boutons rendus
  "Cancel" au lieu de "OK/Cancel". Vérification score (3 boutons
  "+1/+10/Reset") montre 3× "Reset" → bug ancien, jamais détecté
  car les tests fonctionnels passent (id-based).
- Cause : `UI_STR_BUF` unique scratch SDRAM, tous boutons stockent
  strptr → UI_STR_BUF → tous lisent le dernier label uploadé.
- Infrastructure fix posée (`BUTTON_LABELS = $016A00`, 128B) mais
  tentative MVN copy revertée (régression test_oricos_gui_demo).
- À investiguer dans session dédiée.

## [2026-05-31b] — Golden visual test régénéré + mode REGEN_GOLDEN

### Fixed — test_oricos_visual_matches_golden réactivé
- Mode regen ajouté : `REGEN_GOLDEN=1 make test-oricos-visual`
  écrit le golden depuis le frame courant. Pratique pour mettre à
  jour quand fonte/banner/etc change.
- Golden `tests/golden/oricos_boot.ppm` régénéré post-VGA8 + horloge.
- Test compare normal (sans env var) repasse vert. Plus de skip.
- 24/24 suites globales, golden inclus.

## [2026-05-31a] — Dual font VGA8 chrome XVGA fixé (cause : bank byte ld65)

### Fixed — Bug VGA8 upload isolé et résolu
- Cause : ld65 résout les symboles de segment en 16-bit (sans bank
  info) → `#^kernel_charset_xvga` = $00 au lieu de $01 → upload
  lisait bank 0 garbage au lieu de bank 1 VGA8.
- Fix : constante 24-bit explicite `CHARSET_XVGA_SRC = $015C00`
  (avec bank), utilisée par `kernel_tk_font_init`. Cohérent avec le
  pattern existant `CHARSET_SRC = $015800` pour la fonte Atmos.
- 5 lignes de code modifiées.
- Validation oricrobot : chrome XVGA en VGA8 IBM CGA (titres
  OricOS/Editor, menus About/Clear, OK, horloge T:4A) tout lisible.
  Banner mode TEXT Oric 1 reste sur Atmos. 24/24 verts.
- **Leçon** : utiliser constante 24-bit (`= $01XXXX`) plutôt que
  symbole de segment pour les adresses bank-1 référencées via `#^`.

## [2026-05-30z] — Horloge taskbar + infra dual font (VGA8 bug à debug)

### Added — Horloge taskbar "T:HH" (polish UI)
- `kernel_taskbar_draw` ajoute affichage du tick counter en bas-droite
  de la taskbar. Format "T:NN" hex (2 digits, 1 byte). Visible
  immédiatement, polish UI minimal.
- Validation oricrobot : screenshot montre "T:4A" (tick 74).
- 8 tests cyc-bumpés (~1.5×) suite au coût additionnel de l'horloge
  (~800 cyc/redraw taskbar). Sémantique inchangée. 24/24 verts.

### Infrastructure — Dual font (posée, mais bug runtime XVGA upload)
- `data/charset-xvga.bin` : fonte IBM CGA 8×8 domaine public
  (extraite Debian Arabic-VGA8.psf, latin 0-127).
- `handlers.s` : 2e .incbin `kernel_charset_xvga` à bank 1 $5C00.
- `tk.s` : tentative de pointer `kernel_tk_font_init` vers VGA8
  REVERTÉE (rendu carrés blancs en runtime). Cause à investiguer.
  Le binaire contient bien la VGA8 à la bonne adresse (xxd
  confirme) mais l'upload `kernel_vram_write_block` produit
  tout-$FF dans `TK_FONT_ADDR`.
- `data/charset.bin` : restauré Atmos (sans ça, le banner mode
  TEXT Oric 1 ULA était illisible — la fonte Atmos est requise
  par le mode TEXT ULA en bank 0 $B400).
- `tools/gen-font-geos.py` : script Python+PIL pour régénérer
  fontes 8×8 depuis TTF. Conservé pour itération future.
- **Finding** : revenir à dual font quand le bug VGA8 upload sera
  isolé. Pour l'instant chrome XVGA = même fonte Atmos qu'avant.

### Skipped — test_oricos_visual_matches_golden
- Désactivé : golden PPM doit être régénéré au pixel près. À faire
  dans une session dédiée (modifier le test pour dumper plutôt
  que comparer).

## [2026-05-30y] — Fonte 8×8 IBM CGA (RÉTRACTÉE 2026-05-30z)

### Changed — Tentative VGA8 sur charset.bin → reverté
- Replacement de `data/charset.bin` par VGA8 cassait le mode TEXT
  Oric 1 ULA (banner OricOS illisible). VGA8 déplacé vers
  `data/charset-xvga.bin` mais bug runtime (cf. 2026-05-30z).

## [2026-05-30x] — ADR-27 §0quinquies : fast-drag artifact tracé (limitation connue)

### Documented — Limitation connue : fast-drag artifact en --compact
- Validation interactive 2026-05-30w (drag rapide SDL) a révélé des
  bandes horizontales transitoires (y=420..470) pendant le drag.
  Non reproduit par oricrobot (drag discret = propre).
- Hypothèse : timing IRQ — SDL coalesce rafale d'events MOU2 par
  frame, garde B1 wrapper s'enchaîne et un cas de timing non-couvert
  fait leak `bpl`. Théoriquement push/pop équilibrés (invariant
  tient), mais l'observation visuelle suggère un edge case.
- Impact production : nul. `--compact` est un flag dev, aucune app
  ne pose `WM_COMPACT_FLAGS[slot]=$A5`. Mode dormant par défaut.
- Tracé dans `docs/adr/0027-backing-store-fenetre.md §0quinquies`.
  Pour ré-attaquer : compteur IRQ MOU2/frame (existant) +
  commande oricrobot `mouseburst` + reproduction déterministe +
  isolation chemin leak (compose ↔ IRQ ↔ redraw_drag).

## [2026-05-30w] — Finding chrome-direct-FB résolu (WM_NO_BACKING_FLAGS)

### Fixed — Fenêtres système plus rendues en noir en mode --compact
- Table `WM_NO_BACKING_FLAGS[slot]=$A5` (8B `$01690B`). compose
  skip les slots taggés → leur chrome direct framebuffer reste.
- Slots 0/1 (OricOS, Editor, créés au boot) sont taggés.
- task_compact (slot 2) et apps régulières non-taggées → compose
  les copie normalement.
- Validation oricrobot : PPM pixel (105,115) chrome OricOS =
  lightgray, (305,315) chrome Editor = bleu, (61,61) task_compact
  rect = lightgray. **Aucun noir nulle part** (vs avant : rect
  noirs à la place des fenêtres système).
- Cyc-bumpé test_oricos_radio (140k → 200k bootstrap, 280k → 440k).
- 24/24 suites Phosphoric vertes.

## [2026-05-30v] — ADR-27 §0quater C-2 implémentée + validation oricrobot

### Added — `_gfx_xvga_bpl_guard` + B2.c re-livré + ratification
- Helper `_gfx_xvga_bpl_guard` (gfx.s) : si `GFX_BASE_HI ≥ $10` et
  shadow `bpl` ≠ 0, force `bpl=0`. **1 helper couvre les ~36 sites
  kernel direct** (chrome, taskbar, icônes, widgets tk, démos boot).
- Garde insérée en tête de `kernel_gfx_fill_rect`/`line`/`text`/
  `fill_rect16`/`text16`. Coût ~25 cyc/appel, skip si shadow=0.
- B2.c re-livré : `task_compact_entry`, spawn, flag CLI `--compact`,
  test unitaire réactivé (12/12 helloc).
- 2 tests cyc-sensibles bumpés (mainloop_chrome + scrollbar) :
  140k→200k bootstrap, 320k→480k total. Sémantique inchangée.
- 24/24 suites Phosphoric vertes.

### Validated — Oricrobot script de transparence interactive
- `/tmp/adr27_c2_validation.txt` : spawn task_compact, clic menu
  System, mouvements souris exploratoires, peek pixel
  framebuffer XVGA à (61,61) **avant et après interactions**.
- Résultats : `peek $107A1E = $77` (couleur 7 = lightgray) **avant
  ET après clic menu ET après mouvements souris** → transparence
  interactive validée.
- PPM analysé Python PIL : `pixel (60,61) = (170,170,170)` confirme.
- Menu dropdown propre à sa position attendue, pas multiplié comme
  avant C-2. Pas de bandes noires massives.
- Limite connue (= finding WM pré-existant, hors ADR-27) : les
  fenêtres OricOS/Editor restent rendues en noir par le compose-loop
  parce que leur chrome est dessiné directement framebuffer, pas
  dans leur backing store. C'est cosmétique, pas un leak `bpl`.

### Re-ratifiée — ADR-27 option (b)
- Critères moratoire §10 réunis (vu §0quater C-2 implémentée +
  tests verts + validation interactive positive) :
  1. Dossier d'instruction complet (DRAFT avec §0..§0quater).
  2. ≥ 50% impl (A + B1 + B2 + B2.c v2 + C-2).
  3. Cohérence ADR-19/20/21 maintenue.
- Renomme `0027-backing-store-fenetre-DRAFT.md` → `.md` (à venir
  dans le commit doc).

## [2026-05-30u] — ADR-27 §0quater : audit chemins kernel direct

### Added — Dossier d'instruction pour ré-ratification
- Audit `grep jsr kernel_gfx_*` post-revert B2.c : **36 sites
  kernel direct** non-instrumentés (chrome, taskbar, icônes, widgets
  tk, démos boot) qui dessinent dans le framebuffer XVGA en bypassant
  `window_base`/`finish`. Causent le leak `bpl` observé en validation
  interactive.
- 4 stratégies chiffrées : C-1 patch chaque appel, **C-2 à la source
  des helpers GPU (recommandée)**, C-3 entry points WM, C-4 pivot
  option (a) multi-banques.
- Recommandation C-2 : modifier `kernel_gfx_fill_rect16/text16/line/
  fill_rect` pour forcer `bpl=0` si `GFX_BASE_HI ≥ $10` (cible XVGA)
  ET shadow ≠ 0. 1 changement = 36 sites couverts. Coût ~10 cyc/appel.
- Plan d'instruction tracé : tests d'intégration ciblés par chemin
  (menu, taskbar, widgets) en plus du compose-loop.
- Statut : DRAFT (option (b) toujours retenue, ratification après
  C-2 implémentée + tests verts + validation interactive positive).

## [2026-05-30t] — ADR-27 dé-ratifiée : non-transparence interactive

### Removed — Étape B2.c reverté, retour DRAFT
- Validation interactive `--compact` (cf. 2026-05-30s finding) a montré
  que la plomberie compact leak `bpl` vers des chemins kernel non-
  instrumentés (`kernel_menu_draw`, `_wm_draw_one` chrome) → rendu
  corrompu en interaction réelle. Test unitaire trop restreint
  (compose-loop seul, sans interaction) couvrait un cas non
  représentatif.
- Revertés : `task_compact_entry` OricOS, spawn boot, vars test,
  flag CLI `--compact` Phosphoric, test `test_oricos_compact_backing_store`
  désactivé (`#if 0`).
- **Plomberie dormante conservée** (A + B1 + B2 + hardening M=8) :
  shadow, garde IRQ, table flags, helpers, modifs window_base/compose/
  redraw. Comme `WM_COMPACT_FLAGS` reste à 0 partout, comportement
  runtime identique au pré-ADR-27 (24/24 verts).
- Dossier renommé `0027-backing-store-fenetre.md` →
  `-DRAFT.md`. CLAUDE.md §2 : ADR-27 retirée. §3 : ré-ouverte avec
  rétractation et critères de ré-instruction. ADR-31 (clip widget)
  redevient « obsolète à la ratification ADR-27 », pas « redondante ».
- À ré-instruire avant ratification : audit exhaustif des chemins de
  dessin kernel direct + tests d'intégration ciblés (menu, taskbar,
  chrome, dialogues) — pas seulement compose-loop.

## [2026-05-30s] — Hardening gardes shadow `bpl` (M=8 forcé) + finding WM

### Hardening — M=8 explicite dans les 3 gardes shadow `bpl`
- `php/sep #$20/plp` autour des `lda f:GFX_BPL_SHADOW / ora` dans :
  `kernel_wm_redraw` (entrée), `kernel_wm_compose:wcmp_done`,
  `kernel_gfx_window_base:gwb_set_default`. Sûreté générale contre
  callers M=16 (`.smart` ne couvre pas les branchements).

### Finding — compose vs `_wm_draw_one` chrome direct framebuffer
- Validation interactive `--compact` révèle que `task_compact` qui
  boucle `kernel_wm_compose` copie les backing stores vides des
  fenêtres système (OricOS, Editor) → rect noirs à leur place.
- Cause **préexistante ADR-27** : chrome des fenêtres système dessiné
  par `_wm_draw_one` directement dans framebuffer XVGA, pas dans le
  backing store. compose-loop écrase le rendu correct.
- **Hors périmètre ADR-27** : la plomberie compact reste correcte
  (test unitaire `test_oricos_compact_backing_store` 12/12).
- À tracer comme chantier WM séparé (option : faire dessiner le
  chrome dans le backing store des fenêtres système, ou flush
  sélectif). Pas de remise en cause de la ratification ADR-27.

## [2026-05-30r] — ADR-27 RATIFIÉE (option (b))

### Changed — Promotion DRAFT → ratifiée
- `docs/adr/0027-backing-store-fenetre-DRAFT.md` renommé en
  `0027-backing-store-fenetre.md` ; en-tête mis à jour (statut
  RATIFIÉ, date de ratification 2026-05-30q).
- `CLAUDE.md` §2 : ADR-27 ajoutée à la table des décisions figées
  (option (b) stride GPU configurable + backing store compact). §3 :
  entrée déplacée en `~~ADR-27~~ → ratifiée 2026-05-30q`.
- `CLAUDE.md` ADR-31 : reformulée (« rendue redondante à terme par
  ADR-27 (backing store contraint le rendu par construction), mais
  conservée v1 — pas de migration coûteuse pour un cas couvert »).
- `ADR_SUMMARY.md` : nouvelle section ADR-27 (synthèse étapes A/B1/
  B2/B2.c, concurrence option 2, garde IRQ, activation
  `WM_COMPACT_FLAGS[slot]=$A5`, C1/C2 différées).
- Cohérence ADR-19 (SDRAM inchangée) / ADR-20 (stride 512 reste
  défaut) / ADR-21 (extension compatible : SET_BPL opcode sans
  toucher la memory map $0340-$034F).

## [2026-05-30q] — ADR-27 Étape B2.c : activation + test transparence

### Added — Flip compact validé bout-en-bout
- Task de test `task_compact` (gated `TC_CPCT_FLAG=$01EEA0`) : crée
  fenêtre 64×64 à (50,50), active `WM_COMPACT_FLAGS[handle]=$A5`,
  dessine bg bleu + rect rouge en compact stride 32, compose loop.
- Test C `test_oricos_compact_backing_store` : vérifie flag posé,
  pixel framebuffer (61,61) = 7 (rouge), pixel (52,52) = 1 (bleu).
- 12/12 tests `helloc`, 24/24 suites globales.
- Plomberie ADR-27 option (b) validée fonctionnellement → seuil
  moratoire §10 atteint (50%+ d'implémentation + dossier chiffré
  + cohérence ADR-19/20/21) → ratification ADR-27 instructible.

## [2026-05-30p] — ADR-27 Étape B2 : plomberie compact slot (flag inactif)

### Added — Tout le chemin GPU peut basculer en compact, slot par slot
- Table `WM_COMPACT_FLAGS[slot]` ($00 = stride 512 = défaut ; $A5 =
  stride compacte `byte_w = W>>1`). Reste à $00 partout → comportement
  runtime inchangé.
- `kernel_gfx_window_base` pose `bpl` selon le flag du slot owner.
- `kernel_gfx_finish` restaure `bpl=0` en fin de syscall (5 wrappers).
- `kernel_wm_compose` pose `bpl=byte_w` par slot compact avant BLIT,
  restaure en `wcmp_done`.
- `kernel_wm_redraw` force `bpl=0` à l'entrée (garde principale en
  plus de la garde IRQ B1).
- **Comportement** : flag inactif sur tous les slots → no-op partout,
  24/24 suites vertes. Activation = sprint B2.c séparé (test dédié
  ou validation interactive).

## [2026-05-30o] — ADR-27 Étape B1 : garde IRQ `bpl` (transparence)

### Added — `kernel_wm_mouse_step` enveloppé d'une garde transparente
- Wrapper IRQ : si shadow ≠ 0, push shadow + force `bpl=0` (stride
  XVGA 512) le temps des redraws framebuffer, puis restore. Si
  shadow == 0 (cas par défaut), `jmp` direct au body — ~10 cyc.
- Effet runtime nul tant qu'aucun appelant ne touche `set_bpl` :
  la garde est dormante mais prête. 24/24 suites vertes.
- Pré-requis sécurité avant Étape B2 (bascule compact slot 0).

## [2026-05-30n] — ADR-27 Étape A : shadow kernel `bpl` (plomberie passive)

### Added — Pose des fondations pour le flip backing-store compact
- Shadow kernel `GFX_BPL_SHADOW = $016900` (2B) miroirant le registre
  GPU `bpl` non lisible (ADR-27 §0bis option 4).
- Helper `kernel_gfx_get_bpl_shadow` + maintien automatique dans
  `kernel_gfx_set_bpl`. Init shadow = 0 dans `kernel_wm_init`.
- Étape **passive** : aucun appelant ne touche encore `set_bpl`, donc
  shadow reste à 0, comportement runtime identique (24/24 verts).
- Étape B suivante : garde IRQ save/restore dans `kernel_wm_mouse_step`
  + bascule compact slot 0.

### Note `.smart` ca65 (leçon Étape A)
- Une directive `.a16` posée au début d'un helper sans `.a8` refermant
  pollue l'état que `.smart` propage aux helpers voisins → encodage
  immédiat 8 vs 16-bit divergent (a cassé 46 tests sur la 1ère mouture).
- Fix généralisable : entrée arbitraire → `php/sep #$20/.../plp`. La
  routine reste autonome vis-à-vis du contexte M du caller, et `.smart`
  voit M=8 stable.

## [2026-05-30m] — Hot-zones cliquables (pattern GEOS DoIcons)

### Added — `SYS_HOTZONE_SET/CLEAR` + bit 7 du `$DA` MSG_CONTROL
- Rectangles cliquables sans widget chrome (« tap zones »). MSG_CONTROL
  avec `$DA = $80 | id` distingue hotzones (bit 7) des widgets (id 0..15).
  Démo `score` : zone vide sous les boutons → reset. Pattern dérivé de
  `mist64/geos kernal/icon/icon1.s`. MVP n'unifie pas les 3 hit-testers
  existants (refactor structurel séparé) ; ajoute une 4e couche
  orthogonale placée APRÈS widgets pour ne pas casser les apps existantes.

## [2026-05-30l] — Timers d'app (pattern GEOS InitProcesses)

### Added — `SYS_TIMER_SET/CLEAR` + `MSG_TIMER`
- Nouveaux syscalls `$1E/$1F` + table 8 timers. Tick VIA T1 décrémente,
  fire = post `MSG_TIMER` à l'app via `_ml_classify`. SDK helpers
  `oricos_timer_set/clear`. Démo `score` : auto-incrément 30 ticks.
  Pattern dérivé de mist64/geos `process1.s` (cf. session 2026-05-30).

### Leçon réutilisable
- Le dispatcher COP **écrase X** avant d'indexer la syscall_table → tous
  les handlers syscall doivent lire arg1 depuis `DP_SYS_ARG_X` (ZP $11),
  pas depuis le registre X. Documenté dans le code de `sys_timer_set`.

## [2026-05-30k] — Sous-menus cascading (pattern GEOS DoMenu)

### Added — Menus déclaratifs hiérarchiques
- `GU_SUBMENU` + `GU_MENU_OPEN` ajoutés à GenUI (post-clôture ADR-30).
  Un item de top-menu peut ouvrir un submenu caché via cb_hi=$80.
  Pattern dérivé de mist64/geos `kernal/menu/menu1.s` (cf. session de
  lecture GEOS sources 2026-05-30). Cap v1 : 2 top-bar + 2 submenus,
  mono-niveau, dropdown unique (parent disparaît quand fils ouvre — v2
  GeoWorks-style avec MENU_STACK tracé pour itération future). Démo
  ctl_demo : `Edit > Font > [Sans, Serif]`.

## [2026-05-30j] — ADR-30 Étape 5 livrée + ADR-30 clos (GU_FIELD)

### Added — Champ étiqueté gFieldC + clôture ADR-30
- `GU_FIELD` widget : box étiquetée (label gauche + value 2 digits droite),
  non cliquable, mise à jour via `oricos_ctl_set_value(id, value)`.
  `sys_ctl_set_value` étendu pour redessiner le widget après update.
- Démo ctl_demo : compteur de clics menu affiché dans le field.
- 14 widgets exposés en GenUI (~88 % couverture GeoWorks
  GenInteractionClass directe) — objectif ADR-30 atteint (cible 85 %).
- ADR-30 marqué clos, post-mortem dans le dossier §7.6. Étapes futures
  (Vis hierarchy, Document/Application framework, file selector, sous-
  menus cascading) tracées hors scope ADR-30.

## [2026-05-30i] — ADR-30 Étape 4 livrée (GU_SPIN incrémenteur)

### Added — Widget incrémenteur déclaratif (GeoWorks SpinClass)
- `GU_SPIN` widget ajouté à GenUI. Format : rect + max8. Clic haut = +1,
  bas = -1, clamp `[min..max]` (réutilise `GU_HINT_MIN_VALUE` Étape 3).
  Visuel : face lightgray + cadre darkgray + value 2 chars décimaux.
  Démo ctl_demo : spin sous LIST (rel y=124, max=20). Validation
  headless oricrobot positive. 24/24 verts (cyc gui_demo bumpé pour
  bootstrap plus lourd).

## [2026-05-30h] — ADR-30 Étape 2b livrée (MSG_MENU à l'app)

### Added — Dispatch menu actionable
- `EV_MENU_CLICK = 5` + `kernel_event_push_menu` posté quand l'app clique
  un item d'un menu déclaré via `GU_MENU` (callback statique=0). Repack
  `MSG_MENU + $DA = (menu << 4) | item` à la classification. ctl_demo
  déclare un handler qui imprime `"ctl: menu m=X i=Y"` et sort sur
  `App > Quit`. Test verrouille (24/24 verts).
- Aligné GeoWorks `GenInteractionClass` / `OnInteractionClick`.

## [2026-05-30g] — ADR-30 Étape 2 livrée (GU_MENU + GU_MENU_ITEM)

### Added — Menu déclaratif aligné GeoWorks GenPrimary/eMenuC
- Tags `GU_MENU = 12` + `GU_MENU_ITEM = 13` ajoutés à GenUI. Au 1er
  `GU_MENU` dans une table d'app, le kernel bascule `MENU_DYN_ACTIVE`
  et override les menus statiques `System/View`. Strings copiées
  bank app → bank 1 (`MENU_DYN_STR_BUF`, 192 octets). MVP v1 : cap
  2 menus × 2 items, callbacks=0 (clic consommé silencieusement).
- Démo : `ctl_demo` déclare `App > About + Quit`.
- Verrouillage : assertions test_oricos_ctl_demo (`MENU_DYN_*`,
  contenu STR_BUF). 24/24 suites Phosphoric vertes.
- Étape 2b à venir : `MSG_MENU` à l'app sur clic item.

## [2026-05-30f] — Fix bug taskbar focus (onglet slot ≥ 1 non cliquable)

### Fixed — OricOS `kernel_taskbar_hit` `_tbh_advance:`
- Ajoute `.a16` au label `_tbh_advance:` dans `wm.s`. Cause racine :
  ca65 `.smart` encodait `adc #TB_BTN_STRIDE` en immédiat 8-bit (2 octets)
  au lieu de 16-bit (3 octets) parce que le label est atteint via bcc/bcs
  depuis le bounds check M=16 (pas via fall-through, donc `.smart` perd
  l'état M). En M=16 runtime, TB_BTN_X devenait `$8D80` au lieu de `$0080`
  → tous les slots > 0 considérés hors-bounds → onglet « Editor » et au-
  delà non-cliquable. Reproduit headless via `oricrobot` puis verrouillé
  par `test_oricos_taskbar_focus_3_windows`. 24/24 suites Phosphoric vertes.

### Added — Phosphoric `test_oricos_taskbar_focus_3_windows`
- Nouveau test d'intégration (`tests/integration/test_oricos_helloc.c`)
  qui verrouille le fix : boot OricOS + spawn ctl_demo (slot 2 focused),
  clic taskbar onglet Editor (slot 1) à (190, 760), asserte `WM_FOCUS = 1`.

## [2026-05-30e] — ADR-31 ratifiée (clip widget hors rect parent)

### Ratified — ADR-31 option A
- Validation interactive utilisateur positive : resize-down `ctl_demo` →
  widgets dépassants (scrollbar, list) disparaissent proprement au lieu
  de rester peints hors rect window.
- `CLAUDE.md` §3 → §2 (ligne ratifiée ajoutée au tableau, entrée §3
  remplacée par redirect, body d'instruction supprimé).
- `docs/adr/0031-clip-widget-rect-parent-DRAFT.md` renommé en
  `docs/adr/0031-clip-widget-rect-parent.md` (DRAFT retiré, date de
  ratification ajoutée).
- `ADR_SUMMARY.md` mis à jour.

## [2026-05-30d] — ADR-31 Étape 1 livrée (clip widget hors rect parent)

### Added — OricOS `_wm_draw_widget_body` skip widget hors rect parent
- Patch local 15 LOC dans `tk.s` : si `rel.x+w > win.w` ou `rel.y+h > win.h`,
  le widget est skippé au dispatch (`_wdb_clip_skip` : `sep #$20` + `rts`).
  Option A d'ADR-31 (recommandation senior). Élimine le bug visuel
  observé interactivement le 2026-05-30 (widgets peints hors rect window
  après resize-down). 24/24 suites Phosphoric vertes. Ratification ADR-31
  en attente de validation interactive utilisateur (leçon ADR-28).

## [2026-05-30c] — Test ctl_demo durci : assertion min hint (ADR-30 Étape 3)

### Added — Phosphoric `test_oricos_ctl_demo`
- Assertion `WIDGET_MIN_VALUES[3] == 20` (slot SCROLL_V, après les 2 widgets
  chrome auto-attachés titre+close) + `text_buf_contains("ctl: v=66")` :
  vérifie que `GU_HINT_MIN_VALUE 20` est bien posé sur le widget SCROLL_V
  et que `SYS_CTL_GET_VALUE` retourne `raw(46) + min(20) = 66`. Suite
  570→570 verts (10/10 `test-oricos-helloc`, 24/24 suites). Durcit la
  validation Étape 3 (mémoire suivi).

## [2026-05-30b] — ADR-30 Étape 3 ratifiée (GU_HINT_MIN_VALUE)

### Ratified — ADR-30 Étape 3 : `GU_HINT_MIN_VALUE` (attribut min GenValue)
- **Étape 3 d'ADR-30 ratifiée** suite à validation interactive utilisateur
  positive (`ctl_demo` : scrollbar retourne `v=20..60` au lieu de `v=00..40`).
- **Pivot d'instruction** : audit factuel WebFetch a révélé que
  [`gRangeC.def`](https://github.com/bluewaysw/pcgeos/blob/master/Include/Objects/gRangeC.def)
  contient juste « *soon-to-be-dead GenRangeClass... Nuked. 7/7/92 cbh* »
  — GeoWorks a **supprimé `GenRangeClass`** en juillet 1992 car
  `GenValueClass` a déjà `ATTR_GEN_VALUE_MINIMUM`/`MAXIMUM`
  ([`gValueC.def`](https://github.com/bluewaysw/pcgeos/blob/master/Include/Objects/gValueC.def)
  lignes 93-148). OricOS suit le design final : **pas de nouveau widget
  `WG_TYPE_RANGE`**, juste un **hint déclaratif** sur `GU_SCROLL_V/H`.
- **Coût réel** : ~25 LOC asm + 4 LOC SDK + 2 LOC démo (vs ~150 LOC estimés
  pour un `GU_RANGE` séparé). Pivot a divisé le coût par 5.
- Détail OricOS dans [OricOS/CHANGELOG.md](./OricOS/CHANGELOG.md).
- Conforme moratoire CLAUDE.md §10 (audit factuel + impl 100 % +
  cohérence ADR-29 pattern hints).

## [2026-05-30] — ADR-30 Étape 1 ratifiée (GU_LIST) + ADR-31 (DRAFT) ouverte

### Ratified — ADR-30 Étape 1 : GU_LIST (alignement GeoWorks GenList)
- **Étape 1 d'ADR-30 ratifiée** suite à validation interactive utilisateur
  positive (`--ctl-demo` : la liste s'affiche, items cliquables, app reçoit
  `MSG_CONTROL` avec le bon index). Conforme moratoire CLAUDE.md §10
  (audit factuel pré-implémentation de
  [gListC.def](https://github.com/bluewaysw/pcgeos/blob/master/Include/Objects/gListC.def)
  vs [gDListC.def](https://github.com/bluewaysw/pcgeos/blob/master/Include/Objects/gDListC.def),
  alignement sur `GenList` static, items inline = text monikers GeoWorks).
  Étapes 2-5 (MENU, RANGE, SPIN, FIELD) restent à instruire individuellement.
  ADR-30 §7.1 marqué FAIT.

### Added — ADR-31 (DRAFT) ouverte : clip widget hors rect parent
- **ADR-31 (DRAFT) ouverte** : dossier d'instruction sur le bug visuel
  observé en cours de validation d'ADR-30 Étape 1 (widgets restent peints
  hors window rect après resize-down). Bug pré-existant à ADR-30 mais
  révélé par `GU_LIST` (~48 px de hauteur). 3 options chiffrées :
  (A) skip widget hors rect, (B) clip partiel, (C) clip-list architectural
  (lié ADR-27 backing store + ADR-28 §6.5 damage tracking). **Recommandation
  senior tracée : (A)** pour court terme (~15 LOC patch local), (C) tracée
  comme refinement long terme. ADR-31 deviendra obsolète quand ADR-27 sera
  ratifiée (backing store = clip implicite). Dossier :
  `docs/adr/0031-clip-widget-rect-parent-DRAFT.md`.

## [2026-05-30] — ADR-30 (DRAFT) ouverte : roadmap toolbox (alignement GeoWorks)

### Added (Docs / Architecture)
- **ADR-30 (DRAFT) ouverte** : dossier d'instruction sur la roadmap toolbox
  d'OricOS vis-à-vis de la hiérarchie `Gen*` de PC/GEOS. Audit factuel
  WebFetch 2026-05-30 du
  [dossier Include/Objects](https://github.com/bluewaysw/pcgeos/tree/master/Include/Objects) :
  64 `.def` total (40 Gen + 7 Vis + ~18 subsystems). Couverture OricOS
  actuelle = **22 % des classes Gen, 57 % des widgets d'interaction
  directe**. 5 widgets prioritaires identifiés (`GU_LIST`, `GU_MENU`,
  `GU_RANGE`, `GU_SPIN`, `GU_FIELD`), ordonnés par valeur/coût.
  Recommandation senior tracée : option (D) roadmap incrémentale 5 étapes
  indépendantes, chacune ratifiable après validation interactive (leçon
  ADR-29). Cible post-Étape 5 : ~85 % des widgets d'interaction. Au-delà
  hors scope. Conforme moratoire CLAUDE.md §10 (dossier d'instruction
  écrit, référence externe vérifiée par WebFetch). Dossier :
  `docs/adr/0030-roadmap-toolbox-DRAFT.md`. CLAUDE.md §3 + index ADR mis à jour.

## [2026-05-30] — ADR-29 Étape 2 livrée : granularité par widget (GeoWorks complet)

### Added (OricOS kernel + SDK)
- **Étape 2 d'ADR-29 livrée** (refinement post-ratification, conforme
  moratoire §10) : pattern hint déclaratif **par widget** aligné GeoWorks
  `gValueC.def`. Tag `GU_HINT_IMMEDIATE_DRAG_NOTIFY = 10` permet à une app
  C de basculer un scrollbar/view en mode IMMEDIATE individuellement (les
  autres widgets restent au default DELAYED sûr). Tableau `WIDGET_HINTS`
  (`$016320`, 8 × 1B) stocke le hint par widget id. Override global
  `WM_DRAG_NOTIFY_HINT=$A5` conservé comme kill-switch debug.
  **Validation interactive utilisateur positive** : `--ctl-demo` (sans tag
  → default DELAYED) reste fluide, aucune régression vs Étape 1.
  `make tests` vert. ADR-29 §7.2 actée FAIT.

## [2026-05-30] — ADR-29 RATIFIÉE : drag notification hint (GeoWorks-aligned)

### Ratified (Workspace + OricOS)
- **ADR-29 ratifiée** ce jour suite à validation interactive utilisateur sur
  `--ctl-demo` : scrollbar fluide, value 1:1, FORBID_COUNT se libère
  normalement, app reçoit 1 `MSG_CONTROL` à la release au lieu de 60+/sec,
  curseur propre. Bug interactif « fin de course ascenseur » **éliminé**.
  Conforme moratoire CLAUDE.md §10 (audit §8 du dossier) : dossier
  d'instruction avec mtrace4 chiffré + 3 alternatives + reco senior tracée +
  référence externe vérifiée par WebFetch (gValueC.def) + implémentation
  Étape 1 testée headless ET validée interactivement. Fichier renommé
  `docs/adr/0029-drag-notification-hint.md`. CLAUDE.md §2 + index ADR +
  ADR_SUMMARY mis à jour. **Première ratification respectant intégralement la
  leçon ADR-28 « validation interactive AVANT ratification »** — Étape 1
  livrée derrière flag, testée par utilisateur, puis ratifiée.
- **Bug pré-existant `_wm_redraw_ctl` corrigé en même temps** : `draw_cursor`
  (invalidate+save+draw) remplacé par `cursor_blit` (restore+save+draw).
  Trace curseur visible révélée par le changement de timing d'ADR-29 (CPU
  libéré → cadence de redraw plus haute → bug §6.6 latent visible).
- **Refinements post-ratification suivis** : Étape 2 (granularité par
  widget + tag GenUI `GU_HINT_DRAG_NOTIFY_IMMEDIATE`), option CLI
  `--drag-hint-immediate` pour A/B, audit doc apps, extension aux futurs
  GenRange. Aucun non-bloquant pour la décision d'architecture.

## [2026-05-30] — ADR-29 (DRAFT) ouverte : drag notification hint (GeoWorks-aligned)

### Added (Docs / Architecture)
- **ADR-29 (DRAFT) ouverte** : dossier d'instruction sur la sémantique des
  messages pendant un drag de contrôle valeur. Bug interactif « fin de course
  ascenseur » localisé précisément (`mtrace4.log` : PC stuck à `$01:11EE` =
  boucle `kernel_scroll_up.bcc scrl_copy`, FORBID=1 bloqué). Cause racine :
  l'app reçoit `MSG_CONTROL` à chaque MOVED → `ctl_demo` sature print +
  scroll texte. Pattern résolu par GeoWorks il y a 30 ans avec
  `HINT_VALUE_DELAYED_DRAG_NOTIFICATION` (source officielle PC/GEOS lue et
  citée). 3 options chiffrées (statu quo / SymbOS dur / GeoWorks hint+default
  DELAYED), reco senior tracée pour GeoWorks. Implémentation gated par flag
  pour validation interactive **avant** ratification (leçon ADR-28).
  Dossier : `docs/adr/0029-drag-notification-hint-DRAFT.md`. CLAUDE.md §3 +
  index ADR mis à jour.
- **ADR-28 §6.7 rétractée** explicitement dans le fichier source : le quota
  anti-drop button-UP fixe un drop qui n'a jamais lieu. Code reste en place
  (non nocif) mais sans valeur. Le vrai fix du bug est dans ADR-29.

## [2026-05-29] — ADR-28 § 1.2ter rétractation : « famine réfutée » invalide

### Retracted (Docs)
- **Rétractation partielle de §1.2ter** : la conclusion « famine-par-coût
  scrollbar réfutée » repose sur un harnais qui injecte 1 event toutes les
  19 968 cycles (≤ 1 event/frame). En **usage interactif SDL réel** (motion
  60-1000 Hz, drags), la cadence est bien plus dense et le bug bbf067b
  « GUI gelée en fin de course d'ascenseur » **reste reproductible** (confirmé
  par test utilisateur). Les chiffres §1.2ter caractérisent un régime ≤ 1
  event/frame mais ne valent pas comme réfutation du bug famine. §6.6/§6.7
  apportent un vrai gain mesuré headless mais **n'éliminent pas le bug
  d'origine**. Cause racine à ré-instruire avec un harnais à cadence
  représentative (rafale 8+ events/frame). Trace dans ADR-28 §1.2ter et §6.7.
- **Leçon de méthode** : un harnais à cadence non représentative ne réfute
  pas un bug observé interactivement, même quand il passe tous les asserts.
  À ajouter à la discipline d'ADR future.

## [2026-05-29] — ADR-28 Étape 3 revertée (bug interactif), ratification design tient

### Reverted (OricOS kernel)
- **Étape 3 (skip `mouse_step` IRQ + appel en tâche) revertée** suite à bug
  interactif (`--wm-server` SDL : curseur figé, widgets non réactifs ; test
  headless ne reproduisait pas). État courant : Étape 2 pur (passe-plat).
  **Ratification ADR-28 (design option C) tient** — l'implémentation Étape 3
  est buguée, pas l'architecture. Investigation à reprendre. Cf. ADR-28 §7.4.
  Première leçon : les tests headless intégration ne couvrent pas le timing
  SDL réel ; tester en interactif AVANT de valider une ratification doit
  devenir réflexe pour les refactors IRQ ↔ tâche.

## [2026-05-29] — ADR-28 RATIFIÉE (option C, threading WM)

### Ratified (Workspace + OricOS docs)
- **ADR-28 ratifiée**, option C hybride : politique fenêtre + rendu en tâche
  serveur WM (`task_wm_entry`), curseur seul en IRQ. Gated `TC_WM_FLAG=$A5`
  (off par défaut, comportement legacy intact). Audit §10 vérifié : dossier
  d'instruction (§1.2bis/§1.2ter), implémentation testée (Étapes 0/1/2/3 +
  §6.6 + §6.7), cohérence ADR existantes (simplifie ADR-27 §0ter point 5).
  Fichier renommé `docs/adr/0028-threading-window-manager.md`. CLAUDE.md +
  index ADR + ADR_SUMMARY mis à jour. **Première ADR ratifiée avec
  implémentation de référence livrée ET mesurée en amont.**

## [2026-05-29] — ADR-28 Étape 3 : politique WM en contexte tâche (seuil 50%)

### Changed (OricOS kernel)
- **Bascule politique IRQ → serveur** (gated `TC_WM_FLAG=$A5`) : `handlers.s`
  skip `kernel_wm_mouse_step` ; `task_wm_entry` l'appelle après `raw_pop`. La
  politique (hit-test/focus/drag/resize/chrome/callbacks/`redraw_drag`) tourne
  désormais **en contexte tâche** ; seul `cursor_blit` reste en IRQ
  (option C). Test `test-oricos-wm-server` valide la chaîne complète.
  `make tests` vert. **Seuil moratoire 50 % atteint** (CLAUDE.md §10) →
  ratification ADR-28 désormais ouverte (sous réserve campagne GUI +
  validation humaine).
- Limites assumées documentées (refinements non-bloquants) : état souris en
  burst, §6.6 partiellement perdue en mode serveur, §6.7 RAW à porter,
  `php/sei…plp` gfx toujours présents. Cf. ADR-28 §7.4.

## [2026-05-29] — ADR-28 §6.7 : quota EVENT_RING anti-drop button-UP

### Changed (OricOS kernel + Phosphoric test)
- **Quota EVENT_RING** : `event_push_key` et la branche MOVED de `event_push_mouse`
  limités à `ENTRIES-2=14`. Les transitions DOWN/UP gardent la limite pleine 16.
  Garantit que le button-UP **n'est jamais droppé** par saturation — ferme le
  scénario "gel scrollbar interactif" (§1.2ter). Test isolé ajouté à
  `test-oricos-raw-ring`. `make tests` vert.

## [2026-05-29] — ADR-28 Étape 2 : tâche serveur WM passe-plat

### Added (OricOS kernel + Phosphoric test)
- **Tâche serveur WM passe-plat** activable par `TC_WM_FLAG=$A5`. Chaîne
  prouvée IRQ → `RAW_RING` → block/wake → `task_wm_entry` → `EVENT_RING`
  (test `test-oricos-wm-server` : `RAW_WAITER=8`, RAW drainé, event repushé).
  Routing transparent à l'entrée de `kernel_event_push_mouse`/`_key` (mode
  legacy = 6 cyc/push, indétectable). Comportement net identique pour l'app.
  Aucune régression (`make tests` vert). Préparation Étape 3.
- **Leçon timing IRQ** : un essai initial avec `jsr kernel_raw_wake` dans
  `handlers.s` cassait `clock`/`ctl_demo` (~12 cyc supplémentaires détectés).
  Wake colocalisé dans `raw_push_*` à la place — zéro overhead en mode legacy.

## [2026-05-29] — ADR-28 §6.6 : suppression du curseur dupliqué (drag widget)

### Changed (OricOS kernel)
- **`wm_step_drag` (wm.s)** : pendant un drag de widget (`SCROLL_DRAG_ID` armé),
  l'IRQ skip `cursor_blit` (le main loop dessinera le curseur via `_wm_redraw_ctl`).
  **Mesure** : cursor_blit 13 → 1 appel, mouse_step 20,8 % → **6,5 %**, total/event
  drag scrollbar ≈ 34 % → **≈ 16 %** du budget frame. Value 1:1 préservée.
  Non-structurel. `make tests` vert. ADR-28 §6.6 marquée FAIT.

## [2026-05-29] — ADR-28 §1.2ter : mesure main-loop scrollbar (famine réfutée)

### Measured (Phosphoric test + OricOS exports)
- **Chemin main-loop du drag d'ascenseur mesuré** (`test-oricos-scroll-cost`,
  active `task_scr`) : value 1:1 avec les events, ~34 % du budget frame →
  **famine-par-coût réfutée** à ≤ 1 event/frame. Le « gel » interactif est une
  **saturation de ring** (button-UP droppé), pas un manque de cycles. Findings
  ciblés indépendants du refactor : curseur rendu 2×/event (~33 % budget,
  ADR-28 §6.6) ; anti-drop button-UP (§6.7). Conséquence : la famine n'est plus
  l'argument principal de l'Étape 3 ; restent race GPU, sûreté callback, coût
  drag-fenêtre 53 %, curseur dupliqué. `make tests` vert.

## [2026-05-29] — ADR-28 Étape 1 : skip-si-delta-nul (D3)

### Changed (OricOS kernel + Phosphoric test)
- **Skip-si-delta-nul** sur `wm_step_do_drag`/`_wm_do_resize` (wm.s) : un
  `MOUSE_MOVED` sans déplacement réel ne redessine plus (~13000 → 37 cyc,
  prouvé par `test-oricos-wm-cost`). Non-structurel, `make tests` vert.
  **Re-scope honnête** (ADR-28 §7.2) : la mesure localise la famine d'**ascenseur**
  dans la main loop (`_wm_scroll_update`), pas dans l'IRQ (où `mouse_step` en drag
  scrollbar ne fait que `cursor_blit`). D3 aide le drag de **fenêtre/resize** ;
  le fix scrollbar relève de l'Étape 3 (architecture). Finding tracé.

## [2026-05-29] — ADR-28 Étape 0 : RAW input ring (scaffolding)

### Added (OricOS kernel + Phosphoric test)
- **RAW input ring** non câblé (ADR-28 §7.1, première brique du refactor option
  C) : `RAW_RING` bank 1 ($016400) + `kernel_raw_init`/`push`/`pop` (transport
  verbatim via bloc ZP $D0..$D9), destiné à la future tâche serveur WM.
  **Boot et chemin GUI intacts** (réversible). Test `test-oricos-raw-ring`
  (appels kernel isolés, détection retour par pile S). `make tests` vert.
  Plan d'implémentation incrémental complet : ADR-28 §7 (Étapes 0→5, seuil
  moratoire 50 % à l'Étape 3). Étapes 2+ (refactor structurel) en attente de
  confirmation humaine (CLAUDE.md §6).

## [2026-05-29] — ADR-28 (DRAFT) : threading du Window Manager (revue senior)

### Docs / Architecture (oric2)
- **Revue d'architecture senior WM/widgets** → ouverture **ADR-28 (DRAFT)**,
  dossier d'instruction. Constat racine : la politique WM (hit-test, focus,
  Z-order, drag, resize, chrome, callbacks) **et le rendu plein écran**
  s'exécutent dans l'IRQ souris (`kernel_wm_mouse_step`, handlers.s:129), IRQ
  masquées, travail **dupliqué** avec la main loop (`EVENT_RING`/`SYS_MAIN_LOOP`).
  Thèse : cause commune de la famine main loop (ADR-27 §0bis), de la race GPU
  `bpl`/ARG et du danger callback-en-IRQ (incompatible réouverture ADR-15).
  3 options chiffrées (A patches / B serveur WM en tâche / C hybride curseur-IRQ),
  reco senior : C, A en palliatif. Non tranchée (mesure `redraw`/event + arbitrage).
  Couplage : geler le flip compact ADR-27 §0ter jusqu'à arbitrage ADR-28.
  Conforme moratoire CLAUDE.md §10. Fichiers : `docs/adr/0028-…-DRAFT.md`,
  index ADR, CLAUDE.md §3, OricOS/CHANGELOG.

### Mesure / Outillage (Phosphoric — `test-oricos-wm-cost`)
- **Harnais de mesure du coût WM dans l'IRQ** livré (ADR-28 §1.2bis) :
  `tests/integration/test_oricos_wm_cost.c`, boot kernel oric2 + VRAM/GPU en
  mode GUI persistant, injection clic + drags, **coût inclusif** par routine via
  détection de retour sur le pointeur de pile S. Build OricOS : `-Ln kernel.lbl`
  (labels) ajouté au ld65 (kernel.bin inchangé). **Résultats** (budget frame
  19968 cyc) : `mouse_step` **66,8 %/event** (famine quantifiée),
  `redraw_drag` **53,6 %** ≈ `redraw` plein 52,9 % (le drag « incrémental »
  redessine tout → 0 économie), curseur 9–17 %. **Verdict : option C** (curseur
  en IRQ, reste en tâche) — coût IRQ/event ≈ 67 % → ≈ 17 % sans régression
  latence curseur. Borne basse (GPU émulé synchrone), pire sur HDL. Déterministe
  (min==max sur drags). Cible ajoutée à l'agrégat `make tests`.

## [2026-05-28] — ADR-27 §0bis : instrumentation on-target IRQ MOU2/frame

### Mesure / Outillage (Phosphoric 1.22.90-alpha)
- **Compteur d'IRQ MOU2 par frame** livré (env `PHOSPHORIC_MOU2_IRQTRACE=1`,
  `--machine oric2`) : compte les fronts montants de `IRQF_MOU2`, résumé par frame.
  Build SDL OK, `make tests` verts, 0 front parasite au repos (11314 frames headless).
- **Finding (lecture de code)** : le poll SDL draine les events souris **1×/frame,
  après** les cycles CPU → ligne MOU2 assertable **≤ 1×/frame** par construction
  (compteur `multi` ≡ 0). **Réfute** l'hypothèse « famine fréquentielle » du caveat
  §0bis : ≤ 1 IRQ/frame (~2000-4000 cyc) sur 19968 cyc/frame n'affame pas la main
  loop. Cause de la value figée à chercher côté scheduler/`FORBID` ou coût du redraw
  **par event** en main loop. Reste à mesurer : events `MOUSE_MOVED` drainés/frame +
  cycles tâche-app vs handler (2e volet du caveat). Outil de mesure, **aucune
  correction** (D1-D4 non tranchées, moratoire CLAUDE.md §10).

## [2026-05-28] — ADR-27 §0bis : famine main loop pendant drag d'ascenseur rapide

### Instruction / Architecture
- **Nouveau hazard de concurrence documenté** (dossier ADR-27 §0bis) : un drag
  d'ascenseur rapide **fige le thumb** (value bloquée à une valeur intermédiaire)
  pendant que le curseur continue de bouger. Cause racine : **asymétrie de
  traitement** — curseur + drag fenêtre mis à jour dans l'IRQ (`kernel_wm_mouse_step`),
  mais la value du scrollbar via la tâche app (`SYS_MAIN_LOOP` → `_wm_scroll_update`,
  redraw GPU + aller-retour syscall par `MOUSE_MOVED`). En drag rapide, les IRQ
  souris affament la main loop → la value ne suit plus ; elle **rattrape dès l'arrêt
  du mouvement** (donc retard, pas gel logique). Distinct du bug ring/UP corrigé le
  2026-05-27 (que le coalescing ne couvre pas).
- **Reproduction déterministe** (`oricrobot`, TC_SCR_FLAG) : flux `moverel` à cadence
  serrée fige la value ; `run` à souris immobile la fait rattraper d'un coup.
- **Direction retenue par l'humain** : « découpler / alléger l'IRQ ». **À instruire
  avant tout code** (touche le partage IRQ↔syscall du §0bis, moratoire ADR). Aucun
  code ni ADR ratifiée à ce stade.

## [2026-05-27] — Fix GUI gelée après drag d'ascenseur au max (Phosphoric 1.22.89)

### OricOS
- **Bug `kernel_event_push_mouse`** : un drag de scrollbar long inondait le ring
  d'événements (16) de `MOUSE_MOVED` → le button-UP était droppé → `SCROLL_DRAG_ID`
  restait armé → le WM restait bloqué en mode drag, tout clic suivant avalé comme
  scroll → l'app ne répondait plus. Fix : **coalescing des `MOUSE_MOVED`** (maj en
  place au lieu d'empiler). Diagnostic par reproduction déterministe (drag→max :
  value atteint bien 84, mais WM_COUNT bloqué à 3 ; après fix → 2).
- **Collision ZP $6E** (régression 1.22.87) : `GFX_ARG4_LO` ⟷ `EVT_TMP` → `GFX_ARG4`
  déplacé en `$92/$93`.

### Phosphoric
- Test `test_oricos_scroll_drag_max_responsive` (595 tests verts).

## [2026-05-27] — ADR-27 : analyse de concurrence (migration kernel bloquée)

### Instruction / Architecture
- **Analyse de concurrence ADR-27** (dossier §0bis) : la migration vers des backing
  stores compacts est **bloquée sur une décision de concurrence**. Le registre `bpl`
  est un état GPU global persistant ; le handler IRQ dessine le framebuffer
  (`kernel_wm_mouse_step`→`kernel_wm_redraw`) pendant que les syscalls tournent IRQs
  activées. 4 options de résolution instruites (reset-IRQ / sei-cli / stride
  par-commande / shadow ZP), reco = reset-IRQ + fix race ARG.
- **Finding préexistant tracé** (`OS-gpu-race`, BACKLOG P1bis) : race GPU ARG ↔ mouse
  IRQ latente (clobber d'une commande gfx en cours). Tolérée, à corriger ; pré-req au
  flip compact. **Aucun changement de code** (décision de concurrence requise d'abord).

## [2026-05-27] — ADR-27 opt.b : registre BPL GPU configurable (Phosphoric 1.22.88)

### Décision
- L'humain retient l'**option (b)** d'ADR-27 : stride GPU (BPL) configurable +
  backing store compact. Dossier `docs/adr/0027-*-DRAFT.md` mis à jour (option b
  retenue, référence GPU implémentée ≥ amorce, migration kernel = reste).

### Phosphoric
- Nouvel opcode `GPU_OP_SET_BPL` ($08) + `gpu->bpl` persistant (défaut 512).
  BLIT double-stride (src=bpl compact, dst=512 XVGA) ; FILL/LINE/TEXT honorent bpl.
  2 tests GPU (594 verts). Contrainte : aucun port I/O libre → opcode, pas de registre dédié.

### OricOS
- Helper `kernel_gfx_set_bpl` + `GPU_OP_SET_BPL` + ZP `GFX_BPL` ($90/$91).
- Fix régression `.smart` ca65 (helper sans `sep #$20` cassait l'assemblage du
  compositeur). Reste : câbler `bpl` dans `kernel_wm_compose` (migration kernel).

## [2026-05-27] — Audit GPU toolbox senior : ADR-27 ouverte (backing store fenêtre)

### Docs / Architecture
- **Audit exhaustif de la toolbox GPU** (gfx.s + gpu_device.c + chemins compose/syscall).
  Aucun nouveau bug fonctionnel actif. Constats transverses : poll loops ignorent ERR
  (mineur), précondition M=1 non défendue (latent), `vram_poke` borné 24-bit (memory-safe).
- **Finding B → ADR-27 ouverte (DRAFT)** : le backing store (1 banque 64 KiB/slot, stride
  GPU figée 512) plafonne à 128 lignes ; `kernel_wm_compose` sur-lit les banques voisines
  pour une fenêtre > 128 px (limite latente, exposée par le fix BLIT v0.2). Dossier
  d'instruction `docs/adr/0027-backing-store-fenetre-DRAFT.md` (options a/b/c/d chiffrées,
  recommandation (b) stride configurable, **non tranchée** — moratoire §10). Ajout en
  CLAUDE.md §3 + BACKLOG P3. **Aucun changement de code** (mitigation v1 : apps ≤ 128 px).

## [2026-05-27] — GPU BLIT v0.2 : byte_w/byte_h 16-bit (Phosphoric 1.22.87)

### Phosphoric
- `gpu_device.c` `gpu_exec_blit` : ARG3[15:0]=byte_w, ARG4[15:0]=byte_h (fix overflow).
- `test_gpu_device.c` : 2 tests mis à jour + `test_blit_wide_16bit` ajouté (592 tests).

### OricOS
- **Bug A** `kernel_wm_compose` : stores 16-bit (rep #$20) pour byte_w/byte_h → fix
  troncature 8-bit sur fenêtres ≥ 256 px.
- `kernel_gfx_blit` : écriture GPU ARG4 (byte_h) ajoutée.
- `boot.s` : 3 appels BLIT migrés vers encodage v0.2.
- ZP `GFX_ARG4_LO=$6E` / `GFX_ARG4_MID=$6F` alloués.

## [2026-05-26] — GPU toolbox kernel OricOS : 3 bugs wm.s (Z-order, poll loop, curseur) (Phosphoric 1.22.86)

### OricOS (wm.s)
- **Bug `kernel_wm_compose` — Z-order ignoré** : compositor itérait slots 0..N au lieu
  de WM_ZORDER → fenêtres superposées composées dans le mauvais ordre. Fix : itération
  WM_ZORDER[0..N-1].
- **Bug `kernel_wm_compose` — fenêtres minimisées composées** : condition `WM_F_USED`
  seul → fenêtres cachées BLITtées. Fix : `(WM_F_USED | WM_F_VISIBLE)`.
- **Bug `kernel_gfx_fill_rect16` — poll loop manquant** : seul helper sans attente
  GPU_TRIGGER. Fix : ajout poll loop standard (latent v0.1, critique v0.2).
- **Bug `sys_win_flush` — CURSOR_VALID non invalidé** : fond curseur périmé après BLIT.
  Fix : `sta CURSOR_VALID #0` post-compose. 591 tests verts.

## [2026-05-26] — GPU toolbox kernel OricOS déboguée : bug critique kernel_wm_compose (Phosphoric 1.22.85)

### OricOS (wm.s)
- **Bug critique `kernel_wm_compose` — BLIT dst `$000000` au lieu de `$100000`** :
  la compositing loop calculait l'adresse de destination comme `y*512 + x/2` depuis
  `$000000`. Le framebuffer XVGA (ADR-20) est à `$100000`. Résultat : `SYS_WIN_FLUSH`
  ne produisait aucun affichage visible. Fix : `adc #$10` sur `GFX_ARG2_HI`. Tous
  les commentaires internes corrigés.

### Phosphoric (test_oricos_helloc.c)
- Tests `test_oricos_win_draw` et `test_oricos_win_app` mis à jour :
  assertions `$00A032` → `$10A032` (= `$100000 + 80*512 + 50`). 591 tests verts.

## [2026-05-26] — GPU toolbox debuggée : 3 bugs + 3 tests (Phosphoric 1.22.84)

### Phosphoric
- **Bug 1 — `gpu_set_pixel` : no bounds check (SÉVÈRE)** : le helper n'effectuait aucun
  contrôle x/y avant d'écrire en VRAM. Une coordonnée hors-écran (ex. y≥768) pouvait
  corrompre silencieusement la SDRAM simulée au-delà du framebuffer. Corrigé par ajout
  de guards en début de fonction.
- **Bug 2 — `gpu_text_render` : clamp Y prématuré (SÉVÈRE)** : la condition
  `y_start + 7 >= GPU_XVGA_H` stoppait le rendu de TOUS les caractères dès que la
  dernière ligne du glyphe débordait, même si les 6 ou 7 premières lignes étaient
  valides. Corrigé : le `break` ne se déclenche que si `y_start >= GPU_XVGA_H` ;
  les lignes partielles sont clampées par `gpu_set_pixel`.
- **Bug 3 — STATUS register : BUSY/DONE non exclusifs (MEDIUM)** : quand
  `busy=true && err=false`, les bits DONE (0x01) et BUSY (0x80) étaient levés
  simultanément. La sémantique correcte est mutuellement exclusive. Corrigé par
  remplacement du `if/else` en chaîne `if/else if/else`.
- 3 nouveaux tests : `test_status_busy_excludes_done`, `test_text_clamp_y_partial`,
  `test_text_y_out_of_bounds_no_corruption`. 591 tests verts (18/18 gpu_device).

## [2026-05-26] — SP-3.o S.7 v2b : fix régression corps de fenêtre (bloc course 16-bit ca65)

### OricOS
- **Corps de fenêtre effacé au clic d'un contrôle** (régression v2) : le calcul
  de la course de la gouttière dans `_wm_scroll_update` en mode 16-bit (rep/sep +
  immédiats) générait du code corrompu (mauvais tracking de mode ca65) effaçant le
  corps des fenêtres. Réécrit en **8-bit pur**. Vérifié par dump du framebuffer XVGA.

### Phosphoric
- 588 tests verts. EMU_VERSION 1.22.83-alpha.

## [2026-05-26] — SP-3.o S.7 v2 : fix ascenseur mi-course + curseur (critique senior)

### OricOS
- **Ascenseur bloqué à mi-course** : `_wm_scroll_update` clampe `value` à la
  course réelle de la gouttière (`dimension − SCROLL_THUMB_SZ`) au lieu du max
  logique → le pouce atteint le bas.
- **Curseur disparu après redraw ciblé** (régression S.7) : `_wm_redraw_ctl`
  (sei + `kernel_wm_redraw_widget` + `kernel_wm_draw_cursor`) redessine le curseur
  après chaque repaint widget (3 sites : drag ascenseur, store champ texte, MSG_CONTROL).

### Phosphoric
- `test_oricos_scrollbar` : drag jusqu'en bas, assert `val==44`. 588 tests verts.
  EMU_VERSION 1.22.82-alpha.

## [2026-05-26] — SP-3.o S.7 : redraw ciblé (fix scintillement scroll/texte)

### OricOS
- **`kernel_wm_redraw_widget`** : repeint un seul contrôle au lieu du desktop
  entier. Corrige le scintillement plein écran au drag d'ascenseur et à
  l'édition de champ texte (refactor `_wm_draw_widget_body`).

### Phosphoric
- 584 tests verts (comportement inchangé, rendu fluide). EMU_VERSION 1.22.81-alpha.

## [2026-05-26] — Sprint 4 : première vraie app C (clock)

### OricOS
- **App C `clock`** : pilotée par le temps (fenêtre + barre rythmée par
  `SYS_GET_TICKS` + `SYS_YIELD`, dessin GFX). Nouveau syscall `SYS_GET_TICKS`
  ($1D, compteur ticks 8-bit). SDK `oricos_get_ticks` (clobbers a/x/y).

### Phosphoric
- **`test_oricos_clock`** : la boucle temps boucle ("clock: done") → get_ticks
  avance. `test_syscall_table_size` maj ($1D câblé). 584 verts.

## [2026-05-26] — SP-3.o S.6 : démo C contrôles (capstone — arc SP-3.o CLOS)

### OricOS
- **App C `ctl_demo`** : déclare fenêtre + checkbox + ascenseur + champ texte
  (table GenUI), lit la valeur du contrôle touché (`SYS_CTL_GET_VALUE`) sur
  `MSG_CONTROL`. Clôt l'arc SP-3.o (contrôles GeoWorks depuis userland C).
- **Segment `BUNDLES`** ($7000+) : images d'apps sorties du CODE → marge regagnée
  sous le plafond $5400.

### Phosphoric
- **`test_oricos_ctl_demo`** : clic ascenseur → "ctl: v=", close → exit. 583 verts.

## [2026-05-26] — SP-3.o S.5 : tags GenUI déclaratifs des contrôles

### OricOS
- **`GU_CHECK`/`GU_SCROLL_V`/`GU_SCROLL_H`/`GU_RADIO`/`GU_TEXT`** dans
  `sys_ui_define` : contrôles valeur/saisie déclarables en table GenUI (rect +
  extra). Helpers `_sud_rect`/`_sud_attach`. `task_genui` (4 contrôles déclarés).
  `GU_LIST` différé (pointeur blob = bank app). SDK mis à jour.

### Phosphoric
- **`test_oricos_genui`** : vérifie type + valeur initiale des 4 contrôles
  déclarés via SYS_UI_DEFINE. 582 verts.

## [2026-05-26] — SP-3.o S.4c : liste sélectionnable (GenList)

### OricOS
- **`WG_TYPE_LIST`** : liste d'items (blob slots 8 o, selected+14/count+15),
  rendu `kernel_tk_list` (surlignage sélection), `kernel_ctl_list_select`
  (row au clic). `task_list` (3 items).
- **Fix layout** : CODE dépassait `$5000` et écrasait SENTINEL/VERSION ;
  relocalisés en `$016300`/`$016310`. Plafond CODE = TICK_COUNTER `$015400`.

### Phosphoric
- **`test_oricos_list`** : clic 3e item → selected=2. 581 verts. Tests sentinelle
  boot + champ texte mis à jour (nouvelles adresses).

## [2026-05-26] — SP-3.o S.4b : champ texte éditable (GenText/LineEdit)

### OricOS
- **`WG_TYPE_TEXT`** : champ texte éditable. Buffer bank 1 (`TEXT_BUFS`+id*16),
  focus clavier (`TEXT_FOCUS_ID`), édition `_wm_text_edit` (insertion/backspace)
  pilotée par `mlc_key`, rendu `kernel_tk_text_field` (face+cadre+texte+curseur).
  `task_text` (champ maxlen 14).

### Phosphoric
- **`test_oricos_text_field`** : clic (focus) + 'A','B',backspace → "A" (len 1).
  580 verts.

## [2026-05-26] — SP-3.o S.4a : radios mutuellement exclusifs (GenItemGroup)

### OricOS
- **`WG_TYPE_RADIO`** (selected+14/group+15) + `kernel_ctl_radio_select`
  (exclusion par group id) ; dispatch clic MainLoop + desktop IRQ ; rendu case
  colorée. `task_radio` (2 radios, group 1).

### Phosphoric
- **`test_oricos_radio`** : clic radio 1 → exclusion (v1=1, v0=0). 579 verts.

## [2026-05-26] — SP-3.o S.3c : GenView déclaratif + démo C

### OricOS
- **Tag `GU_VIEW`** dans `sys_ui_define` : GenView déclarable dans une table
  GenUI (rect + max scroll). App C `view_demo` : déclare fenêtre + `GU_VIEW`,
  lit `scroll_y` via `SYS_CTL_GET_VALUE` sur `MSG_CONTROL` (modèle GeoWorks).
  SDK : `oricos_ctl_get_value` + `oricos_msg_id`.

### Phosphoric
- **`test_oricos_view_demo`** : GU_VIEW déclaratif + lecture scroll_y en C.
  578 tests verts. `EMU_VERSION` réaligné (1.22.74-alpha).

## [2026-05-26] — SP-3.o S.3 (a+b) : GenView (viewport scrollable managé)

### OricOS
- **`WG_TYPE_VIEW`** : viewport + scrollbar intégré (rendu `kernel_tk_view`) ;
  drag → `scroll_y` (réutilise `SCROLL_DRAG_ID`/S.2). L app lit scroll_y et
  redessine son contenu (modèle GeoWorks). Reste S.3c (GU_VIEW + démo C).

### Phosphoric
- **`test_oricos_genview`** : clic+drag → scroll_y suit. 577 tests verts.

## [2026-05-26] — SP-3.o S.2 : ascenseurs (scrollbars) + thumb-drag

### OricOS
- **`WG_TYPE_SCROLL_V/_H`** + thumb-drag dans le MainLoop (`SCROLL_DRAG_ID`,
  value=clamp(souris-gouttière,0,max)) + rendu gouttière+thumb. Fix conflit drag
  fenêtre↔contrôle (`wm_step_arm_drag` teste le widget d abord).

### Phosphoric
- **`test_oricos_scrollbar`** : clic+drag → la value suit la souris. gui_demo
  durci (MSG_CLOSE via WM_COUNT). 576 tests verts.

## [2026-05-26] — SP-3.o S.1 : API valeur de contrôle + checkbox

### OricOS
- **`SYS_CTL_GET_VALUE` ($1B) / `SYS_CTL_SET_VALUE` ($1C)** : API valeur des
  contrôles. **`WG_TYPE_CHECK`** (checkbox/GenBoolean, value en +14,
  `kernel_ctl_toggle`, garde callback par type → anti-crash). Base ascenseurs (S.2+).

### Phosphoric
- **`test_oricos_ctl_value`** : round-trip SET_VALUE(1)/GET_VALUE → 1. 575 verts.

## [2026-05-26] — BACKLOG : arc SP-3.o cadré (contrôles valeur + ascenseurs + GenView)

### Docs (oric2)
- **`BACKLOG.md`** : arc **SP-3.o** cadré — compléter la famille de contrôles
  « valeur/saisie » manquante vs GeoWorks (le windowing est déjà à parité, les
  widgets sont pauvres). Découpage S.1 API valeur (`SYS_CTL_GET/SET_VALUE`) +
  checkbox, S.2 ascenseurs (scrollbar V/H + thumb-drag), S.3 GenView (viewport
  scrollable managé), S.4 radios/champ texte/liste, S.5 tags GenUI, S.6 démo C.
  Réf : GeoWorks GenScrollbar/GenView/GenBoolean/GenText/GenList. Extension
  mineure d'ADR-26.

## [2026-05-26] — SP-3.n polish : titres + libellés de boutons

### OricOS
- **GenUI chaînes INLINE** (GU_TITLE/GU_BUTTON, stagées bank 1 via _sud_copy_inline)
  → titre + libellé bouton fonctionnent pour les apps C. **Libellés distincts**
  dialogue/alerte (OK/Cancel/Yes/No). gui_demo : titre "Demo C" + bouton "Clic".

### Phosphoric
- **test_oricos_win_app durci** : preuve G.3 via WM_COUNT 3→2 (robuste au scroll
  console) au lieu du texte "sortie". 574 tests verts.

## [2026-05-26] — SP-3.n G.7 (suite) : option --gui-demo (démo GUI visible)

### OricOS
- **`sys_ui_define`** repeint le desktop après création → l'UI déclarée apparaît
  immédiatement. **`apps/gui_demo`** : fenêtre repositionnée (420,420) pour être
  distincte des fenêtres boot.

### Phosphoric
- **Option `--gui-demo`** : lance l'app gui_demo (`./oric1-emu --kernel ... --xvga
  --gui-demo`). La fenêtre déclarée (bouton OK, taskbar "Win2") apparaît à l'écran.
  574 tests verts.

## [2026-05-26] — ADR-26 RATIFIÉE : modèle GUI déclaratif GenUI/SpecUI

### Docs (oric2)
- **ADR-26 ratifiée** (draft → ratifiée) : modèle GUI déclaratif GeoWorks-like
  (UI en tables GenUI/`DB_*`, MainLoop→messages, DoDlgBox/Alert UI-modal,
  GenUI/SpecUI). 3 conditions moratoire §10 remplies (dossier, 100 % impl arc
  SP-3.n testée 574 verts, cohérence). Fichier renommé `0026-modele-gui-declaratif.md`,
  index README mis à jour, CLAUDE.md §2 (ADR-26) + révision note ADR-06.

## [2026-05-26] — SP-3.n G.7 : app C GUI déclarative + MainLoop (arc CLOS)

### OricOS
- **`apps/gui_demo/gui.c`** : app C qui déclare son UI (GenUI fenêtre+bouton) +
  boucle MainLoop + réagit aux messages (MSG_CONTROL → "gui: bouton",
  MSG_CLOSE → sort). `sys_ui_define` refondu : `GU_BUTTON` attache des contrôles
  déclarés. SDK : `oricos_ui_define`/`oricos_main_loop`/`oricos_alert`/`do_dlgbox`.

### Phosphoric
- **`test_oricos_gui_demo`** : l'app C déclare fenêtre+bouton, clic bouton →
  MSG_CONTROL ("gui: bouton"), clic fermeture → MSG_CLOSE ("gui: sortie").
  **Arc SP-3.n (G.1→G.7) clos. 574 tests verts.**

## [2026-05-26] — SP-3.n G.6 : SYS_ALERT (alertes pré-câblées)

### OricOS
- **`sys_alert` ($1A)** : alerte OK / OK-Cancel / Yes-No (type en X). Crée la
  fenêtre modale + boutons puis réutilise la boucle modale de DoDlgBox
  (`jmp ddb_show`). Retour 1 (gauche) / 0 (droite).

### Phosphoric
- **`test_oricos_alert`** : alerte OK-Cancel, clic OK → retour 1 + fermeture
  (WM_MODAL=$FF). 573 tests verts.

## [2026-05-26] — SP-3.n G.5 : SYS_DO_DLGBOX (dialogue modal)

### OricOS
- **`sys_do_dlgbox` ($19)** : command table `DB_*` (style GEOS) → fenêtre modale
  + boutons OK/Cancel + boucle modale ; retour 1/0. **UI-modal** (WM_MODAL) : la
  saisie va au dialogue, la tâche bloque mais rend le CPU (préemption préservée).
- **`kernel_event_wait`** : helper bloquant réutilisable (block/wake).

### Phosphoric
- **`test_oricos_dlgbox`** : clic OK → retour 1 + dialogue fermé (WM_MODAL=$FF).
  572 tests verts.

## [2026-05-26] — SP-3.n G.4 : contrôles → MSG_CONTROL

### OricOS
- **`_ml_classify`** : un clic touchant un contrôle (bouton) de la fenêtre
  (`_wm_widget_hit`) → `MSG_CONTROL` + id du contrôle. L'app réagit via le
  MainLoop (callback kernel conservé en coexistence v1).

### Phosphoric
- **`test_oricos_mainloop_control`** : clic sur le bouton "OK" de la fenêtre 0
  (app-driven) → MSG_CONTROL + id widget. 571 tests verts. (Tag `GU_BUTTON`
  déclaratif reporté à G.7.)

## [2026-05-26] — SP-3.n G.3c : chrome → messages (G.3 complet)

### OricOS
- **`_ml_classify`** : barre de menu (`y < MENU_BAR_H`) → `MSG_MENU` ; case
  fermeture → `MSG_CLOSE` + id ; sinon `MSG_CONTENT`.
- **`WM_APP_DRIVEN`** (posé par `sys_main_loop`) : en mode app-driven, le shell ne
  ferme plus au clic close-box — l'app reçoit `MSG_CLOSE` et décide (GeoWorks).
  Auto-close conservé hors app (test_wm_close_button intact).

### Phosphoric
- **`test_oricos_mainloop_close`** (close-box → MSG_CLOSE + fenêtre ouverte) +
  **`test_oricos_mainloop_menu`** (barre de menu → MSG_MENU). Arc G.3 complet.
  570 tests verts.

## [2026-05-26] — SP-3.n G.3b : SYS_UI_DEFINE (UI déclarative GenUI)

### OricOS
- **`sys_ui_define` ($18)** : l'app passe une **table GenUI** (`GU_WINDOW`/
  `GU_TITLE`/`GU_END`) ; le kernel la parse (`lda [$D0],y`) et crée la fenêtre
  (`kernel_wm_add` + focus). Modèle déclaratif GeoWorks (UI = donnée).
- **Fix race `_ml_classify`** : `WM_ARG_*` partagé avec l'IRQ souris → `php/sei…plp`
  autour du hit-test (ADR-25 Disable/Enable).

### Phosphoric
- **`test_oricos_ui_define`** : table GenUI → fenêtre (handle valide, x==300).
- **`test_oricos_sprint2a` durci** : les compteurs 8-bit TASK_A/B/C wrappent (mod
  256) et peuvent valoir 0 au STP exact → on prouve désormais qu'ils ont été
  non-nuls *pendant* le run (robuste au timing, vs lecture unique fragile).
  568 tests verts.

## [2026-05-26] — SP-3.n G.3a : SYS_MAIN_LOOP (messages sémantiques)

### OricOS
- **`sys_main_loop` ($17)** : modèle GeoWorks — bloque jusqu'à un message
  significatif, traduit les événements bruts (`_ml_classify`) : touche→`MSG_KEY`,
  clic fenêtre→`MSG_CONTENT`+id (hit-test), moved/up→`MSG_NULL` (sautés).
  `kernel_wm_mouse_step` inchangé (focus/drag restent WM-automatiques).

### Phosphoric
- **`test_oricos_mainloop_message`** : move (sauté) puis clic fenêtre →
  MSG_CONTENT + id valide. 567 tests verts.

## [2026-05-26] — SP-3.n G.2 : SYS_GET_NEXT_EVENT + SYS_EVENT_AVAIL

### OricOS
- **`sys_event_avail` ($15)** (non-bloquant) + **`sys_get_next_event` ($16)**
  (bloquant via block/wake ADR-25, `EVENT_WAITER`, réveil IRQ `kernel_event_wake`).
  Le record est copié dans le bloc ZP $D0-$D9. Base du MainLoop (G.3+).

### Phosphoric
- **`test_oricos_event_syscall`** : task_evt bloque sur SYS_GET_NEXT_EVENT,
  réveillée par l'IRQ → reçoit EV_KEY_DOWN / message 'B'. 566 tests verts.

## [2026-05-26] — SP-3.n G.1 : file d'événements unifiée

### OricOS
- **`kernel/modules/event.s`** : file d'événements bank 1 (`EVENT_RING` $015880,
  16×10 o). `kernel_event_push_key`/`_mouse`/`_pop`/`_init`. Alimentée par les IRQ
  KBD2/MOU2 en **coexistence** avec KBD_RING/MOUSE_* (migration progressive →
  aucun consommateur actuel modifié). Base de `SYS_MAIN_LOOP` (G.2).

### Phosphoric
- **`test_oricos_event_queue`** : touche 'A' → EV_KEY_DOWN(msg='A') ; clic
  (250,150) → événement souris dans la file. 565 tests verts.

## [2026-05-26] — ADR-26 (draft) : modèle GUI déclaratif GenUI/SpecUI

### Docs (oric2)
- **`docs/adr/0026-modele-gui-declaratif-DRAFT.md`** : draft d'ADR (NON ratifiée)
  posant le modèle GUI déclaratif GeoWorks-like (UI en tables, MainLoop→messages,
  GenUI/SpecUI, `DoDlgBox` command table). Alternatives écartées (TaskMaster IIgs,
  callbacks kernel, moteur objet Goc), conséquences, conformité moratoire §10
  (ratification bloquée tant que < 50 % impl arc SP-3.n). Indexée README ADR
  (ouvertes/parquées). Révise ADR-06 à la ratification.

## [2026-05-26] — BACKLOG : arc SP-3.n cadré (GUI déclarative GeoWorks)

### Docs (oric2)
- **`BACKLOG.md`** : arc **SP-3.n — Event Manager / Control / Dialog (modèle
  GeoWorks)** gravé. UI déclarative (tables `DB_*`/GenUI), MainLoop → messages
  (retrait callbacks cross-bank), `SYS_DO_DLGBOX`/`SYS_ALERT` table-driven,
  principe GenUI/SpecUI. Découpage G.1→G.7, ~5-6 syscalls (slots `$15-$3F`),
  ADR « modèle GUI déclaratif » à instruire (révise ADR-06). Référence d'art :
  `docs/REFERENCES_ART.md`.

## [2026-05-26] — Doc : note d'architecture des références d'art

### Docs (oric2)
- **`docs/REFERENCES_ART.md`** : note d'architecture « qui inspire quel étage » —
  SymbOS (noyau préemptif à messages), Apple IIgs (mécaniques 65C816 & GrafPort),
  GeoWorks/GEOS (UI déclarative GenUI/SpecUI à messages). Comparaison par axe,
  ce qu'OricOS prend/ne prend pas à chacune, convergence « message », table de
  traçabilité étage→ADR. Informatif (ne ratifie rien).

## [2026-05-25] — SP-3.m G.6 : app C démo fenêtrée (arc SP-3.m clos)

### OricOS
- **`apps/win_hello/win.c`** : première app userland C **fenêtrée** (llvm-mos).
  Crée sa fenêtre (focus), dessine en local, flush, lit le clavier, sort.
- **SDK** : helpers `oricos_win_create`/`oricos_gfx_fill_rect`/`oricos_win_flush`.
- **`SYS_WIN_FLUSH` ($14)** → `kernel_wm_compose` ; `sys_win_create` donne le focus.

### Phosphoric
- **`test_oricos_win_app`** : valide la chaîne GUI×multitâche complète depuis C —
  fenêtre+focus (G.2), dessin local `$080000==$FF` (G.4), composite `$00A032==$FF`
  (G.4bis), clavier au focus → "win_hello: sortie" (G.3), exit → fenêtre fermée
  `WM_COUNT 3→2` (G.5). **Arc SP-3.m clos. 564 tests verts.**

## [2026-05-25] — SP-3.m G.4bis : compositor (backing stores → framebuffer XVGA)

### OricOS
- **`kernel_wm_compose`** : BLITe les backing stores des fenêtres `($06+slot):0000`
  vers le framebuffer XVGA à `dst=y*512+x/2` (stride 512). Le dessin local (G.4)
  apparaît à l'écran → indépendance app ↔ adresse XVGA bouclée (modèle GrafPort).

### Phosphoric
- **`test_oricos_win_draw`** étendu G.4bis : vérifie backing store `$080000==$FF`
  ET framebuffer `$00A032==$FF`. Couleur fenêtre 15 ($FF) ≠ fond desktop ($44) →
  preuve de compositing non-ambiguë. 563 verts.

## [2026-05-25] — SP-3.m G.4 : dessin fenêtré (backing store, coords locales)

### OricOS
- **`kernel_gfx_window_base`** : les `sys_gfx_*` posent `GFX_BASE` = backing store
  de la fenêtre du caller (`($06+slot):0000`) → une app dessine en coords LOCALES
  dans son backing store, **indépendamment de l'adresse XVGA** (modèle GrafPort).
- Validé : `test_oricos_win_draw` — task_wdraw FILL_RECT local → `vram_peek($080000)==$44`.
  563 verts. Suite : G.4bis compositor (BLIT backing stores → XVGA).

## [2026-05-25] — SP-3.m G.3 : clavier → focus (routage)

### OricOS
- **`kernel_kbd_waiter_eligible`** : le clavier va au propriétaire de la fenêtre
  focus (`WM_FOCUS`→`WM_OWNER`) ; tâches sans fenêtre exemptes (préserve task_e) ;
  tâche GUI non-focus retient la touche. `kernel_wm_set_focus` réévalue. Validé
  non-régression (563 verts, task_e exempt OK) ; branche focus → intégration G.6.
  Limite : KBD_WAITER unique (→ polish #1 signaux génériques).

## [2026-05-25] — SP-3.m G.5 : exit → close (fin de v1.a GUI×multitâche)

### OricOS
- **`kernel_wm_close_owner`** appelé par `sys_exit` : la fenêtre d'une tâche se
  ferme automatiquement à sa sortie. task_win crée+sort → fenêtre fermée. Validé :
  handle==2 (créée), puis WM_COUNT==2 + WM_OWNER[2]==0 (fermée). 563 verts.
  **v1.a SP-3.m complet** (fenêtre liée à la tâche : ouvre/ferme). Suite v1.b :
  clavier→focus (G.3), dessin fenêtré (G.4), compositor (G.4bis).

## [2026-05-25] — SP-3.m G.2 : SYS_WIN_CREATE (une app ouvre sa fenêtre)

### OricOS
- **`SYS_WIN_CREATE`** (syscall $13) : une tâche crée sa fenêtre (args ZP $D0-$D7),
  owner = tâche appelante. Backing store SDRAM implicite par slot (($06+slot):0000).
  Validé : task_win obtient handle 2, WM_OWNER[2]==8, WM_COUNT==3. 563 verts.

## [2026-05-25] — SP-3.m G.1 : lien fenêtre↔tâche (WM_OWNER)

### OricOS
- **WM_OWNER** ($015BCD) : pid propriétaire par slot fenêtre ; `kernel_wm_add`
  enregistre `WM_OWNER[id]=TASK_CUR`. Fondation GUI×multitâche (modèle
  backing-store/GrafPort, SP-3.m). Validé : WM_OWNER[0]==1 (fenêtre démo liée à
  task_a). 563 verts.

## [2026-05-25] — OS-2.g v2.b : SYS_SLEEP_MS (sleep bloquant piloté par le timer)

### OricOS
- **`sys_sleep_ms`** : de stub à blocage réel (SLEEP_TICKS[CUR] + block_switch).
  **`kernel_sleep_tick`** (IRQ T1) décrémente et réveille à 0. 2ᵉ source de réveil
  (timer) après le clavier → modèle block/wake (ADR-25) général. task_f dormeuse.

### Phosphoric
- via_t1 : TASK_F_CTR>0 (endormie puis réveillée par le timer), bitmap=$CF. 563 verts.

## [2026-05-25] — OS-2.g v2.b : apps userland comme tâches schedulées

### OricOS
- **`kernel_app_spawn`** (fat.s) : charge un bundle et le lance comme **tâche
  préemptive** (task_create, entry crt0 BANK:$0200) au lieu du JSL boot-context.
  `kernel_app_load` factorisé (front commun avec app_exec legacy). `TC_HELLOC_TASK_FLAG`.
  Bug corrigé : spawn placé après l'init de l'allocateur de banks.

### Phosphoric
- **`test_oricos_helloc_as_task`** : hello_c spawné comme tâche, imprime
  « Hello OricOS from C! » dans $BB80 → **une app C llvm-mos tourne comme tâche
  préemptive schedulée**. 3/3 helloc + 563 verts.

## [2026-05-25] — OS-2.g v2.b : idle task (ferme le trou « dernière tâche »)

### OricOS
- **idle task** : `find_next` borné + saut/fallback `IDLE_PID` → plus de hang si
  toutes les tâches bloquent/sortent. `idle_entry` (WAI, plus basse prio). Validé :
  IDLE_CTR==0 (jamais élue tant que des tâches réelles tournent), bitmap=$4F. 563 verts.

## [2026-05-25] — OS-2.g v2.b/g.5 : blocage réel + ADR-25 RATIFIÉE

### OricOS
- **g.5 `sys_read_char` bloquant** : la tâche passe BLOCKED et rend le CPU
  (`kernel_block_switch`), réveillée par l'IRQ KBD2 (`kernel_kbd_wake` +
  `KBD_WAITER`). Gate `SCHED_ACTIVE` : fallback spin/WAI en contexte boot
  (hello_c). Validé : task_e bloque, autre tâche tourne, 'K' injectée via IRQ →
  task_e lit 'K' + exit. 563 verts. **Fin du spin masqué (deadlock).**

### ADR
- **ADR-25 (modèle de concurrence kernel) RATIFIÉE** (Exec-classique). L'humain a
  approuvé ; 3 conditions du moratoire remplies (dossier + impl g.5/g.6 testée +
  cohérence). `docs/adr/0025-modele-concurrence-kernel.md` (DRAFT retiré),
  CLAUDE.md §2/§3, README index mis à jour. Polish v2.b restant : signaux
  multi-bits, Disable/Enable formels.

## [2026-05-25] — OS-2.g v2.b/g.6 : Forbid/Permit (ADR-25 Exec-classique 1/2)

### OricOS
- **g.6 `kernel_forbid`/`permit`** : compteur FORBID_COUNT ; le COP dispatcher
  forbid à l'entrée / permit à la sortie → syscall non-préemptible (corrige la
  réentrance ZP #2), IRQ actives (pas de deadlock). do_switch skip si FORBID≠0 ;
  yield/exit font permit avant de basculer. Bug A-clobbé détecté+corrigé par les
  tests. 563 verts. Fondation Exec-classique ; g.5 (blocage read_char) = inc. 2.

## [2026-05-25] — OS-2.g v2.a : SYS_EXIT teardown (g.4)

### OricOS
- **g.4 `sys_exit`** (wm.s) : teardown réel (STATE=DEAD + `kernel_bitmap_clear`
  + reschedule) au lieu de STP. Garde-fou `SCHED_ACTIVE` : STP si scheduler
  inactif (app boot-context comme hello_c → test intact). task_d éphémère valide :
  TASK_D_CTR==1 + bitmap=$0F. 563 verts. Limites : fuite page pile, pas d'idle
  task, exit_code ignoré (reportés). Reste : g.5/g.6 block/wake (= ADR-25).

## [2026-05-25] — OS-2.g v2.a : task_create (g.3) + SYS_YIELD réel (g.7)

### OricOS
- **g.3 `kernel_task_create`** (sched.s) : scan bitmap → slot libre, alloc page
  de pile bank 0, init TCB + forge frame d'IRQ initiale. boot crée task_c (pid 3).
  Validé : TASK_C_CTR>0 + bitmap=$0F (frame forgée correcte, RR élit pid 3).
- **g.7 `sys_yield`** (wm.s) : yield coopératif réel (chirurgie de pile →
  frame do_switch → jmp). task_c yield à chaque tour ; système sain → validé.
- 563 tests verts. Reste : g.4 (SYS_EXIT teardown), g.5/g.6 (block/wake = ADR-25).

## [2026-05-25] — OS-2.g v2.a : scheduler N-tâches round-robin (ADR-14)

### OricOS
- **`modules/sched.s`** (nouveau) : `kernel_tcb_ptr` (pid→&tcb 24-bit, index
  (pid-1)*20) + `kernel_sched_find_next` (round-robin READY, wrap, skip).
- **`do_switch`** (handlers.s) : remplace le swap figé 2-tâches par un
  round-robin table-driven (helpers appelés avant le `tcs`). Comportement 1↔2
  préservé avec 2 tâches → 563 tests verts. Implémente ADR-14 (ne présuppose pas
  ADR-25). Reste v2.b : task_create/destroy + block/wake (Forbid/Disable).

## [2026-05-25] — ADR-25 (DRAFT) : dossier modèle de concurrence kernel

### Docs / ADR
- **`docs/adr/0025-modele-concurrence-kernel-DRAFT.md`** : dossier d'instruction
  (moratoire §10 cond. 1) pour le modèle de concurrence du kernel préemptif.
  Recommandation : Exec-classique (`Forbid`/`Disable` + signaux `Wait`/`Signal`,
  réf. AmigaOS Exec / SymbOS, sourcé). Alternatives chiffrées : mutexes (v3),
  messages (v3+), statu quo (écarté), GS/OS coopératif (écarté). **NON ratifié** :
  moratoire cond. 2 non remplie (impl OS-2.g v2 = 0 %).
- `CLAUDE.md` §3 : entrée ADR-25 (ouverte/DRAFT). `docs/adr/README.md` : index MAJ.

## [2026-05-25] — Dette #4 : gardes d'overlap memory map bank 1

### OricOS
- **Pivot après investigation** : migrer les 126 constantes absolues vers `.res`
  casserait l'ABI d'introspection des tests (205 réfs littérales / 7 fichiers).
  À la place : 11 gardes `.assert` (`kernel.s`) → overlap = erreur de build, sans
  bouger une seule adresse. Validé négatif (`WM_MAX=20` → build échoue). 563 verts.

## [2026-05-25] — Réduction dette/bugs (analyse fine OricOS)

### OricOS
- **Fix P1 race ring clavier** (`kbd.s`) : section critique `php;sei…plp` dans
  `kernel_kbd_ring_pop`. Régression du `cli` (deadlock fix) : l'IRQ producteur
  pouvait préempter le pop → RMW perdu sur COUNT + clobber DP_KBD_TMP. Masquage
  bref rend le pop atomique (producteur = IRQ uniquement).
- **Dette P3 largeur M/X** (`wm.s`) : `.a8`/`.i8` en tête des handlers syscall
  (`.smart` ne traverse pas `jsr (table,X)`). Prévient le bug ca65 tracking mode.
- 563 tests verts. Reste (gros rayon) : memory map → `.res`, scheduler ADR-14 (OS-2.g v2).

## [2026-05-25] — oricos.h SSOT numéros de syscalls (revue senior P2)

### OricOS
- **`oricos.h`** : numéros de syscalls stringifiés depuis les `#define SYS_*`
  (`_ORICOS_LDA_SYS`) au lieu de littéraux dupliqués. Conserve l'anti-LTO
  (LDA #imm) et restaure la source unique de vérité. 563 tests verts.

## [2026-05-25] — Fix deadlock SYS_READ_CHAR (revue senior P1)

### OricOS
- **`handlers.s`** : `cli` dans `kernel_cop_handler`. Le COP entre I=1 ; un syscall
  bloquant (`SYS_READ_CHAR`) figeait tout le noyau car l'IRQ KBD2 ne pouvait plus
  remplir le ring → deadlock garanti en usage réel. Conforme ADR-03.
- **`wm.s`** : `WAI` dans `sys_read_char` (dort jusqu'à l'IRQ au lieu de busy-spin).
- **`boot.s`** : suppression du hack de pré-injection clavier (pansement du deadlock).

### Phosphoric
- **`test_oricos_helloc` renforcé** : ne pré-injecte plus ; livre 'A' via le device
  KBD2 quand l'app bloque (`cpu.waiting`), exerçant la chaîne KBD2→IRQ→ring→read_char.
  563 tests verts.

## [2026-05-25] — TC-poc-hello-c : durcissement post-revue senior

### OricOS
- **Repro build (P0)** : `apps/hello_c/Makefile` `-isystem`→`-I` (le SDK versionné
  devient autoritaire vs le `oricos.h` plateforme hors repo qui le shadowait).
- **Repro build (P0)** : `Makefile` — `KERNEL_DEPS = wildcard modules/*.s` ajouté
  aux prérequis de `kernel.o` (éditer un module rebuild le kernel ; fini le
  kernel obsolète testé en silence).
- **Fix racine DBR** : driver console (`console.s`) écrit l'écran bank 0 en
  adressage long (`STA [DP_PCPTR]` + `f:` dans scroll) au lieu de DBR-relatif.
  Corrige `SYS_PRINT_CHAR` ET `SYS_PRINT_STRING` (cassé pour DBR≠0). `phb/plb`
  redondant retiré de `sys_print_char` (wm.s).

### Phosphoric
- Aucune modif source. 563 tests verts (validation des fixes OricOS ci-dessus).

## [2026-05-25] — TC-poc-hello-c : première app C llvm-mos sous OricOS

### OricOS
- **`oricos.h` fix LTO** : syscall literals `"lda #N\n"` au lieu de contrainte `"i"`
  (évite hoisting LTO en LDA ZP qui écrasait ZP kernel).
- **`sys_print_char` fix DBR** : sauvegarde/restaure DBR pour que `kernel_print_char`
  écrive bien en bank0:$BB80 (les apps userland ont DBR≠0).
- **`kernel_app_exec` v0.2** : boucle de copie 16-bit (Y 16-bit) ; supporte bundles >255B.
- Boot TC-poc : `TC_HELLOC_FLAG`, pré-injection kbd 'A', `bundle_hello_c` embarqué.

### Phosphoric
- Test `test_oricos_helloc` : 2/2 PASS. 563 tests verts.

## [2026-05-25] — SP-3.k : icônes desktop

### OricOS
- **SP-3.k — Icônes desktop** : `ICON_TABLE` (4×16B, `$015ADA`), `kernel_icon_add`,
  `kernel_icon_draw_all` (FILL_RECT16 32×32 + TEXT16 label), `_icon_hit` (hit-test),
  callback via `jsr (WM_DP_TMP,X)`. Intégré dans redraw (après clear, avant fenêtres)
  et dans mouse_step (clic vide → _icon_hit → ICON_SELECTED). 2 icônes démo au boot.

### Phosphoric
- Tests `test_wm_icons_init`, `test_wm_icon_hit`, `test_wm_icon_click`. 563 verts.

---

## [2026-05-24] — SP-3.j : dialog modal

### OricOS
- **SP-3.j — Modal** : `WM_MODAL` (`$015AD5`), `kernel_wm_set/clear_modal`,
  auto-clear dans `kernel_wm_close`, blocage clics hors modal. 3 tests. 560 verts.

### Phosphoric
- Tests `test_wm_modal_init/block/close_clears`.

---

## [2026-05-24] — SP-3.i : resize fenêtres par les bords (droit + bas)

### OricOS
- **SP-3.i — Resize fenêtres** : hit-test `_wm_resize_hit` (MARGIN=6 px),
  `_wm_do_resize` (DX/DY → w/h, clamp min 60×40), intégré dans `kernel_wm_mouse_step`.
  `WM_RESIZE_ARMED` + `WM_RESIZE_EDGE` (`$015ACE-$015ACF`), init dans `kernel_wm_init`.
  3 nouveaux tests. 557 tests verts.

### Phosphoric
- Tests `test_wm_resize_init/right_edge/bottom_edge` dans `test_oricos_boot.c`.

---

## [2026-05-24] — SP-3.h : maximize/minimize fenêtres + fix critique chrome

### OricOS
- **SP-3.h — Maximize/minimize dans le chrome** :
  `kernel_wm_maximize` bascule normale↔maximisée (XVGA plein : x=0, y=14,
  w=1024, h=741). Sauvegarde coords dans `WM_SAVED_RECTS` (`$015AA9`, 4×8B)
  via `STA/LDA f:WM_SAVED_RECTS,X` (opcode long,X `$9F`/`$BF`).
  `kernel_wm_minimize` : clear `WM_F_VISIBLE` + `WM_STATE_HIDDEN`.
  Restore depuis taskbar au clic. Drag désactivé sur fenêtre maximisée.
  `_wm_chrome_hit` : hit-test 3 zones chrome (×/□/_).
- **Fix critique `_wm_chrome_hit`** : instructions `sbc #12` dans les labels
  `_crh_test_max` et `_crh_test_min` assemblées en 8-bit par ca65 (tracking
  mode perdu après `sep #$20` d'une branche adjacente). L'opcode `E9 0C`
  tronqué corrompait le ZP $22-$24 (`WM_ARG_TITLE_LO/HI`/`WIN_SLOT`),
  provoquant 4 régressions silencieuses (drag, focus, widgets). Fix : `rep #$20`
  explicite en tête des deux labels.
- Nouvelles constantes : `WM_STATES` ($015AA5), `WM_SAVED_RECTS` ($015AA9),
  `WM_CRH_TMP` ($25-$2A), `WM_STATE_NORMAL/MAXED/HIDDEN`, `BTN_MAX/MIN_OFFSET`.

### Phosphoric
- **v1.22.52-alpha** — 3 nouveaux tests SP-3.h : `test_wm_states_init`,
  `test_wm_maximize` (click □ → `WM_STATES[0]=$01`, `w=1024`),
  `test_wm_minimize_restore` (click _ → HIDDEN, click taskbar → NORMAL).
  554 tests verts (+ 4 régressions SP-3.e/f/g corrigées par le fix ca65).

## [2026-05-24] — SP-3.g : taskbar liste fenêtres + focus au clic

### OricOS
- **SP-3.g — Taskbar fixe bas desktop** : `kernel_taskbar_draw` dessine
  fond darkgray `(0,755,1024,13)` + séparateur blanc `y=755` + boutons par
  fenêtre `WM_F_USED` (lightblue si focus, darkgray sinon). Texte = titre
  SDRAM ou fallback `"WinN\0"` uploadé en SDRAM `$011100`.
- **`kernel_taskbar_hit`** : hit `MOUSE_Y≥755` + BTN_LEFT → `slot=(X-4)/124`
  → `kernel_wm_set_focus(slot)` + `kernel_wm_redraw` + curseur. Priorité
  absolue dans `wm_step_not_drag`, rendu final après `kernel_menu_draw`.
- Constantes ajoutées : `TB_BTN_STRIDE=124`, `TB_WIN_SCRATCH=$015AA0`,
  `TB_WIN_SDRAM=$011100`.

### Phosphoric
- **v1.22.51-alpha** — 2 nouveaux tests : `test_taskbar_render` (pixels
  VRAM fond $88 / bouton focus $99 / séparateur $FF) et `test_taskbar_click`
  (clic `(60,760)` → `WM_FOCUS=0`). **551 tests verts**.

## [2026-05-24] — SP-3.f : chrome de fenêtre (titre + bouton fermer)

### OricOS
- **SP-3.f v0.1 — Titre titlebar** : `kernel_wm_add` uploade le titre en SDRAM
  (`$012000+slot×$100`), `WM_TITLES[slot]=$01`, rendu TEXT16 blanc en titlebar.
- **SP-3.f v0.2 — Bouton fermer** : "X" lightred, zone hit `[win_x+w-12..w-1,y..+13]`,
  `kernel_wm_close` : efface slot, décrémente `WM_COUNT`. Fenêtres "OricOS"/"Editor".
- **GFX_STR_HI** corrigé (`$00→$01` pour bank `$011080`).

### Phosphoric
- **2 nouveaux tests** : `test_wm_window_title` (SDRAM $012000 = "OricOS\0"),
  `test_wm_close_button` (clic zone close → WM_COUNT=1 + flags0 cleared).
- Timing tests menu élargis (160K/175K/200K) pour absorber TEXT16 close button.
- **549 tests verts**. EMU_VERSION → `1.22.50-alpha`.

---

## [2026-05-24] — OS-2.f.v2 clos : table dispatch syscall ADR-17

### Phosphoric
- **OS-2.f.v2** : 3 nouveaux tests d'intégration dans `test_oricos_boot.c` :
  `test_syscall_dispatch_invalid` (cop_invalid → LOG_WARN + A=$FF),
  `test_syscall_yield` (SYS_YIELD no-op, scheduler tourne),
  `test_syscall_table_size` (64 entrées × 2B à `$01:5750`, struct vérifiée).
- **547 tests verts**. EMU_VERSION → `1.22.49-alpha`.

### OricOS
- OS-2.f.v2 clos (implémentation déjà en production dans `kernel.s`).
  Dispatcher `kernel_cop_handler` bank1 $5700, `syscall_table` $5750, 18
  syscalls câblés + 45 × `sys_invalid`. CHANGELOG.md mis à jour.

### Workspace
- BACKLOG.md : OS-2.f.v2 → `~~OS-2.f.v2~~ ✅ clos 2026-05-24`.

---

## [2026-05-24] — PH-2.c/d ADR-18 : suppression effective cœur 6502

### Phosphoric
- **ADR-18 4/6** : suppression effective du cœur 6502. Fichiers supprimés :
  `cpu6502.c` (166 L), `opcodes.c` (572 L), `addressing.c` (113 L),
  `cpu6502.h`, `cpu_internal.h`, `test_cpu.c` (1106 L). ~10 K LOC retirées.
- Migrations de 7 fichiers de tests + 4 fichiers sources vers `cpu65c816_t` mode E.
- `trace_log_instruction(cpu6502_t*)` retirée de `trace.h`/`trace.c`.
- `docs/adr/0018-retrait-6502.md` créé (MADR ADR-18).
- Go/no-go satisfait : **544 tests verts**, boot ROM Oric 1.0 OK, bench ≤ 5 %.
- EMU_VERSION → `1.22.48-alpha`.

## [2026-05-24] — CI GitHub Actions (recalage NOW/P0, programme Phase 1)

- **CI mise en place** sur Phosphoric (build + 570 tests à chaque push/PR, checkout
  OricOS sibling + cc65) et OricOS (build kernel). Garde-fou anti-régression.
- Note : finir ADR-18 (retrait 6502) reste à faire — plus gros que prévu
  (savestate + désassembleur debugger couplés à `cpu6502_t`, pas de désassembleur 816).

## [2026-05-24] — Licence : passage à EUPL-1.2

- Tout le projet (workspace `oric2`, Phosphoric, OricOS) passe sous **EUPL-1.2**
  (European Union Public Licence) © 2026 Bénédicte Marty. Fichier `LICENSE` (texte
  officiel EUPL-1.2) ajouté aux 3 dépôts ; Phosphoric quitte MIT. READMEs + badges
  mis à jour. Exception : ROM Oric 1 (`roms/`) = propriété Tangerine/Oric, hors licence.
- En-têtes **SPDX-License-Identifier: EUPL-1.2** ajoutés à tous les fichiers source (135).

## [2026-05-24] — SP-3.d v0.6 : barre de menu multi (table-driven)

### OricOS
- Barre de menu **table-driven** (`menu_defs`, N menus) : "System" + "View",
  chacun 2 items à callbacks. `MENU_OPEN` = index du menu ouvert ($FF=fermé).

### Phosphoric → 1.22.44-alpha
- Test `menu_second` (2e menu "View"). 570 tests.

## [2026-05-24] — SP-3.d v0.5 : barre de menu déroulant

### OricOS
- **Barre de menu** (`kernel_menu_draw`) en haut de l'écran + menu "System"
  déroulant (items "About"/"Clear"). `kernel_menu_handle_click` ouvre/ferme et
  invoque le callback de l'item. Intercepte le clic avant le window manager.

### Phosphoric → 1.22.43-alpha
- Test `menu_dropdown` (titre→ouvre, item→callback+ferme). 569 tests.

## [2026-05-24] — SP-3.d v0.4 : callbacks de bouton (action au clic)

### OricOS
- **Callbacks de bouton** : chaque widget bouton porte une adresse de callback
  (bank1). Au clic, `_wm_invoke_active_cb` l'appelle via `jsr (vec,X)` ($FC).
  Démo `demo_ok_cb` incrémente un compteur. Les clics font désormais des actions.

### Phosphoric → 1.22.42-alpha
- `tk_button_press` vérifie aussi l'invocation du callback (CB_FLAG). 568 tests.

## [2026-05-24] — SP-3.d v0.3 : bouton cliquable (retour visuel)

### OricOS
- **`_wm_widget_hit`** : hit-test des boutons sous le curseur → `WIDGET_ACTIVE`.
  Le bouton cliqué est dessiné **pressé** (face darkgray vs lightgray). Appelé
  par `kernel_wm_mouse_step` après le focus.

### Phosphoric → 1.22.41-alpha
- Test `tk_button_press` (clic → WIDGET_ACTIVE + face pressée). 568 tests.

## [2026-05-24] — SP-3.d v0.2 : widgets managés (attachés aux fenêtres)

### OricOS
- **Widgets managés** : table de widgets + `kernel_wm_add_widget` +
  `_wm_draw_all_widgets` (hook après `_wm_draw_windows`). Les widgets sont
  attachés à une fenêtre, dessinés à (window.xy + offset relatif) → **persistent
  et suivent leur fenêtre au drag** (corrige la démo flottante v0.1 qui disparaissait).

### Phosphoric → 1.22.40-alpha
- Tests `tk_widgets` (widgets sur fenêtre) + `tk_widget_follows_drag`. Fix harness :
  vram+gpu câblés dans mouse_irq_focus/wm_drag_persistent (évite corruption VIA T1
  par les écritures GPU/VRAM du dessin des widgets). 567 tests.

## [2026-05-24] — SP-3.d v0.1 : toolkit (label / frame / button)

### Phosphoric → 1.22.39-alpha
- **GPU TEXT16** (opcode $07, ADR-21) : texte coords 16-bit packées
  (ARG4 = color<<20|y<<10|x) — texte au-delà de x/y=255 sur XVGA.

### OricOS
- **Toolkit** : `kernel_tk_label` (TEXT16 + upload string SDRAM), `kernel_tk_frame`
  (cadre 2px), `kernel_tk_button` (face+cadre+label). Fonte ASCII uploadée en
  SDRAM $010000 au boot. Démo : label + bouton "OK" à x=400.

### Test
- `test_oricos_tk_widgets`. 566 tests.

## [2026-05-24] — SP-3.e v0.8 : couleur titlebar selon focus

### OricOS
- **Titlebar colorée selon le focus** : fenêtre active = lightblue (9), inactive
  = darkgray (8), dans `_wm_draw_windows` (via `WM_TITLE_COL`). Aucun nouveau
  redraw : clic (full-redraw) et drag (redraw_drag) passent déjà par cette boucle.
- Multi-dirty-rect jugé inutile : un changement de focus déclenche déjà un
  full-redraw qui repeint correctement les 2 titlebars concernées.

### Phosphoric → 1.22.38-alpha
- Test `test_oricos_wm_titlebar_focus`. 565 tests.

## [2026-05-24] — Capture souris XVGA fiable (Phosphoric 1.22.37-alpha)

- Refonte du modèle de capture : démarrage non capturé, **clic = capture ON**
  (pointeur garanti au-dessus → grab confiné fiable), **LCtrl+RShift = relâche**.
  Device MOU2 nourri uniquement si capturé → plus de double curseur (hôte + OS).
  Corrige : double curseur au démarrage, grab clavier non fiable hors fenêtre.

## [2026-05-24] — SP-3.e v0.7 : drag fenêtre incrémental

### OricOS
- **Drag incrémental** : `kernel_wm_redraw_drag` efface seulement l'ancien rect
  de la fenêtre (dirty rect) au lieu du `kernel_gfx_clear` plein écran 393 Ko,
  puis redessine les fenêtres. Drag fluide.

### Phosphoric → 1.22.36-alpha
- Test `test_oricos_wm_drag_no_ghost` : ancienne position effacée (bleu, pas de
  fantôme) + fenêtre à la nouvelle position. 564 tests.

### v0.8 reporté
- Couleur titlebar selon focus + multi-dirty-rect (plusieurs zones sales).

## [2026-05-24] — SP-3.e v0.6 : backing-store curseur

### OricOS
- **Backing-store curseur** : `kernel_wm_cursor_blit` (motion) sauve/restaure la
  zone 8×8 sous le curseur via VRAM I/O — plus de full-redraw (393 Ko) par
  mouvement. Full-redraw conservé sur clic-focus/drag. Curseur fluide.

### Phosphoric → 1.22.35-alpha
- **Fix latent** : masque reg VRAM I/O `$0330-$033F` (`& 0x0F` → `& 0x3F`) —
  les ports VRAM ne fonctionnaient pas (exposé par le backing-store).
- Test `test_oricos_cursor_backing_store` (pas de traînée). 563 tests.

### v0.7 reporté
- Backing-store fenêtre (drag sans full-redraw) + dirty rects.

## [2026-05-24] — SP-3.e v0.5 : capture souris robuste

### Phosphoric → 1.22.34-alpha
- **Fix capture non-déterministe** : les compositeurs (Wayland/X11) ignorent le
  grab si la fenêtre n'a pas le focus pointeur au moment de l'appel. La capture
  est réaffirmée sur `SDL_WINDOWEVENT_FOCUS_GAINED`/`ENTER`, pilotée par un état
  voulu `mouse_capture_wanted`. Validé : curseur hôte capturé/masqué de façon stable.

## [2026-05-24] — SP-3.e v0.5 fix : capture souris + drag borné

### Phosphoric → 1.22.33-alpha
- **Fix capture souris** : `SDL_SetRelativeMouseMode` était annulé par le resize
  de la fenêtre en 1024×768 (posé trop tôt sur la 240×224). (Ré)activé après le
  passage XVGA + `SDL_ShowCursor(DISABLE)`. Curseur hôte désormais capturé/confiné.

### OricOS
- **Fix drag fenêtre** : `WM_DRAG_ARMED` — le drag n'est armé que si le clic a
  atterri sur une fenêtre. Avant, clic sur le vide + glissé déplaçait quand même
  la fenêtre focus.

## [2026-05-24] — SP-3.e v0.5 : relative-mode souris + curseur dessiné

### Phosphoric → 1.22.32-alpha
- **Relative-mode SDL** en `--xvga` : pointeur capturé/confiné, curseur hôte
  masqué, deltas relatifs purs. Bascule via **LCtrl+RShift**.

### OricOS → SP-3.e v0.5
- **`kernel_wm_draw_cursor`** : curseur 6×8 blanc à (MOUSE_X,Y) via FILL_RECT16.
  `wm_mouse_step` redessine desktop + curseur sur tout événement → le curseur
  suit la souris. Curseur initial au boot.

### v0.6 reporté
- Backing-store DMA + redraw incrémental (dirty rects) au lieu du full-clear
  par événement ; couleur titlebar focus.

## [2026-05-24] — SP-3.e v0.4 : main loop persistant + drag fenêtre live

### OricOS → SP-3.e v0.4
- **Mode persistant** : scheduler ne STP plus si `NO_STP_FLAG` ($A5, $01EF00)
  posé par `--kernel` → GUI interactive. Tests gardent le STP (flag non posé).
- **`MOUSE_DX/DY`** : delta par événement (mouse_read lit+clear MOU2 DX/DY) →
  drag propre. `wm_mouse_step` drague la fenêtre focus → suit la souris.

### Phosphoric → 1.22.31-alpha
- `--kernel` pose le flag persistant. Test `test_oricos_wm_drag_persistent`
  (clic→focus, drag→fenêtre (100,100)→(140,130)). 562 verts.

### SP-3.e bouclé (v0.1→v0.4)
- Souris ADR-24 + window manager + event loop IRQ + coords 16-bit + redraw +
  drag live. v0.5 (optim) : backing-store DMA + redraw incrémental.

## [2026-05-24] — SP-3.e v0.3 : coords GPU 16-bit + redraw window manager

### Phosphoric → 1.22.30-alpha
- **GPU `FILL_RECT16`** (opcode $06, ADR-21 v0.2) : coords 16-bit packed 12-bit
  (couvre XVGA). gpu_fill_rect_impl partagé. 1 test.

### OricOS → SP-3.e v0.3
- `kernel_gfx_fill_rect16` (packing 12-bit) + `kernel_wm_redraw` (clear + fenêtres
  aux positions 16-bit, peinture back-to-front, base SDRAM $100000). Appelé au
  boot + sur clic/drag. Desktop XVGA avec fenêtres plein écran **visible**
  (--xvga / --xvga-screenshot).

### v0.4 reporté
- Backing-store DMA + redraw incrémental + drag interactif continu (main loop kernel).

## [2026-05-24] — Déblocage rendu XVGA : desktop OricOS visible

### Phosphoric → 1.22.29-alpha
- **vram_device + gpu_device câblés dans l'émulateur vivant** ($0330/$0340) :
  les commandes VRAM/GPU du kernel s'exécutent en `--kernel` (avant : tests
  standalone seulement).
- **`src/video/hires_oric2_xvga.{c,h}`** : rendu framebuffer XVGA 1024×768×4bpp
  (SDRAM) → ARGB via palette VGA-IBM 16 couleurs (ADR-20). 5 tests.
- **`--xvga-screenshot FILE`** (PPM headless) + **`--xvga`** (SDL live) :
  le desktop OricOS (fenêtres dessinées par le GPU au boot) est **visible**.
- 560 tests verts.

### Débloque
- SP-3.e v0.3 (drag live visible) : il reste les coords GPU packed 16-bit
  (ADR-21 v0.2) pour des fenêtres plein écran + le redraw multi-fenêtre.

## [2026-05-24] — SP-3.e v0.2 : event loop souris IRQ-driven

### OricOS → SP-3.e v0.2
- `kernel_irq_handler` traite l'event MOU2 (lit + `kernel_wm_mouse_step`), puis
  **scheduler gaté sur T1** (`VIA_IFR` bit6) — plus de faux tick sur IRQ souris/clavier.
- `kernel_mouse_init`/`read` activent l'IRQ MOU2 → souris **event-driven** (fin du polling).

### Phosphoric → 1.22.28-alpha
- `test_oricos_mouse_irq_focus` : clic injecté pendant la phase scheduler → IRQ
  MOU2 → focus 1→0. Garde I/O harnais étendu `$036F` (KBD2+MOU2). 555 verts.

### v0.3 reporté
- Drag live + backing-store DMA VRAM cold + redraw multi-fenêtre (bloqué :
  affichage XVGA SDL + coords GPU 16-bit).

## [2026-05-24] — SP-3.e v0.1 : window manager + driver souris

### OricOS → SP-3.e v0.1
- **Driver souris MOU2** (polled) : `kernel_mouse_init`/`kernel_mouse_read`
  (MOU2 `$0360-$036F` → `MOUSE_X/Y/BTN`).
- **Window manager** : table 4 fenêtres (bank 1 `$5900`), `kernel_wm_*`
  (init/add/hit_test topmost/set_focus/move_focused) + `kernel_wm_mouse_step`
  (clic→focus, drag delta). Coords 16-bit (espace XVGA).
- Self-tests déterministes (window table + lecture souris injectée).

### Phosphoric → 1.22.27-alpha
- `test_oricos_boot` : routage mouse2 + injection + 10 assertions SP-3.e. 554 verts.

### v0.2 reporté
- Event loop IRQ-driven, drag live + backing-store DMA VRAM cold, redraw multi-fenêtre.

## [2026-05-24] — ADR-24 : contrôleur souris Oric 2 (prérequis SP-3.e)

### Architecture
- **ADR-24 ratifiée** : contrôleur souris Oric 2 natif (`$0360-$036F`, IRQ MOU2),
  **modèle hybride** absolu (clampé XVGA) + deltas read-clear. Pas de
  paravirtualisation guest (Oric 1 sans souris). Révise ADR-16 (ligne souris).
  `docs/adr/0024-souris-oric2.md`. Direction « souris d'abord » pour SP-3.e.

### Phosphoric → 1.22.26-alpha
- **`src/io/mouse2_device.{c,h}`** : position absolue X/Y 10-bit + deltas +
  boutons G/D/M + IRQ `IRQF_MOU2`. Feed SDL (motion relatif + boutons), gated
  `--machine oric2`. 8 tests. 554 tests verts.

### Reste pour SP-3.e
- Driver souris OricOS + event loop GUI (window table, focus, drag,
  backing-store DMA VRAM cold).

## [2026-05-24] — Debug boot --kernel + optim charset MVN

### Debug session (boot --kernel)
- **Trace 65C816** porté (`trace_log_instruction816`, Phosphoric 1.22.25-alpha) —
  le `--trace` était sur le cœur 6502 inactif.
- Conclusion : **pas de bug** — le boot `--kernel` est correct, juste lent
  (~153K cycles avant STP scheduler, dominé par la copie charset). Validé
  visuellement (bannière OricOS + ligne demo "YABZ").

### OricOS → OS-perf
- **`kernel_install_charset`** optimisé en **MVN** (block move) : copie fonte
  1024 octets ~18K → ~2K cycles. Boot STP ~153K → ~143K cycles. `sep #$30`
  (pas `plp`) pour cohérence du tracking `.smart` ca65.

## [2026-05-24] — PH-bootrom : boot ROM Oric 2 propre

### Phosphoric → 1.22.24-alpha
- **`src/oric2_bootrom.{c,h}`** : `oric2_bootrom_load()` construit une vraie boot
  ROM Oric 2 16 KiB (reset `$C000` CLC;XCE;JML kernel ; trampolines IRQ/NMI/COP
  en ROM `$FF00/$FF10/$FF20` ; vecteurs natifs + émulation).
- **`--kernel`** boote via cette ROM au lieu de patcher `mem.rom[]` + stubs RAM
  épars — chemin de boot représentatif du matériel (puce ROM système).
- Test `test_oricos_bootrom_boots` (boot end-to-end via ROM). 546 tests verts.

### Jalon
- Sprint 2 OricOS clos. PH-bootrom clos. Prochaines options (BACKLOG) :
  SP-3.e (event loop GUI), TC-llvmmos (userland C), SP-3.d (toolkit).

## [2026-05-24] — OS-2.i.v2 : modèle d'erreur kernel (log ring buffer)

### OricOS → OS-2.i (clos)
- Log ring buffer 8 entrées (level, code) en bank 1 `$54E0`, circulaire.
  Codes nommés (`ERR_BANK_EXHAUSTED`/`BAD_SYSCALL`/...), niveaux INFO/WARN/
  ERROR/PANIC. `kernel_log_write`/`kernel_log_init`.
- Points câblés : `kernel_panic`, `cop_invalid` (syscall invalide), `alloc_none`
  (pool épuisé). `SYS_PANIC` journalise.

### Phosphoric → 1.22.23-alpha
- `test_oricos_boot` : 3 assertions (COP invalide → log WARN/ERR_BAD_SYSCALL). 545 tests.

### Jalon
- OS-2.i clos. Jalon courant OricOS → **OS-2.j** (FAT32 SD lecture seule).

## [2026-05-24] — ADR-23 : console flux de caractères, backend interchangeable

### Architecture
- **ADR-23 ratifiée** : le console OricOS est un flux de caractères ; le backend
  d'affichage est interchangeable (Oric 1 text `$BB80` en bootstrap → GPU XVGA
  cible, ADR-20/21). **Règle d'or** : l'ABI (`kernel_print_*`/`SYS_PRINT_*`)
  n'expose jamais géométrie/adresse écran/curseur linéaire/attribut Oric 1.
  Borne la dette du backend Oric 1 à « légère ». `docs/adr/0023-console-flux-caracteres.md`.
- Contrainte clé : pas de syscall texte calqué Oric 1, pas d'app userland
  supposant 40×28 avant le backend GPU (fenêtre de risque = Sprint 4).

## [2026-05-23] — OS-2.e.2 : console CR + scroll up

### OricOS → OS-2.e (clos)
- `kernel_print_char` gère CR (`\r`) : retour début de ligne courante.
- `kernel_scroll_up` : scroll écran d'une ligne quand le curseur dépasse le
  bas (lignes 1..27 → 0..26, dernière ligne effacée, INK restauré). Remplace
  le clamp v0.1.

### Phosphoric → 1.22.22-alpha
- `test_oricos_boot` : 4 assertions validant scroll + CR (self-tests kernel
  exécutés avant `clear_screen`, résultats en bank 1). 545 tests verts.

### Jalon
- OS-2.e clos. Jalon courant OricOS → **OS-2.i** (modèle d'erreur kernel).

## [2026-05-23] — OS-2.d : clavier Oric 2 paravirtualisé (ADR-22)

### Architecture
- **ADR-22 ratifiée** : clavier Oric 2 paravirtualisé hybride (modèle double-ULA
  d'ADR-02). Une source physique → contrôleur KBD2 moderne (hôte OricOS,
  FIFO ASCII + IRQ) **+** matrice Oric 1 virtuelle (guest, compat ADR-10).
  `docs/adr/0022-clavier-oric2-paravirt.md`. Révise la ligne clavier d'ADR-16.

### Phosphoric → 1.22.21-alpha
- **`src/io/kbd2_device.{c,h}`** : contrôleur KBD2 (FIFO 16 ASCII host, IRQ
  `IRQF_KBD2`, matrice virtuelle guest). Registres I/O `$0350-$035F`, gated
  `--machine oric2`. 9 tests device + 1 test intégration. 545 tests verts.

### OricOS → OS-2.d
- Driver clavier réécrit IRQ-driven : `kernel_kbd_poll` draine la FIFO KBD2 →
  ring `$5860` ; `SYS_GET_KEY`/`SYS_READ_CHAR` câblés (étaient stubs). Scan
  matriciel VIA/PSG retiré (keymap déplacée côté contrôleur).

### Gouvernance
- Direction (hybride) + ratification anticipée ADR-22 décidées par l'humain
  (exception moratoire condition 2 : jalon OS-2.d ≤ 4 semaines).

## [2026-05-23] — OS-2.f.v2 + opcodes 65C816 $7C/$FC (Absolute Indexed Indirect)

### Phosphoric → 1.22.20-alpha
- **Opcodes `$7C` (`JMP (a,x)`) et `$FC` (`JSR (a,x)`)** ajoutés au cœur 65C816.
  Mode d'adressage Absolute Indexed Indirect : le vecteur indirect est lu dans la
  **bank programme (PBR)**, pas en bank 0 (sémantique WDC). Mode E (NMOS Oric 1) :
  opcodes illégaux → NOP 3 octets (ADR-11).
- 3 tests unitaires ajoutés. 535 tests verts (532 + 3).

### OricOS → OS-2.f.v2
- **COP handler v0.2** : table de dispatch `syscall_table` (`$01:5750`, 64 entrées),
  18 syscalls v1 ADR-17 routés via `jsr (syscall_table,x)`. Sentinelle erreur `A=$FF`.
- 19 handlers `sys_*` (print, fat, gfx, alloc/free bank, panic, exit/yield, stubs clavier).

### Lien inter-projets
- Le dispatch syscall OricOS v0.2 reposait sur l'opcode `$FC` du golden model.
  Celui-ci tombait dans le `default` (no-op) → tous les syscalls COP silencieusement
  inopérants. Corrigé côté Phosphoric, ce qui débloque OS-2.f.v2.

## [2026-05-23] — Milestone B4 : Compositor logiciel + double ULA (ADR-02)

### Phosphoric → 1.22.19-alpha
- **`compositor_render_to_rgb24()`** ajoutée : conversion ARGB8888→RGB888.
- **`emulator_t`** étendu : `compositor_t compositor`, `compositor_fb_t compositor_output`,
  `bool has_compositor`. Activé si `machine == ORIC_MACHINE_ORIC2`.
- **Pipeline render B4** dans `main.c` : host 240×224 bleu OricOS (title bar 24px
  `#285898`, desktop `#182848`), guest 240×200 ULA Oric 1 positionné à y=24,
  composition et écriture dans `video.framebuffer` avant présentation SDL2.
- 532 tests verts (le golden test valide l'ULA seule, non affecté par le compositor).

---

## [2026-05-23] — Milestone B3 : démonstrateur bascule mode E↔N + guest Oric 1

### OricOS — kernel B3 demo
- Bannière "OricOS B3 Demo / CPU : 65C816 MODE N / MEM : 256KiB (BK0-3)".
- Bloc démo guest : `SEC;XCE` (mode E strict 6502), 32 NOP, `CLC;XCE;SEP #$30`
  (retour mode N). Messages "GUEST: MODE E RUN..." / "GUEST: BACK N OK".
- Jalon B3 validé visuellement sur screenshot headless (240×224 px ULA text mode).

### Phosphoric → 1.22.18-alpha
- Golden frame `tests/golden/oricos_boot.ppm` régénéré (kernel B3).
- `test_oricos_boot.c` + `test_oricos_sd.c` : positions caractères mises à jour
  (+200 bytes, 5 lignes banner B3). Référence banner "OricOS B3 Demo". 532 tests verts.

---

## [2026-05-23] — Corrections analyse senior R1/R2/R5/R6

### Phosphoric → 1.22.17-alpha

**Corrections post-analyse senior :**
- **R1 — ADR-21 CLAUDE.md** : table des 5 commandes GPU corrigée (CLEAR/FILL_RECT/LINE
  passent d'adressage "bank target" à SDRAM 24-bit direct, conformément à ADR-19 v2).
  CLEAR : ARG2=size (octets), ARG3.LO=color. Note de révision v1.1 ajoutée.
- **R2 — gpu_exec_text** : clamp `char_x >= GPU_XVGA_W` et `col` hors [0, GPU_XVGA_W[
  pour prévenir corruption VRAM silencieuse. Nouveaux symboles `GPU_XVGA_W/H` dans le
  header. Test `test_text_clamp_overflow` ajouté.
- **R5 — PEI $D4** : corrigé pour utiliser D+nn (Direct Page) au lieu de la ZP absolue
  en mode N. Conforme WDC W65C816S §A.28. Test `test_pei_uses_direct_page` ajouté.
- **R6 — vram DMA len=0** : `log_warning` émis quand len=0 déclenche un burst de 65536.
- **532 tests** (530 + 2 nouveaux), 0 échec.

---

## [2026-05-09] — Phase 1 PH-2.c.2 sub-1/2/3 : extraction + découplage + migration tests 🧹

### Phosphoric → 1.22.16-alpha (3 sous-commits successifs)

**sub-1 (e3e6384)** — Extraction `opcode_metadata` neutre :
- Nouveau `include/cpu/opcode_metadata.h` + `src/cpu/opcode_metadata.c` :
  contient `opcode_info_t`, `addressing_mode_t` (déplacé de cpu6502.h),
  `opcode_table[256]` (extrait de opcodes.c, ~145 lignes struct littéraux).
- `cpu_internal.h`, `cpu6502.h` : retrait des types déplacés, include
  transitif vers opcode_metadata.h.
- Makefile : `opcode_metadata.c` ajouté à toutes les TEST_*_SRCS.

**sub-2 (1ab1e89)** — Découplage cpu65c816 du cœur 6502 :
- `cpu65c816.h` : include cpu6502.h → cpu_types.h (types neutres uniquement).
- `cpu65c816.c` et `cpu65c816_opcodes.c` : include cpu_internal.h →
  opcode_metadata.h.
- Vérification grep : cpu65c816.c et cpu65c816_opcodes.c n'utilisent
  AUCUNE fonction `cpu_*` du 6502.
- Makefile : `TEST_CPU_CORE_SRCS`, `TEST_CPU816_*_SRCS` débarrassés de
  cpu6502.c, opcodes.c, addressing.c. Cœur 65C816 désormais autonome.

**sub-3 (ef75a3e)** — Migration tests Oric 1 vers 65C816 mode E :
- `test_klaus_dormann.c` : retrait test_6502_core, conservation
  test_65c816_core_e_mode (canari régression mode E ADR-10).
- `test_oric_boot_dual.c` : SUPPRIMÉ (4 tests retirés). Comparaison
  explicite 2 cœurs perd son sens post-suppression 6502. Couverture
  régression mode E préservée par test_klaus + test_cpu65c816_e_mode +
  diff PPM PH-2.b.
- `test_paravirt_demo.c` : Makefile uniquement (utilisait déjà cpu65c816_t).
- Makefile : `TEST_KLAUS_SRCS`, `TEST_PARAVIRT_SRCS` débarrassés des deps
  6502. `TEST_BOOT_DUAL_SRCS` et target supprimés.

### Validation
- Build SDL2 OK.
- 530/530 tests verts (= -5 vs 535 : 1 klaus 6502 + 4 boot_dual retirés).
- 0 FAIL.

### Phase 1 — sprint suivant
- **PH-2.c.2 sub-4** (à risque) : refactor `emulator_t.cpu` (cpu6502_t →
  cpu65c816_t), 69 accès `emu->cpu.X` dans savestate.c (18) + debugger.c
  (30+) + trace.c, profiler.c (15+). Format binaire .ost étendu.
- **PH-2.c.2 sub-5** : suppression effective `cpu6502.c`, `opcodes.c`,
  `addressing.c`, `cpu6502.h`, `cpu_internal.h`, `tests/unit/test_cpu.c`.

---

## [2026-05-09] — Phase 1 PH-2.c.1 : retrait vtable 6502 (ADR-18 étape 1.C, partie 1) 🧹

### Phosphoric → 1.22.13-alpha
- `cpu_core.h/c` : retrait `cpu_core_vtable_6502`, `CPU_KIND_6502`, adaptateurs
  v6502_*. cpu_core.h bascule sur `cpu_types.h` (types neutres).
- `cpu_core_kind_from_string("6502")` : log warning + redirige `CPU_KIND_65C816`
  (rétro-compat CLI `--cpu 6502` ; mode E = 6502 bit-à-bit prouvé PH-2.b).
- `test_cpu_core.c` réécrit : 5 tests sélecteurs + vtable 65C816 (vs 10 avant,
  retrait 8 `matches_direct` 6502 + 3 nouveaux 65c816, net -5 tests).
- 535 tests OK (= -5 vs 540, conforme).

### Découverte / dette PH-2.c.2

Bloc inter-dépendant 6502 ↔ 65C816 plus fort que prévu :
`cpu65c816_opcodes.c → opcode_table[opcodes.c] → cpu_set_flag[cpu6502.c]
→ addressing.c`.

Suppression effective des 3 fichiers cœur 6502 (`cpu6502.c`, `opcodes.c`,
`addressing.c`) reportée à **PH-2.c.2** : extraction préalable de
`opcode_table[256]` vers `src/cpu/opcode_metadata.c` neutre, refactor
`emulator_t.cpu` (69 accès), migration des 3 tests `test_klaus_dormann`,
`test_oric_boot_dual`, `test_paravirt_demo` vers `cpu65c816_t` mode E.

### Phase 1 — sprint suivant
- **PH-2.c.2** (à risque) : suppression effective ~2 K LOC + extractions
  préalables.

---

## [2026-05-09] — Phase 1 PH-2.b : campagne validation 65C816 mode E PASS ✅ (ADR-18 étape 1.B)

### Phosphoric → 1.22.12-alpha
- `src/main.c` : défaut `emu.cpu_kind = CPU_KIND_65C816` (était CPU_KIND_6502).
  Mode E (E=1) reproduit le comportement 6502 strict (cycle-exact + ADR-11
  hybride : bug JMP indirect, opcodes illégaux NMOS = NOP, $EB alias SBC#).
- Tests : 540 OK (aucune régression).

### Validation go/no-go bloquante 1.B → 1.C

| Critère | ROM 1.0 | ROM 1.1 | Tolérance |
|---|---|---|---|
| Diff PPM bit-à-bit (20M cycles) | identique ✓ | identique ✓ | exact |
| Bench overhead 65C816 vs 6502 | +4.6 % | +4.6 % | ≤ 5 % ✓ |
| Suite tests Phosphoric | 540/540 ✓ | (idem) | aucune régression |

**GO pour étape 1.C** : suppression effective cpu6502.c, opcodes.c,
addressing.c, cpu_core_vtable_6502, CPU_KIND_6502, test_cpu.c, réécriture
test_cpu_core.c en test_cpu816_core.c.

### Phase 1 — sprint suivant
- PH-2.c étape 1.C : suppression effective ~2 K LOC.
- PH-2.d étape 1.D : traçage MADR + closure ADR-18.

---

## [2026-05-09] — Phase 1 PH-cleanup-zombie : retrait kernel_hires2_* legacy 🧹

### OricOS → 0.41.0
- `kernel.s` : suppression `kernel_hires2_clear` + `kernel_fill_rect_aligned`
  + `pattern_table` + 14 constantes ZP `HIRES2_*` + appels boot.
  Code mort visuellement depuis ADR-19 v2 (bank `$80` invisible compositor).
  Rendu desktop XVGA = exclusivement GPU blitter via `kernel_gfx_*`.

### Phosphoric → 1.22.11-alpha
- `tests/integration/test_oricos_hires2.c` supprimé (1 test, 20 ASSERTs).
- `Makefile` allégé : retrait `test-oricos-hires2` + `TEST_ORICOS_HIRES2_SRCS`.
- Module `src/video/hires_oric2.c` **conservé** (ADR-12 ULA guest 240×200
  attribute-based, distinct du kernel asm — toujours actif compat Oric 1).

### Validation
- Build OricOS : kernel.bin 57344 bytes.
- Build Phosphoric SDL2 : OK.
- `make tests` : **540 OK** (= -1 vs 541, conforme suppression test_oricos_hires2).

### Phase 1 — sprint suivant
- PH-2.b campagne validation 65C816 mode E par défaut (bloquante 1.B → 1.C).

---

## [2026-05-09] — Phase 1 PH-2.a : décrochage types CPU partagés (ADR-18 étape 1.A) 🧹

### Phosphoric → 1.22.10-alpha
- Création `include/cpu/cpu_types.h` neutre : `memory_t` (forward decl),
  `cpu_flags_t` (FLAG_*), `cpu_irq_source_t` (IRQF_*). Types partagés
  6502 ↔ 65C816, décrochés de `cpu6502.h` selon plan ADR-18 étape 1.A.
- `cpu6502.h` allégé : inclut désormais `cpu_types.h`, conserve les
  définitions cœur (`cpu6502_t`, `addressing_mode_t`, signatures cpu_*).
- Étape additive pure : 0 consommateur à modifier (transitivité d'include).
- Migration effective des consommateurs vers `cpu_types.h` direct prévue
  en étape 1.C (suppression du cœur 6502, post go/no-go 1.B).

### Validation
- 541 tests OK (aucune régression).
- Build SDL2 OK.

### Phase 1 — sprint suivant
- PH-cleanup-zombie (retrait `kernel_hires2_*` legacy ADR-19 v2).
- PH-2.b campagne validation 65C816 mode E par défaut (bloquante 1.B → 1.C).

---

## [2026-05-09] — Programme état-de-l'art Phase 0 close 🏛️

Lancement d'un programme de **remise au top de l'état de l'art** sur 8 semaines, 4 axes parallèles (hygiène d'ingénierie, toolchain & CI moderne, élargissement architectural, process & doc publiable). Phase 0 = décisions bloquantes.

### Décisions cadre
- **ADR-18 modalité** : retrait net du 6502 post-validation (pas de flag LEGACY_6502). Critère go/no-go : 541 tests verts + bench ≤ 5 % + boot interactif ROM 1.0/1.1.
- **CI** : GitHub Actions sur repos publics `benedictemarty/{oric2, oric2-golden-model, OricOS}`.
- **Doc** : site mkdocs Material public sur GitHub Pages.

### ADR ratifiées (3)
- **ADR-16** — Driver model OricOS : hybride event-driven + sync, sans struct ops v1, table dispatch IRQ `$01:5680`, ring buffer kbd 16 keycodes `$01:5860`. Cf. `docs/adr/0016-driver-model.md` et `CLAUDE.md` §2.
- **ADR-17** — ABI syscall userland : 18 syscalls v1, `cop #$AA` + table dispatch `$01:5750`, sentinelle `A=$FF` errno bank 1 `$5760`, versioning par opcode immediate (v2 future = `cop #$AB`). Débloque Sprint 4 userland C. Cf. `docs/adr/0017-abi-syscall-userland.md` et `CLAUDE.md` §2.
- **ADR-18** — Retrait du 6502 dans Phosphoric : retrait net post-validation. DEC-1 actée. Plan d'exécution 4 étapes en Phase 1. Cf. `docs/adr/0018-retrait-6502.md` et `CLAUDE.md` §2.

### ADR parquée v2 (1)
- **ADR-15** — Isolation mémoire post-v1 : parquée v2 avec critères de réouverture explicites (apps non-trusted OU HW-2 mûr OU 2026-12-31). Évite décision prématurée non-instruisable. Cf. `docs/adr/0015-isolation-memoire-post-v1.md` et `CLAUDE.md` §3.

### Décisions stratégiques actées
- **DEC-1** ✅ actée (cf. ADR-18).
- **DEC-2** ✅ actée : HW-1 (contrat HDL ↔ golden model) rédigé en Phase 0/1 (squelette `docs/CONTRACT_HDL.md` créé) ; HDL effectif (HW-2..HW-6, SP-GPU-HDL-1..4) reporté post-programme S9+.
- **DEC-4** ✅ fusionnée avec ADR-15 parquée.

### Discipline d'architecture
- **Moratoire ADR formalisé** dans `CLAUDE.md` §10 : aucune nouvelle ratification sans (1) dossier d'instruction écrit, (2) ≥ 50 % impl, (3) cohérence ADR existantes. Liste blanche : révisions mineures, parking explicite, révisions tooling. Origine : pattern de ratification compulsive observé 2026-05-09 (3 ADR ratifiées + 2 révisées le même jour) malgré recommandation 2026-05-08.

### Documentation produite
- `docs/CONTRACT_HDL.md` — squelette contrat HDL ↔ golden model (HW-1, structure complète, contenu détaillé Phase 1).
- `docs/adr/README.md` — index MADR + procédure de migration progressive.
- `docs/adr/0015-isolation-memoire-post-v1.md`, `0016-driver-model.md`, `0017-abi-syscall-userland.md`, `0018-retrait-6502.md`.
- `CLAUDE.md` mis à jour : ADR-16/17/18 ajoutés §2, parking ADR-15 §3, moratoire §10, révision v0.2.
- `BACKLOG.md` mis à jour : DEC-1/2 actées, ADR ratifiées, HW-1 promu NOW P0, sprints PH-2.a..d + OS-2.f.v2 ajoutés.

### Phase 1 démarrée (S2-S3)
Sprints à exécuter : HW-1 (contenu détaillé), PH-2.a..d (retrait 6502), PH-cleanup-zombie (`kernel_hires2_*`), OS-2.f.v2 (table dispatch syscall), OS-2.d (clavier IRQ-driven, débloqué par ADR-16).

### Aucune régression code
- Pas de modification de code C ou asm en Phase 0. 541 tests OK conservés.
- Modifications limitées à : `CLAUDE.md`, `BACKLOG.md`, `CHANGELOG.md` (workspace, OricOS, Phosphoric), création `docs/CONTRACT_HDL.md`, `docs/adr/*`.

---

## [2026-05-09] — Sprint 3.c v0.4 : 3e fenêtre démo palette ✨

### OricOS → 0.40.0
- Boot kernel : ajout window 3 colorful à (140, 100), 80×60.
- Couleurs distinctes : frame=12=lightred, title=14=yellow,
  body=11=lightcyan — démontre la palette VGA-IBM 16 couleurs.

### Phosphoric → 1.22.9-alpha
- `tests/integration/test_oricos_window.c` : 6 ASSERTs window 3.
- 541 tests OK (= aucune régression).

### Démo PPM finale (1024×768 visualisable)
- Window 1 (20, 10) bleu/lgray + "OS" blanc.
- Window 2 (300, 300) vert/lgray (dragged).
- Window 3 (140, 100) rouge/jaune/cyan ✨.

### Limite v0.1 documentée
- `kernel_window_draw` arguments ZP 8-bit (WIN_X/Y/W/H).
- Contrainte : x+w-1 ≤ 255, y+h-1 ≤ 255.
- Extension 16-bit args prévue v0.5.

---

## [2026-05-09] — Sprint 3.c v0.3 : true drag ✨✨

### OricOS → 0.39.0
- Boot kernel : démo true drag.
  - **BLIT** window 2 depuis (50, 80) vers (300, 300).
  - **FILL_RECT** clear ancienne pos (50, 80, 80, 60) en color 0.
- Window 1 + "OS" titlebar intacts.

### Démo PPM finale ULTIME — 3 actions visuelles
**`/tmp/oricos_window_xvga.ppm`** affiche :
- **Window 1** (20, 10) titlebar BLEUE avec **"OS"** en BLANC ✨
- **Window 2** (300, 300) titlebar VERTE — dragged depuis (50, 80) ✨
- Position (50, 80) tout NOIR — effacée par drag ✨

### Validation
- 541 tests OK.

### Bug d'arithmétique fixé
Confusion `$58 ↔ $18` pour MID byte de dst_addr `$031896` → fix
trivial. Décalage visuel +16 KiB hors zone observable.

### Importance
**True drag = BLIT + CLEAR.** Première opération interactive d'un
window manager. Sprint 3.c v0.4 ajoutera la liste de fenêtres + focus
+ drag depuis events souris/clavier.

### Reportés Sprint 3.c v0.4
- struct window_t (RAM) avec liste fixe 8 windows.
- kernel_window_create/destroy/move/raise/lower.
- close/minimize avec backing-store SDRAM via DMA.
- Event-driven drag (souris/clavier).

---

## [2026-05-09] — Sprint GPU-3 v0.3 : kernel_gfx_text + démo "OS" ✨✨✨

### OricOS → 0.38.0
- **`kernel_gfx_text`** : 5e helper kernel via GPU TEXT.
- **API kernel_gfx_* 100% complète** (5/5) :
  ✅ clear, fill_rect, blit, line, text.
- Mini-fonte 8×8 'O' + 'S' embedded en bank 1 + string "OS\\0".
- Boot kernel : pré-charge fonte/string en SDRAM via
  `kernel_vram_write_block`, puis TEXT pour titrer "OS" en blanc
  dans la titlebar bleue de window 1.

### Démo PPM finale
**`/tmp/oricos_window_xvga.ppm`** affiche maintenant **2 fenêtres
GUI complètes** :
- Window 1 (20, 10) titlebar bleue **avec "OS" en blanc** ✨
- Window 2 (50, 80) clone titlebar verte (BLIT + repaint)

### Validation
- 541 tests OK.
- 6 nouveaux ASSERTs pour les pixels TEXT.

### État architecture finale
```
ADR-21 GPU complet (5 commandes Phosphoric + 5 kernel helpers) :
  Phosphoric : CLEAR + FILL_RECT + BLIT + LINE + TEXT  ✅
  Kernel API : kernel_gfx_clear + fill_rect + blit + line + text ✅

Pipeline end-to-end validé pixel par pixel :
  Boot kernel asm 65C816
    → kernel_vram_write_block (charge fonte SDRAM)
    → kernel_gfx_text (commande GPU TEXT)
    → I/O ports $0340-$034F
    → gpu_device exec (Phosphoric)
    → vram_peek bitmaps + gpu_set_pixel (4bpp mask)
    → SDRAM 16 MiB
    → vram_peek validation + PPM 1024×768 visible
```

### Reportés
- v0.4 : color_bg, fonte taille variable, BLIT pixel-aligned/overlap.
- SP-3.c v0.3 : window list / TCB par fenêtre, true drag, close/min.
- SP-GPU-HDL-1..4 : implémentation HDL ULX3S (~6-8 sem).

---

## [2026-05-09] — Sprint GPU-2 v0.3 : TEXT (ADR-21 complet) ✨✨✨

### Phosphoric (oric2-golden-model)
- **`GPU_OP_TEXT ($05)`** : 5e commande GPU implémentée.
  Rendu fonte 8×8 monochrome (color_fg, transparency intacte).
  Args : base + font_addr + string_addr (null-term) + (x, y, color).
- Test unitaire `test_text_basic_char` valide rendu pixel par pixel
  d'un caractère simple.

### ADR-21 — 100% complet
Toutes 5 commandes GPU ratifiées sont **implémentées et testées** :

| Opcode | Nom | Phosphoric | Kernel API |
|--------|-----|------------|------------|
| $01 | CLEAR | ✅ v0.1 | ✅ kernel_gfx_clear |
| $02 | FILL_RECT | ✅ v0.1 | ✅ kernel_gfx_fill_rect |
| $03 | BLIT | ✅ v0.1 | ✅ kernel_gfx_blit |
| $04 | LINE | ✅ v0.1 | ✅ kernel_gfx_line |
| **$05** | **TEXT** | **✅ v0.1** | ⏳ kernel_gfx_text v0.3 |

### Validation
- 541 tests OK (540 → 541, +1).

### Reportés
- **kernel_gfx_text** côté OricOS (SP-GPU-3 v0.3) : helper asm qui
  configure registres GPU. Demande aussi pré-chargement fonte +
  string en SDRAM avant l'appel — plus lourd à intégrer dans le
  boot, à faire dans une session dédiée.
- v0.4 : color_bg, fonte variable, BLIT pixel-aligned/overlap.

---

## [2026-05-09] — Sprint 3.c v0.2 : Multi-fenêtre via BLIT ✨✨

### OricOS → 0.37.0
- Démo multi-fenêtre :
  - Window 1 (kernel_window_draw) : (20, 10) titlebar blue.
  - **BLIT clone** : window 1 → position (50, 80).
  - **FILL_RECT repaint** titlebar window 2 en green pour distinction.
- 10 ASSERTs supplémentaires (window 2 frame + titlebar green + body).

### Démo PPM
Le PPM `/tmp/oricos_window_xvga.ppm` montre maintenant **2 fenêtres
distinctes** sur fond noir XVGA 1024×768 :
- Window 1 en (20, 10) titlebar bleu.
- Window 2 en (50, 80) titlebar vert (clone via BLIT puis repaint).

### Importance
**Premier multifenêtré OricOS** via le pipeline GPU. Démontre :
1. **BLIT HW** clone/déplace une fenêtre en quelques µs.
2. **Composition multi-couche** : repaint d'un élément d'une fenêtre
   clonée via FILL_RECT.
3. **Pipeline complet** boot kernel → kernel_gfx_blit → I/O GPU →
   gpu_device exec → SDRAM, validé pixel par pixel.

### Implémentation
- Erreur d'arithmétique 80×512 corrigée (40960 vs 41000) : dst_addr
  = $00C000 + 40985 = $016019.
- BLIT v0.1 byte-aligned : x src/dst pairs (20, 50 OK).

### Reportés Sprint 3.c v0.3
- True drag : BLIT + CLEAR pos1 (effacer original).
- Window list / TCB par fenêtre.
- `kernel_window_close/minimize` (backing SDRAM via DMA).
- Title text via `kernel_gfx_text` (dépend SP-GPU-2 v0.3).

---

## [2026-05-09] — Sprint 3.c v0.1 : Window manager basique ✨✨

### OricOS → 0.36.0
- **`kernel_window_draw`** : helper kernel asm 65C816 qui dessine
  1 fenêtre rectangulaire via 6 commandes GPU séquentielles.
- Args ZP : WIN_BASE (24-bit), WIN_X/Y/W/H, WIN_TITLEBAR_H,
  WIN_COLOR_FRAME/TITLE/BODY (4-bit chacun).
- Algorithme :
  1. FILL_RECT body entier (color body).
  2. FILL_RECT titlebar par-dessus (color title).
  3-6. 4 LINEs Bresenham (color frame) pour cadre 1 pixel.

### Boot kernel intégré (démo)
- CLEAR fond noir 32 KiB à $00C000.
- kernel_window_draw(base=$00C000, x=20, y=10, w=80, h=60, titlebar=8,
  frame=0=black, title=1=blue, body=7=lightgray).

### Phosphoric (oric2-golden-model)
- Test `test_oricos_window_draw` : valide 18 pixels assertions
  (4 coins frame + 4 milieux bords + 3 titlebar + 3 body +
  4 hors fenêtre).
- 540 tests OK (539 → 540, +1).

### Importance — première fenêtre GUI OricOS
**Pipeline complet end-to-end** validé du boot kernel asm jusqu'aux
pixels SDRAM via le GPU autonome :
```
Boot kernel (asm 65C816)
  → kernel_window_draw
  → kernel_gfx_fill_rect/line
  → I/O ports $0340-$034F
  → gpu_device (Phosphoric C)
  → exec FILL_RECT + LINE Bresenham
  → vram_device SDRAM
  → vram_peek validation pixel par pixel
```

Sprint 3.c v0.1 démontre **OricOS dessine une vraie fenêtre GUI**.

### Reportés Sprint 3.c v0.2
- `kernel_window_move(id, dx, dy)` via BLIT (drag fenêtre).
- `kernel_window_close` / `kernel_window_minimize` (backing-store
  SDRAM via DMA).
- Multifenêtré (TCB par fenêtre, focus management).
- Title text via `kernel_gfx_text` (dépend SP-GPU-2 v0.3 font ROM).

---

## [2026-05-09] — Sprint GPU-3 v0.2 : kernel_gfx_blit + line ✨

### OricOS → 0.35.0
- **`kernel_gfx_blit`** : copie bloc rectangulaire SDRAM via GPU BLIT.
- **`kernel_gfx_line`** : tracé Bresenham 4bpp via GPU LINE.
- Constantes I/O : GPU_OP_BLIT = $03, GPU_OP_LINE = $04.

### Boot kernel intégré
- BLIT(src=$004000, dst=$008000, byte_w=10, byte_h=8) : 8 lignes × 20
  pixels copiés vers ligne 32+, le rect FILL_RECT répliqué.
- LINE((40,20)→(40,25), color=2=green) : ligne verticale 6 pixels
  green sur fond blue, mix byte = $24.

### API kernel_gfx_* complet
| Helper | Status | GPU opcode |
|--------|--------|------------|
| kernel_gfx_clear | ✅ v0.1 | CLEAR |
| kernel_gfx_fill_rect | ✅ v0.1 | FILL_RECT |
| kernel_gfx_blit | ✅ v0.2 | BLIT |
| kernel_gfx_line | ✅ v0.2 | LINE |
| kernel_gfx_text | ⏳ v0.3 | TEXT (font ROM) |

### Validation
- 539 tests OK. ASSERTs étendus (BLIT + LINE post-STP).

### Importance
**Le kernel OricOS dispose maintenant d'une API graphique complète
pour Sprint 3.c (window manager) :** clear, fill_rect, blit (drag
fenêtre), line (bordures arbitraires). Suffisant pour démarrer le
window manager sans attendre TEXT.

---

## [2026-05-09] — Sprint GPU-2 partiel : BLIT + LINE ajoutés ✨

### Phosphoric (oric2-golden-model)
- **`GPU_OP_BLIT ($03)`** : copie de bloc rectangulaire SDRAM → SDRAM.
  v0.1 limites : src/dst byte-alignés, pas d'overlap, pas de
  transparency. BPL hardcodé GPU_XVGA_BPL=512.
- **`GPU_OP_LINE ($04)`** : tracé Bresenham 4bpp.
  Helper `gpu_set_pixel` mutualise le mask 4bpp gauche/droit.
- 5 tests supplémentaires : BLIT 1 ligne, BLIT rect multi-ligne,
  LINE horizontale, verticale, diagonale.
- 539 tests OK (534 → 539, +5).

### Reportés Sprint GPU-2 v0.3
- **TEXT** : rendu fonte HW (font ROM + string addr). Demande font
  ROM externe + caractère par caractère via blit.
- BLIT v0.2 : alignement pixel-arbitraire, overlap, transparency, ROP.

### État GPU complet (v0.2 partiel)
| Commande | Status | Notes |
|----------|--------|-------|
| CLEAR | ✅ v0.1 | OK |
| FILL_RECT | ✅ v0.1 | x/y/w/h 8-bit |
| BLIT | ✅ v0.1 | byte-aligned, no overlap |
| LINE | ✅ v0.1 | Bresenham 8-bit coords |
| TEXT | ⏳ v0.3 | font ROM à venir |

Avec CLEAR + FILL_RECT + BLIT + LINE, **un window manager basique
peut être construit en SP-3.c** sans attendre TEXT.

---

## [2026-05-09] — Sprint GPU-3 : kernel_gfx_* opérationnels ✨

### OricOS → 0.34.0
- **`kernel_gfx_clear`** : remplit zone SDRAM via GPU CLEAR.
- **`kernel_gfx_fill_rect`** : rectangle 4bpp via GPU FILL_RECT.
- ZP args $70-$78, constantes I/O $000340-$00034F.
- Boot kernel : CLEAR 32 KiB blue + FILL_RECT 8×4 white sur fond blue.

### Phosphoric (oric2-golden-model)
- Test `test_oricos_gpu_clear_then_fill_rect` : pipeline complet
  Boot → kernel_gfx_clear → kernel_gfx_fill_rect → GPU exec → SDRAM.
- io_callback étendu pour routing $0340-$034F (GPU).
- 534 tests OK (533 → 534, +1).

### Importance architecturale
**OricOS utilise désormais le GPU autonome au lieu d'écrire directement
en VRAM.** Pipeline complet validé end-to-end :
```
Boot kernel → kernel_gfx_clear/fill_rect → I/O ports $0340+
           → gpu_device exec → vram_device SDRAM
           → vram_peek validation
```

SP-3.c (window manager) peut maintenant s'appuyer sur cette API.

### Reportés Sprint GPU-3 v0.2
- `kernel_gfx_blit/line/text` (dépend SP-GPU-2 extension Phosphoric).
- IRQ-based wait au lieu de polling busy.
- Cleanup `kernel_hires2_*` legacy (Sprint 3.b cleanup).

---

## [2026-05-09] — Sprint GPU-1 : gpu_device v0.1 implémenté ✨

### Phosphoric (oric2-golden-model)
- **`src/io/gpu_device.{c,h}`** : émulation GPU Blitter HW (ADR-21).
- 2 commandes v0.1 : CLEAR + FILL_RECT synchrones.
- Ports I/O `$0340-$034F` : 4 args 24-bit + status + trigger.
- 7 tests unitaires : init, registres round-trip, CLEAR, FILL_RECT
  (aligné + mask intra-octet 4bpp), opcode inconnu err, CLEAR full
  XVGA framebuffer.

### Validation
- 533 tests OK (526 → 533, +7 nouveaux, aucune régression).

### Importance architecturale
**Première brique GPU autonome opérationnelle**. Le golden model
Phosphoric peut désormais simuler des commandes GPU CLEAR / FILL_RECT
sur la VRAM SDRAM. Le HDL ULX3S à venir (SP-GPU-HDL-1) reproduira
le même comportement.

### Reportés Sprint GPU-2
- Commandes BLIT, LINE, TEXT (3 ops restantes ADR-21).
- BPL configurable (pour résolutions autres que XVGA).
- x/y/w/h 16-bit (limite v0.1 = 255 chacun).
- DMA async + IRQ.

---

## [2026-05-09] — ADR-20 v3 : XVGA 1024×768×4bpp ✨

Avec ADR-19 v2 (VRAM SDRAM unifiée), la contrainte BRAM est levée.
On peut donc viser plus haut que SVGA.

### ADR-20 v3 — XVGA 1024×768×4bpp 16 couleurs
- **Résolution** : 1024×768 pixels (format 4:3 XVGA standard).
- **Profondeur** : 4 bits par pixel = 16 couleurs (palette VGA-IBM).
- **Framebuffer** : 384 KiB linéaires en SDRAM ($000000-$05FFFF).
- **Pixel clock** : 65 MHz (VESA standard XVGA 60Hz).
- **+60% surface vs SVGA**, look "OS pro Win 95-like".

### Évolution résolutions
- v1 : 240×200×3bpp (ADR-12, compat Oric 1, ULA guest)
- v2 : 800×600×4bpp SVGA (ADR-19 v1, BRAM live)
- **v3 : 1024×768×4bpp XVGA** (ADR-19 v2 SDRAM unifiée)

### Capacité fenêtres en XVGA
- 16 MiB SDRAM - 384 KiB framebuffer = **15.6 MiB libres**.
- ~1050 mini fenêtres (200×150) ou ~410 standard (320×240).
- ~40 fenêtres plein écran XVGA backing simultané.

### Effort HDL marginal
- vs SVGA : +30% (PLL ajustée, raster timing standard XVGA).
- HDMI 1024×768 60Hz universellement supporté.

### Documents mis à jour
- **CLAUDE.md §2** : ADR-20 v3 complet, ancien contenu SVGA retiré.
- **MEMORY_MAP §8bis** : framebuffer XVGA $000000-$05FFFF.
- **BACKLOG** : ADR-20 v3 fermée.

---

## [2026-05-09] — ADR-19 v2 + ADR-20 v2 : architecture VRAM simplifiée ✨

Suite à un échange architectural, simplification de la stack VRAM.

### Constat
Avec ADR-21 (GPU Blitter HW autonome), le CPU n'écrit plus de pixels
directement. Les banks 128-159 dédiées VRAM live BRAM (Arch D
hybride) deviennent **redondantes** : le GPU accède SDRAM directement,
pas besoin de "live" exposé au CPU.

### ADR-19 v2 — VRAM en SDRAM unifiée
- **Toute la VRAM** réside en SDRAM 32 MiB (16 MiB exposés v1).
- Hors banking 24-bit du CPU.
- Accès GPU direct + accès CPU via I/O `$0330-$033C` (rare).
- BRAM ECP5 redéployée : caches internes GPU/compositor (line-buffers,
  sprite cache) — invisible côté CPU.

### ADR-20 v2 — Framebuffer SVGA en SDRAM
- v1 : framebuffer 800×600×4bpp en 4 banks live (128-131).
- v2 : framebuffer en **SDRAM offset $000000-$03A97F** (240 KiB linéaires).
- Banks 128-131 **libérés** (réservés legacy v1 kernel_hires2_* à
  retirer Sprint 3.b cleanup).

### Bénéfices
- **+4 MiB RAM utilisable** pour apps (banks 128-191 récupérables).
- **HDL plus simple** : 1 controller SDRAM, BRAM = caches internes.
- **Architecture cohérente** : style Amiga chip RAM unifiée.
- **Capacité fenêtres** : ~420 fenêtres 320×240×4bpp possibles
  (15.76 MiB SDRAM disponible après framebuffer principal).

### Documents mis à jour
- **CLAUDE.md §2** : ADR-19 v2 + ADR-20 v2 complets (l'ancienne v1
  ADR-19 supprimée).
- **MEMORY_MAP §8** : refondu (banks 128-191 = RAM extra apps,
  §8bis = VRAM en SDRAM).
- **BACKLOG** : ADR-19 marquée révisée v2.

### Non modifié
- Le code kernel reste avec `BANK_LIVE_POOL_BASE = $84` (banks 132-159
  pool extra). Cleanup banks 128-131 reporté Sprint 3.b cleanup
  (retirer kernel_hires2_* du boot, étendre pool à $80).
- `vram_device` Phosphoric (Sprint VRAM-1) inchangé.
- `kernel_vram_*` (Sprint VRAM-2) inchangé.

526 tests OK toujours.

---

## [2026-05-09] — RÉORIENTATION MAJEURE : ADR-20 + ADR-21 ratifiées ✨✨✨

### Décisions stratégiques

Suite à un échange architectural avec l'utilisateur, le projet bascule
vers une **architecture GPU-first** plutôt que CPU-direct draw.

#### ADR-20 — Mode HIRES Oric 2 desktop = SVGA 800×600×4bpp
- 800×600 pixels, 16 couleurs simultanées (palette VGA-IBM).
- Layout 4bpp big-endian, 400 octets/ligne, 240 KiB/frame.
- **Localisation : 4 banks live consécutifs (128-131)** dans la VRAM
  live BRAM ECP5 (cf. ADR-19).
- HDMI 800×600 60Hz = 40 MHz pixel clock (ULX3S OK).
- Référence d'art : Amiga ECS, OS/2 Warp, Win 3.x SVGA.

#### ADR-21 — GPU Blitter HW autonome
- Co-processeur graphique HDL ECP5 dédié, exécute les ops de dessin
  en parallèle du CPU.
- Inspiration : Amiga Blitter+Copper, Atari ST DMA blitter, NES PPU.
- 5 commandes v1 ratifiées : **CLEAR, FILL_RECT, BLIT, LINE, TEXT**.
- Registres I/O `$0340-$034F` (16 octets) : opcode + 4 args 24-bit
  + status + trigger + IRQ ctrl.
- Le CPU enqueue commandes via I/O, GPU exécute, IRQ ou polling busy.

### Conséquences immédiates

- **Sprint 3.c retardé** : maintenant dépend de SP-GPU-3 (helpers
  kernel `kernel_gfx_*`). Total +6-8 sem effort HDL pour les 5
  commandes.
- **Sprint 3.b** (`kernel_hires2_clear`, `fill_rect_aligned`) :
  conservé comme fallback legacy mais non plus l'API publique.
- **Pool LIVE allocator** ajusté : `BANK_LIVE_POOL_BASE = $84`
  (= bank 132) puisque banks 128-131 réservés framebuffer SVGA.

### Sprint plan révisé

| Sprint | Contenu | Effort |
|--------|---------|--------|
| ✅ **SP-VRAM-1/2/3** | VRAM hybride + allocator (clos) | — |
| **SP-GPU-1** | Phosphoric `gpu_device` v0.1 (CLEAR + FILL_RECT) | 2-3 j |
| **SP-GPU-2** | Phosphoric extension (BLIT, LINE, TEXT) | 3-5 j |
| **SP-GPU-3** | OricOS kernel `kernel_gfx_*` helpers | 2-3 j |
| **SP-GPU-HDL-1..4** | HDL ULX3S GPU implémentation | 6-8 sem |
| **SP-3.c** | Window manager via GPU | 5-10 j (post-GPU-3) |

### Documents mis à jour
- **CLAUDE.md §2** : ADR-20 + ADR-21 complets avec spec ports I/O,
  commandes, palettes, layouts.
- **MEMORY_MAP §8** : banks 128-131 réservés framebuffer SVGA, banks
  132-159 = pool live fenêtres (28 banks dispos).
- **BACKLOG** : nouveaux sprints SP-GPU-1/2/3 + SP-GPU-HDL-1/2/3/4.
  ADR-20 + ADR-21 fermées dans la liste.

### Code mis à jour
- `kernel.s` : `BANK_LIVE_POOL_BASE` = $84 (bank 132). Démo allocator
  retourne maintenant $84/$85/$86/$85.
- `test_oricos_live_alloc` : ASSERTs ajustées.
- 526 tests OK (aucune régression).

### Importance
**Ce sprint définit la trajectoire long-terme du projet.** OricOS
devient un OS GPU-accélérée, comparable à Amiga/Atari ST. Le CPU se
concentre sur la logique métier ; le GPU dessine de manière autonome.
Trade-off accepté : effort HDL significatif (6-8 semaines pour les 5
commandes), mais résultat = OS rétro 16-bit moderne et fluide.

---

## [2026-05-09] — Sprint VRAM-3 : pool LIVE banks 129-159 ✨

### OricOS → 0.32.0
- **`kernel_alloc_live_bank`** / **`kernel_free_live_bank`** : pool
  séparé pour fenêtres GUI live (banks 129-159 = $81..$9F).
- Bank 128 réservé framebuffer principal HIRES Oric 2 (ADR-12).
- Storage allocator live : `BANK_LIVE_NEXT/FREE_LIST/FREE_TOP` en
  bank 1 zone $015458/$0154C0/$0154D0.
- Boot kernel : démo alloc 3 + free 1 + alloc 1, sentinels à
  BANK_LIVE_DEMO ($015468).

#### Robustesse DMA
- `kernel_vram_dma` : timeout 256 polls dans `vdma_wait` (fix boucle
  infinie potentielle si vram_device absent ou stuck).

### Phosphoric (oric2-golden-model)
- Test `test_oricos_live_alloc_demo` : ASSERT séquence
  $81 $82 $83 $82 (3 alloc + free + alloc LIFO).
- 526 tests OK (+1).

### Architecture
**Deux pools de banks distincts** :
- Pool système (banks 4-127) pour code/data apps.
- Pool live (banks 129-159) pour fenêtres GUI live.

SP-3.c (window manager) pourra allouer 1 bank live par fenêtre
active sans interférer avec le pool d'apps.

---

## [2026-05-09] — Sprint VRAM-2 : kernel API vram_* opérationnelle ✨

### OricOS → 0.31.0
- **`kernel_vram_write_block`** : RAM banking → VRAM cold.
- **`kernel_vram_read_block`** : VRAM cold → RAM banking.
- **`kernel_vram_dma`** : trigger DMA HW SDRAM↔bank (synchrone v0.1).
- ZP args $60-$6D, constantes I/O $000330-$00033C.
- Boot kernel exerce les 3 helpers en séquence (write "VRAM", read
  "ABCD", DMA copy "ABCD"). Validation 3 ASSERTs post-STP.

### Phosphoric (oric2-golden-model)
- Test `test_oricos_vram_write_read_dma` : pré-charge VRAM via
  `vram_poke`, boot kernel, ASSERT post-STP.
- io_callback étendu : $0320-$0327 (SD) + $0330-$033F (VRAM).
- 525 tests OK (524 → 525, +1).

### Importance architecturale
**Premier code kernel utilisant l'I/O VRAM cold.** Les Sprints
suivants (SP-VRAM-3 refactor allocator, SP-3.c window manager)
s'appuieront sur ces helpers comme primitives de base.

### Reportés Sprint VRAM-2 v0.2
- `kernel_vram_alloc(size)` allocator (bumb-only ou bitmap).
- `kernel_vram_blit(src_24, dst_24, w, h)` rectangles 2D.

---

## [2026-05-09] — Sprint VRAM-1 : vram_device implémenté ✨

### Phosphoric (oric2-golden-model)
- `src/io/vram_device.{c,h}` : émulation VRAM cold SDRAM 16 MiB v1
  via I/O ports `$0330-$033C`.
- API : init, cleanup, read/write registers, peek/poke direct.
- Auto-increment ADDR sur DATA + DMA synchrone bidirectionnel
  (SDRAM↔bank).
- 9 tests unitaires couvrent init, address round-trip, auto-inc,
  wrap-around, DMA dans les 2 sens, LEN=0=64KiB, registres.

### Validation
- 524 tests OK (+9 nouveaux, aucune régression).

### Importance
**Première brique d'Arch D opérationnelle.** OricOS pourra accéder à
16 MiB de VRAM cold via I/O, avec DMA HW pour blits massifs sans
cycle CPU. Sprint VRAM-2 (kernel API) suit naturellement.

### Reportés v0.2
- DMA asynchrone (busy bit, IRQ optionnel).
- Extension à 32 MiB (24-bit + bit dans DMA_CTRL).

---

## [2026-05-09] — ADR-19 ratifiée : VRAM hybride (BRAM live + SDRAM cold) ✨

### Décision stratégique architecturale

OricOS adopte une **architecture VRAM à deux niveaux** :
- **VRAM live** : banks 128-159 (2 MiB) en BRAM ECP5, accès CPU
  direct via banking, latence 1 cycle. Compositor matériel ULA host
  (ADR-02) lit ces banks à fréquence pixel HDMI.
- **VRAM cold** : SDRAM 32 MiB accessible via I/O ports $0330-$033C
  (auto-increment + DMA HW pour blit massif). Hors banking 24-bit.

### Justification
- Capacité scalable : 100+ fenêtres iconifiées en SDRAM.
- Performance optimale fenêtre active : `STA al` direct.
- DMA HW pour transferts massifs (drag fenêtre) sans cycle CPU perdu.
- Référence d'art : **Amiga (chip RAM + fast RAM)**, **Apple IIgs
  (system RAM + slot RAM)**.

### Alternatives écartées
- Arch A pure (banking-only) : limité ~3 MiB pratique, pas scalable.
- Arch B I/O VRAM pure : pas d'accès pixel direct, perf dégradée.
- Arch C window mapping : memory remap HDL complexe, latence cache.

### Documents mis à jour
- **`CLAUDE.md` §2 ADR-19** : ratification complète avec spec ports
  I/O et impacts.
- **`docs/MEMORY_MAP.md`** : §8 refondu (banks 128-159 = VRAM live
  BRAM), §9 nouveau (VRAM cold via I/O), §10 ajusté (banks 160-255
  réservés extensions).
- **`BACKLOG.md`** : nouveaux sprints SP-VRAM-1/2/3 ajoutés en
  prérequis SP-3.c. ADR-19 fermée.

### Sprint plan implémentation (à venir)
- **SP-VRAM-1** : Phosphoric `vram_device` simulant 32 MiB SDRAM (1-2j).
- **SP-VRAM-2** : kernel API `vram_read/write/dma` (2-3j).
- **SP-VRAM-3** : refactor allocator pool "live" vs système (1-2j).
- **SP-3.c** ensuite : window manager utilisant Arch D pleinement.

### Importance
**ADR-19 fixe les fondations long-terme** d'OricOS GUI. Sprint 3.c
(window manager) ne démarrera qu'après les SP-VRAM pour s'appuyer
sur la bonne plomberie dès le début (vs refactor douloureux ensuite).

---

## [2026-05-09] — Sprint 3.b v0.2 : kernel_fill_rect_aligned ✨ + fix Phosphoric ASL M=0

### OricOS → 0.30.0
- **`kernel_fill_rect_aligned`** : rectangle 8-px-aligned X.
  Args ZP : gx_start, gx_count, y_start, y_count, color.
- Boot kernel : clear(blue) + fill_rect(red 80×80 centre).

### Phosphoric (oric2-golden-model) → fix opcodes 65C816
**Bug racine trouvé dans Phosphoric** : ASL/LSR/ROL/ROR Accumulator
(`$0A`, `$4A`, `$2A`, `$6A`) ne propageaient PAS le carry low→high
byte en mode M=0 (utilisaient `a8(cpu)` 8-bit même quand A est 16-bit).

**Conséquence** : `lda #$003C; asl asl asl` en M=0 donnait `$00E0`
au lieu de `$01E0`. Toute arithmétique 16-bit basée sur shifts était
silencieusement fausse.

**Découverte** : kernel_fill_rect_aligned d'OricOS calculait `y*90`
via shifts 16-bit. Résultat erroné `$0318` (792) au lieu de `$1518`
(5400) pour y=60. Sentinels asm injectés dans le kernel ont permis
d'isoler le bug DANS Phosphoric, pas OricOS.

**Fix** : branchement M=8bit/M=16bit explicite dans les 4 opcodes
Accumulator, utilisant `cpu->C` 16-bit en M=0.

### Validation
- 515 tests OK (aucune régression du fix Phosphoric).
- Test `test_oricos_hires2_clear_and_rect` : ASSERT 6400 pixels red
  + 41600 pixels blue + 0 autres + frontières strictes 1-px.

### Importance
**Bel exemple de méthodologie pluri-projets** :
- Le golden-model Phosphoric avait un bug latent silencieux.
- Une primitive OricOS l'a réveillé.
- Sentinels asm ont localisé la racine côté Phosphoric.
- Fix en amont → le code OricOS était correct depuis le début.

ADR-12 + ADR-02 maintenant **complètement opérationnels** : kernel
peut dessiner des rectangles arbitraires (granularité 8 pixels en X)
en bank 128, visibles via le compositor.

### Dette technique restante
- Audit shifts/rotations zp/abs en M=0 (possible même bug pour `$06`,
  `$0E`, `$2E` etc.). Pas encore exercé.

### Reportés Sprint 3.b v0.3
- `kernel_pixel_set(x, y, color)` pixel-perfect arbitraire.
- `kernel_blit` pour fontes / icônes.
- Bascule mode TEXT ↔ HIRES via registre I/O.

---

## [2026-05-09] — Sprint 3.b v0.1 : kernel_hires2_clear ✨

### OricOS → 0.29.0
- **`kernel_hires2_clear(A=color)`** : remplit le framebuffer HIRES
  Oric 2 (bank 128, ADR-12) avec une couleur uniforme. 18 000 octets
  écrits via DP indirect long.
- `pattern_table` 8 × 3 octets : pré-calcul `color × $249249`.
- Boot kernel intégré : `kernel_hires2_clear(blue)` tôt dans
  `kernel_entry`. Lazy alloc bank 128 via 1ère écriture (B1.8).

### Phosphoric (oric2-golden-model)
- Test `test_oricos_hires2_clear_fills_blue` :
  - Boot kernel → ASSERT pattern octets aux 4 coins.
  - ASSERT pixels via API hires_oric2_get_pixel.
  - ASSERT 100% des 48 000 pixels ARGB = `0x0000FF`.
- 515 tests OK (514 → 515, +1).

### Importance
**Premier code OricOS qui écrit dans le framebuffer Oric 2.** Le pipeline
complet est opérationnel : kernel asm → bank 128 → render ARGB →
ASSERT visuel. Les futurs sprints 3.c/d/e (window manager, toolkit,
multifenêtré) s'appuieront sur ce socle.

### Reportés Sprint 3.b v0.2
- `kernel_pixel_set` pixel-perfect arbitraire.
- `kernel_fill_rect`, `kernel_blit`.
- Bascule mode TEXT ↔ HIRES via registre I/O.

---

## [2026-05-09] — Sprint 3.a v0.2 : intégration compositor + HIRES Oric 2 ✨

### Phosphoric (oric2-golden-model)
- Test d'intégration `test_compositor_hires_oric2.c` : démontre que
  le framebuffer Oric 2 (bank 128) peut servir de `host` au compositor
  matériel (ADR-02).
- Pipeline complet validé sans toucher aux modules existants (compositor
  et hires_oric2 communiquent via uint32_t* ARGB).
- 3 tests : host seul, host + guest overlay, guest invisible.
- 514 tests OK (511 → 514, +3 intégration).

### Importance
Le compositor matériel double-ULA (ADR-02) peut désormais consommer
un framebuffer Oric 2 (ADR-12) comme `host`, en plus du framebuffer
Oric 1 historique comme `guest` (ULA guest, attribute-based). Toute la
plomberie est en place pour Sprint 3.b (kernel pixels) et au-delà.

### Reportés Sprint 3.a v0.3
- Intégration main loop SDL2 : `--video-mode oric2` rend visible le
  framebuffer Oric 2 en temps réel via SDL2.
- Sans cette étape, validation = PPM dump ; avec, on voit OricOS
  dessiner directement à l'écran.

---

## [2026-05-09] — Sprint 3.a : ADR-12 mode HIRES Oric 2 implémenté ✨

### Phosphoric (oric2-golden-model)
- **`video/hires_oric2.{c,h}`** : implémentation complète ADR-12.
- API : `get_pixel`, `set_pixel`, `clear`, `render_argb`, `export_ppm`.
- Format pixel figé : 8 pixels en 24 bits sur 3 octets, big-endian.
- Bank 128 ($80xxxx), 240×200×3bpp = 18 000 octets/frame.
- Palette 8 couleurs Oric 1 (idem ADR-10 compat stricte).
- 8 tests unitaires (round-trip, layout, palette, clear, isolation
  intra-groupe, bornes, PPM, full line).
- Total tests OK : **511** (503 → 511).

### Importance
- **ADR-12 sort de l'état "vaporware"** identifié au point critique
  architecte senior 2026-05-08 (#1 des 5 ADRs ratifiées sans
  implémentation).
- Building block pour Sprint 3.b/c/d : kernel OricOS pourra écrire
  des pixels en bank 128, le compositor logique pourra dessiner des
  fenêtres, le toolkit pourra rendre des fontes pixel-perfect.

### Reportés Sprint 3.a v0.2
- Intégration compositor (host ULA = HIRES Oric 2).
- Bank framebuffer configurable via registre I/O.
- Double-buffering swap A/B.
- Mode étendu 320×240 (B4 v2).

---

## [2026-05-09] — TC-llvmmos investigation : ADR-05 révisée v2 ✨

### Décision stratégique majeure

**TC-llvmmos** clos par **investigation documentaire** (pas
d'installation, suffit pour trancher). Document complet :
`docs/TC-llvmmos.md`.

#### Constats
- llvm-mos n'implémente **PAS** le mode N 16-bit registres
  (issue #321) ni le banking 24-bit (issue #320), tous deux ouverts
  depuis 2023-2024 sans horizon.
- Le compilateur C génère du code 8-bit linéaire dans tous les
  targets existants (`mos6502`, `mos65c02`, `mosw65c816`, `mos65el02`,
  `rpc8e`).
- Le target `rpc8e` (le plus proche de notre besoin) fait `xce` puis
  `sep #$30` → repasse immédiatement en 8-bit native.

#### Décisions actées
- **ADR-05 v2** (cf. `CLAUDE.md` §2) : userland C llvm-mos en
  **mode N 8-bit native** (M=1, X=1), **apps mono-bank** ≤ 64 KiB
  linéaire. Apps multi-bank ou registres 16-bit → asm 65C816.
- **DEC-3** ratifiée : llvm-mos **conservé** avec contraintes. Pas de
  fallback asm-only complet.
- **Risque "llvm-mos 65C816 immature"** transformé en contrainte
  connue (cf. tableau RISQUES).

#### Sprint 4 décomposé
- `TC-llvmmos-install` (1j) : installer pre-built, vérifier targets.
- `TC-llvmmos-target-oricos` (2-3j) : target custom (clang.cfg +
  crt0.S + link.ld pour bank dédiée).
- `TC-libc` (5-10j) : libc minimal (printf, malloc bank-local).
- `TC-poc-hello-c` (1-2j) : premier hello.c → bundle .oosobj → exec.

#### Importance
**ADR-05 v1 promettait registres 16-bit + banking** pour les apps C —
non tenable. **ADR-05 v2 promet apps C mono-bank ≤ 64 KiB** — tenable
et largement suffisant pour OricOS v1 (un éditeur de texte 60 KiB,
c'est gros sur 8-bit). La roadmap reflète maintenant la réalité
technique. Sprint 3 GUI peut commencer sereinement, Sprint 4 a un
plan concret.

---

## [2026-05-09] — Sprint 2.l v0.2 : APP MULTI-CLUSTER DEPUIS SD ✨✨

### OricOS → 0.28.0
- Boot kernel : ouvre **MULTI.BIN** (bundle 527B sur 2 clusters), charge
  via `fat_read_file` vers $01:8000, exécute via `kernel_app_exec`.
- Bundle synthétique : section CODE à offset 520 → réside dans le 2e
  cluster du bundle. Valide que `app_exec` calcule `DP_SRC = DP_PTR +
  BUNDLE_FOUND_OFFSET` (16-bit add) correctement et copie 7 octets
  depuis le 2e cluster du bundle.
- L'app exécute `ldx #'X' ; lda #1 ; cop #$AA ; rtl` → 'X' à $BBAD.

### Phosphoric (oric2-golden-model)
- Image SD test 166 secteurs : FAT étendue (FAT[6]=7, FAT[7]=EOC) +
  3e entry root dir + clusters 6+7 du bundle MULTI.
- ASSERT `mem[$BBAD] = 'X'` confirme exec multi-cluster.
- 503 tests OK.

### Pipeline complet validé
**Boot → SD → FAT32 → fat_open → fat_read_file → app_exec → exec.**

OricOS peut désormais charger et exécuter une app de taille arbitraire
depuis fichier FAT32 SD :
- 1 cluster (HELLO.BIN, 23B) ✅
- N clusters (MULTI.BIN, 527B) ✅

### Reportés OS-2.l v0.3
- Section CODE > 256 octets (size 16-bit dans copie).
- Sections multiples (DATA, ICON).
- Free bank au RTS app.
- Sandbox / privilege check.

### Importance
Sprint 2.l v0.2 ferme la boucle entière de l'OS pour des **apps réelles
multi-cluster**. Toute la stack du Sprint 2 (driver SD, FAT32 lecture
seule, bundle, app loader) converge ici. **Sprint 3 GUI peut commencer
sur des fondations solides** pour ses propres apps depuis disque.

---

## [2026-05-08] — Sprint 2.j v0.3 : kernel_fat_read_file (multi-cluster) ✨

### OricOS → 0.27.0
- **`kernel_fat_read_file`** : lit un fichier complet en suivant la
  chaîne FAT32 jusqu'à EOC. Boucle `read_cluster` + `next_cluster`,
  avance `DP_PCPTR` de 512 octets entre clusters.
- API : input `FS_FOUND_CLUSTER` (4B) + `DP_PCPTR` (24-bit dest).
  FS_FOUND_CLUSTER préservé à la sortie.
- Boot kernel : ouvre BIG.BIN (1024B, 2 clusters) → charge à $01:7000.

### Phosphoric (oric2-golden-model)
- Image SD test 164 secteurs : root dir avec 2 entries (HELLO + BIG),
  cluster 4 (pattern $AA) + cluster 5 (pattern $55).
- ASSERT `mem[$01:7000..$01:73FF]` valide les 1024 octets contigus.
- 503 tests OK.

### Reportés v0.4
- Fichiers > 64 KiB (propagation DP_PCPTR vers bank).
- BPS != 512, cluster ≥ 16384, subdirectories.

### Importance
OricOS peut désormais charger des fichiers de taille arbitraire
(≤ 64 KiB) depuis SD. **Building block essentiel pour un app loader
complet** : une app multi-cluster pourra être chargée en RAM puis
exécutée. La couche FAT32 v1 lecture seule est complète pour usage
typique (cluster size = 1 secteur, fichiers de quelques KiB).

---

## [2026-05-08] — Sprint 2.j v0.2 : kernel_fat_next_cluster (cluster chain) ✨

### OricOS → 0.26.0
- **`kernel_fat_next_cluster`** : lit l'entry FAT32 d'un cluster donné,
  retourne le cluster suivant dans la chaîne. Permet la traversée de
  fichiers multi-cluster.
- API : input `FS_QUERY_CLUSTER` (4B), output `FS_NEXT_CLUSTER` (4B,
  >= $0FFFFFF8 = EOC).
- Hypothèse v0.2 : BPS=512, cluster < 16384.

### Phosphoric (oric2-golden-model)
- Image SD test : LBA 32 contient une vraie FAT FAT32 partielle
  (FAT[0..5] avec media descriptor + reserved + EOC + chaîne 4→5→EOC).
- ASSERT `kernel_fat_next_cluster(4) → 5`.
- 503 tests OK.

### Reportés v0.3
- `kernel_fat_read_file` (itération sur la chaîne complète).
- BPS != 512 (rare en FAT32 mais spec autorise 512/1024/2048/4096).
- Cluster >= 16384 (FAT spread sur plusieurs secteurs).
- Subdirectories.

### Importance
Building block essentiel pour charger des fichiers > 1 cluster.
Sans la chaîne FAT, OricOS ne lit que les 512 premiers octets d'un
fichier — la v0.3 pourra lire des fichiers de taille arbitraire en
bouclant sur `fat_next_cluster` jusqu'à EOC.

---

## [2026-05-08] — Sprint 2.j.5/6 : APP CHARGÉE DEPUIS FAT32 SD ✨✨

### OricOS → 0.25.0
- `kernel_fat_read_cluster` : lit 1 cluster (v0.1, SPC=1) vers
  DP_PCPTR. LBA = FDS + cluster - 2.
- Boot kernel intégré : après fat_open OK, charge le bundle vers
  $01:6200 puis appelle kernel_app_exec dessus → app exécutée
  écrit 'Z' supplémentaire à $BBAC.

### Pipeline complet validé
1. Boot kernel
2. SD device émulé (Phosphoric I/O $0320-$0327)
3. kernel_sd_read_block ↔ image hôte
4. kernel_fat_init valide FAT32 + parse champs
5. kernel_fat_open localise fichier 8.3 dans root dir
6. kernel_fat_read_cluster charge contenu en mémoire
7. kernel_app_exec valide bundle + alloc bank + copy + JSL
8. App exécute en bank dédiée + syscall vers kernel

**OricOS peut charger et exécuter une app depuis FAT32 SD** sans
qu'elle soit embedded dans le kernel.

### Phosphoric (oric2-golden-model)
- Image SD test 162 secteurs avec bundle hello à cluster 3.
- ASSERT mem[$BBAB]='Z' (bundle inline) + mem[$BBAC]='Z' (bundle SD).
- 503 tests OK.

### Démo
"OricOS v0.7" + "**YABZZ**" — deux 'Z' = exec inline + exec from SD.

### Importance
Sprint 2.j clos. Tous les fondamentaux OS sont en place :
- Multitâche préemptif (TCB scheduler).
- Mécanisme syscall (COP).
- Driver clavier matrice.
- Driver console (print_char/print_string).
- Bank allocator (free list LIFO).
- Modèle erreur kernel (panic + hex8).
- App loader (validate + alloc + copy + JSL).
- App standalone (pipeline asm + bundle + .incbin).
- **Stockage FAT32 SD lecture seule + chargement app depuis disque**.

Sprint 3 (GUI) théoriquement débloqué — tous les blocs OS de base sont
fonctionnels.

---

## [2026-05-08] — Sprint 2.j.4 : kernel_fat_open (root dir lookup 8.3) ✨

### OricOS → 0.24.0
- `kernel_fat_open` : scanne le root dir cluster (16 entries 32B),
  match filename 11B en `DP+$40..$4A`, extrait first_cluster + size
  dans `FS_FOUND_CLUSTER`/`FS_FOUND_SIZE`. `FS_OPEN_RESULT = $00` OK
  ou `$01` NOT_FOUND.
- Skip LFN ($0F), volume label ($08), directory ($10).
- v0.1 : 1 secteur root dir max ; suppose `FS_ROOT=2` (LBA root = FDS).

### Phosphoric (oric2-golden-model)
- Image FAT32 test étendue 1 → 161 secteurs (root dir au bloc 160).
- ASSERT FS_OPEN_RESULT, FS_FOUND_CLUSTER ($3), FS_FOUND_SIZE
  ($DEADBEEF) après lookup "HELLO   BIN" au boot kernel.
- 503 tests OK.

### Importance
Le kernel sait désormais **localiser un fichier dans la racine FAT32**
par son nom 8.3. Plus que la lecture du contenu (OS-2.j.5) et l'intégration
loader (OS-2.j.6) avant que les apps puissent être chargées depuis SD.

---

## [2026-05-08] — Sprint 2.j.3 : parse complet boot sector FAT32

### OricOS → 0.23.0
- `fat_parse_boot_sector` : parse 6 champs FAT32 (BPS, SPC, RSC, NFAT,
  SPF 4B, ROOT 4B) depuis FS_BUFFER. Calcule FS_FDS (first data sector)
  = RSC + NFAT*SPF par boucle d'addition.
- v0.1 limite arithmétique à 16-bit (disque < 32 MiB).

### Phosphoric (oric2-golden-model)
- Test `test_oricos_fat_init_validates_fat32_signature` étendu : ASSERT
  les 6 champs + FDS calculé.
- 503 tests OK.

---

## [2026-05-08] — Sprint 2.j.2 : kernel_fat_init (signature FAT32)

### OricOS → 0.22.0
- `kernel_fat_init` : lit boot sector via sd_read_block, valide signature
  `"FAT32"` à offset $52. Stocke résultat à `FS_INIT_RESULT` ($016160).
- Buffer FS distinct ($015F60) du buffer test sd_read_block ($015D40).

### Phosphoric (oric2-golden-model)
- `tests/integration/test_oricos_sd.c` étendu :
  - `create_fat32_test_image` : génère image SD avec boot sector FAT32
    minimal (champs BPS/SPC/reserved/num_fats/SPF/root_cluster + sig).
  - Test `test_oricos_fat_init_validates_fat32_signature` : ASSERT mem
    [$015F60+$52..+$56] = "FAT32" et FS_INIT_RESULT = 0.
  - Test sd_read_block étendu : ASSERT FS_INIT_RESULT = 1 sur image
    non-FAT32 (pattern A..Z).
- 503 tests OK (+1 nouveau).

---

## [2026-05-08] — Sprint 2.j.1 : test fonctionnel SD validé

### Phosphoric (oric2-golden-model)
- `tests/integration/test_oricos_sd.c` : pipeline complet validé.
  Image SD test 512B (pattern A..Z) → driver kernel sd_read_block au
  boot → ASSERT mem[$01:5D40..] contient le pattern.
- `make test-oricos-sd` target dédiée.
- 502 tests OK (+1 nouveau).

### OricOS — pas de change code
- Driver `kernel_sd_read_block` (livré en 2.j.0) prouvé fonctionnel
  end-to-end.

---

## [2026-05-08] — Sprint 2.j.0 : driver bloc SD minimal

### Phosphoric (oric2-golden-model)
- **`include/io/sd_device.h` + `src/io/sd_device.c`** : émulation device
  SD bloc minimal (8 registres I/O à $0320-$0327, LBA 24-bit,
  CTRL/DATA, lecture 512B).
- **Option `--sd-image FILE`** : binde une image SD au device émulé.
- Hook io_read/write_callback. Cleanup sd_close à emulator_cleanup.

### OricOS → 0.21.0 (Sprint 2.j.0)
- **`kernel_sd_read_block (LBA 16-bit en $30/$31, dest via DP_PCPTR)`** :
  lit 1 bloc 512 octets via device SD.
- Constantes SD_LBA_*/CTRL/DATA. Test au boot : sd_read_block(LBA=0,
  dest=$01:5D40). Sans image : 512 zéros. Avec image : contenu.
- 501 tests OK.

### Reportés OS-2.j.1+
- Test fonctionnel image SD.
- Parser FAT32 boot sector + root dir.
- Charger app depuis SD au lieu de .incbin.

---

## [2026-05-08] — Sprint 2.m.1 : première app asm standalone ✨

### OricOS → 0.20.0
- **`apps/hello/hello.s`** : première app userland standalone, source
  asm 65C816 mode N, position-independent.
- **`apps/hello/Makefile`** + **`tools/oricos-bundle.py`** : pipeline
  build complet asm → flat binary → bundle .oosobj.
- Top Makefile build apps avant kernel ; `.incbin` du bundle dans
  `kernel.s`.
- **Premier exécutable userland** d'OricOS dont la source vit hors
  du kernel. Démontre la viabilité du pipeline d'apps tierces.
- Sprint 4 (llvm-mos C) utilisera le même pipeline avec un compilo C.

### Demo
"OricOS v0.7" + "YABZ" inchangé visuellement, mais le 'Z' provient
désormais d'une app standalone (`apps/hello/hello.s`) buildée
indépendamment du kernel et embedded via `.incbin`.

### Tests
501 tests OK.

---

## [2026-05-08] — Sprint 2.l.1 : APP LOADER COMPLET ✨

### OricOS → 0.19.0
- **`kernel_app_exec`** orchestre validate + find_code + alloc_bank +
  copy code section + JSL self-modifying.
- **App test** = `ldx #'Z'; lda #$01; cop #$AA; rtl` exécutée en bank 7.
- ASSERT côté Phosphoric : `mem[$BBAB] = 'Z'` écrit par l'app via
  syscall SYS_PRINT_CHAR depuis bank dédiée. `BUNDLE_APP_BANK = $07`.
- **Démo SDL2** : "OricOS v0.7" + "YABZ" — Y du kernel syscall + AB hex
  + **Z de l'app loader**.

### Notes techniques
- Bug ld65 contourné : `STA al` sur label CODE génère bank=$00 par
  défaut. Workaround = `STA [dp]` long indirect avec bank explicite
  en DP zero page.
- 501 tests OK. Golden frame régénérée.

---

## [2026-05-08] — Sprint 2.l.0 : kernel_bundle_find_code (parse sections)

### OricOS → 0.18.0
- `kernel_bundle_find_code` : parse sections du bundle, trouve CODE,
  retourne size + offset 16-bit dans BUNDLE_FOUND_SIZE/OFFSET.
- Constantes BNL_HDR_SIZE / BNL_SEC_SIZE / BNL_SEC_* offsets.
- Test au boot : sur bundle_test inline, SIZE=$0002, OFFSET=$0010.
  ASSERT mem[$015498..$01549B] = 02 00 10 00.
- v0.1 (kernel_app_exec) reportée.

### Notes
- `cpx` n'a pas de long-absolute. Utilise tmp DP zero page pour cpx zp.
- 501 tests OK.

---

## [2026-05-08] — Bug majeur Phosphoric P mode N corrigé + OS-2.k.1 finalisé

### Phosphoric (oric2-golden-model)
- **Bug critique 65C816** : COP/BRK/IRQ/NMI/PHP/PLP/RTI manipulaient
  cpu->P avec masque mode E (`& ~FLAG_BREAK | FLAG_UNUSED`) en toutes
  circonstances. En mode N, ces bits sont X/M (index/accumulator width).
  Résultat : RTI/PLP/COP corrompaient les flags width.
- **Manifestation OricOS** : `cop #$AA` syscall → RTI restorait X=0
  (16-bit) → `ldy #$00` du caller consommait 2 bytes au lieu de 1 →
  crash $00:0000.
- **Fix** : conditionnement sur `cpu->E` aux 6 emplacements ; en mode N
  push/pull P entier sans masque, troncation X/Y si transition X 0→1.
- **Test isolé** ajouté (`test_dp_indirect_long_y_bank1_validate_pattern`).
- Référence : WDC W65C816S §7, §A.21/A.32.

### OricOS → 0.17.0 (Sprint 2.k.1 finalisé)
- `kernel_bundle_validate` ré-activé, ASSERT mem[$01549C]=$00 OK.
- 501 tests passent (500 → +1).

---

## [2026-05-08] — Sprint 2.k.1 : format bundle apps (ADR-08 partiel)

### OricOS → 0.16.0 (Sprint 2.k.1)
- Spec format bundle "OOS\x01" : header 8B + section entries 8B + data.
- `kernel_bundle_validate` code écrit (magic + version check).
- `bundle_test` inline header + section CODE 2B placeholder.
- 500 tests OK.

### Known issue tracée
- **PH-bug-dp-indirect-Y-bank1** : crash mystérieux dans `lda [dp],Y`
  quand DP_PTR_BK=$01 et offset spécifique du bundle_test. Print_string
  utilise le même opcode sans souci. Position-dépendant. Validate
  désactivé au boot en attendant fix. À investiguer via test unitaire
  isolé dans Phosphoric.

---

## [2026-05-08] — Sprint 2.g.1 : refactor scheduler TCB-based (ADR-14)

### ADR-14 ratifiée
- Table fixe **16 TCBs** (20 octets chacun) en bank 1 $5C00.
- **Bitmap free 16 bits** à $5B00 pour réutilisation PID après mort.
- Champs : PID, STATE, PRIO, PARENT, saved_S 16-bit, entry_pc, code_bank,
  data_bank, stack_bank, flags, name 8B.
- Réf. : SymbOS/Z80 (8) et GS/OS/65C816 (32). 16 = compromis OricOS.
- Cf. workspace `CLAUDE.md` §2.

### OricOS → 0.15.0 (Sprint 2.g.1)
- Init TCB_1 (task A RUNNING) + TCB_2 (task B READY) au boot, bitmap $07.
- Scheduler do_switch refactor : sauve/charge via TCB_1_S/TCB_2_S au
  lieu de globales TASK_A_S/TASK_B_S. Met à jour STATE à chaque swap.
- TASK_CUR sémantique 0/1 → 1/2 (PID).
- v0.2 reporté : `task_create` dynamique, alloc PID via bitmap scan,
  scheduler avec priorité réelle.

### Validation
- Test 2.a + test visuel pixel-identique au golden : zéro régression.
- 501 tests OK.

---

## [2026-05-08] — PH-CI-visual : test visuel golden frame OricOS

### Phosphoric (oric2-golden-model)
- `tests/integration/test_oricos_visual.c` : boot OricOS jusqu'à STP,
  render frame, compare pixel-par-pixel à `tests/golden/oricos_boot.ppm`.
  FAIL au 1er offset divergent avec (x, y, channel).
- `make test-oricos-visual` target dédiée.
- Test gated : SKIP si kernel.bin OU golden absent.
- 501 tests OK (500 → +1).

### Why
Prévient régressions render type H4 (fonte char manquante en bank 0
$B400-$B7FF). Test FAIL détecté empiriquement quand 1 byte du golden
est corrompu.

---

## [2026-05-08] — PH-debug816 : extension debugger 65C816

### Phosphoric (oric2-golden-model)
- `cpu816_get_state_string` : formate l'état CPU mode E/N pour le
  debugger. Affiche tous les registres natifs (C 16-bit, X/Y/S 16-bit,
  D, DBR, PBR, P avec flags MX en mode N).
- Debugger `regs`/`r` route selon `cpu_kind`. `set <reg> <val>` étendu
  pour 65C816 : C/A, X, Y, S, PC, P, D, DB, PB.
- 2 tests unitaires + Makefile fix (TEST_DEBUGGER_SRCS / TEST_COVERAGE_SRCS
  incluent maintenant cpu65c816.* et cpu_core.c).
- 500 tests OK (498 → +2).

---

## [2026-05-08] — Sprint 2.i.1 : modèle erreur kernel

### OricOS → 0.14.0
- `kernel_panic` (A=code) : "PANIC <hex>" + STP + stocke à PANIC_CODE.
- `kernel_print_hex8` / `kernel_print_nibble` : helpers hex chars.
- Test : `print_hex8(#$AB)` au boot → "AB" en VRAM. Démo SDL2 : "YAB".
- v0.2 (reportés) : log ring buffer, SYS_PANIC syscall, stack trace.

---

## [2026-05-08] — Sprint 2.h.1 : bank allocator avec free

### OricOS → 0.13.0
- **Free list LIFO 16 entries** : `BANK_FREE_LIST` ($0154A0) +
  `BANK_FREE_TOP` ($0154B0).
- `kernel_alloc_bank` étendu : pop free list d'abord, sinon bump.
- `kernel_free_bank` (nouvelle routine) : push avec drop si pleine.
- Test : alloc 3 / free 1 / alloc → LIFO retourne le freed bank.
- v0.2 (reportée) : bitmap allocator pour fragmentation arbitraire.

### Phosphoric — pas de change code
- Test `test_oricos_sprint2a` étendu : ASSERT `mem[$015463] = $05`
  pour valider le LIFO.

---

## [2026-05-08] — Bug fixes Phosphoric (PH-fix-txs + PH-fix-dp-indirect)

### Phosphoric (oric2-golden-model)
- **PH-fix-dp-indirect** : 8 opcodes DP indirect 16-bit `(dp)` ajoutés
  ($12/$32/$52/$72/$92/$B2/$D2/$F2). Helper `addr816_dp_indirect`.
  2 tests unitaires.
- **PH-fix-txs** (faux positif) : investigation conclue — comportement
  WDC correct, le "bug" était dans OricOS. Code Phosphoric inchangé,
  commentaire WDC §A.32 ajouté. 2 tests de non-régression.
- 498 tests OK (494 → +4 nouveaux).

### OricOS → 0.12.0 (cleanup)
- `kernel_entry` : `ldx #$FF; txs` → `lda #$01FF; tcs` (stack page 1).
- `kernel_print_char` : `STA [dp]` long indirect → `STA (dp)` natif
  (suite à PH-fix-dp-indirect).

---

## [2026-05-08] — Sprint 2.f.1 : COP syscall (ADR-13 ratifiée)

### ADR-13 ratifiée
- Mécanisme syscall : `COP #imm` + table dispatch.
- Numéro syscall en `A`, args en `X`/`Y`.
- Vecteur $00FFE4 mode N → trampoline bank 0 $0150 → kernel handler bank 1 $5700.
- Référence : GS/OS sur Apple IIgs.
- Cf. workspace `CLAUDE.md` §2 (déplacée de §3).

### OricOS → 0.11.0 (Sprint 2.f.1)
- Segment `COP_HANDLER` à $5700.
- `kernel_cop_handler` v0.1 : dispatch hardcoded (cmp #$01 → SYS_PRINT_CHAR).
- v0.2 (reporté) : table de pointers indexée.

### Phosphoric (oric2-golden-model)
- `--kernel` installe trampoline COP $0150 + vecteurs $00FFE4 / $00FFF4.
- Test `test_oricos_boot` étendu : ASSERT `mem[$BBA8] = 'Y'` après COP au boot.

### Demo
"OricOS v0.7" ligne 1 + "Y" ligne 2 (sortie du `cop #$AA` syscall).

---

## [2026-05-08] — Sprint 2.d.1 : driver clavier OricOS

### OricOS → 0.9.0 (Sprint 2.d.1)
- Routines PSG bus (`psg_set_reg/write_data/read_data`) via VIA PCR.
- `kernel_kbd_init` (DDRA/DDRB/PSG R7) + `kernel_kbd_scan` (8 colonnes
  via PSG R14) + matrice 8 octets à `$015470`.
- Appel du scan depuis IRQ T1 handler (période T1 augmentée 512→4096
  cycles pour laisser ~3000 cycles/slot aux tasks).

### Phosphoric — pas de change code
- Test `test_oricos_sprint2a` valide implicitement le driver clavier.
- Bug latent identifié : TXS en mode N + X=1 copie seulement low byte
  de X dans S → stack OricOS tourne en bank 0 page 0 par chance.
  Tracé en dette technique (`PH-fix-txs`).

### oric2-golden-model
- Repo dédié créé sur github (benedictemarty/oric2-golden-model, privé)
  et gitea (chipinette/oric2-golden-model, privé). Phosphoric officiel
  reset à a934cc8 (état pré-Oric 2). Le travail Oric 2 + OricOS
  integration tests vivent désormais sur oric2-golden-model.

---

## [2026-05-08] — Backlog révisé suite point critique architecte

### Added
- **`BACKLOG.md`** racine workspace : source de vérité priorisée
  (NOW / NEXT / LATER), liste sprints OS-2.d à OS-2.m, dépendances,
  estimations en jours-semaines réalistes.
- **6 ADRs ouvertes** dans `CLAUDE.md` §3 :
  - ADR-13 Mécanisme de syscall (COP / WAI / call gate)
  - ADR-14 Format TCB et structure interne tâche
  - ADR-15 Isolation mémoire post-v1 (MMU custom HDL ?)
  - ADR-16 Driver model (event/polling/IRQ)
  - ADR-17 API kernel publique exposée à userland (ABI)
  - ADR-18 Sort du 6502 Phosphoric (retrait / cohabitation)
- **4 décisions stratégiques** en attente (DEC-1 à DEC-4).
- **Phase 7** dans `Phosphoric/ROADMAP` : outillage post point critique
  (PH-debug816, PH-CI-visual, PH-bootrom).

### Changed
- **Roadmap OricOS révisée** (`OricOS/CLAUDE.md` §7) : insertion
  Sprints 2.d → 2.m comme prérequis stricts avant Sprint 3 GUI. La
  version d'origine sautait directement de 2.c à 3, ce qui n'était
  pas viable (manque clavier, console générique, syscalls, TCB,
  allocator avec free, modèle d'erreur).

### Why
Point critique architecte senior 2026-05-08 a identifié :
- **5 ADRs vaporware** ratifiées sans implémentation (05, 06, 07,
  08, 09).
- **Kernel 380 LOC asm** = proof-of-concept, pas un OS.
- **Sprints <1 jour** non réalistes vs charge restante.
- **Tests sans validation visuelle** : bug H4 fonte passé inaperçu.
- **6 décisions implicites** non documentées.

Ce commit cristallise les recommandations sans changement de code
(documentation seule). Les sprints OS-2.d à OS-2.i deviennent
prérequis explicites pour Sprint 3.

---

## [2026-05-08] — Démo OricOS visible (kernel autonome)

### Diagnostic & fix H4 — fonte char absente
Symptôme : `oric1-emu --kernel build/kernel.bin` affichait écran noir
malgré banner "OricOS v0.7" écrit en VRAM (test unitaire OK).
Cause : `video.c:get_charset_byte()` lit la fonte depuis bank 0
`$B400-$B7FF` ; sans la ROM Oric 1 (skippée par `--kernel`), zone à
zéro → tous les chars rendus en pixels noirs.

### Phosphoric → 1.22.8-alpha
- **`--kernel FILE`** : charge kernel.bin OricOS, force `oric2`/`65C816`,
  installe trampolines RESET/IRQ/NMI bank 0 + vecteurs natifs E/N.
- Boucle main loop tolérante au STP en mode kernel (fenêtre SDL2 reste).
- Test `test_oricos_sprint2a` étendu (attribute byte INK + banner).
- 494 tests passent (=).

### OricOS → 0.8.0
- **`data/charset.bin`** : fonte 1024 oct extraite de `basic11b.rom`
  offset `$3B78` (autonomie totale vis-à-vis ROM Oric 1).
- **Segment `CHARSET`** dans kernel.cfg à `$5800` ; `kernel_install_charset`
  copie la fonte vers `$00B400` au boot.
- **Attribute byte `$07`** (INK 7 blanc) écrit à `$00BB80` pour banner visible.

### Demo
```
cd OricOS && make
cd Phosphoric && ./oric1-emu --kernel ../OricOS/build/kernel.bin
```
"OricOS v0.7" affiché en blanc, scheduler préemptif round-robin actif
en arrière-plan (VIA T1).

---

## [2026-05-08] — Bootstrap workspace Oric 2

- ADR-01 à ADR-12 ratifiées (cf. `CLAUDE.md` §2).
- DAT v1.0 IEEE 42010 (`docs/DAT.md`).
- MEMORY_MAP.md v1.0 (`docs/MEMORY_MAP.md`).
- Phosphoric importé depuis git.nagominosato.fr:6775/chipinette/Phosphoric.
- OricOS sous-projet créé (kernel asm 65C816 + ld65/ca65).
- Track B clos : 65C816 cycle-exact + memory map 256K + paravirt + compositor.
- Sprints OricOS 0 → 2.c+ : boot, scheduler préemptif, VIA T1, bank
  allocator, driver console minimal, fonte embarquée.
