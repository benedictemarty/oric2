# CHANGELOG — Workspace Oric 2

CHANGELOG commun aux 3 sous-projets (`Phosphoric`, `OricOS`, `oric2`
docs). Format Keep a Changelog v1.0.0.

Entrées détaillées par sous-projet :
- [Phosphoric/CHANGELOG](./Phosphoric/CHANGELOG)
- [OricOS/CHANGELOG.md](./OricOS/CHANGELOG.md)

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
