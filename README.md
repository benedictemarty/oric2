# Oric 2

> Une **chimère 8/16 bits rétrocompatible Oric 1**, conçue de zéro pour FPGA ULX3S — avec son propre OS multitâche graphique et un guest Oric 1 virtualisé nativement.

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](Phosphoric/LICENSE)
[![Phosphoric](https://img.shields.io/badge/Phosphoric-1.22.16--alpha-green)](https://github.com/benedictemarty/oric2-golden-model)
[![OricOS](https://img.shields.io/badge/OricOS-v0.41--alpha-green)](https://github.com/benedictemarty/OricOS)
[![Tests](https://img.shields.io/badge/tests-530%2F530-brightgreen)](Phosphoric/VERSION_TRACKING)
[![Phase](https://img.shields.io/badge/programme-état--de--l'art%20Phase%201-orange)](BACKLOG.md)

---

## Vision

**Oric 2** est une machine *« chimère »* qui réinvente l'Oric 1 (1983) avec les libertés du présent : un CPU **WDC 65C816** au lieu du 6502, **256 banks** de mémoire au lieu de 64 KiB, un **GPU blitter HDL autonome** alimentant une sortie **HDMI 1024×768**, du **multitâche préemptif** style SymbOS — tout en restant **bit-à-bit compatible Oric 1** sur l'ancien logiciel via un mode émulation matériel.

Trois couches indissociables :

```
┌─────────────────────────────────────────────────────────┐
│            APPS userland C (llvm-mos) + asm            │
├─────────────────────────────────────────────────────────┤
│   OricOS — multitâche préemptif, GUI multifenêtré       │
│   FAT32 SD, audio AY+SID, syscalls, GPU helpers          │
├──────────────────────────┬──────────────────────────────┤
│  ULA host (XVGA)         │  ULA guest (Oric 1, 240×200) │
│  Compositor matériel     │  Mode émulation 65C816 E=1   │
├──────────────────────────┴──────────────────────────────┤
│  WDC 65C816  ·  GPU Blitter  ·  SDRAM 32 MiB  ·  HDMI   │
│  Cible HDL : Lattice ECP5 LFE5U-85F  ·  ULX3S            │
└─────────────────────────────────────────────────────────┘
```

Le projet livre **simultanément** un **émulateur logiciel cycle-exact** (le *golden model*) et la **spécification HDL** correspondante. L'émulateur sert de référence comportementale pour le port FPGA.

---

## Sous-projets

| Repo | Rôle | Statut | Lien |
|---|---|---|---|
| **Phosphoric** | Émulateur cycle-exact C11. Couvre Oric 1 historique **et** Oric 2 (golden model). | v1.22.9-alpha · 541 tests · 0 fuite mémoire | [oric2-golden-model](https://github.com/benedictemarty/oric2-golden-model) |
| **OricOS** | Système d'exploitation natif Oric 2, kernel asm 65C816 + apps. | v0.40-alpha · Sprint 3.c | [OricOS](https://github.com/benedictemarty/OricOS) |
| **oric2** (ce repo) | Hub de spécification : ADR, BACKLOG, contrat HDL, memory map, document d'architecture. | Phase 0 close · Phase 1 active | vous êtes ici |

---

## Architecture en bref

| Aspect | Choix | ADR |
|---|---|---|
| CPU | WDC 65C816, mode E (compat 6502 strict) + mode N (16 bits, banking) | [ADR-01](CLAUDE.md#adr-01--cpu--65c816), [ADR-11](CLAUDE.md#adr-11--sémantique-du-mode-e-vis-à-vis-du-nmos-6502-ratifiée-2026-05-07) |
| Vidéo | Compositor matériel double ULA — ULA host XVGA 1024×768×4bpp + ULA guest 240×200 attribute-based | [ADR-02](CLAUDE.md#adr-02--compositor-matériel-double-ula), [ADR-12](CLAUDE.md#adr-12), [ADR-20](CLAUDE.md#adr-20) |
| GPU | Blitter HW autonome, 5 commandes (CLEAR/FILL_RECT/BLIT/LINE/TEXT) via I/O `$0340-$034F` | [ADR-21](CLAUDE.md#adr-21) |
| OS | Multitâche **préemptif strict** (réf SymbOS), 16 TCB, COP+table syscalls | [ADR-03](CLAUDE.md#adr-03), [ADR-13](CLAUDE.md#adr-13), [ADR-14](CLAUDE.md#adr-14) |
| ABI userland | 18 syscalls v1, `cop #$AA`, sentinelle `A=$FF`, llvm-mos C 8-bit native | [ADR-17](docs/adr/0017-abi-syscall-userland.md), [ADR-05](CLAUDE.md#adr-05) |
| Drivers | Hybride event-driven (clavier/audio) + sync (FAT32/console/GPU sync v1) | [ADR-16](docs/adr/0016-driver-model.md) |
| Mémoire | Banking 24 bits, 16 MiB SDRAM unifiée, BRAM ECP5 = caches GPU | [ADR-04](CLAUDE.md#adr-04), [ADR-19](CLAUDE.md#adr-19) |
| Stockage | FAT32 sur carte SD (SPI) | [ADR-07](CLAUDE.md#adr-07) |
| Audio | AY-3-8912 (compat Oric 1) + extension SID-like | [ADR-09](CLAUDE.md#adr-09) |
| Compat | Mode émulation **bit-à-bit** Oric 1 — ROM originale boote sans modification | [ADR-10](CLAUDE.md#adr-10) |
| Cible HDL | Lattice ECP5 LFE5U-85F sur carte ULX3S | [docs/CONTRACT_HDL.md](docs/CONTRACT_HDL.md) |

---

## Démarrage rapide

```bash
# 1. Cloner les 3 repos
git clone --recurse-submodules https://github.com/benedictemarty/oric2.git
cd oric2
git clone https://github.com/benedictemarty/oric2-golden-model.git Phosphoric
git clone https://github.com/benedictemarty/OricOS.git

# 2. Build de l'émulateur
cd Phosphoric
sudo apt-get install build-essential libsdl2-dev   # Debian/Ubuntu
make SDL2=1

# 3. Lancer une ROM Oric 1 historique
./oric1-emu -r roms/basic10.rom         # ORIC-1 BASIC 1.0
./oric1-emu -r roms/basic11b.rom        # Atmos BASIC 1.1

# 4. Suite de tests (doit afficher 541/541)
make tests
```

OricOS se construit avec **cc65** (ca65 + ld65 ≥ 2.19) :

```bash
cd OricOS
make             # produit build/kernel.bin
```

---

## Documentation

| Document | Contenu |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | Instructions tactiques + ADR ratifiées (§2) + ADR ouvertes (§3) + moratoire (§10). **Source de vérité.** |
| [`BACKLOG.md`](BACKLOG.md) | Sprints NOW/NEXT/LATER + risques + dette + décisions stratégiques. |
| [`CHANGELOG.md`](CHANGELOG.md) | Journal commun aux 3 sous-projets (format Keep a Changelog). |
| [`docs/adr/`](docs/adr/) | ADR au format MADR (un fichier par décision, migration progressive). |
| [`docs/CONTRACT_HDL.md`](docs/CONTRACT_HDL.md) | **Contrat HDL ↔ golden model** — interface comportementale et temporelle entre Phosphoric et l'implémentation FPGA. |
| [`docs/MEMORY_MAP.md`](docs/MEMORY_MAP.md) | Layout 24 bits exact des banks. |
| [`docs/DAT.md`](docs/DAT.md) | Document d'Architecture (IEEE 42010, en cours). |
| [`docs/TC-llvmmos.md`](docs/TC-llvmmos.md) | Investigation toolchain C 65C816. |

---

## Statut

**Phase 0 du programme « état de l'art » close** (2026-05-09) — 4 ADR (15/16/17/18) tranchées, 1 parquée v2 avec critères de réouverture, **moratoire ADR formalisé** (CLAUDE.md §10), squelette du contrat HDL livré.

**Phase 1 active** : retrait du cœur 6502 historique (ADR-18), table dispatch syscall (OS-2.f.v2), driver clavier IRQ-driven (OS-2.d). Suivez le journal en temps réel sur [`CHANGELOG.md`](CHANGELOG.md) et [`BACKLOG.md`](BACKLOG.md).

Le programme complet (8 semaines, 4 axes parallèles) couvre : assainissement, CI multi-OS + sanitizers + fuzzing, libc + GDB stub + audio + window manager, migration ADR/MADR + DAT + site mkdocs.

---

## Statuts visuels

| Phase 0 | Phase 1 | Phase 2 | Phase 3 | Phase 4 | Phase 5 |
|:---:|:---:|:---:|:---:|:---:|:---:|
| ✅ close | 🟡 active | ⏳ | ⏳ | ⏳ | ⏳ |
| Décisions cadre | Assainissement | Toolchain & CI | Modernisation produit | Process & doc | Bilan |

---

## Contribuer

Le projet suit une logique agile rigoureuse :

1. **Lire** [`CLAUDE.md`](CLAUDE.md) avant toute modification — c'est la source de vérité tactique.
2. **Petits commits atomiques** : `[Workspace|Phosphoric|OricOS] <scope> <description>`.
3. **Tests en vert** systématiquement : `make tests` côté Phosphoric, build kernel côté OricOS.
4. **Référencer les ADR** en commentaire si la décision n'est pas évidente.
5. **Pas de gros refactor sans demande explicite.**
6. **Pas de nouvelle ADR sans dossier d'instruction écrit** — moratoire (CLAUDE.md §10).

Voir [`Phosphoric/CONTRIBUTING.md`](https://github.com/benedictemarty/oric2-golden-model/blob/main/CONTRIBUTING.md).

---

## Glossaire express

- **ULA** — *Uncommitted Logic Array*. Génère le timing vidéo + lecture VRAM + couleur attribute-based.
- **Mode E / Mode N** — État du 65C816 : émulation 6502 strict (E=1) versus natif 16 bits (E=0).
- **XCE** — *Exchange Carry/Emulation*, l'instruction qui bascule entre les deux.
- **Bank** — Page de 64 KiB dans l'espace 24 bits du 65C816.
- **Compositor** — Logique (SW dans Phosphoric, HW en HDL) qui mixe les framebuffers host et guest à la sortie vidéo.
- **OricOS** — Système d'exploitation natif d'Oric 2.
- **Guest** — Instance Oric 1 virtualisée dans une fenêtre OricOS.
- **Golden model** — Phosphoric en tant que référence comportementale pour le futur HDL.
- **Paravirtualisation** — Stratégie où le guest s'exécute nativement sur le CPU (mode E ici), avec coopération minimale du kernel.

Glossaire complet : [`CLAUDE.md` §8](CLAUDE.md#8-glossaire).

---

## Licence

- **Phosphoric** : MIT © 2026 chipinette ([LICENSE](Phosphoric/LICENSE)).
- **OricOS** et **workspace `oric2`** : licence à finaliser. Probablement MIT ou Apache 2.0 — décision business courante.

---

## Références techniques

- [WDC W65C816S Datasheet](https://www.westerndesigncenter.com/wdc/documentation/w65c816s.pdf)
- [The Western Design Center 65C816 Programming Manual](http://www.6502.org/tutorials/65c816opcodes.html)
- [Klaus Dormann's 6502 functional test](https://github.com/Klaus2m5/6502_65C02_functional_tests)
- [ULX3S documentation](https://github.com/emard/ulx3s)
- [Lattice ECP5 LFE5U Datasheet](https://www.latticesemi.com/Products/FPGAandCPLD/ECP5)
- [SymbOS](https://www.symbos.de/) — référence multitâche préemptif fenêtré sur 8 bits.
- [Apple IIgs GS/OS](https://en.wikipedia.org/wiki/GS/OS) — référence OS sur 65C816 avec mode legacy.

---

*Projet maintenu par [@benedictemarty](https://github.com/benedictemarty) · Inspiré par la machine Oric 1 (Tangerine Computer Systems, 1983).*
