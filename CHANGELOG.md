# CHANGELOG — Workspace Oric 2

CHANGELOG commun aux 3 sous-projets (`Phosphoric`, `OricOS`, `oric2`
docs). Format Keep a Changelog v1.0.0.

Entrées détaillées par sous-projet :
- [Phosphoric/CHANGELOG](./Phosphoric/CHANGELOG)
- [OricOS/CHANGELOG.md](./OricOS/CHANGELOG.md)

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
