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

### ADR-05 — Langage d'implémentation OricOS : asm + C llvm-mos (ratifiée 2026-05-07)
- **Kernel + drivers** en assembleur 65C816 natif (ca65 ou équivalent) : performances critiques, contrôle direct du banking.
- **Userland (apps, libs)** en C compilé par **llvm-mos** (toolchain LLVM ciblant 6502/65C816).

Alternatives écartées : tout asm (maintenance et apps lourdes), tout C (perfs kernel dégradées sur boucles serrées et drivers temps réel).

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

### ADR-13 — Mécanisme de syscall
**Question** : comment une app userland appelle le kernel ?
- (a) `COP #imm` + table de syscalls indexée par `imm` (style 65C816 natif).
- (b) `WAI` + protocole signal (peu idiomatique).
- (c) Call gate via JSL vers stub kernel exporté.

**Impact** : ABI kernel/userland, base de tout Sprint 4. Bloque OS-2.f.

### ADR-14 — Format TCB et structure interne tâche
**Question** : quelle représentation pour une tâche kernel ?
- Champs minimaux : `pid`, `state` (running/ready/blocked/zombie),
  `prio`, `regs_save` (A/X/Y/P/PC/PBR/DBR/D/S 16-bit), `stack_bank`,
  `code_bank`, `data_bank`, `parent_pid`.
- Layout en bank 1 dédiée ? Table fixe N tâches max ou liste chaînée ?

**Impact** : refactor scheduler obligatoire. Bloque OS-2.g.

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
