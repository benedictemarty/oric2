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
| ~~OS-2.j~~ | ✅ **clos 2026-05-08 (v0.6)** : pipeline complet SD → FAT32 → fat_open → fat_read_cluster → app_exec → exec en bank dédiée. App chargée depuis fichier "HELLO.BIN" sur image SD FAT32 et exécutée. Démo SDL2 : "YABZZ" (deux Z = bundle inline + bundle SD). v0.2 reportés : cluster chain (fichier > 1 cluster), subdir, fichier > 32 MiB. | — | OricOS | — |
| ~~OS-2.k~~ | ✅ **clos 2026-05-08** (v0.1 : spec + `kernel_bundle_validate` fonctionnel après fix bug PH P-mode-N). | — | OricOS | — |
| ~~PH-bug-dp-indirect-Y-bank1~~ | ✅ **clos 2026-05-08** : pas un bug `[dp],Y`. Vraie cause = COP/BRK/IRQ/NMI/PHP/PLP/RTI corrompaient bits X/M de P en mode N (masque mode E `& ~FLAG_BREAK \| FLAG_UNUSED`). RTI post-COP restaurait X=0 → `ldy` du caller consommait 2 bytes au lieu de 1. Fix : conditionnement sur `cpu->E` aux 6 emplacements. |
| ~~OS-2.l~~ | ✅ **clos 2026-05-08** : v0.1 livré. `kernel_app_exec` complet : validate + find_code + alloc + copy + JSL self-modifying. App test en bank 7 écrit 'Z' via syscall. v0.2 (free bank exit, sandbox) reporté. | — | OricOS | — |
| ~~OS-2.m~~ | ✅ **clos 2026-05-08** : `apps/hello/hello.s` source standalone, pipeline build via ca65+ld65+`tools/oricos-bundle.py`. Bundle .oosobj embedded dans kernel via `.incbin`, exec via `kernel_app_exec`. Premier userland OricOS hors kernel. | — | OricOS | — |

### Priorité 2bis — Toolchain userland C

| ID | Titre | Estim. | Pré-req |
|----|-------|--------|---------|
| **TC-llvmmos** | Installer llvm-mos, valider 65C816 mode N support | 2-5 j | — |
| **TC-libc** | libc minimal (printf via syscalls, malloc bank-based) | 5-10 j | TC-llvmmos, OS-2.f |

---

## LATER — backlog priorisé bas (Q3 2026+)

### Priorité 3 — GUI

| ID | Titre | Notes |
|----|-------|-------|
| **OS-3.a** | Compositor logique au-dessus du compositor matériel | Bloqué par ADR-15 |
| **OS-3.b** | Window manager basique (1 fenêtre + focus) | |
| **OS-3.c** | Event loop (clavier, timer, IRQ) | Pré-req OS-3.a, OS-2.d |
| **OS-3.d** | Toolkit minimal (frame, label, button) | |

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
| **DEC-3** | llvm-mos 65C816 mode N : si bloquant → fallback asm-only userland v1 ? | Avant Sprint 4 |
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

---

## RISQUES SURVEILLÉS

| Risque | Impact | Probabilité | Mitigation actuelle |
|--------|--------|-------------|---------------------|
| llvm-mos 65C816 mode N immature | 🔴 bloque userland C | 🔴 haute | Aucune ; PoC TC-llvmmos prioritaire |
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
