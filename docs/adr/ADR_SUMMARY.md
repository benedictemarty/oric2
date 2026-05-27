# ADR Ratifiées — Résumé complet (Oric 2 / OricOS)

> Extrait du §2 de `CLAUDE.md` (2026-05-27). Source de vérité : ce fichier.
> Toute modification d'une ADR passe par discussion explicite avec l'humain.

Ces décisions sont **non-négociables** sans nouvelle discussion explicite.

---

### ADR-01 — CPU : 65C816
Le CPU cible est le WDC 65C816. Il fournit :
- Le **mode émulation** (E=1) : comportement strict 6502 cycle-exact, pour exécution native du code Oric 1.
- Le **mode natif** (E=0) : registres 16 bits, banking, pour OricOS.
- L'instruction **XCE** comme mécanisme de bascule, support direct de la paravirtualisation.

Alternatives écartées : 6502 strict (insuffisant pour OricOS), 65C02 (pas de mode natif 16 bits).

### ADR-02 — Compositor matériel double ULA
Le rendu vidéo de l'Oric 2 instancie **deux ULA en parallèle** :
- Une **ULA host** générant le framebuffer principal d'OricOS (modes étendus 320×240+).
- Une **ULA guest** générant le framebuffer Oric 1 (240×200, attribute-based, comportement Oric 1 strict).

Un **compositor** mixe les deux à la sortie selon la position de la fenêtre guest. Le guest Oric 1 tourne donc à pleine vitesse sans charge CPU pour son rendu.

Cette décision conditionne toute la stratégie d'implémentation : pas d'émulation logicielle de l'Oric 1 dans OricOS.

### ADR-03 — Multitâche : préemptif strict (ratifiée 2026-05-07)
OricOS implémente un scheduler **préemptif** dirigé par le timer (tick périodique, sauvegarde de contexte complète : registres 65C816, P, PBR/DBR/D, S). Round-robin avec niveaux de priorité simples. Référence d'art : **SymbOS** sur Z80. Garanties : latence audio prévisible, I/O serviceables sans yield explicite, kernel jamais bloqué par une app récalcitrante.

Alternatives écartées : coopératif (risque blocage I/O), hybride (complexité conceptuelle inutile pour la phase v1).

### ADR-04 — Isolation mémoire : bank-based v1 (ratifiée 2026-05-07)
Pas de MMU/MPU matérielle dans la v1. L'isolation s'appuie sur le banking 65C816 natif (PBR pour le code, DBR pour les données, D pour zero-page). Le kernel alloue les banks par process. **Pas de protection matérielle** entre processes — OricOS est un OS « de confiance » dans cette version. Le HDL ULX3S reste simple côté memory map.

Alternatives écartées : MMU custom (coûte BRAM, retarde B2), MPU à segments (sans précédent 65C816).

### ADR-05 — Langage d'implémentation OricOS : asm + C llvm-mos (ratifiée 2026-05-07, **révisée v2 2026-05-09**)
- **Kernel + drivers** en assembleur 65C816 natif (ca65 ou équivalent) : performances critiques, contrôle direct du banking.
- **Userland (apps, libs)** en C compilé par **llvm-mos** (toolchain LLVM ciblant 6502/65C816), en **mode N 8-bit native** (M=1, X=1, après XCE par le kernel). **Apps mono-bank** : limitées à 1 bank de 64 KiB linéaire (code+data), pas de pointer cross-bank, pas de registres 16-bit dans le code généré.

**Révision v2 (2026-05-09) — TC-llvmmos** :
- llvm-mos ne supporte ni le mode 16-bit registres ni le banking 24-bit dans le compilateur C (issues llvm-mos #320, #321, pas d'horizon de livraison).
- v2 acte la contrainte : **apps C = mono-bank 8-bit native**. Apps multi-bank ou exigeant registres 16-bit restent **en asm 65C816** (ca65).
- Implication : DEC-3 ratifiée (cf. BACKLOG). Pas de dépendance à des features llvm-mos non-implémentées pour Sprint 4.

### ADR-06 — Modèle GUI : SymbOS-like (ratifiée 2026-05-07)
GUI multifenêtrée préemptive sur 8/16-bit, drag & drop, menus contextuels, taskbar. Référence directe : **SymbOS**.

> **Révision (ADR-26, 2026-05-26)** : ADR-06 reste valide pour le noyau/WM. Le **modèle d'API applicative** est précisé par **ADR-26** (déclaratif GenUI/SpecUI + MainLoop à messages, GeoWorks-like).

### ADR-07 — Système de fichiers : FAT32 SD + hostfs émulateur (ratifiée 2026-05-07)
- **Hardware ULX3S** : FAT32 sur carte SD via SPI.
- **Émulateur Phosphoric** : option `--hostfs DIR` en alternative à une image SD FAT32.

### ADR-08 — Packaging apps natives : bundle léger (ratifiée 2026-05-07)
Format binaire OricOS = **header magique + table des sections** (code / data / icône / manifest). Inspiré d'AmigaOS Hunk simplifié.

### ADR-09 — Audio : AY-3-8912 + extension SID-like (ratifiée 2026-05-07)
- **AY-3-8912** conservé (compat Oric 1 stricte ADR-10) : 3 voies + bruit + enveloppe.
- **Extension SID-like** ajoutée : 3 voies supplémentaires, filtres, samples PCM 4-8 bits.

### ADR-10 — Compatibilité ascendante Oric 1 stricte
En mode émulation, le comportement doit être **indistinguable d'un Oric 1 réel** pour tout logiciel existant. Aucune régression tolérée.

### ADR-11 — Sémantique du mode E vis-à-vis du NMOS 6502 (ratifiée 2026-05-07)
Le 65C816 en mode E adopte un comportement **hybride pragmatique** :
- **Bug `JMP ($xxFF)`** : reproduit (conforme NMOS).
- **Opcodes illégaux NMOS** : traités comme NOP. Exception : `$EB` (alias SBC immediate) conservé.
- **Séquence RMW interne** : suit le WDC, non observable côté software pur.

### ADR-12 — Mode HIRES Oric 2 (ratifiée 2026-05-08)
- **Dimensions** : 240×200 pixels. **Encodage** : 3 bits par pixel direct (8 couleurs).
- **Framebuffer** : 240×200 × 3 bits = 18 000 octets/frame.
- **ULA guest Oric 1** non concernée — reste 240×200 attribute-based (ADR-10).

### ADR-13 — Mécanisme de syscall : COP + table (ratifiée 2026-05-08)
OricOS expose ses services via l'opcode **`COP #imm`** (opcode `$02`) en mode N. Numéro de syscall en `A`. Vecteur COP mode N : `$00FFE4`. Dispatch via table de pointers 16-bit en bank 1 (`syscall_table` à `$01:5750`).

### ADR-14 — Format TCB et table tâches (ratifiée 2026-05-08)
Table fixe de 16 TCBs en bank 1 à `$01:5C00`. Layout TCB (20 octets) : pid, state, prio, parent_pid, saved_S, entry_pc, code_bank, data_bank, stack_bank, flags, name. Bitmap d'allocation à `$01:5B00`.

### ADR-16 — Driver model OricOS (ratifiée 2026-05-09)
Modèle hybride : **Classe 1** IRQ-driven (clavier KBD2, souris MOU2, GPU async) ; **Classe 2** sync/blocking (FAT32, console, GPU sync v1, bank alloc). Pas de struct ops formelle en v1. Cf. `docs/adr/0016-driver-model.md`.

### ADR-17 — ABI kernel publique (ratifiée 2026-05-09)
18 syscalls v1 via `cop #$AA`. Entrée : `A`=syscall num, `X`/`Y`=args 8-bit. Sortie : `A`=retour, `A=$FF`=erreur. Table dispatch à `$5750`. Cf. `docs/adr/0017-abi-syscall-userland.md`.

### ADR-18 — Retrait du cœur 6502 (ratifiée 2026-05-09)
Le cœur 6502 historique est **retiré** au profit du 65C816 mode E unique. Cf. `docs/adr/0018-retrait-6502.md`.

### ADR-19 — VRAM en SDRAM unifiée (ratifiée 2026-05-09, révisée v2)
Toute la VRAM réside en SDRAM ULX3S (16 MiB via 24-bit address). Accès GPU direct ; accès CPU via MMIO `$0330-$033C`. Banks 128-255 libérés pour RAM extra apps.

### ADR-20 — Mode HIRES desktop : 1024×768×4bpp XVGA (ratifiée 2026-05-09, révisée v3)
- **Résolution** : 1024×768 pixels. **Profondeur** : 4 bpp = 16 couleurs.
- **Framebuffer** : 512 octets/ligne × 768 lignes = 384 KiB à SDRAM `$100000`.
- **Pixel clock** : 65 MHz (VESA XVGA standard).
- **Palette VGA-IBM** : 16 couleurs fixes (black→white, indices 0..15).

### ADR-21 — GPU Blitter HW autonome (ratifiée 2026-05-09)
Co-processeur graphique HDL ECP5. Registres I/O `$0340-$034F`. 7 opcodes : NOP($00), CLEAR($01), FILL_RECT($02), BLIT($03), LINE($04), TEXT($05), FILL_RECT16($06), TEXT16($07). v0.1 synchrone, v0.2 async avec IRQ. Cf. `docs/adr/` (ADR-21 intégré dans CLAUDE.md).

### ADR-22 — Clavier Oric 2 paravirtualisé hybride (ratifiée 2026-05-23)
Registres `$0350-$035F`. IRQ-driven côté hôte, matrice 8×8 virtuelle côté guest. Cf. `docs/adr/0022-clavier-oric2-paravirt.md`.

### ADR-23 — Console OricOS : flux de caractères, backend interchangeable (ratifiée 2026-05-24)
ABI publique (`kernel_print_char`/`print_string`) sans géométrie. Backend v1 = mode texte Oric 1 (bootstrap) ; cible = `kernel_gfx_text` GPU sur XVGA. Cf. `docs/adr/0023-console-flux-caracteres.md`.

### ADR-24 — Contrôleur souris Oric 2 (ratifiée 2026-05-24)
Registres `$0360-$036F`. Position absolue 10-bit + deltas read-clear. IRQ dédiée `IRQF_MOU2`. Cf. `docs/adr/0024-souris-oric2.md`.

### ADR-25 — Modèle de concurrence kernel : Exec-classique (ratifiée 2026-05-25)
`Forbid`/`Permit` suspend le context-switch (IRQ continuent). `Disable`/`Enable` pour RMW courts. Blocage/réveil via `STATE=BLOCKED` + driver wake (`READY`). Cf. `docs/adr/0025-modele-concurrence-kernel.md`.

### ADR-26 — Modèle GUI déclaratif GenUI/SpecUI (GeoWorks-like) (ratifiée 2026-05-26)
UI déclarative (tables `GU_WINDOW`/`GU_BUTTON`/…). `SYS_MAIN_LOOP` bloque et retourne des messages sémantiques (`MSG_KEY`/`MSG_CLOSE`/…). GenUI/SpecUI séparés. Syscalls `$15-$1A` ajoutés. Cf. `docs/adr/0026-modele-gui-declaratif.md`.

---

*Extrait automatique — 2026-05-27. Conserver en cohérence avec `CLAUDE.md` §2.*
