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
| **OS-2.g** | Refactor TCB-based scheduler (struct `task_t`, N tâches, états) | 5-7 j | OricOS | ADR-14 |
| **OS-2.h** | Bank allocator bitmap avec free | 2-3 j | OricOS | — |
| **OS-2.i** | Modèle d'erreur kernel (panic codes, kernel log ring buffer) | 2-3 j | OricOS | — |

### Priorité 1bis — Outillage Phosphoric

| ID | Titre | Estim. | Pré-req |
|----|-------|--------|---------|
| **PH-debug816** | Étendre debugger REPL pour 65C816 (B, D, PBR, DBR, S 16-bit, M/X/E) | 2 j | — |
| **PH-CI-visual** | CI visuelle : pHash sur frame golden, comparer à chaque test | 2 j | — |
| **PH-bootrom** | Refactor `--kernel` pour utiliser une boot ROM Oric 2 propre (pas patch `mem.rom[]`) | 1 j | — |

---

## NEXT — itération suivante (estim. 2026-05-23 → 2026-06-30)

### Priorité 2 — Stockage & ressources

| ID | Titre | Estim. | Pré-req |
|----|-------|--------|---------|
| **OS-2.j** | FAT32 SD lecture seule (ULX3S SPI + hostfs Phosphoric) | 10-15 j | — |
| **OS-2.k** | Format bundle apps (header + sections) — ADR-08 implémenté | 3-5 j | ADR-08, ADR-17 |
| **OS-2.l** | App loader (parse bundle, alloc bank, exec) | 5-7 j | OS-2.k, OS-2.f, OS-2.h |
| **OS-2.m** | Première app "hello" en asm 65C816 (PoC userland sans llvm-mos) | 2-3 j | OS-2.l |

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
| **ADR-14** | Format TCB et structure interne tâche |
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
| Tests d'intégration sans validation visuelle | 🟠 bugs render passent | 🔴 haute (cf. H4 fonte) | PH-CI-visual en NOW |

---

## DETTE TECHNIQUE IDENTIFIÉE

| Item | Origine | Plan |
|------|---------|------|
| **Bug TXS Phosphoric mode N + X=1** : copie seulement low byte de X dans S → S=$00xx au lieu de $01xx pour ldx #$FF;txs. Stack OricOS tourne page 0 par chance. Reproductible via test 2.a (`TASK_A_S=$00F8`). | Sprint 2.d.1 | **PH-fix-txs** : revoir `cpu816_opcodes.c:841` vs WDC W65C816S §A.32. Sans urgence — kernel actuel non impacté fonctionnellement, mais incorrect sémantiquement. |
| **8 opcodes DP indirect 65C816 NON IMPLÉMENTÉS dans Phosphoric** : $12 ORA (dp), $32 AND, $52 EOR, $72 ADC, $92 STA, $B2 LDA, $D2 CMP, $F2 SBC. Le decoder les traite via default case avec `opcode_table[op].size` (table 6502 où ces slots sont `???` size=1) → consume 0 bytes operand → corruption décodage. Contournement Sprint 2.e : utiliser `[dp]` long indirect (opcodes $03/$07/$13/$17/...). | Sprint 2.e.1 | **PH-fix-dp-indirect** : ajouter les 8 opcodes dans `cpu65c816_opcodes.c` + helper `addr816_dp_indirect` (16-bit pointer en DP+dp, écrit à DBR:addr). Mettre à jour opcode_table size=2 pour ces 8 slots. Priorité moyenne (contournement actuel fonctionne mais coûteux en clarté). |
| `--kernel` patch `mem.rom[]` (pollue ADR-10) | Demo visible | PH-bootrom |
| Cohabitation 6502/65C816 sans politique de retrait | B1 cohabitation | DEC-1 / ADR-18 |
| `kernel_print_banner` = 12 STA hardcodés | Sprint 2.c PoC | OS-2.e remplace |
| `kernel_alloc_bank` bump-only sans free | Sprint 2.b PoC | OS-2.h remplace |
| Compositor B4 = modèle simulé pas pipeline | B4 PoC | HW-4 |
| Paravirt B3 = stub | B3 PoC | OS-5 + HW |
| 3 globales `TASK_*_S` au lieu de TCB | Sprint 1.b PoC | OS-2.g |

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
