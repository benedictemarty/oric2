# Contrat HDL ↔ Golden Model — Oric 2 / Phosphoric

> Document normatif. Définit l'interface comportementale et temporelle entre
> le **golden model** (Phosphoric, émulateur C) et l'**implémentation HDL**
> cible (ULX3S LFE5U-85F).
>
> Toute divergence entre golden model et HDL est un défaut à instruire :
> soit le HDL doit converger, soit le contrat est mis à jour formellement.
>
> Statut : **squelette v0.1** ratifié 2026-05-09 (HW-1 Phase 0). Contenu
> détaillé section par section produit en Phase 1 du programme état-de-l'art.

---

## Méta

| Champ | Valeur |
|---|---|
| Version contrat | 0.1 (squelette) |
| Plateforme HDL cible | ULX3S — Lattice ECP5 LFE5U-85F-6BG381 |
| Plateforme golden model | Phosphoric (C11, Linux/macOS/Windows) |
| Référence ADR | ADR-01 (CPU), ADR-02 (compositor double ULA), ADR-09 (audio), ADR-10 (compat Oric 1), ADR-11 (sémantique mode E), ADR-12 (HIRES Oric 2), ADR-19 v2 (VRAM SDRAM unifiée), ADR-20 v3 (XVGA), ADR-21 (GPU Blitter HW) |
| Critère d'évolution v0.1 → v1.0 | Toutes sections détaillées + jeux de stimulus partagés + premier module HDL HW-2 valide une section. |

---

## 0. Préambule

Le contrat distingue 3 catégories d'attributs par interface :

- **Comportement fonctionnel** : ce que l'interface fait (résultat observable).
  Le golden model et le HDL doivent être **fonctionnellement équivalents**.
- **Timing** : combien de cycles consomme l'opération. Le golden model
  *modélise* le timing du HDL ; l'écart toléré est borné par section.
- **Effets de bord observables** : flags, IRQ, registres status. Doivent
  matcher bit-à-bit.

Toute opération qui touche le bus mémoire ou les I/O Oric 1 ($0300-$031F)
**doit** rester strictement compatible Oric 1 (ADR-10) côté HDL et golden
model.

---

## 1. CPU 65C816

### 1.1 Modes
- **Mode E** (E=1) : compatible 6502 strict. Cycle-exact contre Klaus
  Dormann. Comportement hybride documenté ADR-11 (bug `JMP ($xxFF)`
  reproduit, opcodes illégaux NMOS = NOP, `$EB` alias SBC#).
- **Mode N** (E=0) : 65C816 natif, registres 16-bit selon M/X.
- **XCE** : bascule.

### 1.2 Bus et timing
*À détailler Phase 1 :*
- Phases d'horloge, signals VPA/VDA/MX/E.
- Cycles par opcode mode E et mode N (table de référence WDC).
- Tolérance golden model vs HDL : **0 cycle** en mode E (cycle-exact),
  **≤ 1 cycle** en mode N par opcode (à confirmer après bench).

### 1.3 IRQ / NMI / Reset / ABORT
- Niveaux : IRQ niveau actif bas, NMI front, RES, ABORT (mode N seul).
- Vecteurs E : `$00FFFA-FF` standard.
- Vecteurs N : `$00FFE4-FF` standard.

### 1.4 Banking
- PBR pour le code, DBR pour les données, D pour zero-page.
- Wrap mode E : pile et zero-page wrap dans bank 0 page 0/1.

---

## 2. Mémoire et banking

### 2.1 Espace d'adressage 24-bit
*À détailler Phase 1, références `docs/MEMORY_MAP.md` :*
- Layout banks 0-255.
- Bank 0 : RAM système + I/O ($0300-$03FF).
- Bank 1 : kernel OricOS.
- Banks 2-127 : RAM générale.
- Banks 128-255 : RAM extra apps (post-ADR-19 v2, anciennement VRAM live).

### 2.2 SDRAM controller
*À détailler Phase 1 :*
- Capacité : 16 MiB exposée via address 24-bit (32 MiB physique ULX3S).
- Latence d'accès burst typique (cycles).
- Throughput pic (MB/s).
- Interface : signals SDRAM (CKE, CS, RAS, CAS, WE, BA, A, DQ, DQM).

### 2.3 BRAM ECP5 caches internes
- Réservée caches GPU (line-buffer scanout, sprite cache, font cache,
  command queue) et caches compositor.
- **Invisible côté CPU** post-ADR-19 v2.

---

## 3. Interface VRAM (I/O ports $0330-$033C)

### 3.1 Registres
Table existante CLAUDE.md §2 ADR-19 v2.

### 3.2 Sémantique
*À détailler Phase 1 :*
- Auto-increment ADDR sur read/write `VRAM_DATA`.
- DMA RAM ↔ VRAM bidirectionnel.
- Bit `VRAM_DMA_CTRL.7` = busy : à respecter par CPU.
- Comportement bord : `LEN=0` → 64 KiB transfert (déjà testé Phosphoric).

### 3.3 Tolérances timing
- DMA throughput cible : à mesurer. Golden model simule synchrone instantané v0.1.
- Tolérance v1 : à fixer après prototype HDL controller SDRAM.

---

## 4. Interface GPU Blitter (I/O ports $0340-$034F, ADR-21)

### 4.1 Registres et opcodes
Table existante CLAUDE.md §2 ADR-21.

### 4.2 Sémantique des 5 commandes
*À détailler Phase 1 :*
- `CLEAR` (opcode `$01`)
- `FILL_RECT` (`$02`)
- `BLIT` (`$03`)
- `LINE` (`$04`, Bresenham)
- `TEXT` (`$05`)

Pour chaque : précondition args, comportement clipping, format pixel layout
(4bpp big-endian, 2 px/octet), gestion masque intra-octet.

### 4.3 Modèle d'exécution

**Golden model v0.1** : synchrone bloquant. CPU écrit registres, trigger,
poll busy, retour. Pas de queue, pas d'IRQ done.

**HDL cible v1** : asynchrone avec queue. CPU enqueue commande (max
profondeur N à définir), GPU exécute en parallèle, IRQ done sur fin
+ flag bit dans `GPU_STATUS`.

**Convergence requise** : golden model **doit** être étendu en
v0.2/v0.3 pour modéliser l'async (latence par commande, queue, IRQ).
Sinon écart structurel masqué jusqu'au port HDL.

### 4.4 Tolérances timing
*À détailler après prototype HDL :*
- Latence CLEAR XVGA (393 KiB writes) : cible ~3.8 ms à 100 MHz GPU.
- Latence FILL_RECT, BLIT, LINE, TEXT : à mesurer par taille et complexité.
- Tolérance golden model vs HDL : ± 20 % cycles, **0 différence fonctionnelle**.

---

## 5. ULA host + ULA guest (ADR-02)

### 5.1 ULA host
- Génère framebuffer principal OricOS (XVGA 1024×768×4bpp, ADR-20 v3).
- Lit SDRAM via line-buffer BRAM (1 BRAM 18Kb, scanline 512B).
- Pixel clock 65 MHz.

### 5.2 ULA guest
- Génère framebuffer Oric 1 virtualisé (240×200 attribute-based, compat
  stricte ADR-10).
- Lit RAM bank dédiée guest.

### 5.3 Compositor
*À détailler Phase 1 :*
- Mixe scanout host et guest selon position fenêtre guest.
- Mode mélange : remplacement (host pixel sauf zone guest).
- Position fenêtre guest configurable via I/O register (à spécifier).

### 5.4 Tolérances timing
- Scanout XVGA 1024×768@60Hz : 60 FPS strict. Tolérance : 0.
- Phase H/V sync : VESA standard.

---

## 6. VIA 6522 (compat Oric 1 stricte)

### 6.1 Périmètre
- Adresses bank 0 `$0300-$030F`.
- Comportement Oric 1 strict (ADR-10) : registres, timers T1/T2, IFR/IER,
  Port A/B, callbacks.
- Source IRQ : VIA T1 utilisée par OricOS scheduler tick + clavier scan.

### 6.2 Tolérance
- Cycle-exact mode E (compat ROM Oric 1).
- Mode N : tolérance ≤ 1 cycle (à confirmer).

---

## 7. SD SPI controller (ADR-07)

*À détailler Phase 1 :*
- Mode SPI : 0 ou 3 (à fixer).
- Fréquence init / runtime.
- Commandes supportées : CMD0, CMD8, CMD17, CMD18, ACMD41, etc.
- Throughput cible.
- Interface CPU : à définir (I/O port direct ou DMA).

---

## 8. HDMI tx (ADR-20 v3)

*À détailler Phase 1 :*
- Source : framebuffer XVGA 1024×768@60Hz (ULA host) + ULA guest mixée.
- Pixel clock 65 MHz, TMDS rate 650 MHz.
- Audio HDMI : à décider (out-of-scope v1 ou inclus).

---

## 9. Audio (ADR-09)

### 9.1 AY-3-8912 (compat Oric 1)
- Compat stricte ADR-10. Comportement, timing, sortie audio identiques.

### 9.2 Extension SID-like
- 3 voies + filtres + samples PCM 4-8 bits.
- Spec à formaliser Phase 1.

---

## 10. Comportement attendu du golden model

### 10.1 Rôle
- Référence comportementale et fonctionnelle.
- **Pas** référence cycle-exact pour HDL (sauf mode E pour CPU).
- Validateur de tests partagés HDL ↔ Phosphoric.

### 10.2 Exigences sur Phosphoric
1. Reproduire les comportements documentés ici, **bit-à-bit**.
2. Modéliser les latences en ordre de grandeur (queue GPU async, DMA SDRAM busy).
3. Exposer un mode `--hdl-strict` qui force les tolérances HDL exactes
   (à implémenter Phase 1, programme état-de-l'art).
4. Émettre des stimulus reproductibles (savestate étendu record/replay
   déterministe — Phase 3 task).

### 10.3 Tests de conformité partagés
*À produire Phase 1 + HW-2 :*
- Jeu de stimulus binaires `tests/conformance/*.bin`.
- Sortie attendue (registres, framebuffer, audio buffer, IRQ events).
- Run sur Phosphoric → produit golden output.
- Run sur HDL ECP5 (simulation Verilator + ULX3S réel) → comparaison.
- Critère pass : 100 % match comportemental, ± tolérances timing par section.

---

## 11. Évolution du contrat

### 11.1 Versioning
- v0.1 (actuel) : squelette ratifié, sections vides ou stub.
- v0.5 : sections détaillées Phase 1 + jeux stimulus initiaux.
- v1.0 : premier module HDL HW-2 (port CPU 65C816 ECP5) valide une section.

### 11.2 Procédure de modification
- Modification mineure (clarification, typo) : commit direct avec mention.
- Modification majeure (interface, timing) : doc d'instruction écrit,
  validation conjointe golden model + HDL avant merge.
- Toute découverte de divergence est tracée dans un registre
  `docs/CONTRACT_HDL_DIVERGENCES.md` (à créer dès la première occurrence).

---

## Annexe — Références

- WDC W65C816S Datasheet
- The Western Design Center 65C816 Programming Manual
- Lattice ECP5 LFE5U Datasheet
- ULX3S Schematics & Reference
- VESA DMT Standard (1024×768@60Hz)
- ADR-01 à ADR-21 (CLAUDE.md §2)
- `docs/MEMORY_MAP.md`
- `docs/DAT.md` (Architecture Description, IEEE 42010, en cours)
