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
| `$01` | `CLEAR` | base SDRAM 24-bit | size (octets) 24-bit | LO = color (0..15) | — |
| `$02` | `FILL_RECT` | base SDRAM 24-bit | LO=x, MID=y (8-bit v0.1) | LO=w, MID=h (8-bit v0.1) | LO = color (0..15) |
| `$03` | `BLIT` | src_addr 24-bit | dst_addr 24-bit | LO=byte_w, MID=byte_h (8-bit v0.1) | flags (unused v0.1) |
| `$04` | `LINE` | base SDRAM 24-bit | LO=x1, MID=y1 (8-bit v0.1) | LO=x2, MID=y2 (8-bit v0.1) | LO = color (0..15) |
| `$05` | `TEXT` | base SDRAM 24-bit | font_addr 24-bit | string_addr 24-bit (null-term) | LO=x, MID=y, HI=color_fg (8-bit v0.1) |

> **Note de révision v1.1 (2026-05-23)** : suite à la ratification d'ADR-19 v2 (VRAM SDRAM unifiée), les
> commandes CLEAR/FILL_RECT/LINE opèrent sur des adresses SDRAM 24-bit directes et non plus sur des
> banks cibles ($80..$83). Le tableau ci-dessus reflète l'implémentation réelle de Phosphoric v0.1.
> En v0.2 les coordonnées x/y/w/h passeront en 16-bit via le format packed XVGA.

**Format packed v0.2 (implémenté 2026-05-24, `FILL_RECT16` opcode `$06`)** :
- Packing **12-bit par coordonnée** (couvre XVGA 1024×768, marge à 4095) :
  `ARG2 = y<<12 | x`, `ARG3 = h<<12 | w` (chaque coord 0..4095).
- `FILL_RECT` (opcode `$02`, 8-bit) conservé ; `FILL_RECT16` (`$06`) ajouté pour
  les fenêtres plein écran. `kernel_gfx_fill_rect16` + `kernel_wm_redraw` côté OricOS.
- BLIT/LINE/TEXT 16-bit : reportés (même schéma de packing 12-bit).

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

### ADR-22 — Clavier Oric 2 paravirtualisé hybride (ratifiée 2026-05-23)

L'entrée clavier Oric 2 suit le modèle **double-ULA d'ADR-02** : une source
physique de touches alimente deux chemins matériels distincts.

- **Hôte OricOS** : contrôleur clavier Oric 2 natif **IRQ-driven**, registres
  `$0350-$035F` (bank 0). FIFO d'ASCII (keymap faite côté HW/modèle, pas dans le
  kernel) ; IRQ sur touche ; l'hôte ne scanne plus la matrice.
- **Guest Oric 1** : matrice 8×8 virtuelle alimentée par le même contrôleur,
  lue via le chemin VIA PB / PSG R14 normal (compat stricte ADR-10 préservée).
  Routage par le bit `route_to_guest` de `KBD2_CTRL`.

**Registres** : `$0350` STATUS (data_ready/overflow/guest_focus), `$0351` DATA
(pop FIFO ASCII), `$0352` CTRL (IRQ enable / clear / route_to_guest), `$0353`
MOD (SHIFT/CTRL/FUNCT/CAPS), `$0354-$035F` réservé v2.

**Impact** : révise la ligne clavier d'ADR-16 (source VIA T1 scan → IRQ KBD2) ;
ring kernel `$5860` et ABI syscall (ADR-17 SYS_GET_KEY/READ_CHAR) inchangés.
Phosphoric modélise `src/io/kbd2_device.{c,h}` (gated `--machine oric2`).

Alternatives écartées : (a) matrice Oric 1 réutilisée (scan ~830 cyc > tick,
pas de séparation host/guest, dette HDL reportée) ; (b) clavier natif seul
(casse ADR-10, le guest n'a plus de matrice). Cf. `docs/adr/0022-clavier-oric2-paravirt.md`.

### ADR-23 — Console OricOS : flux de caractères, backend interchangeable (ratifiée 2026-05-24)

Le console kernel OricOS est un **flux de caractères** ; le backend d'affichage
est un **détail d'implémentation interchangeable**. Backend v1 = mode texte
Oric 1 (`$BB80`, ULA, 40×28) en **bootstrap** ; cible = `kernel_gfx_text` (GPU,
ADR-21) sur framebuffer XVGA (ADR-20).

**Règle d'or** : l'ABI publique (`kernel_print_char`/`print_string`,
`SYS_PRINT_CHAR`/`SYS_PRINT_STRING`) ne doit **jamais** exposer de géométrie
(40×28), d'adresse écran (`$BB80`), de curseur en adresse linéaire, ni de
sémantique d'attribut sériel Oric 1. Ces éléments restent privés au backend.

**Contraintes de bordure** : (1) aucun futur syscall de positionnement/couleur
calqué Oric 1 (cibler le modèle XVGA, via révision ADR-17) ; (2) pas d'app
userland supposant 40×28/`$BB80`/attributs avant que le backend GPU console
existe (fenêtre de risque = Sprint 4) ; (3) `$BB80`/40×28 = détail privé (seules
fuites tolérées : assertions de test) ; (4) à terme le mode texte Oric 1
redevient exclusivement guest (ADR-02).

Justification : la dette du backend Oric 1 reste **légère et bornée** (~140
lignes asm, ABI déjà agnostique) tant que la règle d'or tient ; migration GPU =
réécriture locale derrière l'API. Alternatives écartées : (a) figer le texte
Oric 1 comme mode officiel Oric 2 (attribute clash, incompatible XVGA) ; (b)
bascule GPU immédiate (retarde le boot/debug). Cf. `docs/adr/0023-console-flux-caracteres.md`.

### ADR-24 — Contrôleur souris Oric 2 (ratifiée 2026-05-24)

Périphérique de pointage **Oric 2 natif** (l'Oric 1 n'a pas de souris → pas de
paravirtualisation guest), prérequis de SP-3.e (event loop GUI focus/drag).

- **Registres `$0360-$036F`** (bank 0) : `$0360` STATUS (event + boutons G/D/M),
  `$0361-$0364` X/Y **position absolue 10-bit** clampée XVGA, `$0365` BUTTONS,
  `$0366` CTRL (IRQ enable / clear event), `$0367-$0368` DX/DY deltas signés
  **read-clear**, `$0369-$036F` réservé v2.
- **Modèle hybride** : v0.1 OricOS lit la position absolue (driver minimal) ;
  les deltas restent dispo pour accélération/usage avancé v2 **sans nouvelle ADR**.
- **IRQ** dédiée `IRQF_MOU2`, event-driven (ADR-16 classe 1, comme le clavier).
- Phosphoric : `src/io/mouse2_device.{c,h}`, alimenté par les events SDL souris,
  gated `--machine oric2`. Révise ADR-16 (ajout ligne souris).

Alternatives écartées : absolu seul (couple device↔résolution, fermé) ; deltas
seuls (+code kernel, overflow). Cf. `docs/adr/0024-souris-oric2.md`.

### ADR-20 — Mode HIRES Oric 2 desktop : 1024×768×4bpp XVGA (ratifiée 2026-05-09, **révisée v3 2026-05-09** : SVGA → XVGA après simplification ADR-19 v2)

Avec GPU Blitter HW (ADR-21) qui décharge le CPU + VRAM SDRAM unifiée (ADR-19 v2) qui retire la contrainte BRAM, OricOS vise une résolution desktop **XVGA** :

- **Résolution** : 1024×768 pixels (format 4:3 XVGA standard).
- **Profondeur** : 4 bits par pixel = 16 couleurs simultanées.
- **Palette** : 16 couleurs fixes style VGA-IBM (v2 indexable 16 sur 4096).
- **Layout pixel** : 2 pixels groupés en 8 bits, big-endian :
  - Octet n bits [7:4] = pixel 0 (gauche)
  - Octet n bits [3:0] = pixel 1 (droite)
- **Adressage** : 1024/2 = **512 octets/ligne** × 768 lignes = **393 216 octets/frame** (= 384 KiB).

**Localisation** : framebuffer en SDRAM (cf. ADR-19 v2). Adresse `$000000-$05FFFF` (= 0..393 215, 384 KiB linéaires contigus).

**HDMI** : pixel clock 1024×768@60Hz = **65 MHz** (VESA standard). LFE5U-85F PLL trivial (≪ limite 250 MHz).

**Évolution résolutions** :
- v1 : 240×200×3bpp (= ADR-12, mode HIRES Oric 2 compat ULA guest).
- v2 : 800×600×4bpp SVGA (= ADR-20 v2, libéré BRAM grâce à ADR-21).
- **v3 : 1024×768×4bpp XVGA** (= ADR-20 v3, libéré contrainte BRAM via ADR-19 v2 SDRAM unifiée).

Justification :
- **Format 4:3 productif** standard, supporté universellement (HDMI/DVI/CRT).
- **786 432 pixels** = +60% surface vs SVGA, look "OS pro" (Win 95, OS/2 Warp).
- **384 KiB framebuffer** = 2.4% des 16 MiB SDRAM v1. Négligeable.
- **Pixel clock 65 MHz** : trivial pour ULX3S.
- **Effort HDL +30%** vs SVGA (PLL ajustée, raster timing standard).

Alternatives écartées :
- 800×600 SVGA (ADR-20 v2) : moins productif que XVGA, marge HDL trop grande inutilisée.
- 1280×720 HD 16:9 : pixels rectangulaires moins fidèles retro 8/16-bit (préfère 4:3).
- 1280×1024 SXGA : plus exigeant HDL (108 MHz), peu de gain visuel.
- 1920×1080 FHD : effort HDL +200%, format 16:9 non-retro, surdimensionné.
- XVGA 8bpp 256 couleurs : reportable v2 si demande de couleurs riches émerge.

Implications :
- **Phosphoric** : module `video/hires_oric2_xvga.{c,h}` (à créer SP-GPU-1) ou refactor `hires_oric2.c` original. Render 1024×768×4bpp ARGB.
- **OricOS kernel** : constantes `HIRES2X_W=1024`, `HIRES2X_H=768`, `HIRES2X_BPL=512`, `HIRES2X_FB_SIZE=393216`.
- **HDL ULX3S** : raster controller à 65 MHz pixel clock, PLL ajustée. Compositor lit 512 octets de SDRAM par scanline via line-buffer BRAM (1 BRAM 18Kb = 2 KiB suffit).
- **GPU performance** : CLEAR XVGA = 384 KiB writes. À 100 MHz GPU = 3.8 ms (1/4 frame 60Hz). OK fluide.

**Capacité fenêtres** (avec 16 MiB SDRAM, framebuffer 384 KiB) :
- Disponible backing-stores : 15.6 MiB.
- Mini fenêtre 200×150×4bpp = 15 KiB → ~1050.
- Standard 320×240×4bpp = 38 KiB → ~410.
- Plein écran 1024×768×4bpp = 384 KiB → ~40.

**Palette VGA-IBM standard (ADR-20)** :

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

### ADR-19 — VRAM en SDRAM unifiée (ratifiée 2026-05-09, **révisée v2 2026-05-09** suite ADR-21)
**v1 (caduque)** : architecture hybride avec BRAM live (banks 128-159) + SDRAM cold via I/O. Les banks live offraient un accès CPU pixel-direct rapide.

**v2 (actuelle)** : avec ratification d'ADR-21 (GPU Blitter HW autonome), le CPU n'écrit **plus** de pixels directement — il envoie des commandes au GPU. Les banks 128-159 dédiées VRAM live perdent leur raison d'être. Architecture **simplifiée** :

**Toute la VRAM** réside en **SDRAM ULX3S (32 MiB, v1 expose 16 MiB via 24-bit address)**, accessible :
- **Par le GPU Blitter HW (ADR-21)** : accès direct, pleine vitesse SDRAM. Commandes graphiques opèrent directement en SDRAM.
- **Par le CPU** : via I/O ports MMIO `$0330-$033C` (auto-increment + DMA). Usage minoritaire : debug, init, fallback exceptionnel.

**Banks 128-255 (8 MiB) libérés** pour usage RAM extra (apps gourmandes, code paging, buffers utilisateur, ROM cartouche). Total RAM banking accessible CPU = **191 banks × 64 KiB ≈ 12 MiB** (vs 8 MiB dans v1).

**BRAM ECP5 ULX3S** redéployée :
- v1 : 32 banks live (= 2 MiB) accessible CPU + cache compositor.
- v2 : caches internes GPU/compositor uniquement (line-buffers raster, sprite cache, font cache, command queue). **Invisible côté CPU**.

**Ports I/O VRAM (inchangés v1→v2)** :

| Adresse | Registre | R/W | Description |
|---------|----------|-----|-------------|
| `$0330-$0332` | `VRAM_ADDR_{LO,MID,HI}` | R/W | adresse 24-bit dans SDRAM |
| `$0333` | `VRAM_DATA` | R/W | byte courant, **auto-increment ADDR** |
| `$0334` | `VRAM_DMA_CTRL` | R/W | bit 0 trigger, bit 1 dir, bit 7 busy |
| `$0335-$0337` | `VRAM_DMA_SRC_{LO,MID,HI}` | R/W | source 24-bit |
| `$0338-$033A` | `VRAM_DMA_DST_{LO,MID,HI}` | R/W | dest 24-bit |
| `$033B-$033C` | `VRAM_DMA_LEN_{LO,HI}` | R/W | longueur 16-bit |

DMA HW utile pour :
- Init initial (kernel charge fonts/icônes en SDRAM au boot).
- Migration RAM ↔ VRAM (apps qui veulent pré-calculer en RAM puis envoyer au GPU).
- Debug / dumps.

Justification v2 :
- **Cohérence avec GPU autonome** : 1 seul espace VRAM, pas de hiérarchie live/cold redondante.
- **+4 MiB RAM utilisable** pour apps (banks 128-191 libérés).
- **HDL plus simple** : 1 controller SDRAM, BRAM = caches internes, pas exposée CPU.
- **Architecture moderne unifiée** : style consoles / Amiga (chip RAM = SDRAM unifiée, pas de niveaux multi-tier exposés au CPU).

Alternatives écartées :
- v1 hybride BRAM live + SDRAM cold : redondant avec GPU autonome.
- "1 bank live scratch" : compromis sans valeur claire.

Implications projet :
- **Phosphoric** : `vram_device.{c,h}` reste valide (Sprint VRAM-1, 16 MiB simulés). v2 pourrait étendre à 32 MiB.
- **OricOS kernel** : helpers `kernel_alloc_live_bank` / `free_live_bank` (Sprint VRAM-3) restent valides mais sémantique élargie : pool RAM extra (banks 128-191) pour apps gourmandes, plus dédié uniquement à VRAM.
- **MEMORY_MAP §8/9/10** : refondus.
- **Sprint 3.b kernel_hires2_*** : code conservé en legacy fallback (écrit en bank 128 mais bank 128 = RAM normale, plus visible compositor). Effectivement obsolète, à retirer en Sprint 3.b cleanup.

### ADR-11 — Sémantique du mode E vis-à-vis du NMOS 6502 (ratifiée 2026-05-07)
Le 65C816 en mode E adopte un comportement **hybride pragmatique** :
- **Bug `JMP ($xxFF)`** : reproduit (le high byte est lu à `(ptr & 0xFF00) | ((ptr+1) & 0xFF)`, conforme NMOS).
- **Opcodes illégaux NMOS** (LAX, SAX, DCP, RLA, SLO, etc.) : traités comme NOP (consomment leur taille d'opérande, sans effet). Exception : `$EB` est un alias officieux de SBC immediate, conservé.
- **Séquence RMW interne** (read-read-modify-write vs read-modify-write) : suit le WDC, non observable côté software pur.

Justification : aligne le mode E sur le comportement de facto du cœur 6502 Phosphoric (golden model), évite d'ajouter des opcodes illégaux dans les deux cœurs, préserve les détecteurs CPU dépendant du bug JMP indirect. Le HDL ULX3S devra reproduire ce comportement hybride.

Alternatives écartées :
- (a) Strict NMOS — exigerait d'ajouter les opcodes illégaux dans les deux cœurs ; gain compat infinitésimal.
- (b) WDC strict — exigerait de retirer le bug JMP indirect du 6502 existant ; casse `test_cpu_indirect_jmp_bug` et potentiellement quelques logiciels Oric 1.

### ADR-16 — Driver model OricOS (ratifiée 2026-05-09)

OricOS adopte un **modèle hybride** pour ses drivers. Pas de struct ops formelle en v1 ; le kernel reste monolithique avec convention de nommage `kernel_<drv>_<op>`.

**Classe 1 — IRQ-driven event queue** (latence faible critique) :

| Driver | Source IRQ | Event queue | Wakeup userland |
|---|---|---|---|
| Clavier (OS-2.d) | IRQ contrôleur KBD2 (`$0350-$035F`, ADR-22) | ring buffer 16 keycodes en bank 1 `$5860` | `SYS_GET_KEY` (non-bloquant), `SYS_READ_CHAR` (bloquant) |
| Souris (SP-3.e) | IRQ contrôleur MOU2 (`$0360-$036F`, ADR-24) | état X/Y absolu + boutons + deltas | event loop GUI (focus/drag) |
| Audio AY (OS-4.a) | VIA T2 ou tick NMI | feed AY registers depuis buffer | non exposé v1 |
| GPU async (ADR-21 v2) | GPU IRQ done | flag bit + callback | (futur) |
| Timer ms | NMI tick scheduler | TCB blocked list | wake on counter |

**Classe 2 — Sync/blocking** : FAT32 (`kernel_fat_*`), console (`kernel_print_*`), GPU sync v1 (`kernel_gfx_*`), bank alloc (`kernel_alloc_bank`).

**Classe 3 — Polling idle** : aucun en v1, réservé futur.

**Mécanisme IRQ formalisé** :
```
IRQ matériel mode N → vecteur $00FFEE
  → trampoline bank 0 ($0140 : JML $01:5600)
  → kernel_irq_dispatch (bank 1 $5600)
       lit VIA_IFR
       cas T1 (timer)  → kernel_kbd_scan + kernel_sched_tick
       cas T2 (audio)  → kernel_audio_tick (futur)
       cas autre       → ignore + ack
       RTI
```
Table dispatch IRQ à **`$01:5680`** (8 entrées × 2B = 16B), indexée par bit IFR. Drivers s'enregistrent dynamiquement via `driver_init`.

**Convention ring buffer events** : slot fixe en bank 1 (~16 entrées × N bytes), head/tail 8-bit zero-page kernel, sentinelle pop=0 cohérente avec ADR-17.

Référence d'art : SymbOS (8-bit, IRQ-driven hybride).

Alternatives écartées :
- **(a) Tout IRQ-driven** : refactor FAT32/console/bank alloc en async, surcoût sans bénéfice. Rejette pattern existant.
- **(c) Struct ops formelle dès v1** : surcoût indirection asm 65C816 (jsr (vtable,X) au lieu de jsr label) sans modules dynamiques. Coût RAM ~10-20B par driver.

**v2 ouvertures parquées** : struct ops si modules dynamiques chargeables, driver discoverability runtime, hot-reload debug. Réouverture déclenchée par besoin réel module dynamique ou multi-OS host.

### ADR-17 — ABI kernel publique exposée à userland (ratifiée 2026-05-09)

OricOS expose 18 syscalls v1 stables aux apps userland (asm + C llvm-mos), via `cop #$AA` + table dispatch en bank 1.

**Mécanisme d'entrée** : `cop #imm` où `imm` = signature ABI version.
- v1 = `$AA` (déjà en place v0.1)
- v2 future = `$AB` (versioning par opcode immediate, statique, dispatch séparé, apps v1 préservées éternellement)

**Convention d'appel canonique (ABI v1)** :
- **Entrée** : `A` = numéro syscall (0..63), `X` et `Y` = args 8-bit
- **Sortie** : `A` = valeur retour 8-bit, `Y` = high byte si retour 16-bit
- **Préservés par le kernel** : `X` (sauf retour multi-byte), `D` (DPR), `DBR`, `PBR`, pile
- **Args > 2 bytes** : bloc d'args en zero-page kernel-réservée `$D0-$DF` (8 bytes)

**Convention d'erreur (sentinelle)** :
- `A = $FF` → erreur, code dans variable kernel `errno` (bank 1 `$5760`, exposée par `SYS_GETERRNO` ou lecture directe DBR=0).
- `A = $00..$FE` → succès (valeur ou status).
- Justification : compat immédiate llvm-mos C sans intrinsics ni inline asm. `if (sys_x() == 0xFF) handle_error();` trivial.

**Liste syscalls v1 (18 syscalls)** :

| # | Nom | Args | Retour |
|---|---|---|---|
| `$01` | SYS_PRINT_CHAR | X=char | — |
| `$02` | SYS_PRINT_STRING | X/Y=str_ptr (DBR-rel) | — |
| `$03` | SYS_READ_CHAR | bloquant | A=char |
| `$04` | SYS_EXIT | X=exit_code | n/a |
| `$05` | SYS_YIELD | — | — |
| `$06` | SYS_GET_KEY | non-bloquant | A=keycode/0 |
| `$07` | SYS_FAT_OPEN | X/Y=name_ptr | A=fd, $FF=err |
| `$08` | SYS_FAT_READ | bloc zp | A=nbytes, $FF=err |
| `$09` | SYS_FAT_CLOSE | X=fd | — |
| `$0A` | SYS_PANIC | X/Y=code | n/a |
| `$0B` | SYS_ALLOC_BANK | — | A=bank, $FF=err |
| `$0C` | SYS_FREE_BANK | X=bank | — |
| `$0D` | SYS_GFX_CLEAR | bloc zp | A=$00/$FF |
| `$0E` | SYS_GFX_FILL_RECT | bloc zp | A=$00/$FF |
| `$0F` | SYS_GFX_BLIT | bloc zp | A=$00/$FF |
| `$10` | SYS_GFX_LINE | bloc zp | A=$00/$FF |
| `$11` | SYS_GFX_TEXT | bloc zp | A=$00/$FF |
| `$12` | SYS_SLEEP_MS | X/Y=ms16 | — |

Slots `$00`, `$13-$3F` réservés extensions v1+. Slots `$40-$7F` réservés futur. Slots `$80-$FF` réservés signaux/contrôle système.

**Dispatch v0.2 (table)** :
- Table `syscall_table` à bank 1 `$5750`, 64 entrées × 2 octets = 128 B.
- Handler : `cmp #$40 ; bcs sys_invalid ; asl A ; tax ; jsr (syscall_table,X)`.
- v0.1 actuelle (cmp/bne hardcoded sur SYS_PRINT_CHAR) sera migrée en table dès Phase 1.

**Bundle versioning** : header `version=$01` (offset +4) déjà en place ADR-08. Apps v1 marquées version=1, ABI v2 future utilisera version=2.

Alternatives écartées :
- **(α) Carry flag** style GS/OS : élégant asm, mais llvm-mos C n'expose pas le carry. Bloque agilité Sprint 4 userland C.
- **(γ) Y=errno + A=value** : ABI calling convention non standard llvm-mos, intrinsics requis.
- **Liste minimale 10 syscalls** : couvre Sprint 4 strict mais re-design ABI à prévoir pour ajouter GFX/bank alloc.

**Impact** : ABI = contrat long terme. Toute app compilée avec `cop #$AA` doit fonctionner sur toute version v1.x du kernel. Cohérent avec ADR-13 (mécanisme syscall) et ADR-08 (bundle).

### ADR-18 — Sort du 6502 dans Phosphoric (ratifiée 2026-05-09)

Le cœur 6502 historique de Phosphoric est **retiré** post-validation, au profit du 65C816 mode E unique. Décision actée dans le cadre du programme « état de l'art » (DEC-1 close).

**Modalité retenue** : retrait net post-validation. Pas de flag compile-time `LEGACY_6502`. Pas de cohabitation perpétuelle.

**Plan d'exécution (Phase 1 du programme)** :

1. **Étape 1.A — Décrochage des types partagés** (~½ j) : extraire `memory_t`, `cpu_irq_source_t`, `cpu_flags_t`, `IRQF_*` de `cpu6502.h` vers `include/cpu/cpu_types.h` neutre. Tous les consommateurs (debugger, trace, profiler, emulator.h, tests) incluent `cpu_types.h`.
2. **Étape 1.B — Campagne de validation 65C816 mode E** (~1 j, **bloquante**) : bascule défaut `emu.cpu_kind = CPU_KIND_65C816`. Migration `test_klaus_dormann.c`, `test_oric_boot_dual.c`, `test_paravirt_demo.c` vers mode E. Audit shifts/rotations zp/abs M=0 (dette identifiée).
3. **Étape 1.C — Suppression effective** (~½ j, après go) : `src/cpu/cpu6502.c` (166), `src/cpu/opcodes.c` (572), `src/cpu/addressing.c` (113), `include/cpu/cpu6502.h`, `cpu_core_vtable_6502`, `CPU_KIND_6502`, `tests/unit/test_cpu.c` (1103, redondant avec test_cpu65c816_e_mode.c). Réécriture `test_cpu_core.c` (199) en `test_cpu816_core.c`.
4. **Étape 1.D — Traçage** (~½ j) : ADR-18 → §2 (ce paragraphe), `docs/adr/0018-retrait-6502.md` MADR, BACKLOG, CHANGELOG.

**Critère go/no-go (1.B → 1.C)** : **541 tests verts + bench ≤ 5 % + boot interactif ROM Oric 1 1.0 et 1.1**. Triple filet conforme contrainte CLAUDE.md §7. Si l'un échoue, on diagnostique, on ne supprime pas.

Justification : mode E déjà validé (Klaus Dormann passe, ROM Oric 1 boote). Cohabitation indéfinie ferait ~10 K LOC de surface de maintenance dupliquée pour gain de protection régressive marginal vu la couverture de tests existante. Vtable `cpu_core_vtable_t` permet le retrait chirurgical.

Alternatives écartées :
- **(b) Flag compile-time `LEGACY_6502`** : maintient les 2 cœurs disponibles via `-DLEGACY_6502`. Redoute un retour arrière qu'aucun signal technique ne justifie. Coût maintenance résiduel.
- **(c) Cohabitation perpétuelle** : dette de maintenance permanente, deux suites de tests à maintenir, aucun bénéfice après validation 1.B.

**Impact** : surface de maintenance Phosphoric divisée par ~2 (~10 K LOC supprimées). HDL ULX3S devra implémenter un seul cœur (65C816) avec mode E hybride pragmatique (ADR-11). DEC-1 actée.

---

## 3. Décisions ouvertes (ADR à instruire — NE PAS trancher unilatéralement)

Si une tâche force la main sur l'une de ces décisions, **arrête et demande**. Ne choisis pas par défaut.

Ces ADR ont été ouvertes le **2026-05-08** suite au point critique architecte
senior (cf. `BACKLOG.md` §annexe). Elles correspondent à des décisions
prises tacitement dans les premiers sprints OricOS et qu'il faut
expliciter avant d'avancer.

### ~~ADR-13~~ → ratifiée 2026-05-08, déplacée vers §2 (option a : COP + table)

### ~~ADR-14~~ → ratifiée 2026-05-08, déplacée vers §2 (table fixe 16 + bitmap free)

### ~~ADR-16~~ → ratifiée 2026-05-09, déplacée vers §2 (modèle hybride event-driven + sync, sans struct ops v1)

### ~~ADR-17~~ → ratifiée 2026-05-09, déplacée vers §2 (18 syscalls, COP `$AA`, sentinelle `A=$FF`, table dispatch)

### ~~ADR-18~~ → ratifiée 2026-05-09, déplacée vers §2 (retrait net 6502 post-validation, DEC-1 actée)

### ADR-15 — Isolation mémoire post-v1 (parquée v2 — 2026-05-09)

**Statut** : décision **parquée** au programme état-de-l'art Phase 0. Pas tranchée maintenant car aucun des 3 instruments d'instruction n'est disponible : pas de HDL ULX3S existant pour mesurer le budget BRAM/LUT, pas d'apps non-trusted en v1 (ADR-04 « OS de confiance »), pas de prototype des 3 alternatives.

**Question initiale** : à quoi ressemble la v2 d'ADR-04 ?
- (a) MMU custom HDL ECP5 (translation table par bank, BRAM).
- (b) MPU à segments avec privilege bits (kernel/user).
- (c) Banking matériel étendu avec tags d'accès.

**Critères de réouverture** : ADR-15 doit être réouverte dès qu'**au moins l'un** des 3 jalons suivants est atteint :

1. **Apps non-trusted ratifiées** (ouverture marketplace OricOS, code tiers exécuté, ou guest Oric 1 enrichi) — bascule modèle « OS de confiance » vers « OS multi-tenant ».
2. **HDL ULX3S à maturité HW-2** (port 65C816 ECP5 fonctionnel + contrat HW-1 figé) — budget BRAM/LUT restant connu, MMU custom vs MPU chiffrable.
3. **Date plancher 2026-12-31** — sécurité temporelle, force la réouverture même si jalons 1/2 pas atteints.

**Préparation préalable** : draft `docs/adr/0015-isolation-v2-DRAFT.md` à initier en Phase 4 du programme (Q3 2026), pré-instruisant les 3 options avec données budgétaires ECP5 LFE5U-85F (416 KB BRAM dont caches GPU/compositor déjà alloués), références projets retro avec MMU custom, coûts HDL typiques.

**Impact** : multitâche robuste, exécution apps non-trusted.

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

### Moratoire de ratification ADR (instauré 2026-05-09)

Aucune nouvelle ADR ne peut être ratifiée tant que **toutes** les conditions suivantes ne sont pas réunies :

1. **Dossier d'instruction écrit** : contexte chiffré, ≥ 2 alternatives chiffrées (coût/bénéfice), recommandation senior tracée. Pas de ratification verbale.
2. **Implémentation prête** : ≥ 50 % de l'implémentation de référence existe en code testé, OU jalon dur ≤ 4 semaines justifie la ratification anticipée.
3. **Cohérence ADR existantes** : pas de contradiction non-explicite avec les ADR §2 ratifiées. Toute révision rétroactive (vN→vN+1) suit la même règle.

**Liste blanche** :
- **Révisions mineures** d'une ADR ratifiée (clarifications, typos, mises à jour de chiffres factuels) : permises sans dossier complet, mention au CHANGELOG.
- **Parking explicite** d'une ADR ouverte vers v2/v3 avec critères de réouverture : permis (n'engage pas d'implémentation).
- **Révisions techniques bloquantes** suite à découverte d'incompatibilité tooling (e.g. ADR-05 v2 suite TC-llvmmos) : permises avec dossier raccourci documenté.

**Audit** : toute nouvelle ratification doit citer ce paragraphe et confirmer la conformité aux 3 conditions. Une ratification non-conforme est traitée comme bug d'architecture à corriger immédiatement.

Origine : critique architecte 2026-05-08 (cf. `BACKLOG.md` annexe) recommandait de geler les nouvelles ratifications jusqu'à 50 % d'implémentation. Le 2026-05-09, 3 ADR (19, 20, 21) ont été ratifiées en une journée, dont 2 révisées la même journée — pattern préoccupant. Le moratoire formalise le frein.

---

*Dernière révision : v0.2 — Phase 0 du programme état-de-l'art (2026-05-09) : ratification ADR-16 (driver model), ADR-17 (ABI syscall), ADR-18 (retrait 6502), parking ADR-15 v2, instauration du moratoire ADR.*
