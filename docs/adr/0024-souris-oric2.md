# ADR-24 — Contrôleur souris Oric 2

- **Statut** : **ratifiée 2026-05-24** (ratification anticipée, exception moratoire condition 2 : prérequis SP-3.e ≤ 4 semaines ; modèle hybride absolu+delta validé par l'humain)
- **Date** : 2026-05-24
- **Décideurs** : bmarty (humain — « souris d'abord » + modèle hybride absolu/delta, 2026-05-24), Claude Code
- **Jalon déclencheur** : SP-3.e (event loop GUI multifenêtré + focus + drag) — nécessite une source de pointage ; aucun driver souris n'existe.
- **ADR liées** : ADR-16 (driver model — classe 1 IRQ-driven), ADR-22 (clavier KBD2, même esprit de device Oric 2 natif), ADR-20 (desktop XVGA 1024×768).

---

## Contexte

SP-3.e (GUI SymbOS-like : focus fenêtre, drag) exige du **pointage**. L'Oric 1
n'avait pas de souris standard ; il n'y a donc **rien à virtualiser pour le
guest** (contrairement au clavier, ADR-22). La souris est un **périphérique
Oric 2 natif pur**, exposé uniquement à l'hôte OricOS.

Plutôt que piloter la GUI au clavier (palliatif), on spécifie un contrôleur
souris matériel — cohérent avec le modèle device Oric 2 (KBD2, GPU, VRAM) et
avec la cible HDL ULX3S (souris USB → contrôleur).

### Chiffrage

- Desktop XVGA 1024×768 (ADR-20) → coordonnées **10 bits** par axe (0..1023 / 0..767).
- Plage I/O bank 0 libre : `$0360-$036F` (VIA `$0300`, Microdisc `$0310`,
  SD `$0320`, VRAM `$0330`, GPU `$0340`, KBD2 `$0350` déjà alloués).
- Driver model ADR-16 classe 1 (IRQ-driven event queue) : la souris est à
  latence faible critique, comme le clavier.

---

## Options envisagées

### (a) Pas de souris — GUI au clavier
- ✅ Zéro nouveau hardware.
- ❌ Pointage au clavier (Tab/flèches) non-SymbOS, palliatif ; drag peu naturel ;
  re-travail quand la souris arrivera.
- **Écartée** (choix humain : souris d'abord).

### (b) Contrôleur **position absolue** seule
Le contrôleur maintient X/Y absolus clampés ; l'OS lit directement.
- ✅ OS minimal. ⚠️ Couple le device à la résolution vidéo ; pas d'accélération.
- Bon mais fermé aux évolutions sans nouvelle ADR.

### (c) Contrôleur **deltas relatifs** seuls (style PS/2 brut)
- ✅ Découple input/vidéo, fidèle HW. ❌ +code kernel (accumulation/clamp/accél),
  gestion overflow. v0.1 plus lourd.

### (d) **Hybride : absolu (clampé) + deltas read-clear** **[retenue]**
Le contrôleur maintient la position absolue X/Y 10-bit (clampée écran) **et**
expose des registres delta signés `DX/DY` (accumulés depuis la dernière lecture,
**read-clear**).
- ✅ **v0.1 utilise l'absolu** → driver OS minimal (lit X/Y directement).
- ✅ Les deltas restent disponibles pour l'**accélération de pointeur / usage
  avancé futur** (v2) **sans nouvelle ADR**.
- ✅ Cohérent ADR-16 (IRQ-driven) + ADR-22 (device Oric 2 + CTRL).
- ⚠️ Un peu plus de HW/modèle (2 registres delta + accumulateur signé).
- **Retenue** (choix humain 2026-05-24).

---

## Décision (proposée)

### Contrôleur souris Oric 2 — registres `$0360-$036F` (bank 0, DBR=0)

| Adresse | Registre | R/W | Description |
|---------|----------|-----|-------------|
| `$0360` | `MOU2_STATUS` | R | bit7 = `event` (mouvement/bouton depuis dernier clear), bit0 = bouton gauche, bit1 = droit, bit2 = milieu |
| `$0361` | `MOU2_X_LO` | R | position X bits [7:0] (absolue, 0..1023) |
| `$0362` | `MOU2_X_HI` | R | X bits [9:8] |
| `$0363` | `MOU2_Y_LO` | R | position Y bits [7:0] (0..767) |
| `$0364` | `MOU2_Y_HI` | R | Y bits [9:8] |
| `$0365` | `MOU2_BUTTONS` | R | bitmask boutons (bit0=G, bit1=D, bit2=M) |
| `$0366` | `MOU2_CTRL` | R/W | bit0 = IRQ enable, bit1 = clear event (W, auto-reset) |
| `$0367` | `MOU2_DX` | R | delta X signé depuis dernière lecture (**read-clear**) |
| `$0368` | `MOU2_DY` | R | delta Y signé depuis dernière lecture (**read-clear**) |
| `$0369-$036F` | réservé v2 | — | molette, curseur HW (sprite), sensibilité |

- **Position absolue** maintenue par le contrôleur, **clampée** à `[0,1023]×[0,767]`
  (modèle hybride : v0.1 OricOS lit X/Y absolus).
- **Deltas `DX/DY`** : accumulés (signés, saturés `[-128,127]`) depuis la dernière
  lecture ; la lecture remet à 0 (read-clear). Disponibles pour accélération v2.
- **IRQ** dédiée `IRQF_MOU2` : assertée quand `event` set ET IRQ enable.
  Lire `MOU2_STATUS` ou écrire `CTRL` bit1 clear l'event (deassert IRQ).
- **Pas de matrice virtuelle guest** (l'Oric 1 n'a pas de souris).

### Impact

- **Phosphoric** : `src/io/mouse2_device.{c,h}` (position, boutons, IRQ).
  Alimenté par les événements SDL souris (motion → X/Y, boutons). Gated
  `--machine oric2`. Routage I/O `$0360-$036F`. `IRQF_MOU2` (nouvelle source).
- **OricOS kernel** : driver souris IRQ-driven (lit X/Y/boutons), exposé à
  l'event loop GUI (SP-3.e). Éventuel `SYS_GET_MOUSE` (extension ABI v1+,
  slot réservé ADR-17 `$13-$3F`) — à trancher à l'impl.
- **ADR-16** : ajoute une ligne classe 1 (souris OS-2.x / SP-3.e | IRQ MOU2 |
  état X/Y/boutons | event loop GUI).

---

## Conformité moratoire (CLAUDE.md §10)

1. **Dossier écrit** : ce document (contexte chiffré, 3 alternatives chiffrées,
   recommandation + décision humaine de direction). ✅
2. **Jalon dur ≤ 4 semaines** : prérequis immédiat de SP-3.e (jalon GUI courant) →
   exception applicable. Impl de référence produite juste après ratification. ⚠️ humain.
3. **Cohérence ADR** : aucun conflit ; calque ADR-22/ADR-16. ✅

---

## Conséquences

- **Positives** : pointage matériel propre, GUI SymbOS-like naturelle, cohérent
  device Oric 2 + cible HDL.
- **Négatives** : nouveau device à modéliser/tester + nouvelle source IRQ ;
  décale légèrement SP-3.e (mais le débloque proprement).
- **Ouvert** : curseur matériel (sprite HW) reporté v2 (le compositor/GPU pourra
  dessiner le curseur) ; molette/accélération v2.

*Ratifiée 2026-05-24. Reportée dans CLAUDE.md §2. Révision coordonnée d'ADR-16 (ligne souris).*
