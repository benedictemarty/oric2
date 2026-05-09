# BACKLOG — Workspace Oric 2

> Source de vérité pour le travail à venir. Issu d'un point critique
> architecte senior OS du 2026-05-08 (cf. annexe §6 ci-dessous).
> Met à jour la ROADMAP affichée dans Phosphoric/ROADMAP et OricOS/CLAUDE.md §7.

**Convention** : sprints estimés en charge nominale single-developer
sessions. Les sprints "ratifiés clos" précédents (Sprint 0 → 2.c+) ont
duré <1 jour ; les sprints à venir sont calibrés en **semaines réelles**
parce qu'ils touchent des fondamentaux non-triviaux.

---

## NOW — itération courante (2026-05-08 → 2026-05-22)

### Priorité 1 — Fondations OS bloquantes (avant tout sprint GUI)

| ID | Titre | Estim. | Sprint | Pré-req |
|----|-------|--------|--------|---------|
| **OS-2.d** | Driver clavier Oric 1 (matrice VIA PB) | 3-5 j | OricOS | — |
| **OS-2.e** | Driver console générique (`print_char`, `print_string`, cursor, scroll) | 2-3 j | OricOS | — |
| ~~OS-2.f~~ | ✅ **clos 2026-05-08** (v0.1 : COP handler avec dispatch hardcoded SYS_PRINT_CHAR. Table dispatch reportée v0.2). | — | OricOS | — |
| ~~OS-2.g~~ | ✅ **clos 2026-05-08** (v0.1 : TCB table 16 + bitmap, scheduler refactored, ADR-14 ratifiée. v0.2 task_create dynamique reporté). | — | OricOS | — |
| ~~OS-2.h~~ | ✅ **clos 2026-05-08** (v0.1 : free list LIFO 16 entries. Bitmap reportée v0.2). | — | OricOS | — |
| ~~OS-2.i~~ | ✅ **clos 2026-05-08** (v0.1 : kernel_panic + print_hex8 + print_nibble. Log ring + SYS_PANIC reportés v0.2). | — | OricOS | — |

### Priorité 1bis — Outillage Phosphoric

| ID | Titre | Estim. | Pré-req |
|----|-------|--------|---------|
| ~~PH-debug816~~ | ✅ **clos 2026-05-08** : `cpu816_get_state_string` mode E/N, debugger `regs`+`set` étendus, 2 tests unitaires. | — | — |
| ~~PH-CI-visual~~ | ✅ **clos 2026-05-08** : test_oricos_visual + golden PPM + comparaison pixel-perfect. Empêche régressions render comme H4 (fonte). | — | — |
| **PH-bootrom** | Refactor `--kernel` pour utiliser une boot ROM Oric 2 propre (pas patch `mem.rom[]`) | 1 j | — |

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
| **TC-llvmmos-install** | Installer llvm-mos pre-built, vérifier targets dispo | 1 j | — |
| **TC-llvmmos-target-oricos** | Créer target `oricos` custom (clang.cfg + crt0.S + link.ld) | 2-3 j | TC-llvmmos-install |
| **TC-libc** | libc minimal (printf via syscalls, malloc bank-local) | 5-10 j | TC-llvmmos-target-oricos, OS-2.f |
| **TC-poc-hello-c** | Premier hello.c → bundle .oosobj → exec sur OricOS | 1-2 j | TC-libc |

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
| **SP-GPU-1** | Phosphoric : `src/io/gpu_device.{c,h}` simulant le GPU Blitter (ADR-21). v0.1 : commande CLEAR + FILL_RECT synchrone. Tests unitaires. | ADR-21 ✅ |
| **SP-GPU-2** | Phosphoric : étendre gpu_device avec BLIT, LINE, TEXT. Tests étendus. | SP-GPU-1 |
| **SP-GPU-3** | OricOS kernel : helpers `kernel_gfx_clear`, `kernel_gfx_fill_rect`, etc. Refactor `kernel_hires2_*` vers `kernel_gfx_*` (ou les marquer legacy). | SP-GPU-1 |
| **SP-GPU-HDL-1** | HDL ULX3S : controller GPU minimal (CLEAR + FILL_RECT). | SP-GPU-3 |
| **SP-GPU-HDL-2** | HDL ULX3S : BLIT engine. | SP-GPU-HDL-1 |
| **SP-GPU-HDL-3** | HDL ULX3S : LINE Bresenham. | SP-GPU-HDL-2 |
| **SP-GPU-HDL-4** | HDL ULX3S : TEXT engine + font ROM. | SP-GPU-HDL-3 |
| **SP-3.c** | Compositor logique : 1 fenêtre rectangulaire avec frame + title bar via GPU commands `kernel_gfx_*`. | SP-GPU-3 |
| **SP-3.d** | Toolkit minimal : font HIRES, label, button | Pré-req SP-3.b |
| **SP-3.e** | Event loop multifenêtré + focus + drag (SymbOS-like). Backing-stores en VRAM cold via DMA. | SP-3.c, OS-2.d, SP-VRAM-2 |

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
| **HW-1** | Définir contrat HDL ↔ golden model (interface BRAM, timing) | 🔴 écart Phosphoric/HDL |
| **HW-2** | Port 65C816 ECP5 | |
| **HW-3** | ULA host + guest en HDL | |
| **HW-4** | Compositor HDL réel | Prérequis OS-5 |
| **HW-5** | SD SPI controller | |
| **HW-6** | HDMI tx | |

---

## DÉCISIONS STRATÉGIQUES EN ATTENTE

| Décision | Question | Échéance souhaitée |
|----------|----------|--------------------|
| **DEC-1** | Sort du 6502 dans Phosphoric : retrait après B1.6 stable ou cohabitation perpétuelle ? | Avant Sprint 3 |
| **DEC-2** | Track HDL : démarrer en parallèle ou attendre OS v1 ? | Q2 2026 |
| ~~DEC-3~~ | ✅ **actée 2026-05-09** : llvm-mos **conservé** pour userland v1 mais en mode N **8-bit native** uniquement (apps mono-bank). Pas de fallback asm-only complet. Apps multi-bank → asm 65C816. Cf. ADR-05 v2 + `docs/TC-llvmmos.md`. | — |
| **DEC-4** | ADR-04 v2 : MMU custom HDL ou MPU à segments ? | Q4 2026 |

---

## ADR OUVERTES (à instruire — cf. CLAUDE.md §3)

Identifiées suite point critique 2026-05-08. Ne pas trancher
unilatéralement.

| ADR | Sujet |
|-----|-------|
| ~~ADR-13~~ | ✅ **ratifiée 2026-05-08 (option a : COP + table)**, déplacée vers `CLAUDE.md` §2 |
| ~~ADR-14~~ | ✅ **ratifiée 2026-05-08** (table fixe 16 + bitmap free, layout 20B), déplacée vers `CLAUDE.md` §2 |
| **ADR-15** | Stratégie d'isolation mémoire post-v1 (MMU custom HDL ?) |
| **ADR-16** | Driver model (event-driven / polling / IRQ-driven) |
| **ADR-17** | API kernel publique exposée à userland (call gates, ABI) |
| **ADR-18** | Sort du 6502 Phosphoric (cf. DEC-1) |
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
