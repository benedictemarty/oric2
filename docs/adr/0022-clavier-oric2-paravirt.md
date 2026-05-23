# ADR-22 — Clavier Oric 2 paravirtualisé hybride

- **Statut** : **ratifiée 2026-05-23** (ratification anticipée, exception moratoire condition 2 : jalon OS-2.d ≤ 4 semaines)
- **Date** : 2026-05-23
- **Décideurs** : bmarty (humain — direction hybride + ratification du dossier, 2026-05-23), Claude Code
- **Jalon déclencheur** : OS-2.d (driver clavier) — jalon courant, estimé 3-5 j (≤ 4 semaines → exception moratoire condition 2 applicable).
- **ADR liées** : ADR-02 (double-ULA, modèle paravirtualisation matériel), ADR-10 (compat Oric 1 stricte), ADR-16 (driver model — **révision requise**, voir §Impact).

---

## Contexte

OS-2.d doit livrer l'entrée clavier d'OricOS. Le plan initial (BACKLOG OS-2.d,
ADR-16) prévoyait que l'hôte OricOS **scanne la matrice Oric 1** (VIA PB[0:2]
col select + PSG R14 rows), réutilisant le clavier hérité (rétrocompat ADR-10).

Le point soulevé : un OS moderne (OricOS, multitâche/GUI SymbOS-like, ADR-06)
qui fait du **polling matriciel bas niveau** pour son propre clavier est
incohérent — le scan matriciel est une contrainte Oric 1 historique, pas un
choix d'architecture Oric 2. Par ailleurs le guest Oric 1 (ADR-02) **doit**
voir une matrice (compat stricte ADR-10).

L'analogie directe est **ADR-02** : une source physique (le rendu) alimente
deux consommateurs via des chemins matériels distincts (ULA host moderne +
ULA guest attribute-based). Le clavier suit la même logique : **une source
physique de touches → un contrôleur Oric 2 moderne (hôte) + une matrice
virtuelle Oric 1 (guest)**.

### Chiffrage de la contrainte

- Matrice Oric 1 : 8 colonnes × 8 rangées = 64 touches, active low, scan via
  VIA + PSG (~830 cycles/scan complet d'après commentaire kernel.s actuel).
- Scan à chaque tick IRQ (512 cycles de période visés) → le scan matriciel
  **consomme plus que la période du tick** : dette de perf déjà identifiée.
- Plage I/O bank 0 libre : `$0350-$035F` (VIA `$0300`, Microdisc `$0310`,
  VRAM `$0330`, GPU `$0340` déjà alloués).

---

## Options envisagées

### (a) Matrice Oric 1 réutilisée (plan BACKLOG/ADR-16 initial)

OricOS scanne la matrice VIA/PSG comme l'Oric 1. Le code `kernel_kbd_scan`
existe déjà.

- ✅ Coût OricOS faible (infrastructure scan déjà amorcée).
- ✅ Aucun nouveau HW à modéliser dans Phosphoric.
- ⚠️ Scan matriciel ~830 cyc > période tick 512 cyc → dette perf.
- ⚠️ Pas de séparation host/guest : un seul clavier matriciel partagé, routage
  par focus non trivial (qui consomme la touche ?).
- ⚠️ Sur HDL ULX3S il faut **de toute façon** une couche soft IO USB→matrice
  (cf. `docs/DAT.md` « Clavier USB via FT232 + soft IO ? »). On reporte le
  problème sans le résoudre.
- **Écartée** (incohérence OS moderne + dette reportée).

### (b) Clavier Oric 2 natif seul

Nouveau contrôleur Oric 2 (FIFO scancodes/ASCII + IRQ), pas de matrice du tout.

- ✅ Propre et moderne pour l'hôte.
- ❌ Le guest Oric 1 n'a plus de matrice à lire → **casse ADR-10** (compat
  stricte : le logiciel Oric 1 lit la matrice via VIA/PSG).
- **Écartée** (viole une contrainte dure).

### (c) Hybride paravirtualisé **[retenue, choix humain 2026-05-23]**

Contrôleur clavier Oric 2 natif pour l'hôte OricOS (FIFO ASCII + IRQ) **+**
matrice Oric 1 virtuelle pour le guest, alimentée par le même contrôleur.
Miroir exact d'ADR-02 (double-ULA).

- ✅ Cohérence architecturale avec ADR-02 (paravirtualisation matérielle).
- ✅ Hôte OricOS moderne : pas de scan, IRQ + pop FIFO (~quelques cycles).
- ✅ Guest Oric 1 voit sa matrice normale → ADR-10 préservée.
- ✅ Routage par focus explicite (bit `route_to_guest` du contrôleur).
- ⚠️ Coût : nouveau device Phosphoric `kbd2_device.{c,h}`, révision ADR-16
  (source clavier : VIA T1 scan → IRQ contrôleur KBD2), réécriture du driver
  clavier OricOS (plus simple que le scan + edge-detect + keymap).
- **Retenue**.

---

## Décision (proposée)

### Contrôleur clavier Oric 2 — registres `$0350-$035F` (bank 0, DBR=0)

| Adresse | Registre | R/W | Description |
|---------|----------|-----|-------------|
| `$0350` | `KBD2_STATUS` | R | bit7 = `data_ready` (FIFO non vide), bit6 = `overflow`, bit0 = `guest_focus` |
| `$0351` | `KBD2_DATA` | R | pop tête FIFO → keycode ASCII v1. La **lecture avance la FIFO**. $00 si vide. |
| `$0352` | `KBD2_CTRL` | R/W | bit0 = IRQ enable, bit1 = clear FIFO (W, auto-reset), bit2 = `route_to_guest` |
| `$0353` | `KBD2_MOD` | R | bitmask modificateurs courants : bit0=SHIFT, bit1=CTRL, bit2=FUNCT, bit3=CAPS |
| `$0354-$035F` | réservé v2 | — | scancode brut, repeat rate, LED état, mapping reconfigurable |

- **FIFO host** : profondeur 16 (alignée ADR-16 ring buffer kernel). Le
  contrôleur traduit (col,row) physique → ASCII **côté HW/modèle** (la keymap
  quitte le kernel). `overflow` set si FIFO pleine.
- **IRQ** : ligne dédiée KBD2. Entrée dans la table dispatch IRQ ADR-16
  (`$01:5680`). Le handler lit `KBD2_DATA` jusqu'à FIFO vide, push dans le ring
  kernel `$5860`.
- **Matrice virtuelle guest** : le contrôleur maintient une matrice 8×8 Oric 1
  (layout historique) reflétant l'état courant des touches. Quand
  `route_to_guest=1`, cette matrice alimente le chemin VIA PB / PSG R14 que lit
  le guest (exactement comme un Oric 1 réel). Quand `route_to_guest=0`, les
  touches vont seulement à la FIFO host.

### Impact OricOS kernel

- Le driver host devient **IRQ-driven pur** : handler KBD2 → pop FIFO → push
  ring `$5860`. Plus de `kernel_kbd_scan` matriciel pour l'hôte (la keymap et
  l'edge-detect disparaissent du kernel, déplacés dans le contrôleur).
- `kernel_kbd_init` : configure `KBD2_CTRL` (IRQ enable), au lieu de DDR/PSG.
- `SYS_GET_KEY` (non-bloquant) / `SYS_READ_CHAR` (bloquant) : inchangés
  conceptuellement — lisent le ring `$5860` (ADR-16).
- Le code VIA/PSG scan existant (`kernel_kbd_scan`, `KBD_MATRIX`) est **retiré**
  ou conservé en legacy mort (à trancher à l'impl).

### Impact Phosphoric (golden model)

- Nouveau `src/io/kbd2_device.{c,h}` : FIFO ASCII + IRQ + matrice virtuelle
  guest. Réutilise la table char→(row,col) de `io/keyboard.c` pour la matrice
  guest ; pousse l'ASCII direct dans la FIFO host.
- Routage I/O `$0350-$035F` (gated `--machine oric2`).
- Source des touches : événements SDL existants (réutilise `oric_keyboard`).

### Impact ADR-16 (révision requise)

La ligne « Clavier (OS-2.d) | VIA T1 | ring buffer 16 keycodes `$5860` |
SYS_GET_KEY/READ_CHAR » devient :

« Clavier (OS-2.d) | **IRQ contrôleur KBD2 (`$0350-$035F`)** | ring buffer 16
keycodes `$5860` | SYS_GET_KEY/READ_CHAR ». Le ring kernel et l'ABI syscall
(ADR-17) sont **inchangés** — seule la source amont change.

---

## Conformité moratoire (CLAUDE.md §10)

1. **Dossier écrit** : ce document (contexte chiffré, 3 alternatives chiffrées,
   recommandation tracée + décision humaine de direction du 2026-05-23). ✅
2. **Implémentation prête / jalon dur** : OS-2.d est le jalon courant, estimé
   3-5 j (BACKLOG) ≤ 4 semaines → exception applicable. L'impl de référence
   (kbd2_device + driver) sera produite immédiatement après ratification. ⚠️ à
   confirmer par l'humain (ratification anticipée).
3. **Cohérence ADR** : pas de contradiction avec ADR-02/06/10/17. Révision
   coordonnée d'ADR-16 (source clavier) documentée ci-dessus. ✅

---

## Conséquences

- **Positives** : architecture clavier cohérente avec le modèle double-ULA ;
  OricOS moderne ; guest Oric 1 préservé ; keymap hors kernel.
- **Négatives** : nouveau device à modéliser + tester ; révision ADR-16 ;
  travail HDL ULX3S supplémentaire (contrôleur KBD2 + matrice virtuelle) à
  cadrer (hors v1 émulateur).
- **Ouvert** : routage focus multi-fenêtres (qui a le focus clavier) — relève
  du window manager (Sprint 3) ; le bit `route_to_guest` est le hook bas niveau.

---

*Ratifiée 2026-05-23. Reportée dans CLAUDE.md §2. Révision coordonnée d'ADR-16 (ligne clavier).*
