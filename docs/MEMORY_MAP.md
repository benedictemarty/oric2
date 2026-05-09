# Oric 2 — Memory Map Specification

**Version** : 1.0 (B2 v0.1, ratifiée 2026-05-08)
**Statut** : spécification formelle de la memory map du système Oric 2.
**Auteur** : bmarty
**Référence** : DAT §4.1 (vue logique), ADR-04 (banking), ADR-10 (compat Oric 1).

> Ce document définit l'organisation de l'espace d'adressage 24 bits du
> 65C816 sur Oric 2. La v0.1 (B2) couvre les **256 KiB minimum** requis
> par la roadmap (banks 0-3). Les banks 4-255 sont réservés à des
> extensions futures et alloués paresseusement à la demande.

---

## 1. Vue d'ensemble

L'Oric 2 a un espace d'adressage 24 bits = 16 MiB max théorique. Le
système est organisé en **banks de 64 KiB** (convention WDC 65C816).

| Plage banks | Taille | Usage | Présence v0.1 |
|---|---|---|---|
| `$00xxxx` | 64 K | **Compat Oric 1** : RAM principale, I/O, ROM Oric 1 (guest) | ✅ Toujours |
| `$01xxxx` | 64 K | **Kernel OricOS + ROM système** | ✅ Si `--machine oric2` |
| `$02xxxx` | 64 K | **Code apps utilisateur (initial pool)** | ✅ Si `--machine oric2` |
| `$03xxxx` | 64 K | **Données apps utilisateur (initial pool)** | ✅ Si `--machine oric2` |
| `$04xxxx`-`$0Fxxxx` | 768 K | Pool kernel/apps additionnel | Lazy alloc |
| `$10xxxx`-`$7Fxxxx` | 7 MiB | Code et données apps utilisateur étendu | Lazy alloc |
| `$80xxxx`-`$BFxxxx` | 4 MiB | **Framebuffers OricOS, buffers fenêtres** | Lazy alloc |
| `$C0xxxx`-`$FFxxxx` | 4 MiB | Réservé / extensions futures | Lazy alloc |

**256 KiB minimum (B2 v0.1)** = banks 0, 1, 2, 3 alloués lors du boot
en mode oric2.

---

## 2. Bank 0 — Compatibilité Oric 1 stricte (ADR-10)

Bank 0 est **inviolable** : c'est l'espace d'adressage Oric 1 d'origine,
identique bit-à-bit à un Oric 1 réel. Il sert à exécuter le guest
Oric 1 en mode E (XCE bascule, paravirtualisation matérielle ADR-01).

```
Bank 0 ($000000-$00FFFF)
┌──────────────────────────────────────────────────┐
│ $0000-$00FF   Zero page (Oric 1 + guest)         │
│ $0100-$01FF   Stack (mode E forcé page 1)        │
│ $0200-$02FF   System variables Oric 1            │
│ $0300-$030F   VIA 6522 (16 registres mappés)     │
│ $0310-$031B   Microdisc WD1793 FDC (si présent)  │
│ $031C-$031F   ACIA 6551 sériel (si présent)      │
│ $0400-$BFFF   RAM utilisateur Oric 1 (48 KiB)    │
│               + RAM écran ($BB80-$BFDF, 1.1 KiB) │
│ $C000-$FFFF   ROM BASIC 1.0/1.1 (16 KiB)         │
│               OU upper_ram + overlay Microdisc   │
└──────────────────────────────────────────────────┘
```

**Règles** :
- Le 6502 Phosphoric et le 65C816 mode E voient cette même mémoire
  via `memory_read` / `memory_write` (16-bit address).
- Klaus B1.5 et boot dual B1.6 valident l'invariance.
- En mode Oric 2 (`--machine oric2`), bank 0 reste exactement identique.
  Aucun comportement Oric 1 ne change.

---

## 3. Bank 1 — Kernel OricOS

Bank 1 héberge le noyau OricOS et ses tables de descripteurs. Ce bank
est essentiel : il contient le code asm 65C816 du kernel (ADR-05),
les vecteurs natifs, le scheduler préemptif (ADR-03), et les structures
critiques.

```
Bank 1 ($010000-$01FFFF)
┌──────────────────────────────────────────────────┐
│ $0000-$00FF   Direct page kernel (D pointe ici)  │
│ $0100-$03FF   Stack kernel (3 × 256 octets)      │
│ $0400-$07FF   Tables descripteurs tâches         │
│ $0800-$0FFF   Buffers I/O kernel                 │
│ $1000-$1FFF   Driver registres (mappage virtuel) │
│ $2000-$EFFF   Code kernel + libs système         │
│ $F000-$FFFF   ROM système (vecteurs natifs +     │
│               handlers RES/NMI/IRQ/COP/BRK)      │
└──────────────────────────────────────────────────┘
```

**Vecteurs natifs en bank 1 (ROM)** :
- `$01FFE0-$01FFE5` : vecteurs natifs (NMI/RES/IRQ/COP/BRK/ABORT)
- À noter : le 65C816 *lit toujours les vecteurs depuis bank 0*. Les
  adresses `$00FFE_` (mode N) et `$00FFF_` (mode E) en bank 0 doivent
  donc **router** vers les handlers en bank 1. Ce routing est implémenté
  par convention : bank 0 contient des stubs courts (`JML $01FE..`) qui
  sautent en bank 1.

---

## 4. Bank 2 — Code apps utilisateur (initial pool)

Bank 2 est le premier bank de code utilisateur. Les apps (bundle léger
ADR-08) sont chargées par le kernel dans ce bank, ou dans des banks
supplémentaires si bank 2 est plein.

```
Bank 2 ($020000-$02FFFF)
┌──────────────────────────────────────────────────┐
│ $0000-$00FF   Direct page de l'app courante      │
│ $0100-$01FF   Stack utilisateur (page 1, mode E  │
│               compatible si l'app revient en E)  │
│ $0200-$FFFF   Code app + ressources statiques    │
└──────────────────────────────────────────────────┘
```

**Multi-app** : chaque app a son propre bank de code. Le kernel alloue
un bank libre via son bank allocator (ADR-04 sans MMU — alloc simple).

---

## 5. Bank 3 — Données apps utilisateur (initial pool)

Bank 3 héberge les données dynamiques (heap) des apps. Une app peut
référencer plusieurs banks de données via DBR (ADR-04).

```
Bank 3 ($030000-$03FFFF)
┌──────────────────────────────────────────────────┐
│ $0000-$FFFF   Heap apps utilisateur (par DBR)    │
└──────────────────────────────────────────────────┘
```

---

## 6. Banks 4-15 — Pool kernel/apps additionnel (lazy)

Banks `$04xxxx` à `$0Fxxxx` (768 KiB) sont alloués paresseusement
quand le kernel manque de bank pour une nouvelle app ou un buffer
système. Pas de structure imposée — c'est un pool fongible.

---

## 7. Banks 16-127 — Pool apps étendu (lazy)

Banks `$10xxxx` à `$7Fxxxx` (7 MiB) sont l'espace d'adressage utilisateur
étendu pour les grosses apps ou les structures de données massives
(ex: tampons audio long, framebuffers temporaires, caches I/O).

---

## 8. Banks 128-191 — RAM extra apps (ADR-19 v2)

Banks `$80xxxx` à `$BFxxxx` (64 banks × 64 KiB = **4 MiB**) sont
**RAM extra disponible aux apps** suite à la révision ADR-19 v2
(2026-05-09).

**Évolution v1 → v2** :
- v1 : banks 128-159 = VRAM live BRAM (accès pixel direct CPU).
- v2 : avec GPU autonome (ADR-21), CPU n'écrit plus de pixels
  directement. Banks 128-191 **libérés** pour usage RAM extra.

Cas d'usage typiques :
- **Apps gourmandes** : code/data dépassant 64 KiB (ex. compilateur,
  éditeur évolué). 1 app peut occuper plusieurs banks via paging.
- **Buffers utilisateur** : grands tampons documents, images, son.
- **Code paging** : apps qui chargent dynamiquement des modules.

Allocateur :
- `kernel_alloc_live_bank` / `kernel_free_live_bank` (Sprint VRAM-3)
  gèrent le pool banks 132-191 (28 banks) avec free list LIFO.
  Note : nom historique "live" conservé pour compat code, sémantique
  élargie en "RAM extra" en v2.
- Banks 128-131 réservés v1 (framebuffer SVGA en SDRAM, ADR-20 v2)
  → **disponibles pour apps custom v2** (à formaliser ADR si besoin).

---

## 8bis. VRAM en SDRAM (ADR-19 v2 + ADR-20 v3)

**Toute la VRAM** réside en SDRAM ULX3S (32 MiB physiques, v1 expose
16 MiB via I/O 24-bit). **Hors banking CPU**.

Localisation framebuffer principal **XVGA** (ADR-20 v3) :
- **SDRAM offset $000000-$05FFFF** : 384 KiB linéaires (1024×768×4bpp).
- 512 octets/ligne × 768 lignes = 393 216 octets contigus.
- Aucune contrainte de bank-cross (1 seul espace SDRAM).

Le **GPU Blitter HW** (ADR-21) lit/écrit cette zone directement à
pleine vitesse SDRAM. Le **compositor HDL** raster scan via line-buffer
cache BRAM (1 ligne lookahead).

Le **CPU** accède la VRAM uniquement via I/O ports `$0330-$033C`
(usage minoritaire : init, debug, fallback).

---

## 9. VRAM "cold" SDRAM via I/O (ADR-19)

La **VRAM "cold"** réside dans la **SDRAM ULX3S (32 MiB)** et est
accessible **uniquement via I/O ports MMIO** depuis le CPU
(non mappée dans le banking 24-bit).

Ports I/O (bank 0 / DBR=0) :

| Adresse | Registre | R/W | Description |
|---------|----------|-----|-------------|
| `$0330` | `VRAM_ADDR_LO`  | R/W | Adresse SDRAM bits [7:0] |
| `$0331` | `VRAM_ADDR_MID` | R/W | bits [15:8] |
| `$0332` | `VRAM_ADDR_HI`  | R/W | bits [23:16] (16 MiB adressable, top 16 MiB réservés) |
| `$0333` | `VRAM_DATA`     | R/W | byte courant, **auto-increment ADDR** après chaque accès |
| `$0334` | `VRAM_DMA_CTRL` | R/W | W bit 0 = trigger DMA, bit 1 = direction (0=SDRAM→bank, 1=bank→SDRAM); R bit 7 = busy |
| `$0335` | `VRAM_DMA_SRC_LO`  | R/W | DMA source adresse low |
| `$0336` | `VRAM_DMA_SRC_MID` | R/W | DMA source adresse mid |
| `$0337` | `VRAM_DMA_SRC_HI`  | R/W | DMA source adresse high (banking si bank→SDRAM) |
| `$0338` | `VRAM_DMA_DST_LO`  | R/W | DMA dest adresse low |
| `$0339` | `VRAM_DMA_DST_MID` | R/W | DMA dest adresse mid |
| `$033A` | `VRAM_DMA_DST_HI`  | R/W | DMA dest adresse high |
| `$033B` | `VRAM_DMA_LEN_LO`  | R/W | DMA longueur low |
| `$033C` | `VRAM_DMA_LEN_HI`  | R/W | DMA longueur high (max 64 KiB par burst) |

Usage typique :
- **Backing-stores** des fenêtres iconifiées ou occluded.
- **Sprites/fontes/icônes** précalculées chargées au boot.
- **Streams** (vidéo, animations) lus pendant le rendu.
- **Code apps en pagination** (apps > 64 KiB peuvent paginer leur code).

**DMA** est l'opération clé pour les transferts massifs :
- Drag fenêtre : DMA cold backing → live bank, ~µs au lieu de 38 000
  cycles CPU pour une fenêtre 320×240×4bpp.
- Restore from icon : DMA cold backing → live bank.
- Sauvegarder fenêtre avant focus change : DMA live bank → cold
  backing.

Le DMA **bloque le CPU** pendant le transfert (busy bit set). Pour
les blits asynchrones, le kernel polle `VRAM_DMA_CTRL` ou attend une
interruption (à définir).

---

## 10. Banks 160-255 — Réservé / extensions futures (lazy)

Banks `$A0xxxx` à `$FFxxxx` (96 banks × 64 KiB = 6 MiB) sont
**réservés pour extensions futures** et ne doivent pas être utilisés
par le code v1. Cas d'usage prévus :
- ROM cartouche (banks 160-191 : 2 MiB)
- DMA buffers spécialisés
- Memory-mapped peripherals étendus
- Banks privilégiés (sécurité, si ADR-04 v2 introduit MPU)

---

## 11. Implémentation Phosphoric (golden model)

### 11.1 Mode machine

`emulator_t.machine` = `ORIC_MACHINE_ORIC1` (défaut) ou
`ORIC_MACHINE_ORIC2`. Sélectionné par CLI `--machine oric1|oric2`.

- **Mode oric1** : seul bank 0 actif (compat Oric 1 strict, no-op vis-à-vis
  du B1.6 boot dual). `extra_banks[]` reste NULL.
- **Mode oric2** : banks 0-3 alloués au boot (RAM 192 KiB additionnelle).
  Banks 4-255 toujours lazy.

### 11.2 Allocation banks

- Bank 0 : `mem.ram` + `mem.rom` (existant, jamais alloué via `extra_banks`).
- Banks 1-3 (oric2) : alloués au reset via `memory_write24(BANK*0x10000, 0)`
  ou via une nouvelle fonction `memory_alloc_bank(mem, bank)`. Les banks
  sont initialisés à zéro (calloc).
- Banks 4-255 : lazy alloc à la première écriture (mécanisme B1.8 inchangé).

### 11.3 Vecteurs Oric 2

Mode oric2 attend que les vecteurs en bank 0 (`$00FFE_` et `$00FFF_`)
soient initialisés par OricOS. Si non initialisés (test sans OricOS),
COP/BRK/IRQ se comportent comme du Oric 1 (bank 0 ROM contient les
vecteurs Oric 1 originaux).

### 11.4 Gating B2 v0.1

- Aucun changement de comportement guest Oric 1 (bank 0).
- `--machine oric2` débloque l'allocation banks 1-3 mais ne charge
  aucun OricOS — l'utilisateur exécute du code 65C816 brut dans ces
  banks.
- Les sous-jalons OricOS (Track séparé) construiront les binaires kernel
  qu'on chargera en bank 1 et au-delà.

---

## 11. Évolution du document

| Version | Date | Changements | Auteur |
|---|---|---|---|
| 1.0 | 2026-05-08 | Spec initiale B2 v0.1 — banks 0-3 (256 KiB minimum). | bmarty |

Modifications futures attendues :
- B2 v1.1 : raffinement bank 1 (ROM système + RAM kernel exact split)
- B3 : ajout des stubs de routing vecteurs bank 0 → bank 1
- B4 : spécification compositor HW + accès banks 128-191
- v2 (post-OricOS v1) : apps multi-bank, packaging-time relocation
