# DAT — Document d'Architecture du système Oric 2

**Format** : ISO/IEC/IEEE 42010:2011 (Architecture description)
**Version** : 1.0 (ratifiée)
**Date** : 2026-05-07
**Auteur** : bmarty
**Statut** : Architecture description ratifiée pour la v1 du système Oric 2.

> Ce document est la description d'architecture autorisée du système Oric 2.
> Toute modification d'une décision (ADR) ratifiée doit être traçable et discutée
> explicitement, conformément au CLAUDE.md racine §10. Le présent document est
> le compagnon stratégique du CLAUDE.md (instructions tactiques pour l'agent
> Claude Code) — l'un n'invalide jamais l'autre.

---

## 0. Métadonnées

| Champ | Valeur |
|---|---|
| Identification du système | Oric 2 (machine chimère rétrocompatible Oric 1) |
| Version du document | 1.0 |
| Cycle de vie | Architecture description initiale, post-ratification ADR |
| Standard de référence | ISO/IEC/IEEE 42010:2011 |
| Source de vérité tactique | `/home/bmarty/oric2/CLAUDE.md` |
| Sous-projets | Phosphoric (golden model), OricOS (système d'exploitation), HDL ULX3S (cible matérielle) |
| Cible matérielle finale | FPGA ULX3S (ECP5, BRAM, SDRAM, HDMI tx, SD slot) |
| Prochain jalon ouvert | B1.7 (mode natif 65C816) |

---

## 1. Contexte et objectifs

### 1.1 Mission

Concevoir et réaliser **Oric 2**, une machine 8/16-bit moderne en trois couches indissociables :

1. **Hardware Oric 2** — CPU 65C816, ULA étendue, **compositor matériel double ULA**, banking mémoire 24-bit, sortie HDMI.
2. **OricOS** — système d'exploitation multitâche graphique, natif Oric 2.
3. **Guest Oric 1** — machine Oric 1 historique virtualisée dans une fenêtre d'OricOS, exécutée nativement par le 65C816 en mode émulation (paravirtualisation matérielle via XCE).

L'objectif final est l'implémentation HDL sur **ULX3S**. Phosphoric, l'émulateur Oric 1 multiplateforme en C, devient le **modèle exécutable de référence** (golden model) du HDL.

### 1.2 Périmètre

**Inclus dans ce DAT** :
- L'architecture des trois couches (HW + OS + guest).
- Les contraintes de compatibilité ascendante avec l'Oric 1 d'origine (ADR-10).
- La stratégie d'implémentation Phosphoric → HDL ULX3S.
- Les ADR ratifiées au 2026-05-07.

**Hors périmètre du DAT v1.0** :
- Spécifications de bas niveau (registres, bit-level memory map, signaux HDL) — du ressort des documents techniques de chaque sous-projet.
- Plan de mise en œuvre détaillé — couvert par la roadmap (cf. `Phosphoric/ROADMAP`).
- Choix de licence — décision business à prendre séparément.
- API publique d'OricOS — sujet d'un Software Development Kit (SDK) document à venir.

### 1.3 Parties prenantes (Stakeholders)

| Stakeholder | Rôle | Concerns principaux |
|---|---|---|
| **Utilisateur Oric 1 historique** | Joue à des jeux Oric 1 / Atmos sur Oric 2 | Compat stricte (ADR-10), aucune régression |
| **Utilisateur OricOS** | Utilise Oric 2 comme machine moderne | Réactivité, stabilité, GUI ergonomique |
| **Développeur d'apps OricOS** | Écrit du code natif pour Oric 2 | SDK, doc, toolchain disponible |
| **Développeur du noyau OricOS** | Maintient kernel/drivers | Architecture claire, tests reproductibles |
| **Intégrateur HDL ULX3S** | Porte vers FPGA | Modèle de référence cycle-exact, séparation des préoccupations |
| **Mainteneur Phosphoric** | Maintient l'émulateur PC | Multiplateforme préservé, dépendances minimales |
| **Communauté Oric historique** | Logiciels et docs Oric 1 existants | Préservation patrimoniale, interop disquettes |
| **Project owner (bmarty)** | Décisions architecturales | Cohérence, traçabilité, agilité |

### 1.4 Concerns (préoccupations transverses)

| ID | Concern | Stakeholders concernés | Adressé par |
|---|---|---|---|
| C1 | Compatibilité Oric 1 stricte | Utilisateur historique, communauté | ADR-01, ADR-10, ADR-11 |
| C2 | Cycle accuracy en mode émulation | Communauté, intégrateur HDL | ADR-10, B1.5, B1.6 |
| C3 | Performance OricOS sur 65C816 | Utilisateur OricOS, dév kernel | ADR-01, ADR-03, ADR-05 |
| C4 | Latence I/O et audio | Utilisateur OricOS, dév kernel | ADR-03 (préemptif), ADR-09 |
| C5 | Multiplateforme Phosphoric | Mainteneur Phosphoric | Contrainte dure §5 CLAUDE.md |
| C6 | Coût et complexité HDL ULX3S | Intégrateur HDL | ADR-04 (pas de MMU), ADR-02 |
| C7 | Maintenabilité du code OS | Dév kernel et apps | ADR-05, ADR-08 |
| C8 | Ergonomie GUI | Utilisateur OricOS | ADR-06, ADR-02 |
| C9 | Persistance et interop fichiers | Utilisateur OricOS, dév | ADR-07 |
| C10 | Qualité et expressivité audio | Utilisateur OricOS | ADR-09 |
| C11 | Traçabilité des décisions | Project owner, dév | DAT, CLAUDE.md, CHANGELOGs |
| C12 | Compatibilité visuelle pixel-exact | Utilisateur historique | ADR-02 (ULA guest dédiée) |

---

## 2. Stakeholders × Concerns — matrice de couverture

| Concern | C1 | C2 | C3 | C4 | C5 | C6 | C7 | C8 | C9 | C10 | C11 | C12 |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| Utilisateur Oric 1 | ●●● | ●●● | | | | | | | | | | ●●● |
| Utilisateur OricOS | | | ●●● | ●●● | | | ● | ●●● | ●●● | ●●● | | |
| Dév apps | | | ● | ● | | | ●●● | ● | ●● | | ● | |
| Dév kernel | | ● | ●●● | ●●● | | | ●●● | | ● | ●● | ●● | |
| Intégrateur HDL | ●● | ●●● | | | | ●●● | ● | | | ●● | | ●●● |
| Mainteneur Phosphoric | ●● | ●● | | | ●●● | | ●● | | ● | | ●● | |
| Communauté | ●●● | ●●● | | | ● | | | | ● | | ● | ●●● |
| Project owner | ●● | ●● | ●● | ● | ● | ● | ●● | ● | ● | ● | ●●● | ● |

(●●● très important / ●● important / ● secondaire / vide non concerné)

---

## 3. Viewpoints

Cinq viewpoints sont définis pour le système Oric 2.

### 3.1 Viewpoint logique

- **Audience** : tous stakeholders.
- **Concerns adressés** : C1, C3, C7, C8.
- **Model kinds** : diagrammes de couches, diagrammes de composants, table d'allocation mémoire conceptuelle.
- **Conventions** : couches verticales (top = utilisateur, bottom = HW), composants nommés à la responsabilité.
- **Méthodes d'analyse** : revue d'API, analyse de couplage entre couches.

### 3.2 Viewpoint comportemental

- **Audience** : dév kernel, intégrateur HDL, communauté.
- **Concerns adressés** : C1, C2, C4.
- **Model kinds** : machines à états (modes E/N), diagrammes de séquence (XCE, virtualisation guest, IRQ).
- **Conventions** : transitions étiquetées par évènement causal, états avec invariants.

### 3.3 Viewpoint physique

- **Audience** : intégrateur HDL, mainteneur Phosphoric.
- **Concerns adressés** : C5, C6, C9, C10, C12.
- **Model kinds** : block diagrams hardware, memory map, sound chip topology, I/O routing.
- **Conventions** : entités matérielles (ECP5 BRAM, SDRAM, SD, HDMI tx, GPIO), interconnexions explicites.

### 3.4 Viewpoint compatibilité

- **Audience** : utilisateur historique, communauté, intégrateur HDL.
- **Concerns adressés** : C1, C2, C12.
- **Model kinds** : matrices opcodes 6502 ↔ 65C816 mode E, table des bugs reproduits, table des cas d'écart documentés.
- **Conventions** : statut « identique / divergent justifié / divergent à corriger » par opcode et timing.

### 3.5 Viewpoint implémentation

- **Audience** : intégrateur HDL, mainteneur Phosphoric, project owner.
- **Concerns adressés** : C2, C5, C6, C7, C11.
- **Model kinds** : pipeline Phosphoric → HDL, organisation des sous-projets, traçabilité ADR ↔ code ↔ tests.
- **Conventions** : chaque décision d'implémentation référence une ADR.

---

## 4. Views

### 4.1 Vue logique

**Couches du système Oric 2** (top-down) :

```
┌──────────────────────────────────────────────────────────────────────┐
│  Apps OricOS natives                  ┌───────────────────────────┐  │
│  (bundle léger, ADR-08)               │  Guest Oric 1 (fenêtre)   │  │
│  Toolchain : llvm-mos (ADR-05)        │  ROM Oric 1 BASIC 1.0/1.1 │  │
├──────────────────────────────────────┤  exécutée native en mode E │  │
│  Bibliothèques système OricOS        │  (XCE bascule, ADR-01,    │  │
│  (libc, GUI, audio, FS — C/asm)      │   compat ADR-10)          │  │
├──────────────────────────────────────┤                           │  │
│  Kernel OricOS                        │                           │  │
│  (asm 65C816, ADR-05)                 │                           │  │
│   - Scheduler préemptif (ADR-03)      │                           │  │
│   - VFS (FAT32, hostfs émul., ADR-07) │                           │  │
│   - Window manager SymbOS-like (ADR-06)│                          │  │
│   - Audio mixer (AY+SID, ADR-09)      │                           │  │
│   - Bank allocator (ADR-04)           │                           │  │
└───────────────────────────────────────┴───────────────────────────┘  │
                              ▲                                         │
┌─────────────────────────────┴─────────────────────────────────────┐  │
│  Hardware Oric 2                                                  │  │
│   - CPU 65C816 (ADR-01) — modes E (compat 6502 strict ADR-11(c))  │  │
│                            et N (16-bit, banking 24-bit)          │  │
│   - Compositor matériel + ULA host + ULA guest (ADR-02)           │  │
│   - VIA 6522 (compat Oric 1) + extensions Oric 2                  │  │
│   - AY-3-8912 + extension SID-like (ADR-09)                       │  │
│   - SD slot SPI / FAT32 (ADR-07)                                  │  │
│   - HDMI tx (sortie compositée)                                   │  │
└───────────────────────────────────────────────────────────────────┘
```

**Composants logiciels OricOS** :

- **Kernel** : tâches, ordonnancement, IPC minimal (signaux, files de messages), gestion banks.
- **Drivers** : ULA host, ULA guest, VIA, AY/SID, microdisc (legacy), SD/FAT32, clavier, joystick, série, imprimante.
- **Window manager** : gestion fenêtres, focus, événements, compositor logique au-dessus du compositor HW.
- **VFS** : abstraction FAT32 / hostfs / Sedoric overlay (futur).
- **Toolkit GUI** : widgets, dialogues, layout. Style SymbOS-like (ADR-06).
- **Mixer audio** : mélange canaux AY et SID, priorité par stream, latence courte (ADR-04 préemptif → audio servi sur tick).

**Allocation mémoire conceptuelle** (espace 24-bit, 16 MiB max théorique sur 65C816) :

| Plage | Usage |
|---|---|
| Bank 00 ($00xxxx) | Compat Oric 1 strict — RAM principale, I/O, ROM Oric 1 lorsqu'utilisée par le guest |
| Bank 01-0F | Kernel OricOS + ROM système |
| Bank 10-7F | Code et données apps utilisateur |
| Bank 80-BF | Buffers de fenêtres OricOS, framebuffer host |
| Bank C0-FF | Réservé / extensions futures |

(allocation ratifiable plus finement par un document « Memory Map Spec » ultérieur ; elle reste indicative ici.)

### 4.2 Vue comportementale

#### 4.2.1 Modes du CPU et bascule

```
        ┌──────────────────────────────────────┐
        │           Mode N (natif, E=0)        │
        │  - Registres 16-bit (M=0, X=0 OK)    │
        │  - Banking 24-bit utilisable         │
        │  - Vecteurs natifs ($FFE_)           │
        │  - OricOS s'y exécute                │
        └──────────────────────────────────────┘
                  ▲                 │
        XCE       │                 │     XCE
        (avec C=0)│                 │     (avec C=1)
                  │                 ▼
        ┌──────────────────────────────────────┐
        │           Mode E (émulation, E=1)    │
        │  - M=X=1 forcés                      │
        │  - S verrouillé en page $01          │
        │  - X.high = Y.high = 0               │
        │  - Vecteurs émulation ($FFFA/C/E)    │
        │  - Comportement 6502 strict (ADR-10)  │
        │  - Bug JMP indirect reproduit (ADR-11)│
        │  - Opcodes illégaux NMOS = NOP        │
        │  - Le guest Oric 1 s'y exécute        │
        └──────────────────────────────────────┘
                  ▲
                  │
                  │ RES (signal reset)
                  │  → E=1, P[I]=1, PC = vec($00FFFC)
```

#### 4.2.2 Paravirtualisation guest Oric 1

OricOS lance le guest Oric 1 en :
1. Allouant les banks 00 (RAM) et la ROM Oric 1 mappée.
2. Configurant l'**ULA guest matérielle** sur la fenêtre OricOS dédiée.
3. Préparant les vecteurs émulation pour intercepter NMI/IRQ vers le kernel host.
4. Exécutant **XCE** pour passer en mode E.
5. Sautant à l'entry point Oric 1 ($F88B pour BASIC 1.0).

Le guest s'exécute **nativement** (pas d'émulation logicielle dans OricOS). Sa sortie vidéo est générée par l'ULA guest matérielle (240×200, attribute-based), composée par le compositor matériel dans la fenêtre que le window manager OricOS lui a allouée.

#### 4.2.3 Cycle scheduler OricOS (ADR-03 préemptif)

```
Timer tick (50 Hz typique)
        ↓
   IRQ vector ($FFEE en mode N)
        ↓
   Save context (A/X/Y/S/D/DBR/PBR/PC/P) sur stack tâche courante
        ↓
   Update tâche statistiques + timeslice
        ↓
   Sélectionne tâche prête de plus haute priorité (round-robin niveau)
        ↓
   Restore context tâche élue
        ↓
   RTI → reprise de la tâche élue
```

Le scheduler ne suspend **jamais** le guest Oric 1 si celui-ci est tâche élue (le guest est mode E, le scheduler est mode N — la transition est faite par XCE en sortie de scheduler).

### 4.3 Vue physique (ULX3S)

```
                    ┌──────────────────────────────────────┐
                    │           ULX3S (ECP5 LFE5U-85F)     │
                    │                                      │
   ┌──────────┐     │   ┌──────────────────────────────┐  │
   │  HDMI tx │ ◀───┼───┤  Compositor + ULA host + guest│  │
   │  (Pmod)  │     │   │  (ADR-02)                     │  │
   └──────────┘     │   └──────┬────────────────┬───────┘  │
                    │          │                │          │
                    │   ┌──────▼──────┐  ┌──────▼──────┐   │
                    │   │  CPU core   │  │ Sound mixer │   │
                    │   │  65C816     │◀─│ AY+SID      │   │
                    │   │  (ADR-01)   │  │ (ADR-09)    │   │
                    │   └──────┬──────┘  └──────┬──────┘   │
                    │          │                │          │
                    │   ┌──────▼─────────┐  ┌───▼─────┐    │
                    │   │  VIA + I/O bus │  │ Audio DAC│   │
                    │   └──────┬─────────┘  │ (Pmod)   │   │
                    │          │            └─────────┘    │
                    │   ┌──────▼─────────┐                 │
                    │   │  Memory ctrl   │                 │
                    │   │  (BRAM + SDRAM)│                 │
                    │   └──┬─────────┬───┘                 │
                    │      │         │                     │
                    │  ┌───▼───┐ ┌───▼────┐  ┌─────────┐  │
                    │  │ BRAM  │ │ SDRAM  │  │ SD slot │  │
                    │  │ ~512K │ │ 32 MB  │◀─│ (SPI)   │  │
                    │  └───────┘ └────────┘  │ ADR-07  │  │
                    │                        └─────────┘  │
                    └──────────────────────────────────────┘
                                                  ▲
                              Clavier USB ─────────┘ (via FT232 + soft IO ?)
```

**Allocation mémoire physique** (indicative, à raffiner) :
- BRAM ECP5 ~512 KiB : framebuffers double, ZP rapide, registres I/O, regs ULA.
- SDRAM 32 MiB : RAM système OricOS + apps, pool de banks 65C816.
- SD card : storage FAT32 (ADR-07), illimité de fait.

**Cibles de timing** :
- Cœur 65C816 : ~14 MHz pour suivre HDMI 720×480p tile-based ; perf concrète à mesurer en synthèse.
- ULA guest émule le timing PAL Oric 1 (1 MHz pixel clock effectif depuis le guest) sans charger le CPU host.

### 4.4 Vue compatibilité

#### 4.4.1 Matrice de compatibilité 65C816 mode E ↔ NMOS 6502

L'option ADR-11(c) **hybride pragmatique** définit le contrat :

| Catégorie | Mode E Oric 2 | Reference |
|---|---|---|
| 151 opcodes officiels 6502 | Identiques (sémantique + timing) | Validé par Klaus B1.5 |
| Bug `JMP ($xxFF)` page wrap | Reproduit (lit high byte à `(ptr & 0xFF00) | 0`) | ADR-11(c) |
| Opcodes illégaux NMOS (LAX/SAX/DCP/...) | NOP avec consommation taille opérande | ADR-11(c) |
| Alias officieux `$EB` (SBC imm) | Conservé (commun NMOS/WDC) | ADR-11(c) |
| Séquence RMW interne | Comportement WDC (read-read-modify-write) | Non observable software |
| Décimal post-RESET | D=0 | Datasheet WDC §6 |
| Vecteurs en mode E | $FFFA (NMI) / $FFFC (RES) / $FFFE (IRQ/BRK) | Identique 6502 |
| Bit B dans P (registre interne) | Toujours 0, n'apparaît qu'à la valeur pushed | Aligné 6502 Phosphoric |

#### 4.4.2 Logiciels Oric 1 ciblés

| Catégorie | Statut compat |
|---|---|
| ROM BASIC 1.0 (Oric 1) | Boot validé Phosphoric, équivalence 65C816 mode E B1.6 |
| ROM BASIC 1.1 (Atmos) | Détection auto (Phosphoric), à revalider sous 65C816 |
| Programmes BASIC sur cassette TAP | Phosphoric OK (CLOAD), même comportement attendu sous 65C816 mode E |
| Sedoric (disquette) | Phosphoric OK avec `--disk-rom`, idem |
| Démos exigeant `JMP ($xxFF)` bug | OK (bug reproduit ADR-11) |
| Logiciels exigeant opcodes illégaux NMOS | **NON supporté** (rare ; aucun cas connu sur Oric 1) |

#### 4.4.3 Tests de validation

| Test | Périmètre | Statut |
|---|---|---|
| Suite `test_cpu` Phosphoric (74 tests) | Opcodes 6502 unitaires | ✅ Vert (sur les deux cœurs cible B1.4i) |
| Klaus Dormann functional test | Tous opcodes officiels + BCD + RMW + JMP bug | ✅ Vert sur 6502 et 65C816 mode E (B1.5) |
| Boot ROM BASIC 1.0 dual-cœur | Équivalence bit-à-bit après 1 M cycles | ✅ Vert (B1.6) |
| Klaus extended 65C02 | Opcodes 65C02 supplémentaires (BBR/BBS/RMB/SMB...) | À évaluer (pertinent en B1.7+ uniquement si on supporte ces opcodes côté natif) |

### 4.5 Vue implémentation

#### 4.5.1 Workflow Phosphoric → HDL ULX3S

```
                  ┌────────────────────────────────┐
                  │  Phosphoric (C portable)       │
                  │   - Cœur 6502 (golden 6502)    │
                  │   - Cœur 65C816 (golden 816)   │
                  │   - Modèle ULA, VIA, AY        │
                  │   - Tests : Klaus, ROM, unit   │
                  └─────────────┬──────────────────┘
                                │
                       Spécification
                       comportementale
                                │
                                ▼
                  ┌────────────────────────────────┐
                  │  HDL ULX3S (Verilog/SpinalHDL) │
                  │   - 65C816 IP                  │
                  │   - ULA host + guest + comp.   │
                  │   - VIA, AY, SID, FDC          │
                  │   - Tests : co-simulation      │
                  │     contre Phosphoric          │
                  └─────────────┬──────────────────┘
                                │
                                ▼
                          Synthèse ECP5
                                │
                                ▼
                          Bitstream ULX3S
```

Phosphoric joue trois rôles :
1. **Lab d'expérimentation** des décisions architecturales avant gravage HDL.
2. **Référence comportementale** : le HDL doit reproduire son comportement, validé par co-simulation.
3. **Émulateur multiplateforme persistant** post-Oric 2 (utilisateurs sans ULX3S).

#### 4.5.2 Organisation des sous-projets (à terme)

```
/home/bmarty/oric2/
├── CLAUDE.md             ← instructions tactiques agent
├── docs/
│   ├── DAT.md            ← ce document
│   └── (futurs : memory map spec, SDK OricOS, HDL spec, …)
├── Phosphoric/           ← émulateur C, golden model (existant)
├── OricOS/               ← noyau + apps (à venir, post-B1)
└── HDL/                  ← projet FPGA ULX3S (à venir, post-B4)
```

(Chaque sous-projet a son propre git, conformément à l'instruction utilisateur globale.)

#### 4.5.3 Traçabilité ADR ↔ code ↔ tests

| ADR | Implémentation actuelle | Tests d'attestation |
|---|---|---|
| ADR-01 (CPU 65C816) | `Phosphoric/src/cpu/cpu65c816{,_opcodes}.c` | `test_cpu65c816*`, Klaus, boot_dual |
| ADR-02 (compositor + double ULA) | À implémenter (B4) | À écrire |
| ADR-03 (préemptif) | À implémenter (OricOS) | À écrire |
| ADR-04 (banking) | Mode N à venir (B1.7+) | À écrire |
| ADR-05 (asm + llvm-mos) | À mettre en place (toolchain OricOS) | Build infrastructure |
| ADR-06 (GUI SymbOS-like) | À implémenter (OricOS) | UX tests |
| ADR-07 (FAT32 SD + hostfs) | hostfs OK dans Phosphoric, FAT32 à venir | `test_storage` (existant), nouveau pour FAT32 |
| ADR-08 (bundle léger) | À spécifier | Loader test |
| ADR-09 (AY + SID-like) | AY OK dans Phosphoric, SID à venir (B4 HDL puis Phosphoric) | `test_audio` à étendre |
| ADR-10 (compat Oric 1) | Phosphoric 6502 cycle-exact, 65C816 mode E B1.4 | Klaus, boot_dual, suite Oric 1 |
| ADR-11 (mode E hybride) | `cpu65c816_opcodes.c` | `test_jmp_indirect_page_wrap_bug_adr11` + Klaus |

---

## 5. Correspondences (cohérence inter-vues)

| Correspondance | Sens | Vérification |
|---|---|---|
| Vue logique ↔ vue physique | Chaque composant logique mappe sur une entité matérielle (pas de feature OricOS sans support HW) | Revue lors B2 (modèle mémoire) et B4 (compositor) |
| Vue comportementale ↔ vue compat | Toutes les transitions XCE et tous les opcodes mode E sont conformes ADR-10 et ADR-11 | Klaus B1.5 + B1.6 (validé) |
| Vue logique ↔ vue compat | Le guest Oric 1 utilise exclusivement les composants compat (ULA guest, ROM Oric 1 originale, mode E) | Boot ROM dual-cœur B1.6 (validé) |
| Vue compat ↔ vue physique | Le HDL final doit reproduire les bugs ADR-11(c) au sens du golden model | Co-simulation Phosphoric ↔ HDL (post B4) |
| Vue implémentation ↔ vue logique | Chaque ADR est traçable à au moins un fichier source ou un sous-projet | Table §4.5.3 |
| Vue physique ↔ vue implémentation | Le HDL implémente strictement la spec comportementale Phosphoric | Co-simulation à mettre en place |

---

## 6. Architecture rationale

Toutes les décisions d'architecture sont consignées comme ADR dans `/home/bmarty/oric2/CLAUDE.md` §2 (ratifiées) et §3 (ouvertes). Au 2026-05-07 : **11 ADR ratifiées, 0 ADR ouverte**.

| ADR | Décision | Rationale concise | Statut |
|---|---|---|---|
| ADR-01 | CPU = 65C816 | Mode E pour compat Oric 1 native, mode N pour OricOS 16-bit, XCE pour paravirt | Ratifiée |
| ADR-02 | Compositor + double ULA | Le guest Oric 1 tourne pleine vitesse, rendu HW, pas de charge CPU host | Ratifiée |
| ADR-03 | Multitâche préemptif | Latence I/O et audio garantie ; SymbOS prouve faisable sur 8-bit | Ratifiée 2026-05-07 |
| ADR-04 | Banking, pas de MMU | Simplicité HDL ULX3S ; OricOS de confiance v1 ; ne bloque pas B2 | Ratifiée 2026-05-07 |
| ADR-05 | Kernel asm + userland C llvm-mos | Perf kernel maximale, userland portable et productif | Ratifiée 2026-05-07 |
| ADR-06 | GUI SymbOS-like | État de l'art fenêtré sur 8-bit, cohérent ADR-03 | Ratifiée 2026-05-07 |
| ADR-07 | FAT32 SD + hostfs émulateur | Interop universelle avec PC ; hostfs déjà opérationnel dans Phosphoric | Ratifiée 2026-05-07 |
| ADR-08 | Bundle léger (header + sections) | Simple à parser en asm, supporte icône/manifest, pas d'overkill ELF | Ratifiée 2026-05-07 |
| ADR-09 | AY + SID-like | Compat Oric 1 + extension expressive moderne ; IP HDL existante | Ratifiée 2026-05-07 |
| ADR-10 | Compatibilité Oric 1 stricte | Préservation patrimoniale, raison d'être Oric 2 | Ratifiée |
| ADR-11 | Mode E hybride pragmatique | Aligne sur Phosphoric 6502 actuel (bug JMP reproduit, illégaux NOP) | Ratifiée 2026-05-07 |

Pour le détail des alternatives et raisons de leur écartement, voir CLAUDE.md §2.

---

## 7. Glossaire

(Cf. CLAUDE.md §8 pour la liste détaillée. Termes spécifiques à ce DAT :)

- **Architecture description** : ensemble structuré d'artefacts décrivant l'architecture d'un système, conforme à ISO/IEC/IEEE 42010.
- **Concern** : intérêt de stakeholder vis-à-vis du système (ex : performance, sécurité, coût).
- **Stakeholder** : individu, équipe ou organisation qui a un intérêt dans le système.
- **Viewpoint** : modèle d'observation du système, adressant un ensemble de concerns. Définit comment construire une vue.
- **View** : description de l'architecture sous l'angle d'un viewpoint, instanciée pour le système concret.
- **Correspondence** : règle de cohérence reliant deux ou plusieurs vues.
- **Architecture rationale** : justification des décisions d'architecture (ADR + texte explicatif).

---

## 8. Références

### 8.1 Standards et méthodes
- ISO/IEC/IEEE 42010:2011 — *Systems and software engineering — Architecture description*.
- WDC W65C816S Datasheet, *The Western Design Center*.
- The Western Design Center 65C816 Programming Manual.

### 8.2 Documents projet
- `/home/bmarty/oric2/CLAUDE.md` — instructions tactiques agent (source de vérité ADR).
- `/home/bmarty/oric2/Phosphoric/CLAUDE.md` — guide Phosphoric (build, tests, archi interne).
- `/home/bmarty/oric2/Phosphoric/CHANGELOG`, `ROADMAP`, `VERSION_TRACKING`, `CIRRUS_OS` — bookkeeping.

### 8.3 Code et tests
- Repo Phosphoric : `https://git.nagominosato.fr:6775/chipinette/Phosphoric` (origin).
- Klaus Dormann tests : `https://github.com/Klaus2m5/6502_65C02_functional_tests`.

### 8.4 Inspirations architecturales
- **SymbOS** (Z80) — référence multitâche préemptif fenêtré sur 8-bit.
- **GS/OS** (Apple IIgs, 65C816) — OS sur 65C816 avec mode legacy.
- **AmigaOS Intuition / Hunk** — modèle GUI et format binaire.
- **Oricutron** — émulateur Oric 1 de référence pour comportement matériel détaillé.
- **ULX3S documentation** — `https://github.com/emard/ulx3s`.

---

## 9. Méta — évolution du document

- Ce document est versionné dans le repo workspace `/home/bmarty/oric2/`.
- Toute modification d'une ADR ratifiée passe par discussion explicite avec le project owner et mise à jour du DAT et du CLAUDE.md §2.
- Toute nouvelle ADR (ADR-12 et au-delà) sera ouverte dans le CLAUDE.md §3 avant d'être ratifiée et intégrée ici.
- La granularité du DAT peut évoluer : un sous-document spécialisé (Memory Map Specification, SDK OricOS, HDL Spec) peut être détaché si une section dépasse ~300 lignes.

| Version | Date | Changements | Auteur |
|---|---|---|---|
| 1.0 | 2026-05-07 | Version initiale ratifiée. Ratification ADR-03 à ADR-09 (sauf ADR-10 et ADR-11 déjà ratifiées). | bmarty + Claude |

---

*Fin du DAT v1.0.*
