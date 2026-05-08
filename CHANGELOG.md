# CHANGELOG — Workspace Oric 2

CHANGELOG commun aux 3 sous-projets (`Phosphoric`, `OricOS`, `oric2`
docs). Format Keep a Changelog v1.0.0.

Entrées détaillées par sous-projet :
- [Phosphoric/CHANGELOG](./Phosphoric/CHANGELOG)
- [OricOS/CHANGELOG.md](./OricOS/CHANGELOG.md)

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
