# Références d'art OricOS — qui inspire quel étage

**Version** : 1.0
**Date** : 2026-05-26
**Auteur** : bmarty
**Statut** : Note d'architecture (informatif). Compagnon du [DAT](./DAT.md) et des ADR.

> Cette note documente les **trois systèmes de référence** d'OricOS et **à quel
> étage** chacun inspire le design. Elle ne ratifie aucune décision : elle trace
> le *prior art* derrière les ADR existantes et l'arc GUI à venir (SP-3.n).
> Conforme au CLAUDE.md racine §5.5 (« toute décision d'architecture trace à
> une ADR ») et §10 (moratoire).

---

## 0. Thèse

OricOS ne copie aucune des trois références : il **fusionne le meilleur étage de
chacune**. C'est sa singularité — aucune des trois n'a eu les trois propriétés à
la fois sur une machine de cette classe.

> **OricOS = noyau préemptif à messages (SymbOS) + mécaniques 65C816 & GrafPort
> (Apple IIgs) + GUI déclarative GenUI/SpecUI à messages (GeoWorks).**

| Référence | CPU / époque | Étage emprunté | Pas emprunté |
|-----------|--------------|----------------|--------------|
| **SymbOS** | Z80 8-bit, 2006 | **Noyau** : préemptif, microkernel, messages | UI impérative |
| **Apple IIgs** | **65C816** 16-bit, 1986 | **Mécaniques CPU** : COP, banking, GrafPort | Coopératif mono-app, UI impérative |
| **GeoWorks / GEOS** | 6502 (C64) → x86 (Ensemble) | **Modèle d'UI applicative** : déclaratif, GenUI/SpecUI, messages | Moteur objet complet (« Goc »), géométrie auto-managée |

---

## 1. Comparaison par axe

| Axe | SymbOS | Apple IIgs (GS/OS + Toolbox) | GeoWorks / GEOS |
|-----|--------|------------------------------|-----------------|
| Multitâche | **Préemptif** (timer) ⭐ | Coopératif, mono-app foreground | Coopératif (Ensemble = multi-app coop) |
| Architecture noyau | **Microkernel + managers + messages** ⭐ | Monolithe ROM + Toolbox | OO : objets + messages, kernel coop |
| Mémoire | Banking multi-bank | **Memory Manager handles relogeables** ⭐ | Handles + objets (Ensemble) ; banks (C64) |
| Modèle GUI | WM Win95-like, **impératif** | Toolbox riche, **impératif** (`NewControl`) | **Déclaratif GenUI/SpecUI** ⭐ |
| Dialogues | Fenêtres + contrôles codés | `ModalDialog` + contrôles construits | **`DoDlgBox` + command table** (donnée) ⭐ |
| Communication | **Messages inter-processus** ⭐ | Appels procéduraux + event queue | **Messages objets** ⭐ |
| Sépare look / logique | Non | Non | **Oui** (générique / spécifique) ⭐ |
| Sophistication GUI | ●●○ | ●●● | ●●●● |
| Sophistication noyau | ●●●● | ●●○ | ●●○ |

⭐ = propriété qu'OricOS emprunte à cette référence.

---

## 2. Ce qu'OricOS prend à chacune

### 2.1 SymbOS → le noyau

Référence d'ADR-03 (multitâche préemptif) et ADR-06 (GUI fenêtrée). SymbOS
*prouve* qu'un microkernel préemptif + messages + GUI multifenêtrée tient sur du
Z80 — donc largement sur 65C816 + GPU blitter + 16 Mio SDRAM.

- **Déjà pris** : préemption (ADR-03), Forbid/Permit + block/wake (ADR-25),
  drivers event-driven (ADR-16).
- **Pas encore pris** : le **passage de messages inter-processus** générique.
  OricOS a des syscalls (COP) + rings dédiés, pas encore d'IPC message
  universel. C'est le **polish #1** (signaux multi-bits génériques d'ADR-25) +
  l'arc Event Manager (SP-3.n).

### 2.2 Apple IIgs → les mécaniques 65C816

Même CPU → transfert direct des mécaniques *liées au silicium* :

- **COP** comme trap de syscall (ADR-13) — l'IIgs utilisait un dispatcher JSL ;
  OricOS fait plus propre (COP est *fait pour ça*).
- **Banking** PBR/DBR/D (ADR-04), **mode E/N** (ADR-01, XCE).
- **Modèle GrafPort** de QuickDraw II : backing store par fenêtre + dessin en
  **coordonnées locales**, compositing à la sortie. C'est le « niveau QDF »
  d'OricOS (G.4/G.4bis, SP-3.m).
- **Pas pris** : multitâche coopératif mono-app (OricOS = préemptif) ; UI
  impérative `NewControl` (OricOS = déclaratif, cf. GeoWorks).

L'IIgs est l'**intrus** côté modèle logiciel (procédural, pas message-based) :
on ne le garde que pour ce qui touche au 65C816, pas pour son modèle applicatif.

### 2.3 GeoWorks / GEOS → le modèle d'UI applicative

Le plus avancé des trois côté UI ; direction GUI choisie pour OricOS (arc SP-3.n).

- **UI déclarative** : l'app décrit menus/dialogues/contrôles comme des *tables
  de données*, le système rend + exécute. Continu avec l'existant
  (`menu_defs`, `ICON_TABLE` déjà table-driven).
- **GenUI / SpecUI** : UI générique déclarée par l'app / rendu « spécifique »
  (look) changeable **sans recompiler les apps**. Signature de GeoWorks Ensemble.
- **`DoDlgBox` + command table** (façon `DB_*` de GEOS C64) : le dialogue *est*
  une donnée, exécuté par un seul syscall modal.
- **MainLoop → messages** sémantiques (pas de callback cross-bank).
- **Pas pris** : le moteur objet complet (« Goc »), la géométrie auto-managée et
  les fontes vectorielles Nimbus-Q (surdimensionnés v1 — cueillir brique par
  brique). GEOS C64 (6502) prouve la faisabilité 8-bit ; Ensemble donne le
  modèle conceptuel.

---

## 3. La convergence : le message comme colonne vertébrale

**SymbOS et GeoWorks pointent tous deux vers les messages** — SymbOS au niveau
noyau (IPC inter-processus), GeoWorks au niveau UI (messages objets). OricOS a
déjà semé ce modèle dans ADR-16 (drivers event-driven) et le cadrage Event
Manager (MainLoop → messages). Le « modèle message » est donc la **colonne
vertébrale commune** des deux références préférées d'OricOS : il est cohérent de
l'adopter **du noyau jusqu'à l'app**, l'Apple IIgs servant uniquement de banque
de mécaniques 65C816.

---

## 4. Synthèse

Aucune des trois références n'a réuni préemption + 65C816 natif + UI déclarative :

- **SymbOS** : préemptif ✓ — mais UI impérative.
- **Apple IIgs** : 65C816 ✓ — mais coopératif + UI impérative.
- **GeoWorks** : UI déclarative ✓ — mais coopératif.

OricOS vise **l'union** : préemptif (SymbOS) + 65C816 natif & GrafPort (IIgs) +
UI déclarative GenUI/SpecUI à messages (GeoWorks). Jamais réalisée sur cette
classe de machine, mais chaque brique est *prouvée faisable séparément* par l'une
des trois.

---

## 5. Traçabilité

| Étage | ADR / arc | Référence d'art |
|-------|-----------|-----------------|
| Multitâche préemptif | ADR-03 | SymbOS |
| Modèle de concurrence (Forbid/Permit, messages) | ADR-25 | AmigaOS Exec / SymbOS |
| Driver model event-driven | ADR-16 | SymbOS |
| Mécanisme syscall (COP + table) | ADR-13 | GS/OS (Apple IIgs) |
| ABI syscall userland | ADR-17 | GS/OS (Apple IIgs) |
| Banking / mode E-N | ADR-01, ADR-04 | WDC 65C816 / Apple IIgs |
| GrafPort (backing store, coords locales) | SP-3.m (G.4/G.4bis) | QuickDraw II (Apple IIgs) |
| GUI fenêtrée | ADR-06 | SymbOS |
| **Modèle GUI déclaratif (GenUI/SpecUI, DoDlgBox)** | **ADR à instruire (arc SP-3.n)** | **GeoWorks Ensemble / GEOS C64** |

---

*v1.0 — 2026-05-26. Note informative ; toute décision reste portée par les ADR.*
