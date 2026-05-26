# BACKLOG — Workspace Oric 2

> Source de vérité pour le travail à venir. Issu d'un point critique
> architecte senior OS du 2026-05-08 (cf. annexe §6 ci-dessous).
> Met à jour la ROADMAP affichée dans Phosphoric/ROADMAP et OricOS/CLAUDE.md §7.

**Convention** : sprints estimés en charge nominale single-developer
sessions. Les sprints "ratifiés clos" précédents (Sprint 0 → 2.c+) ont
duré <1 jour ; les sprints à venir sont calibrés en **semaines réelles**
parce qu'ils touchent des fondamentaux non-triviaux.

---

## NOW — itération courante (2026-05-09 → fin Phase 1, programme état-de-l'art)

### Priorité 0 — Programme état-de-l'art (kickoff 2026-05-09)

Cf. `~/.claude/projects/-home-bmarty-oric2/memory/programme_remise_au_top.md` (mémoire). Programme 8 semaines, 4 axes parallèles. Phase 0 close 2026-05-09 ; Phase 1 (assainissement) démarre.

| ID | Titre | Estim. | Phase | Pré-req |
|----|-------|--------|-------|---------|
| ~~PHASE0~~ | ✅ **clos 2026-05-09** : décisions cadre (ADR-18 retrait net, CI=GitHub Actions public, doc=GitHub Pages) + ratification ADR-16/17/18 + parking ADR-15 v2 + DEC-1/DEC-2 actées + moratoire ADR formalisé + squelette `docs/CONTRACT_HDL.md` créé. | — | Phase 0 | — |
| **HW-1** | Contrat HDL ↔ golden model — détail des sections vides du squelette `docs/CONTRACT_HDL.md`. **Promu NOW P0 suite DEC-2.** | 4-5 j | Phase 1 | PHASE0 ✅ |
| **PH-2.a** | Étape 1.A ADR-18 — décrochage types partagés (extraire `memory_t`, `cpu_irq_source_t`, `cpu_flags_t`, `IRQF_*` vers `include/cpu/cpu_types.h`). | ½ j | Phase 1 | PHASE0 ✅ |
| **PH-2.b** | Étape 1.B ADR-18 — campagne validation 65C816 mode E par défaut. **Bloquante** : 541 tests verts + bench ≤ 5 % + boot interactif ROM 1.0/1.1. | 1 j | Phase 1 | PH-2.a |
| **PH-2.c** | Étape 1.C ADR-18 — suppression effective cpu6502.c, opcodes.c, addressing.c, cpu_core_vtable_6502, test_cpu.c, réécriture test_cpu_core.c. | ½ j | Phase 1 | PH-2.b ✅ go |
| **PH-2.d** | Étape 1.D ADR-18 — création `docs/adr/0018-retrait-6502.md` MADR + traçage CHANGELOG. | ½ j | Phase 1 | PH-2.c |
| **PH-cleanup-zombie** | Retrait `kernel_hires2_*` zombie (legacy ADR-19 v2, jamais appelé en boot courant). | ½ j | Phase 1 | — |
| ~~OS-2.f.v2~~ | ✅ **clos 2026-05-24** : table dispatch 64 entrées × 2B à `$01:5750`, 18 syscalls câblés + 45 × `sys_invalid`. Dispatcher v0.2 (`kernel_cop_handler`). Tests : `test_syscall_dispatch_invalid`, `test_syscall_yield`, `test_syscall_table_size`. | — | Phase 1 | PHASE0 ✅ |

### Priorité 1 — Fondations OS bloquantes (avant tout sprint GUI)

| ID | Titre | Estim. | Sprint | Pré-req |
|----|-------|--------|--------|---------|
| ~~OS-2.d~~ | ✅ **clos 2026-05-23** (ADR-22 ratifiée : clavier Oric 2 paravirtualisé hybride. Contrôleur KBD2 `$0350-$035F` IRQ-driven, ring 16 keycodes `$5860`, SYS_GET_KEY/READ_CHAR câblés. Modèle Phosphoric `kbd2_device`. Matrice virtuelle guest différée OS-5.a). | — | OricOS | ADR-16/22 ✅ |
| ~~OS-2.e~~ | ✅ **clos 2026-05-23** (print_char/print_string/cursor v0.1 + OS-2.e.2 : CR `\r` + scroll up `kernel_scroll_up`. Reporté : attribut couleur par ligne). | — | OricOS | — |
| ~~OS-2.f~~ | ✅ **clos 2026-05-08** (v0.1 : COP handler avec dispatch hardcoded SYS_PRINT_CHAR. Table dispatch reportée v0.2). | — | OricOS | — |
| ~~OS-2.g~~ | ✅ **clos 2026-05-08** (v0.1 : TCB table 16 + bitmap, scheduler refactored, ADR-14 ratifiée. v0.2 task_create dynamique reporté). | — | OricOS | — |
| ~~OS-2.h~~ | ✅ **clos 2026-05-08** (v0.1 : free list LIFO 16 entries. Bitmap reportée v0.2). | — | OricOS | — |
| ~~OS-2.i~~ | ✅ **clos 2026-05-08 (v0.1) + 2026-05-24 (v0.2)** : v0.1 kernel_panic + print_hex8. v0.2 : log ring buffer `$54E0` (8 entrées level/code), codes nommés (ERR_*), wiring panic/cop_invalid/alloc_none, SYS_PANIC. | — | OricOS | — |

### Priorité 1bis — Outillage Phosphoric

| ID | Titre | Estim. | Pré-req |
|----|-------|--------|---------|
| ~~PH-debug816~~ | ✅ **clos 2026-05-08** : `cpu816_get_state_string` mode E/N, debugger `regs`+`set` étendus, 2 tests unitaires. | — | — |
| ~~PH-CI-visual~~ | ✅ **clos 2026-05-08** : test_oricos_visual + golden PPM + comparaison pixel-perfect. Empêche régressions render comme H4 (fonte). | — | — |
| ~~PH-bootrom~~ | ✅ **clos 2026-05-24** : `src/oric2_bootrom.{c,h}` (`oric2_bootrom_load`) — image ROM 16K (reset $C000 + trampolines IRQ/NMI/COP en ROM + vecteurs natifs/émulation). `--kernel` boote via cette ROM au lieu de patcher `mem.rom[]` + stubs RAM. Test `test_oricos_bootrom_boots`. 546 tests. | — | — |

---

## NEXT — itération suivante (estim. 2026-05-23 → 2026-06-30)

### Priorité 2 — Stockage & ressources

| ID | Titre | Estim. | Pré-req |
|----|-------|--------|---------|
| ~~OS-2.j~~ | ✅ **clos 2026-05-08 (v0.8)** : pipeline FAT32 lecture seule complet : `fat_init` + `fat_open` + `fat_read_cluster` + `fat_next_cluster` (v0.2) + `fat_read_file` (v0.3, multi-cluster jusqu'à EOC). Permet de charger un fichier de taille arbitraire (≤ 64 KiB). v0.4 reportés : fichiers > 64 KiB, BPS != 512, cluster ≥ 16384, subdirs. | — | OricOS | — |
| ~~OS-2.k~~ | ✅ **clos 2026-05-08** (v0.1 : spec + `kernel_bundle_validate` fonctionnel après fix bug PH P-mode-N). | — | OricOS | — |
| ~~PH-bug-dp-indirect-Y-bank1~~ | ✅ **clos 2026-05-08** : pas un bug `[dp],Y`. Vraie cause = COP/BRK/IRQ/NMI/PHP/PLP/RTI corrompaient bits X/M de P en mode N (masque mode E `& ~FLAG_BREAK \| FLAG_UNUSED`). RTI post-COP restaurait X=0 → `ldy` du caller consommait 2 bytes au lieu de 1. Fix : conditionnement sur `cpu->E` aux 6 emplacements. |
| ~~OS-2.l~~ | ✅ **clos 2026-05-09 (v0.2)** : v0.1 (validate + find_code + alloc + copy + JSL) + **v0.2 app multi-cluster depuis SD** : pipeline `fat_open → fat_read_file → app_exec` validé sur bundle 527B avec section CODE à offset 520 (cross-cluster). v0.3 reporté : code > 256B, sections multiples, free bank, sandbox. | — | OricOS | — |
| ~~OS-2.m~~ | ✅ **clos 2026-05-08** : `apps/hello/hello.s` source standalone, pipeline build via ca65+ld65+`tools/oricos-bundle.py`. Bundle .oosobj embedded dans kernel via `.incbin`, exec via `kernel_app_exec`. Premier userland OricOS hors kernel. | — | OricOS | — |

### Priorité 2bis — Toolchain userland C

| ID | Titre | Estim. | Pré-req |
|----|-------|--------|---------|
| ~~TC-llvmmos~~ | ✅ **clos 2026-05-09** : investigation documentaire (cf. `docs/TC-llvmmos.md`). Constats : llvm-mos n'implémente PAS le mode N 16-bit registres (issue #321) ni le banking 24-bit (issue #320), tous deux ouverts depuis 2023-2024. ADR-05 révisée v2 : userland C mode N **8-bit native** (M=1, X=1), apps **mono-bank** ≤ 64 KiB linéaire. DEC-3 actée : llvm-mos conservé avec contraintes. Installation effective + PoC reportés au Sprint 4 (sous-tâches **TC-llvmmos-install**, **TC-llvmmos-target-oricos**, **TC-poc-hello-c**). | — | — |
| ~~**TC-llvmmos-install**~~ | ✅ **clos 2026-05-25** : llvm-mos v23.0.1 installé dans `$HOME/llvm-mos`. Target `mosw65816` validé : génère code 8-bit natif (M=1,X=1), conforme ADR-05 v2. PoC `int main()` compilé + désassemblé. | — | — |
| ~~**TC-llvmmos-target-oricos**~~ | ✅ **clos 2026-05-25** : target `mos-oricos` créé dans `$HOME/llvm-mos` + SDK versionné dans `OricOS/tools/oricos-sdk/`. crt0.S (`.init.000` DBR=PBR + `.call_main` + `.after_main` SYS_EXIT+RTL), link.ld (code@$0200, ZP imag-regs $89-$A8), oricos.h (18 syscalls ADR-17). App `apps/hello_c/hello.c` compilée → bundle .oos 600B valide. 563 tests Phosphoric verts. | — | — |
| ~~**TC-libc**~~ | ✅ **clos 2026-05-25** : `liboricos.a` (printf/sprintf/strlen/memset/memcpy/strcpy/strcat/strcmp/itoa/malloc/free/calloc). malloc = bump bank-local $__heap_start→$FFFF. `apps/test_libc/test_libc.oos` 9068B valide. |
| ~~**TC-poc-hello-c**~~ | ✅ **clos 2026-05-25** : hello_c (llvm-mos) s'exécute sous OricOS dans Phosphoric. Fix oricos.h (LTO LDA #N), fix sys_print_char (DBR userland), fix kernel_app_exec (copie 16-bit). Test `test_oricos_helloc` : 2/2 PASS. 563 tests verts. | — | — |

---

## LATER — backlog priorisé bas (Q3 2026+)

### Priorité 3 — GUI

| ID | Titre | Notes |
|----|-------|-------|
| ~~SP-3.a~~ | ✅ **clos 2026-05-09 (v0.2)** : implémentation ADR-12 (mode HIRES Oric 2 240×200×3bpp) + intégration compositor matériel (ADR-02). Module `video/hires_oric2.{c,h}` (8 tests unit) + `tests/integration/test_compositor_hires_oric2.c` (3 tests intégration). Pipeline validé : bank 128 → render ARGB → compositor host → compose → output. ADR-12 sort de l'état "vaporware". v0.3 reportés : intégration main loop SDL2 (`--video-mode oric2`), bank configurable, double-buffer. |
| ~~SP-3.b~~ | ✅ **clos 2026-05-09 (v0.2)** : `kernel_hires2_clear` + `kernel_fill_rect_aligned` (rectangles 8-px-aligned X) + boot kernel intégré (clear blue + rect red 80x80 centre). Bug Phosphoric ASL/LSR/ROL/ROR M=0 trouvé et fixé en cours de route (cf. Phosphoric/CHANGELOG). v0.3 reporté : `pixel_set` arbitraire, blit, bascule TEXT↔HIRES via registre I/O. | Pré-req SP-3.a ✅ |
| ~~SP-VRAM-1~~ | ✅ **clos 2026-05-09** : `src/io/vram_device.{c,h}` simulant 16 MiB SDRAM via I/O `$0330-$033C` + DMA synchrone. 9 tests unitaires (init, address round-trip, auto-increment, wrap, DMA bidirectionnel, LEN=0=64K). 524 tests OK (+9). | ADR-19 ✅ |
| ~~SP-VRAM-2~~ | ✅ **clos 2026-05-09** : `kernel_vram_write_block`, `kernel_vram_read_block`, `kernel_vram_dma` (OricOS) + test boot intégration (Phosphoric). 525 tests OK (+1). | SP-VRAM-1 ✅ |
| ~~SP-VRAM-3~~ | ✅ **clos 2026-05-09 (v0.2)** : pool LIVE banks 132-159 (= $84..$9F, 28 banks) suite ADR-20 ratifiée (banks 128-131 réservés framebuffer SVGA 800×600×4bpp). `kernel_alloc_live_bank`/`free_live_bank` (LIFO+bump). Robustesse DMA : timeout 256 polls. 526 tests OK. | SP-VRAM-2 ✅ |
| ~~SP-GPU-1~~ | ✅ **clos 2026-05-09** : `src/io/gpu_device.{c,h}` simulant GPU Blitter (ADR-21). v0.1 : CLEAR + FILL_RECT synchrones. Ports I/O $0340-$034F. 7 tests unitaires (init, args round-trip, CLEAR, FILL_RECT aligned, FILL_RECT mask intra-octet, opcode inconnu err, CLEAR full XVGA). 533 tests OK (+7). | ADR-21 ✅ |
| ~~SP-GPU-2~~ | ✅ **clos 2026-05-09 (v0.3)** : **ADR-21 complet** — toutes 5 commandes GPU implémentées : CLEAR + FILL_RECT + BLIT + LINE + **TEXT** (fonte 8×8 mono, null-terminated, color_fg only). 13 tests gpu_device unitaires. 541 tests OK. v0.4 reporté : color_bg, fonte variable, BLIT pixel-aligned/overlap. | SP-GPU-1 ✅ |
| ~~SP-GPU-3~~ | ✅ **clos 2026-05-09 (v0.3)** : **API kernel_gfx_* 100% complète** : `clear`, `fill_rect`, `blit`, `line`, `text`. Boot kernel pré-charge fonte + string en SDRAM puis appelle TEXT pour titrer "OS" sur la titlebar de window 1. 541 tests OK. v0.4 reporté : IRQ-based wait, refactor `kernel_hires2_*` legacy, fonte complète ASCII. | SP-GPU-1 ✅, SP-GPU-2 ✅ |
| **SP-GPU-HDL-1** | HDL ULX3S : controller GPU minimal (CLEAR + FILL_RECT). | SP-GPU-3 |
| **SP-GPU-HDL-2** | HDL ULX3S : BLIT engine. | SP-GPU-HDL-1 |
| **SP-GPU-HDL-3** | HDL ULX3S : LINE Bresenham. | SP-GPU-HDL-2 |
| **SP-GPU-HDL-4** | HDL ULX3S : TEXT engine + font ROM. | SP-GPU-HDL-3 |
| ⚙️ **SP-3.c** | v0.3 ✅ (clos 2026-05-09) : window manager basique + multi-fenêtre via BLIT + true drag (BLIT + CLEAR pos1) + title text "OS" via TEXT. PPM XVGA finale affiche 2 fenêtres distinctes (window 1 (20,10) avec "OS" titlebar bleu + window 2 (300,300) titlebar vert dragged depuis (50,80)). 541 tests OK. v0.4 reporté : window list / TCB par fenêtre, focus, close/minimize (DMA backing), event-driven drag. | SP-GPU-3 ✅ |
| **SP-3.d** | Toolkit minimal : font HIRES, label, button | Pré-req SP-3.b |
| ⚙️ **SP-3.e** | **v0.1 + v0.2 ✅ (2026-05-24)** : v0.1 ADR-24 souris + window manager (table 4 fenêtres `$5900`, hit-test/focus/move) + `kernel_wm_mouse_step`. v0.2 **event loop IRQ-driven** (IRQ MOU2 dans `kernel_irq_handler`, scheduler gaté sur T1 ; test injection clic→focus). **v0.3 ✅ (2026-05-24)** : GPU FILL_RECT16 (coords 16-bit packed, ADR-21 v0.2) + kernel_wm_redraw (clear + fenêtres aux positions 16-bit) + affichage XVGA débloqué (vram/gpu live, hires_oric2_xvga, --xvga/--xvga-screenshot). Desktop visible. **v0.4 ✅ (2026-05-24)** : main loop kernel persistant (NO_STP_FLAG posé par --kernel) + delta souris par événement → **drag fenêtre live** (clic→focus, drag→move+redraw, visible --xvga). v0.5 ✅ (2026-05-24)** : curseur dessiné par l'OS (kernel_wm_draw_cursor) + relative-mode SDL (pointeur capturé, LCtrl+RShift). v0.6 ✅ : backing-store curseur (motion). v0.7 ✅ : drag incrémental (dirty rect). v0.8 ✅ (2026-05-24)** : couleur titlebar selon focus (multi-dirty-rect jugé inutile : focus-change = full-redraw correct). | SP-3.c, OS-2.d, SP-VRAM-2 |
| ~~**SP-3.f**~~ | ✅ **clos 2026-05-24** : chrome de fenêtre — titre dans titlebar + bouton fermer (×). v0.1 : `kernel_wm_add` reçoit `WM_ARG_TITLE_LO/HI`, upload en SDRAM (`$012000+slot×$100`), `WM_TITLES[slot]=$01`, rendu TEXT16 blanc à `win_x+4,win_y+3`. v0.2 : "X" lightred à `win_x+w-10,win_y+3`, `_wm_close_btn_hit` (zone `[w-12..w-1,y..y+13]`), `kernel_wm_close` (clear slot+WM_COUNT--+focus rebind). Fix GFX_STR_HI `$00→$01`. Tests Phosphoric : `test_wm_window_title` + `test_wm_close_button`. 549 tests verts. | SP-3.e ✅ |
| ~~**SP-3.g**~~ | ✅ **clos 2026-05-24** : taskbar fixe bas desktop XVGA. `kernel_taskbar_draw` : fond darkgray `(0,755,1024,13)` + séparateur blanc `y=755` + boutons par slot `WM_F_USED` (`btn_x=4+i×124`, `w=120`, `h=10`, `y=757`) lightblue si focus sinon darkgray, texte titre SDRAM ou fallback `"WinN\0"` (`$011100`). `kernel_taskbar_hit` : hit `MOUSE_Y≥755` + BTN_LEFT → `slot=(X-4)/124` → `kernel_wm_set_focus+redraw+cursor`. Intégré en fin de `kernel_menu_draw` (après dropdown) et priorité absolue dans `wm_step_not_drag`. Tests : `test_taskbar_render` + `test_taskbar_click`. 551 tests verts. | SP-3.f ✅ |
| ~~**SP-3.h**~~ | ✅ **clos 2026-05-24** : maximize (□) et minimize (_) dans le chrome des fenêtres. `kernel_wm_maximize` bascule normale↔maximisée (x=0,y=14,w=1024,h=741), sauvegarde coords dans `WM_SAVED_RECTS` ($015AA9, 4×8B) via `STA f:WM_SAVED_RECTS,X` (long,X `$9F`). `kernel_wm_minimize` : clear `WM_F_VISIBLE` + `WM_STATE_HIDDEN`. `kernel_taskbar_hit` restaure au clic. `_wm_chrome_hit` retourne 0/1/2/3 (none/×/□/_). Drag désactivé sur fenêtre maximisée. **Fix critique** : `rep #$20` explicite dans `_crh_test_max/_crh_test_min` (bug tracking mode ca65 assemblait `sbc #12` en 8-bit → corruption ZP $22-$24 → 4 régressions corrigées). `WM_CRH_TMP` ($25-$2A) scratch dédié. Tests : `test_wm_states_init` + `test_wm_maximize` + `test_wm_minimize_restore`. 554 tests verts. | SP-3.g ✅ |

| ⚙️ **SP-3.m — GUI × multitâche** (arc, cadré 2026-05-25 ; pré-req multitâche OS-2.g v2.b ✅) | **Objectif** : une app C tourne comme tâche, possède une fenêtre, dessine dedans, reçoit le clavier au focus, sa fenêtre se ferme à l'exit. **Décision de conception (ADR-06/19/21)** : modèle **backing-store + compositing façon GrafPort** (cf. QuickDraw II IIgs) — chaque fenêtre = un backing store SDRAM + descripteur `{base24, w, h}` ; l'app dessine en **coordonnées LOCALES** (handle fenêtre + x/y), le kernel résout `base+offset`, le compositor (GPU BLIT, ADR-21) place les backing stores sur le XVGA selon position/z-order. **L'app est indépendante de l'adresse physique XVGA** (pas de direct-to-screen ni d'update-events). **Découpage** : **G.1** `WM_OWNER[8]` (pid propriétaire par slot) + lien au spawn. **G.2** `SYS_WIN_CREATE` (alloue fenêtre + backing store SDRAM + descripteur, retourne handle). **G.3** clavier→focus (`WM_FOCUS`→`WM_OWNER`→destinataire ; READ_CHAR d'une tâche non-focus bloque). **G.4** dessin fenêtré : `SYS_GFX_*` ciblent le backing store du caller en coords locales (clip = taille buffer). **G.4bis** compositor `kernel_wm_compose` (BLIT backing stores → XVGA, sur redraw move/focus/contenu) — remplace le redraw direct actuel. **G.5** exit→close (teardown ferme la fenêtre owner). **G.6** app C démo fenêtrée. **Risques** : compositor (G.4bis, le plus lourd, = raison d'être d'ADR-21) ; clavier↔focus↔blocage (G.3) ; `KBD_WAITER` unique (OK v1 1-focus ; polish #1 signaux génériques le lève) ; churn tests WM. **Effort** ~2-3 sem (v1.a G.1+G.2+G.5, v1.b G.3+G.4+G.4bis, puis G.6). **CLOS 2026-05-25** : G.1 ✅, G.2 ✅, G.3 ✅, G.4 ✅, G.4bis ✅, G.5 ✅, G.6 ✅ (`apps/win_hello/win.c` + `SYS_WIN_FLUSH`, `test_oricos_win_app` — chaîne complète depuis C, 564 verts). **Arc SP-3.m terminé.** | SP-3.h ✅, OS-2.g v2.b ✅ |

| ⚙️ **SP-3.n — Event Manager / Control / Dialog (modèle GeoWorks)** (arc, cadré 2026-05-26 ; pré-req SP-3.m ✅) | **Objectif** : exposer aux apps C une couche GUI **déclarative et événementielle** façon **GeoWorks/GEOS** (préférée au TaskMaster impératif d'Apple IIgs — cf. `docs/REFERENCES_ART.md`). **Principe directeur** : l'app **déclare** son UI comme des *tables de données* (menus/dialogues/contrôles), le système **rend + exécute** ; séparation **GenUI/SpecUI** (UI générique déclarée / rendu « look » spécifique changeable sans recompiler les apps). **Bascule de modèle** : on remplace les callbacks kernel (`_wm_invoke_active_cb` `jsr (vec,X)` cross-bank) par un **MainLoop → messages** sémantiques traités par un `switch` côté app. **Tout au niveau QDF** (backing stores, coords locales, compositor — jamais d'adresse XVGA exposée). **Découpage** : **G.1** file d'événements bas niveau bank 1 (ring 10B/event : what/message/mods/where_x16/where_y16/when ; alimentée par IRQ KBD2/MOU2 → unifie ring clavier `$5860` + `kernel_wm_mouse_step`). **G.2** `SYS_MAIN_LOOP` (bloquant via block/wake ADR-25) + `SYS_EVENT_AVAIL` (non-bloquant) ; MainLoop traite drag/menu/close en **un événement par appel** (état gardé entre appels → pas de boucle longue sous `Forbid`, préemption saine) et rend des messages (`MSG_MENU_ITEM`/`MSG_ICON`/`MSG_KEY`/`MSG_CONTENT`/`MSG_CLOSE`) ; migration de `kernel_wm_mouse_step` ; **retrait des callbacks kernel**. **G.3** modèle déclaratif d'UI (structure GenUI : fenêtre + menus + contrôles en table) ; SpecUI v1 = rendu GEOS/Ensemble au niveau QDF (réutilise `menu_defs`/`ICON_TABLE`/widgets managés). **G.4** contrôles génériques (bouton `GenTrigger`, checkbox `GenBoolean`) déclarés ; tracking dans MainLoop → `MSG_CONTROL`+id ; l'app lit l'état final (pas de `NewControl`/`SetValue` un par un). **G.5** `SYS_DO_DLGBOX` + **command table** (façon `DB_*` GEOS C64 : `DB_POSITION/DB_TEXT/DB_CHECKBOX/DB_OK/DB_CANCEL/DB_END`) → exécute le dialogue modalement (réutilise WM_MODAL/SP-3.j) et rend l'item terminal + état des contrôles. **G.6** `SYS_ALERT` (command tables pré-câblées OK / OK-Cancel / Yes-No). **G.7** app C démo : menu + dialogue déclaratifs, boucle MainLoop. **ABI** : ~5-6 syscalls dans les slots réservés `$15-$3F` (enveloppe ADR-17 existante ; moins que le cadrage IIgs écarté). **ADR à instruire** : « modèle GUI déclaratif GenUI/SpecUI (GeoWorks-like) » — révise/étend ADR-06 (SymbOS-like → déclaratif) ; à ratifier à ≥50 % d'impl (moratoire §10). **Risques** : churn tests (migration menus/widgets/modal vers MainLoop+déclaratif → filet de tests obligatoire) ; `KBD_WAITER` unique → motive enfin les **signaux multi-bits génériques** (polish #1, ADR-25) ; encodage command tables en C llvm-mos (tableau `const` d'octets — trivial, on prend l'esprit déclaratif pas le moteur objet « Goc »). **Effort** ~3-4 sem (G.1+G.2 événements ; G.3+G.4 déclaratif/contrôles ; G.5+G.6 dialogues ; G.7 démo). **Avancement 2026-05-26** : **G.1 ✅** (`event.s`, `EVENT_RING` $015880 16×10 o, push_key/_mouse/_pop, IRQ postent en coexistence, 565 verts). **G.2 ✅** (`SYS_GET_NEXT_EVENT` $16 bloquant block/wake + `SYS_EVENT_AVAIL` $15, `EVENT_WAITER`+`kernel_event_wake`, record en ZP $D0-$D9, `test_oricos_event_syscall`, 566 verts). **G.3a ✅** (`SYS_MAIN_LOOP` $17 : événements bruts → messages sémantiques `MSG_KEY`/`MSG_CONTENT` via `_ml_classify`+hit-test, skip MSG_NULL ; `kernel_wm_mouse_step` inchangé = focus/drag WM-auto ; `test_oricos_mainloop_message`, 567 verts). **G.3b ✅** (`SYS_UI_DEFINE` $18 : table GenUI déclarative `GU_WINDOW`/`GU_TITLE`/`GU_END` parsée → fenêtre créée ; fix race `_ml_classify` WM_ARG via ADR-25 Disable/Enable ; sprint2a durci compteurs 8-bit ; `test_oricos_ui_define`, 568 verts). **G.3c ✅** (`_ml_classify` classe le chrome : barre de menu→`MSG_MENU`, case fermeture→`MSG_CLOSE` ; `WM_APP_DRIVEN` : en mode app le shell ne ferme plus, l'app décide — GeoWorks ; `test_oricos_mainloop_close`/`_menu`, 570 verts). **Arc G.3 (a/b/c) complet.** **G.4 ✅** (`_ml_classify` : clic sur contrôle bouton `_wm_widget_hit`→`MSG_CONTROL`+id ; callback kernel en coexistence v1 ; `test_oricos_mainloop_control`, 571 verts ; tag `GU_BUTTON` déclaratif reporté à G.7). **G.5 ✅** (`SYS_DO_DLGBOX` $19 : command table GEOS `DB_*`→ fenêtre modale + boutons OK/Cancel + boucle modale ; **UI-modal** WM_MODAL + tâche bloque via `kernel_event_wait` mais rend le CPU ; `test_oricos_dlgbox`, 572 verts ; task-modal parqué v2). **G.6 ✅** (`SYS_ALERT` $1A : OK/OK-Cancel/Yes-No pré-câblés, type en X ; réutilise la boucle modale DoDlgBox via `jmp ddb_show` ; `test_oricos_alert`, 573 verts ; texte message + libellés distincts = polish). **G.7 ✅ — ARC SP-3.n CLOS** (`apps/gui_demo/gui.c` : app C déclare son UI GenUI fenêtre+bouton via `GU_BUTTON` + boucle MainLoop, réagit `MSG_CONTROL`/`MSG_CLOSE` ; `sys_ui_define` refondu ; SDK `oricos_ui_define`/`_main_loop`/`_alert`/`_do_dlgbox` ; `test_oricos_gui_demo`, 574 verts). **Couche GUI déclarative GeoWorks complète depuis C (G.1→G.7).** Suite : ratifier ADR-26 (≥50 % impl atteint), polish (libellés boutons distincts, texte message DoDlgBox/Alert, tag `GU_BUTTON` label, retrait callback kernel, signaux multi-bits #1). **Avancement final** : ADR-26 ✅ ratifiée (2026-05-26) ; polish titres+libellés ✅ (chaînes GenUI inline, libellés OK/Cancel/Yes/No, gui_demo "Demo C"/"Clic" visibles SDL `--gui-demo`) ; v1.22.70-alpha. | SP-3.m ✅, ADR-25 ✅ |

| ⚙️ **SP-3.o — Contrôles valeur + ascenseurs + GenView (widgets GeoWorks)** (arc, cadré 2026-05-26 ; pré-req SP-3.n ✅) | **Objectif** : compléter la famille de **contrôles « valeur/saisie »** qui manque vs GeoWorks (cf. comparatif widget/fenêtrage : windowing déjà à parité, widgets pauvres). Aujourd'hui OricOS n'a que le cliquable (bouton/menu/dialogue) ; manquent **ascenseurs, sliders, champ texte, listes, radios** + l'**API de valeur de contrôle**. **Réf. d'art** : GeoWorks `GenScrollbar`/`GenView`/`GenBoolean`/`GenItemGroup`/`GenText`/`GenList` ; Control Manager IIgs (scroll bar, LineEdit, List Manager). **Cohérent ADR-26** (mêmes principes : contrôles déclarés en GenUI, interactions → `MSG_CONTROL` via MainLoop, rendu QDF). **Découpage** : **S.1** **API valeur de contrôle** : `SYS_CTL_GET_VALUE`/`SYS_CTL_SET_VALUE` (slots ABI réservés `$1B`+) + champ `value`/`max` dans le record widget (16B → étendre ou réutiliser couleur/cb) ; **implémente la checkbox** (`WG_TYPE_CHECK`, `GenBoolean`) : clic → toggle value → `MSG_CONTROL`+id, l'app lit la valeur. **S.2** **Ascenseurs** (`WG_TYPE_SCROLL_V/_H`, `GenScrollbar`) : rendu gouttière+thumb (+flèches), hit-test (thumb/gouttière/flèche), **thumb-drag dans le MainLoop** (état gardé entre appels, comme le drag de fenêtre → 1 événement/appel, pas de boucle longue sous Forbid) → `MSG_CONTROL`+nouvelle valeur. **S.3** **GenView** (le gros confort) : zone de contenu **scrollable managée** reliant fenêtre + 1-2 scrollbars + viewport ; v1 = l'app dessine son contenu, le système gère les scrollbars et rend la position ; clip du contenu à la charge de l'app. **S.4** autres contrôles valeur : radios (`GenItemGroup`, exclusion mutuelle), **champ texte éditable** (`GenText`/LineEdit : curseur, insertion, backspace via clavier au focus), liste (`GenList`). **S.5** tags déclaratifs GenUI : `GU_CHECK`/`GU_SCROLL_V`/`GU_SCROLL_H`/`GU_RADIO`/`GU_TEXT`/`GU_LIST` (chaînes inline comme GU_BUTTON). **S.6** démo C : fenêtre avec ascenseur + checkbox + champ texte, l'app lit les valeurs. **Risques** : thumb-drag (réutilise le pattern drag fenêtre, balisé) ; **GenView** = le plus lourd (coordination viewport/scroll/contenu, clip) ; champ texte (gestion curseur/édition, interaction clavier-au-focus G.3) ; record widget 16B peut-être trop court pour value+max+state (étendre WIDGET_ENTSZ → churn WIDGET_TABLE) ; churn tests WM/widgets. **Effort** ~3-4 sem (S.1+S.2 valeur+scrollbar ; S.3 GenView ; S.4 texte/liste/radio ; S.5+S.6 déclaratif+démo). **Ordre d'impact** : S.1 (débloque tout) → S.2 (ascenseurs, demandé) → S.3 (GenView) → S.4. **Avancement 2026-05-26** : **S.1 ✅** (`SYS_CTL_GET/SET_VALUE` $1B/$1C + `WG_TYPE_CHECK`, `kernel_ctl_toggle`, 575 verts). **S.2 ✅** (ascenseurs `WG_TYPE_SCROLL_V/_H` value(+14)/max(+15) ; thumb-drag MainLoop `SCROLL_DRAG_ID` + `_wm_scroll_update` value=clamp(souris-gouttière,0,max) ; rendu `kernel_tk_scrollbar` ; fix conflit drag fenêtre↔contrôle dans `wm_step_arm_drag` ; `test_oricos_scrollbar`, 576 verts). Reste S.3 (GenView)→S.6. **ADR** : extension mineure d'ADR-26 (nouveaux types de contrôle, même modèle) ; mention CHANGELOG, pas de nouvelle ADR sauf si GenView introduit un modèle de viewport non-trivial (à réévaluer en S.3). | SP-3.n ✅, ADR-26 ✅ |

### Priorité 4 — Audio

| ID | Titre |
|----|-------|
| **OS-4.a** | Driver AY-3-8912 (compat Oric 1) |
| **OS-4.b** | Extension SID-like (synth 6 voies, filtres) |

### Priorité 5 — Guest Oric 1

| ID | Titre | Notes |
|----|-------|-------|
| **OS-5.a** | Routage clavier guest | Nécessite compositor HDL réel |
| **OS-5.b** | Partage ULA guest dans fenêtre OricOS | |
| **OS-5.c** | Démo BASIC 1.0 dans une fenêtre OricOS | |

### Priorité 6 — Track HDL ULX3S

| ID | Titre | Risque si différé |
|----|-------|-------------------|
| ~~HW-1~~ | ✅ **promu NOW P0 le 2026-05-09 (DEC-2)**. Squelette `docs/CONTRACT_HDL.md` créé Phase 0 ; détail Phase 1. | — |
| **HW-2** | Port 65C816 ECP5 — démarrage S9+ (post-programme état-de-l'art) | |
| **HW-3** | ULA host + guest en HDL — démarrage S9+ | |
| **HW-4** | Compositor HDL réel — démarrage S9+ | Prérequis OS-5 |
| **HW-5** | SD SPI controller — démarrage S9+ | |
| **HW-6** | HDMI tx — démarrage S9+ | |

---

## DÉCISIONS STRATÉGIQUES EN ATTENTE

| Décision | Question | Échéance souhaitée |
|----------|----------|--------------------|
| ~~DEC-1~~ | ✅ **actée 2026-05-09 (Phase 0 programme état-de-l'art)** : retrait net 6502 post-validation (cf. ADR-18 ratifiée CLAUDE.md §2). Critère go/no-go : 541 tests verts + bench ≤ 5 % + boot interactif ROM 1.0/1.1. Exécution Phase 1 du programme. | — |
| ~~DEC-2~~ | ✅ **actée 2026-05-09 (Phase 0 programme état-de-l'art)** : HW-1 (contrat HDL ↔ golden model) rédigé en Phase 0/1 (squelette `docs/CONTRACT_HDL.md` créé). Implémentation HDL effective (HW-2..HW-6, SP-GPU-HDL-1..4) reportée post-programme S9+ pour préserver focus single-developer. | — |
| ~~DEC-3~~ | ✅ **actée 2026-05-09** : llvm-mos **conservé** pour userland v1 mais en mode N **8-bit native** uniquement (apps mono-bank). Pas de fallback asm-only complet. Apps multi-bank → asm 65C816. Cf. ADR-05 v2 + `docs/TC-llvmmos.md`. | — |
| ~~DEC-4~~ | ✅ **fusionnée avec ADR-15 parquée v2 (2026-05-09)** : critères de réouverture explicites (apps non-trusted OU HW-2 mûr OU 2026-12-31). Cf. CLAUDE.md §3. | — |

---

## ADR OUVERTES (à instruire — cf. CLAUDE.md §3)

Identifiées suite point critique 2026-05-08. Ne pas trancher
unilatéralement.

| ADR | Sujet |
|-----|-------|
| ~~ADR-13~~ | ✅ **ratifiée 2026-05-08 (option a : COP + table)**, déplacée vers `CLAUDE.md` §2 |
| ~~ADR-14~~ | ✅ **ratifiée 2026-05-08** (table fixe 16 + bitmap free, layout 20B), déplacée vers `CLAUDE.md` §2 |
| **ADR-15** | **Parquée v2 2026-05-09** (Phase 0 programme état-de-l'art). Critères de réouverture : apps non-trusted ratifiées OU HW-2 mûr OU 2026-12-31. Préparation draft `docs/adr/0015-isolation-v2-DRAFT.md` en Phase 4. Cf. `CLAUDE.md` §3. |
| ~~ADR-16~~ | ✅ **ratifiée 2026-05-09 (Phase 0)** : modèle hybride event-driven (clavier/audio/GPU async) + sync (FAT32/console/GPU sync v1), pas de struct ops formelle v1, table dispatch IRQ `$01:5680`, ring buffer kbd 16 keycodes `$01:5860`. Cf. `CLAUDE.md` §2. |
| ~~ADR-17~~ | ✅ **ratifiée 2026-05-09 (Phase 0)** : 18 syscalls v1, COP `#$AA` + table dispatch `$01:5750`, convention sentinelle `A=$FF` errno bank 1 `$5760`, versioning par opcode immediate (v2 future = `#$AB`). Débloque Sprint 4 userland C. Cf. `CLAUDE.md` §2. |
| ~~ADR-18~~ | ✅ **ratifiée 2026-05-09 (Phase 0)** : retrait net 6502 post-validation. DEC-1 actée. Plan d'exécution 4 étapes en Phase 1 du programme. Critère go/no-go : 541 tests verts + bench ≤ 5 % + boot interactif ROM. Cf. `CLAUDE.md` §2. |
| ~~ADR-19~~ | ✅ **ratifiée 2026-05-09**, **révisée v2 2026-05-09** suite ADR-21 : VRAM en SDRAM unifiée (32 MiB hors banking, accès GPU direct + I/O CPU). Banks 128-191 redeviennent RAM extra apps. BRAM ECP5 = caches GPU internes invisibles CPU. Cf. `CLAUDE.md` §2. |
| ~~ADR-20~~ | ✅ **ratifiée 2026-05-09**, **révisée v3 2026-05-09** : mode HIRES Oric 2 desktop = **XVGA 1024×768×4bpp** 16 couleurs, framebuffer en SDRAM. v1 240×200×3bpp (compat ULA guest), v2 SVGA, **v3 XVGA actuelle**. Cf. `CLAUDE.md` §2. |
| ~~ADR-21~~ | ✅ **ratifiée 2026-05-09 (GPU Blitter HW autonome, 5 commandes v1 : CLEAR/FILL_RECT/BLIT/LINE/TEXT, ports I/O $0340-$034F)**, déplacée vers `CLAUDE.md` §2. |

---

## RISQUES SURVEILLÉS

| Risque | Impact | Probabilité | Mitigation actuelle |
|--------|--------|-------------|---------------------|
| ~~llvm-mos 65C816 mode N immature~~ | ✅ **clos 2026-05-09** | — | TC-llvmmos investigation : ADR-05 révisée v2 (apps C = mode N 8-bit native, mono-bank). Risque transformé en contrainte connue. |
| HDL ULX3S diverge du golden model | 🔴 refactor massif | 🔴 haute | Aucune ; HW-1 prioritaire |
| Bank allocator → fragmentation | 🟠 OS crash après N apps | 🟠 moyenne | OS-2.h en NOW |
| Pas de driver clavier = OS non-interactif | 🔴 démo finale impossible | 🔴 haute | OS-2.d en NOW |
| Pas d'isolation mémoire ADR-04 v1 | 🟠 multitasking fragile | 🟠 assumée v1 | DEC-4 prévue Q4 2026 |
| ~~Tests d'intégration sans validation visuelle~~ | 🟠 bugs render passent | ✅ **clos 2026-05-08** | test_oricos_visual + golden PPM mis en place |

---

## DETTE TECHNIQUE IDENTIFIÉE

| Item | Origine | Plan |
|------|---------|------|
| ~~Bug TXS Phosphoric~~ | Sprint 2.d.1 | ✅ **clos 2026-05-08** : faux positif. Comportement WDC correct (SEP #$10 force X high=0, TXS copie X entier). Fix côté OricOS : utiliser TCS au lieu de TXS pour stack page 1. |
| ~~8 opcodes DP indirect manquants~~ | Sprint 2.e.1 | ✅ **clos 2026-05-08** : 8 opcodes ajoutés ($12/$32/$52/$72/$92/$B2/$D2/$F2) + helper `addr816_dp_indirect`. 2 tests unitaires. OricOS print_char simplifié vers STA (dp) natif. |
| `--kernel` patch `mem.rom[]` (pollue ADR-10) | Demo visible | PH-bootrom |
| ~~Bug `kernel_fill_rect_aligned` offset_initial~~ | ✅ **clos 2026-05-09** : root cause = bug Phosphoric (ASL/LSR/ROL/ROR Accumulator M=0 ne propageaient pas le carry low→high). Fixé en branchant M=8bit/M=16bit dans les 4 opcodes. Découvert via sentinels asm OricOS. |
| Shifts/rotations zp/abs en M=0 : possible bug similaire à ASL A si on shifte une valeur 16-bit en mémoire. | À auditer | Pas encore exercé par OricOS. Audit à prévoir avant un usage massif d'arithmétique 16-bit avec shifts mémoire. |
| Cohabitation 6502/65C816 sans politique de retrait | B1 cohabitation | DEC-1 / ADR-18 |
| `kernel_print_banner` = 12 STA hardcodés | Sprint 2.c PoC | OS-2.e remplace |
| ~~`kernel_alloc_bank` bump-only sans free~~ | Sprint 2.b PoC | ✅ **clos 2026-05-08 (OS-2.h.1)** : free list LIFO 16 entries. Bitmap reportée v0.2. |
| Compositor B4 = modèle simulé pas pipeline | B4 PoC | HW-4 |
| Paravirt B3 = stub | B3 PoC | OS-5 + HW |
| ~~3 globales `TASK_*_S` au lieu de TCB~~ | Sprint 1.b PoC | ✅ **clos 2026-05-08 (OS-2.g.1)** : TCB table 16 + bitmap (ADR-14). |
| **Polish concurrence OS-2.g v2.b** (ADR-25) — à reprendre, non bloquant | OS-2.g v2.b (2026-05-25) | (1) **Signaux multi-bits génériques** : remplacer `KBD_WAITER` (signal dégénéré, 1 seul attendeur clavier) par `SIG_PENDING[16]`/`SIG_WAIT[16]` + `task_wait(mask)`/`task_signal(pid,bits)` → plusieurs attentes simultanées + IPC. (2) **`Disable`/`Enable` formels** : remplacer les `sei/php/plp` ad hoc (ring_pop, etc.) par des primitives nestées propres. (3) **Migrer le chemin JSL d'`app_exec` → `app_spawn`** et **retirer les fallbacks `SCHED_ACTIVE`** (boot-context spin/WAI dans `sys_read_char`, STP dans `sys_exit`) une fois (1) en place. (4) **Free-list de pages de pile** : `task_destroy`/teardown fuit la page de pile (`STACK_NEXT_PAGE` bump-only). (5) **Idle « dernière tâche »** : `find_next` retombe sur l'idle, mais valider le cas « toutes les tâches bloquées/sorties » par un test dédié. (6) **`SYS_SLEEP_MS` ms→ticks** : v1 interprète l'arg en ticks ; ajouter la conversion ms 16-bit. |

---

## ANNEXE — Point critique architecte 2026-05-08

Backlog réorganisé suite à critique senior :

1. **5 ADRs vaporware** (05, 06, 07, 08, 09) — ratifiées sans
   implémentation. Recommandation : geler ratifications nouvelles
   jusqu'à 50% impl.
2. **Kernel = 380 LOC asm** — proportions toy. Sprints 2.d→2.i
   ajoutés pour combler les fondamentaux manquants (clavier, console,
   syscall, TCB, allocator, panic).
3. **Sprints précédents <1 jour** — pas réalistes vs charge restante.
   Re-calibrage estimations en jours-semaines.
4. **Tests sans validation visuelle** — bug H4 fonte est passé à
   travers 494 tests. Ajout PH-CI-visual.
5. **Decisions stratégiques implicites** non documentées — DEC-1
   à DEC-4 et ADR-13 à ADR-18 ouvertes.

Sprints OS-2.d → OS-2.i sont **prérequis stricts** pour Sprint 3 GUI ;
la roadmap d'origine sautait directement de 2.c à 3, ce qui n'était
pas viable.
