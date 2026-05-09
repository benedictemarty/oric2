# CLAUDE.md — Phosphoric / Projet Oric 2

> Document directeur pour Claude Code. Ce fichier est la **source de vérité** pour les décisions d'architecture, les contraintes, et le périmètre courant. À lire en début de chaque session avant toute modification de code.

---

## 1. Identité du projet

**Phosphoric** est un émulateur Oric 1 multiplateforme écrit en C. Son rôle évolue : il devient le **modèle exécutable de référence** (golden model) du projet **Oric 2**, une machine "chimère" rétrocompatible Oric 1, destinée à terme à une implémentation HDL sur FPGA ULX3S.

**Oric 2** est défini par trois couches indissociables :
- Le **hardware Oric 2** : 65C816, ULA étendue, compositor matériel, banking mémoire, sortie HDMI.
- **OricOS** : système d'exploitation multitâche graphique, natif Oric 2.
- Le **guest Oric 1** : machine Oric 1 virtualisée tournant dans une fenêtre d'OricOS, exécutée nativement par le 65C816 en mode émulation (paravirtualisation matérielle via XCE).

Phosphoric doit modéliser ces trois couches en logiciel, validation comportementale avant le port HDL.

---

## 2. Décisions d'architecture figées (ADR ratifiées)

Ces décisions sont **non-négociables** sans nouvelle discussion explicite avec l'humain. Elles ne peuvent pas être contournées par convenance d'implémentation.

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

### ADR-10 — Compatibilité ascendante Oric 1 stricte
En mode émulation, le comportement doit être **indistinguable d'un Oric 1 réel** pour tout logiciel existant. Aucune régression tolérée. Boot sans OricOS = comportement Oric 1 pur.

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
- L'ADR-05 v1 sous-entendait registres 16-bit + banking 24-bit pour les apps C. Investigation TC-llvmmos (cf. `docs/TC-llvmmos.md`) montre que **llvm-mos ne supporte ni le mode 16-bit registres ni le banking 24-bit dans le compilateur C** (issues llvm-mos #320, #321 ouvertes depuis 2023-2024, pas d'horizon de livraison).
- Le compilateur C génère du code 8-bit linéaire dans tous les targets existants (`mos6502`, `mos65c02`, `mosw65c816`, `mos65el02`, `rpc8e`). Le crt0 du target rpc8e fait `xce` puis `sep #$30` → mode 8-bit native.
- v2 acte la contrainte : **apps C = mono-bank 8-bit native**. Apps multi-bank ou exigeant registres 16-bit restent **en asm 65C816** (ca65).
- Implication : DEC-3 ratifiée (cf. BACKLOG). Pas de dépendance à des features llvm-mos non-implémentées pour Sprint 4.

Alternatives écartées : tout asm (maintenance et apps lourdes), tout C (perfs kernel dégradées sur boucles serrées et drivers temps réel), attendre llvm-mos #320/#321 (horizon multi-années).

### ADR-06 — Modèle GUI : SymbOS-like (ratifiée 2026-05-07)
GUI multifenêtrée préemptive sur 8/16-bit, drag & drop, menus contextuels, taskbar. Référence directe d'implémentation : **SymbOS**. Cohérent avec ADR-03 (préemptif). Adapté au compositor double ULA d'ADR-02 (la fenêtre guest Oric 1 est une fenêtre OricOS comme une autre, alimentée par l'ULA guest matérielle).

Alternatives écartées : Intuition-like (mémoire excessive), GEM-like (moins moderne), custom (réinvention coûteuse).

### ADR-07 — Système de fichiers : FAT32 SD + hostfs émulateur (ratifiée 2026-05-07)
- **Hardware ULX3S** : FAT32 sur carte SD via SPI (slot natif ULX3S). Lib FAT32 65C816 à écrire ou porter.
- **Émulateur Phosphoric** : option `--hostfs DIR` (déjà existante) en alternative à une image SD FAT32. Le VFS abstraction layer de Phosphoric route entre les deux.

Une **option Sedoric overlay (lecture seule)** sera ajoutée plus tard pour booter des disquettes Oric 1 historiques depuis OricOS (extension naturelle d'ADR-10), mais hors v1.

### ADR-08 — Packaging apps natives : bundle léger (ratifiée 2026-05-07)
Format binaire OricOS = **header magique + table des sections** (code / data / icône / manifest). Concaténation simple, parseable directement en asm 65C816, inspiré d'AmigaOS Hunk simplifié. Permet icônes/métadonnées sans complexité ELF.

Alternatives écartées : flat binaire .com (resources externes coûteuses à gérer), relocatable (overkill v1, peut être ajouté plus tard).

### ADR-09 — Audio : AY-3-8912 + extension SID-like (ratifiée 2026-05-07)
- **AY-3-8912** conservé (compat Oric 1 stricte ADR-10) : 3 voies + bruit + enveloppe.
- **Extension SID-like** ajoutée : 3 voies supplémentaires, filtres, samples PCM 4-8 bits. IP HDL existante pour ULX3S. Standard expressif 8-bit.

Alternatives écartées : AY seul (pauvre pour OricOS), OPL2 (lourd à driver depuis 8-bit), Paula wavetable (moins iconique).

### ADR-14 — Format TCB et table tâches (ratifiée 2026-05-08)

OricOS gère ses tâches via une **table fixe de 16 TCBs** (Task Control
Block) en bank 1 à `$01:5C00`, indexée par PID 8-bit (1..16, 0 réservé
pour invalid).

**Layout TCB** (20 octets) :
```
+0  pid         (1B)  : 0=invalid, 1..16=valid
+1  state       (1B)  : 0=DEAD, 1=READY, 2=RUNNING, 3=BLOCKED, 4=ZOMBIE
+2  prio        (1B)  : 0..7 priority (0=highest)
+3  parent_pid  (1B)  : 0=kernel root
+4  saved_S     (2B)  : stack pointer 16-bit (mode N)
+6  entry_pc    (2B)  : PC initial (respawn)
+8  code_bank   (1B)  : PB initial
+9  data_bank   (1B)  : DBR initial
+10 stack_bank  (1B)  : v1 = 0 (stacks page-based en bank 0)
+11 flags       (1B)  : kernel/user, signal pending, etc.
+12 name        (8B)  : nom debug
                       ─────
                       20B
```

**Allocation** : bitmap 16 bits (2 octets) à `$01:5B00`. `kernel_task_create`
scanne le bitmap pour trouver un slot libre, set le bit, init le TCB.
`kernel_task_destroy` clear le bit (PID réutilisable).

**Scheduling** : round-robin avec priorité v1 (skip tasks de prio plus
basse si une plus haute READY). Sauvegarde S dans `tcb[CUR].saved_S`,
charge `tcb[NEXT].saved_S`. Mode N implicite (mode E réservé au guest
Oric 1, ADR-02).

**Total RAM** : 322 octets en bank 1 ($01:5B00-$01:5D3F).

Référence d'art : SymbOS (8 tâches Z80), GS/OS (32 tâches 65C816). 16
choisi comme compromis pour OricOS GUI multifenêtré + apps utilisateur.

Alternatives écartées :
- **8 tasks** (SymbOS strict) : trop limitant pour GUI + apps simultanées.
- **32 tasks** (GS/OS) : overkill pour OricOS v1.
- **256 tasks** (PID 8-bit full) : 5 KiB RAM, surdimensionné.
- **Liste chaînée** : 10-20% overhead vs table fixe pour N ≤ 32, pas de
  gain réel à cette échelle.

Ouvert : ADR-17 (API kernel publique syscalls liés à tâches —
`task_create`/`destroy`/`yield`/`wait`).

### ADR-13 — Mécanisme de syscall : COP + table (ratifiée 2026-05-08)

OricOS expose ses services à l'userland via l'opcode **`COP #imm`**
(opcode `$02`) en mode N. Le `imm` est une signature/version (`$AA` =
OricOS magic, ignorée pour dispatch v1) ; le **numéro de syscall est
passé en `A`** ; les args en `X`/`Y` selon convention par syscall.

Le vecteur COP mode N est `$00FFE4` (WDC standard). Phosphoric
implémente déjà ce vecteur ; OricOS y installe un trampoline bank 0
(`$0150 → JML $01:5700`) qui route vers `kernel_cop_handler` en bank 1
segment `COP_HANDLER`.

Le handler dispatch via une **table de pointers 16-bit en bank 1**
(`syscall_table` à `$01:5750`, indexée par syscall num × 2). Chaque
entrée pointe vers une routine kernel locale qui consomme les args et
retourne via RTI.

Référence : GS/OS sur Apple IIgs (65C816, COP-based syscalls).

Alternatives écartées :
- **Call gate JSL** : pas de privilege boundary, layout kernel figé
  dans userland, incompatible avec ADR-15 future (MMU).
- **WAI + signal** : pas conçu pour syscall, mélange ordonnanceur
  et appel système.

Impact ouvert : **ADR-17** (liste complète syscalls v1 + ABI versioning)
reste ouverte ; sera tranchée quand le besoin de syscalls se diversifie
(au-delà de print_char pour Sprint 4 userland C).

### ADR-12 — Mode HIRES Oric 2 (ratifiée 2026-05-08)
Le mode vidéo HIRES de l'Oric 2 (utilisé par OricOS) est :

- **Dimensions** : 240×200 pixels (identiques à l'Oric 1 HIRES historique).
- **Encodage** : **3 bits par pixel direct** (8 couleurs par pixel,
  palette = 8 couleurs RGB fixes, identique à l'Oric 1). Pas de notion
  ink/paper séparée — chaque pixel sélectionne directement sa couleur.
- **Framebuffer** : 240×200 × 3 bits = 18 000 octets/frame. Layout
  recommandé : 8 pixels groupés en 24 bits sur 3 octets alignés.
- **Localisation mémoire** : banks 128-191 (cf. `docs/MEMORY_MAP.md`).
- **ULA guest Oric 1** non concernée — elle reste 240×200 attribute-based
  pour la compat stricte (ADR-10).

Justification :
- Simplicité HDL ULX3S : 3 fetches consécutifs par 8 pixels, palette à
  8 entrées trivialement câblée en combinatoire.
- Framebuffer compact (18 KiB) → laisse de la marge BRAM ECP5 pour le
  double-buffering et plusieurs fenêtres OricOS.
- Compatibilité visuelle Oric 1 stricte (mêmes couleurs).
- Scrolling pixel-perfect : pas d'attribute clash.

Alternatives écartées :
- ink/paper indépendants par pixel (7 bpp) : alignement non-trivial,
  ~42 KiB/frame, gain de flexibilité marginal vs (1).
- Bitmap + plan attribut (9 bpp) : ~54 KiB/frame, fetches HW doublés,
  surdimensionné pour le besoin v1.
- Half-attribute Spectrum-like (5 bpp ~) : compromis qui n'est ni
  élégant ni nécessaire.

Modes vidéo additionnels (mode TEXT 40×28 compat Oric 1, modes
étendus 320×240+ pour le desktop OricOS) restent ouverts à de futurs
ADR (notamment lors de B4 v2).

### ADR-21 — GPU Blitter HW autonome (ratifiée 2026-05-09)
OricOS adopte une **architecture GPU command-based** où un co-processeur graphique HDL ECP5 dédié exécute les opérations de dessin en parallèle du CPU 65C816. Le CPU envoie des commandes via I/O ports, le GPU les exécute en accédant directement à la mémoire (BRAM live + SDRAM cold via Arch D / ADR-19).

**Inspiration** : Amiga Blitter + Copper, Atari ST DMA blitter, NES PPU, Apple IIgs Mega II VOC. Le CPU se concentre sur la logique métier ; le GPU s'occupe du dessin.

**Registres I/O ($0340-$034F)** en bank 0 :

| Adresse | Registre | R/W | Description |
|---------|----------|-----|-------------|
| `$0340` | `GPU_CMD_OP` | W | opcode 8-bit |
| `$0341-$0343` | `GPU_ARG1_{LO,MID,HI}` | W | argument 1 (24-bit) |
| `$0344-$0346` | `GPU_ARG2_{LO,MID,HI}` | W | argument 2 |
| `$0347-$0349` | `GPU_ARG3_{LO,MID,HI}` | W | argument 3 |
| `$034A-$034C` | `GPU_ARG4_{LO,MID,HI}` | W | argument 4 |
| `$034D` | `GPU_STATUS` | R | bit 7=busy, 6=err, 5=queue_full, 0=done |
| `$034E` | `GPU_TRIGGER` | W | any value → enqueue command |
| `$034F` | `GPU_INT_CTRL` | R/W | IRQ enable/clear |

**Commandes v1 ratifiées (5 opcodes)** :

| Opcode | Nom | ARG1 | ARG2 | ARG3 | ARG4 |
|--------|-----|------|------|------|------|
| `$01` | `CLEAR` | bank target ($80..$83) | color (0..15) | — | — |
| `$02` | `FILL_RECT` | bank target | (x, y) packed | (w, h) packed | color |
| `$03` | `BLIT` | src_addr 24-bit | dst_addr 24-bit | (w, h) packed | flags (mask, ROP) |
| `$04` | `LINE` | bank target | (x1, y1) | (x2, y2) | color (Bresenham) |
| `$05` | `TEXT` | font_addr 24-bit | string_addr 24-bit | (x, y) packed | (color_fg, len) |

**Format packed** :
- (x, y) : `LO` = x_lo, `MID` = x_hi (1 bit) | y_lo (7 bits), `HI` = y_hi (3 bits) — supporte x ≤ 1023, y ≤ 1023. v1 800×600 utilise x ≤ 799, y ≤ 599.
- (w, h) : pareil, w_max = h_max = 1023.

**Commandes v2 reportées** :
- `SPRITE_DEF` / `SPRITE_MOVE` : sprites HW (cursor souris, anim).
- `SCROLL` : smooth scroll fenêtre.
- `COPPER_LIST` : raster effects (couleur/scroll changeant par scanline).

**Modèle d'utilisation typique** (depuis kernel ou app C) :
```
Helper : kernel_gfx_clear(bank=$80, color=4)
  → écrit registres, trigger, poll busy.

Asynchrone :
  trigger commande, le CPU fait autre chose,
  reçoit IRQ "done" en fin d'exec.
```

Le GPU exécute la commande **en parallèle du CPU** : pendant que le GPU dessine, le CPU peut faire la logique applicative. Le GPU lit/écrit dans la mémoire (bank live BRAM ou SDRAM cold) sans intervention CPU.

**Implications projet** :
- **Phosphoric** : ajouter `src/io/gpu_device.{c,h}` simulant le GPU. v0.1 synchrone (commandes "instantanées"), v0.2 async avec cycles.
- **OricOS kernel** : helpers `kernel_gfx_clear`, `kernel_gfx_fill_rect`, `kernel_gfx_blit`, `kernel_gfx_line`, `kernel_gfx_text`. Refactor de `kernel_hires2_*` (Sprint 3.b) vers cette API.
- **HDL ULX3S** : implémentation GPU ECP5 — **projet majeur (~6-8 semaines)** pour les 5 commandes. Décomposition par command :
  - SP-GPU-HDL-1 : CLEAR + FILL_RECT (~2 semaines)
  - SP-GPU-HDL-2 : BLIT (~2 semaines)
  - SP-GPU-HDL-3 : LINE Bresenham (~1 semaine)
  - SP-GPU-HDL-4 : TEXT avec font ROM (~2 semaines)

**Conséquences sur ADR existantes** :
- **ADR-12** (HIRES Oric 2 240×200×3bpp) : reste valide pour **ULA guest** uniquement (compat Oric 1). Plus utilisée pour desktop.
- **ADR-19** (VRAM hybride) : conservée. BRAM live = framebuffer principal cible GPU. SDRAM cold = backing-stores + assets (fontes, sprites).
- **Sprint 3.b** (`kernel_hires2_clear`, `kernel_fill_rect_aligned`) : code conservé comme **legacy fallback** mais l'API publique kernel devient `kernel_gfx_*`.

Alternatives écartées :
- (a) **Approche B hybride progressif** (v1 CPU+DMA, v2 GPU) : refactor kernel douloureux à v2.
- (b) **CPU-only** : trop lent pour résolution > 320×240 et apps fluides.

### ADR-20 — Mode HIRES Oric 2 desktop : 800×600×4bpp SVGA (ratifiée 2026-05-09)
Avec GPU Blitter HW (ADR-21) qui décharge le CPU, OricOS vise une résolution desktop **ambitieuse SVGA** :

- **Résolution** : 800×600 pixels (format 4:3 SVGA standard).
- **Profondeur** : 4 bits par pixel = 16 couleurs simultanées.
- **Palette** : 16 couleurs fixes style VGA-IBM (v2 indexable 16 sur 4096).
- **Layout pixel** : 2 pixels groupés en 8 bits, big-endian :
  - Octet n bits [7:4] = pixel 0 (gauche)
  - Octet n bits [3:0] = pixel 1 (droite)
- **Adressage** : 800/2 = **400 octets/ligne** × 600 lignes = **240 000 octets/frame**.
- **Localisation** : **4 banks live BRAM** (128-131) selon ADR-19.

**Layout 4 banks (préliminaire, à figer en SP-GPU-1)** :
- Option A — **lignes alignées par bank** (recommandé HDL) :
  - Bank 128 : lignes 0..162 (163 lignes × 400 = 65 200 octets, padding 336 bytes)
  - Bank 129 : lignes 163..325
  - Bank 130 : lignes 326..488
  - Bank 131 : lignes 489..599 (111 lignes, 44 400 octets utilisés)
- Option B — packing strict 240 KiB linéaires sans padding (ligne cross-bank).

**Palette VGA-IBM standard** :

| Idx | Nom | RGB |
|-----|-----|-----|
| 0 | black | (0,0,0) |
| 1 | blue | (0,0,170) |
| 2 | green | (0,170,0) |
| 3 | cyan | (0,170,170) |
| 4 | red | (170,0,0) |
| 5 | magenta | (170,0,170) |
| 6 | brown | (170,85,0) |
| 7 | lightgray | (170,170,170) |
| 8 | darkgray | (85,85,85) |
| 9 | lightblue | (85,85,255) |
| 10 | lightgreen | (85,255,85) |
| 11 | lightcyan | (85,255,255) |
| 12 | lightred | (255,85,85) |
| 13 | lightmagenta | (255,85,255) |
| 14 | yellow | (255,255,85) |
| 15 | white | (255,255,255) |

**HDMI** : 800×600 60Hz = 40 MHz pixel clock. Largement supporté ULX3S (LFE5U-85F PLL OK).

Justification :
- **Ambitieuse mais réaliste** avec GPU autonome (ADR-21). Le CPU n'a plus à dessiner.
- **Référence d'art** : Amiga ECS (640×512×4bpp), OS/2 Warp, Win 3.1 (640×480×16 ou SVGA add-on).
- **Productivité** : ≈ 6× plus de surface qu'un Oric 1 historique.
- **Pixel-square 4:3** : fidèle aux standards 80s/90s.

Alternatives écartées :
- 320×240×4bpp : trop modeste vu que GPU n'est plus bottleneck.
- 640×480×4bpp : VGA standard, mais moins ambitieux.
- 1024×768×4bpp : 393 KiB = 6 banks live, raster timing 65 MHz tendu LFE5U-85F. Reportable v2.

Implications :
- **Phosphoric** : module `video/hires_oric2_svga.{c,h}` (ou réutilisation refactorée du module ADR-12).
- **OricOS kernel** : constantes `HIRES2X_W=800`, `HIRES2X_H=600`, `HIRES2X_BPL=400`, `HIRES2X_FB_BANKS=4`.
- **HDL ULX3S** : raster controller multi-bank (lit 4 banks BRAM séquentiellement par scanline).

### ADR-19 — VRAM hybride : BRAM live + SDRAM cold via I/O (ratifiée 2026-05-09)
OricOS adopte une architecture VRAM **à deux niveaux** pour combiner accès pixel direct rapide et capacité de stockage massive :

**VRAM "live"** :
- Banks **128-159** (32 banks × 64 KiB = **2 MiB**) physiquement implémentés en **BRAM ECP5** côté HDL ULX3S.
- Accès CPU **direct** via banking 24-bit (`STA al`, `STA [dp],Y`, etc.). Latence 1 cycle.
- Capacité visée : framebuffer principal `host` (mode HIRES Oric 2) + 4-8 backing-stores fenêtres simultanément en focus.
- Le **compositor matériel ULA host** lit ces banks à fréquence pixel HDMI.

**VRAM "cold"** :
- **SDRAM ULX3S** (32 MiB), accessible **uniquement via I/O ports MMIO** depuis le CPU (pas dans le banking 24-bit).
- Ports en bank 0 (DBR=0) :
  - `$0330` `VRAM_ADDR_LO`  (R/W)
  - `$0331` `VRAM_ADDR_MID` (R/W)
  - `$0332` `VRAM_ADDR_HI`  (R/W) — adresse 24-bit dans SDRAM
  - `$0333` `VRAM_DATA`     (R/W, **auto-increment ADDR**)
  - `$0334` `VRAM_DMA_CTRL` (W = trigger, R bit 7 = busy)
  - `$0335-$0337` `VRAM_DMA_SRC_{LO,MID,HI}` — adresse source 24-bit (interprétée selon `VRAM_DMA_CTRL` bit 0/1 = SDRAM/bank live)
  - `$0338-$033A` `VRAM_DMA_DST_{LO,MID,HI}` — adresse dest 24-bit
  - `$033B-$033C` `VRAM_DMA_LEN_{LO,HI}` — longueur 16-bit (max 64 KiB par DMA)
- Usage : backing-stores fenêtres iconifiées, sprites/fontes/icônes, streams, code apps en pagination.
- **DMA matériel** : copie SDRAM ↔ banks live (2 MiB BRAM) sans bouger le CPU. Drives de blits massifs (drag fenêtre, restore from icon).

**Banks 160-191** : redeviennent **"Réservés extensions futures"** (pas VRAM directe). Cf. ADR-12 inchangée mais avec localisation framebuffer = bank 128 (live).

Justification :
- Capacité scalable : 32 MiB SDRAM → 100+ fenêtres iconifiées + assets gros.
- Performance optimale fenêtre active : `STA al` direct.
- DMA HW pour transferts massifs sans cycle CPU perdu (déplacement fenêtre = quelques µs, pas 38 000 cycles).
- Référence d'art : **Amiga (chip RAM + fast RAM)**, **Apple IIgs (system RAM + slot RAM)**.

Alternatives écartées :
- (a) **Arch A pure (banking-only)** : limité à ~3-4 MiB pratique, pas scalable pour 100+ fenêtres iconifiées.
- (b) **Arch B I/O VRAM pure** : pas d'accès pixel direct, freine apps qui dessinent point-par-point (tous les pixels via I/O port = 3-4× cycles).
- (c) **Arch C window mapping** : memory remap HDL complexe, latence cache, debugging non-trivial.

Implications projet :
- **Phosphoric** : ajouter `src/io/vram_device.{c,h}` (32 MiB simulés en heap, ports `$0330-$033C`).
- **OricOS kernel** : helpers `kernel_vram_read_block`, `kernel_vram_write_block`, `kernel_vram_dma`. Allocator `kernel_alloc_bank` distingue pool "live" (banks 128-159) du pool système (banks 4-127).
- **HDL ULX3S** : DMA controller SDRAM↔BRAM, address latch + auto-increment, refresh SDRAM, controller raster.
- **MEMORY_MAP.md §8** : refonte (live BRAM banks 128-159, cold SDRAM via I/O) — cf. ADR-19 §implications.

### ADR-11 — Sémantique du mode E vis-à-vis du NMOS 6502 (ratifiée 2026-05-07)
Le 65C816 en mode E adopte un comportement **hybride pragmatique** :
- **Bug `JMP ($xxFF)`** : reproduit (le high byte est lu à `(ptr & 0xFF00) | ((ptr+1) & 0xFF)`, conforme NMOS).
- **Opcodes illégaux NMOS** (LAX, SAX, DCP, RLA, SLO, etc.) : traités comme NOP (consomment leur taille d'opérande, sans effet). Exception : `$EB` est un alias officieux de SBC immediate, conservé.
- **Séquence RMW interne** (read-read-modify-write vs read-modify-write) : suit le WDC, non observable côté software pur.

Justification : aligne le mode E sur le comportement de facto du cœur 6502 Phosphoric (golden model), évite d'ajouter des opcodes illégaux dans les deux cœurs, préserve les détecteurs CPU dépendant du bug JMP indirect. Le HDL ULX3S devra reproduire ce comportement hybride.

Alternatives écartées :
- (a) Strict NMOS — exigerait d'ajouter les opcodes illégaux dans les deux cœurs ; gain compat infinitésimal.
- (b) WDC strict — exigerait de retirer le bug JMP indirect du 6502 existant ; casse `test_cpu_indirect_jmp_bug` et potentiellement quelques logiciels Oric 1.

---

## 3. Décisions ouvertes (ADR à instruire — NE PAS trancher unilatéralement)

Si une tâche force la main sur l'une de ces décisions, **arrête et demande**. Ne choisis pas par défaut.

Ces ADR ont été ouvertes le **2026-05-08** suite au point critique architecte
senior (cf. `BACKLOG.md` §annexe). Elles correspondent à des décisions
prises tacitement dans les premiers sprints OricOS et qu'il faut
expliciter avant d'avancer.

### ~~ADR-13~~ → ratifiée 2026-05-08, déplacée vers §2 (option a : COP + table)

### ~~ADR-14~~ → ratifiée 2026-05-08, déplacée vers §2 (table fixe 16 + bitmap free)

### ADR-15 — Isolation mémoire post-v1
**Question** : à quoi ressemble la v2 d'ADR-04 ?
- (a) MMU custom HDL ECP5 (translation table par bank, BRAM).
- (b) MPU à segments avec privilege bits (kernel/user).
- (c) Banking matériel étendu avec tags d'accès.

**Impact** : multitâche robuste, exécution apps non-trusted. Pour Q4 2026.

### ADR-16 — Driver model
**Question** : comment un driver est structuré dans OricOS ?
- IRQ-driven pur (handler + queue ?).
- Polling depuis idle task ?
- Hybride avec callback registration ?
- Quelle interface (struct ops ? jump table ?) ?

**Impact** : forme tous les drivers à venir (clavier, FAT32, audio).

### ADR-17 — API kernel publique exposée à userland
**Question** : quelle ABI stable expose-t-on à une app C ?
- Liste minimale syscalls v1 (`open/read/write/close/exec/exit/alloc_bank/...`).
- Convention d'appel (registres, stack frame).
- Versioning de l'ABI.

**Impact** : ABI = contrat long terme. Décide avec ADR-13.

### ADR-18 — Sort du 6502 dans Phosphoric
**Question** : maintient-on la cohabitation 6502/65C816 indéfiniment ?
- (a) Retrait du 6502 après B1.6 stable (mode E remplace tout).
- (b) Cohabitation perpétuelle (régression-protection).
- (c) Bascule conditionnelle compile-time (`-DLEGACY_6502`).

**Impact** : surface de maintenance Phosphoric, dette doublée actuellement.

---

## 4. Roadmap et jalon courant

### Track B — Prototype Phosphoric (en cours)

- **B1 — Cœur 65C816 dans Phosphoric** ← **JALON COURANT**
- B2 — Modèle mémoire bank-based 256K minimum
- B3 — Démonstrateur bascule mode + virtualisation guest (stub kernel)
- B4 — Compositor logiciel + double ULA

### Track A — Document d'architecture (DAT)
À rédiger en parallèle, format IEEE 42010. Pas le sujet de Claude Code pour l'instant — Claude Code se concentre sur l'implémentation B1.

### Critères d'acceptation B1

- Le CPU core supporte l'intégralité du jeu d'instructions 65C816 (modes E et N).
- En mode émulation, la suite de tests Oric 1 existante de Phosphoric passe **à 100 %** sans régression.
- Cycle-accurate vérifié contre la suite Klaus Dormann 6502 (ou équivalent à valider) en mode émulation.
- Les nouveaux opcodes 65C816 sont couverts par des tests unitaires dédiés.
- Le flag E et la bascule XCE sont implémentés et testés.
- Aucun changement de comportement pour la ROM Oric 1 originale.

---

## 5. Contraintes dures

À respecter sans exception. Une PR qui les viole doit être refusée.

1. **Compat Oric 1 stricte** en mode émulation. Le ROM original boote et fonctionne identiquement. Aucune régression sur la suite de tests existante.
2. **Ne jamais modifier l'image ROM Oric 1**. Elle est immuable.
3. **Multiplateforme préservé**. Pas de dépendance OS-spécifique non portable. Le code doit compiler sur Linux, macOS, Windows comme actuellement.
4. **Cycle accuracy en mode émulation**. Pas de relaxation pour gagner en vitesse d'implémentation.
5. **Toute décision d'architecture trace à une ADR**. Si une décision n'a pas d'ADR, elle est ouverte (cf. §3) et doit être posée à l'humain.
6. **Nouvelles fonctionnalités gated derrière flags explicites**. Aucun changement de comportement caché. L'Oric 2 ne s'active jamais en mode legacy.
7. **Pas de gros refactor sans demande explicite**. Modifications minimales et chirurgicales.
8. **Aucune dépendance externe nouvelle sans validation**. Phosphoric doit rester léger et auto-contenu.

---

## 5bis. Layout du repo et commandes

Le code source de Phosphoric est importé sous `Phosphoric/` (cloné depuis `https://git.nagominosato.fr:6775/chipinette/Phosphoric`). Un `CLAUDE.md` détaillé existe dans ce sous-répertoire — **le lire en complément** pour l'architecture interne (sous-systèmes 6502, VIA, ULA, AY, Microdisc, etc.), le framework de tests, et le détail de tous les targets Make.

Toutes les commandes ci-dessous s'exécutent depuis `Phosphoric/` :

```bash
make SDL2=1              # Build standard
make DEBUG=1 SDL2=1      # Build debug (-g -O0)
make tests               # Lance toutes les suites de tests (doit passer à 100 % avant commit)
make test-cpu            # Suite CPU isolée (cf. CLAUDE.md de Phosphoric pour la liste complète)
make valgrind            # Détection de fuites mémoire
make clean
```

Lancer un binaire :

```bash
./oric1-emu -r roms/basic10.rom        # ORIC-1 BASIC 1.0
./oric1-emu -r roms/basic11b.rom       # Atmos BASIC 1.1 (auto-détecté)
```

Le compteur de tests autoritatif est dans `Phosphoric/VERSION_TRACKING`. Le critère B1 (§4) « 100 % de la suite Oric 1 sans régression » s'évalue contre `make tests`.

## 6. Conventions de code et workflow

### Style
- Le code est en **C**. Suivre rigoureusement les conventions déjà en place dans Phosphoric (indentation, nommage, organisation des fichiers, headers). Lire les fichiers existants avant d'écrire.
- Les noms de symboles 65C816 suivent la nomenclature WDC officielle (registres A, X, Y, S, D, DBR, PBR, P, PC ; flags N, V, M, X, D, I, Z, C, E ; instructions canoniques).
- Commentaires en français acceptés, mais les noms de symboles, fonctions et types restent en anglais (cohérence avec le code existant).

### Organisation
- Le cœur 65C816 est isolé dans son propre module. Le 6502 existant reste fonctionnel et indépendant tant que la transition n'est pas complète, ou est remplacé proprement si plus simple — **à valider avant exécution**.
- Les nouveaux modes Oric 2 (ULA étendue, banking, registres I/O Oric 2) sont gated derrière un flag de mode machine explicite.

### Workflow
- **Petits commits atomiques**. Un changement logique = un commit. Message clair, format `[B1] Implement XCE instruction and E flag handling`.
- **Tests systématiquement**. Tout nouveau opcode, tout nouveau comportement, a un test unitaire.
- **Référencer les ADR en commentaire** quand une décision n'est pas évidente : `/* ADR-01: emulation mode preserves 6502 cycle counts */`
- **Avant tout refactor structurel**, demander confirmation à l'humain.

---

## 7. Critères de validation continue

À chaque modification, vérifier dans cet ordre :

1. La compilation passe sur la plateforme de dev.
2. La suite de tests Oric 1 existante passe à 100 %.
3. Les nouveaux tests ajoutés passent.
4. (Si pertinent) Bench comparatif de performance — pas de régression > 5 % sans justification.
5. La ROM Oric 1 boote visuellement et exécute un programme BASIC simple.

---

## 8. Glossaire

- **ULA** — Uncommitted Logic Array. Sur Oric 1, gère le timing vidéo, la lecture VRAM, la génération couleur attribute-based.
- **ULA host** — Instance ULA générant le framebuffer principal d'OricOS (modes Oric 2 étendus).
- **ULA guest** — Instance ULA dédiée au rendu de la machine Oric 1 virtualisée.
- **Mode E (émulation)** — État du 65C816 où il se comporte en 6502 strict.
- **Mode N (natif)** — État 16 bits du 65C816 utilisé par OricOS.
- **XCE** — Exchange Carry/Emulation : instruction 65C816 qui bascule entre mode E et N.
- **Bank** — Page de 64K dans l'espace d'adressage 65C816. PBR pour les instructions, DBR pour les données.
- **PBR / DBR / DPR** — Program Bank Register, Data Bank Register, Direct Page Register.
- **Compositor** — Logique (logicielle dans Phosphoric, matérielle en HDL) qui combine framebuffers host et guest à la sortie vidéo.
- **OricOS** — Système d'exploitation natif Oric 2.
- **Guest** — Instance Oric 1 virtualisée dans une fenêtre OricOS.
- **Host** — La plateforme Oric 2 / OricOS qui héberge le guest.
- **Golden model** — Phosphoric sert de référence comportementale pour le futur HDL.
- **Paravirtualisation** — Stratégie de virtualisation où le guest s'exécute nativement sur le CPU (mode E ici), avec coopération minimale du kernel host.

---

## 9. Références techniques

- **WDC W65C816S Datasheet** — référence opcodes et timings.
- **The Western Design Center 65C816 Programming Manual** — sémantique détaillée des modes E/N, vecteurs, banking.
- **The Oric Hardware Manual** (community) — comportement ULA, VIA 6522, timing PAL.
- **Klaus Dormann's 6502 functional test** — suite de validation cycle-exact en mode émulation.
- **ULX3S documentation** — cible HDL finale (ECP5, BRAM, HDMI tx, SDRAM).
- **IEEE 42010** — standard d'architecture appliqué au DAT (track A).
- **SymbOS sources** — référence multitâche préemptif fenêtré sur 8 bits.
- **GS/OS** — référence OS sur 65C816 avec mode legacy.

---

## 10. Méta — Évolution de ce document

- Ce document est versionné dans le repo et fait partie intégrante du projet.
- Toute modification d'une ADR ratifiée passe par discussion explicite avec l'humain et mise à jour de §2.
- Toute ADR ouverte (§3) qui se ferme rejoint §2 avec sa date de ratification.
- Le jalon courant (§4) est mis à jour à chaque transition B1→B2→B3→B4.
- Si Claude Code identifie une décision implicite non documentée, il l'ajoute en §3 comme ADR ouverte plutôt que de trancher.

---

*Dernière révision : v0.1 — initialisation projet Oric 2.*
