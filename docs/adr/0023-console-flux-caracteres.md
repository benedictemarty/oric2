# ADR-23 — Console OricOS : flux de caractères, backend d'affichage interchangeable

- **Statut** : **ratifiée 2026-05-24**
- **Date** : 2026-05-24
- **Décideurs** : bmarty (humain), Claude Code
- **ADR liées** : ADR-10 (compat Oric 1 stricte), ADR-12 (HIRES Oric 2 ULA guest),
  ADR-17 (ABI syscall), ADR-20 (desktop XVGA), ADR-21 (GPU Blitter, commande TEXT).
- **Conformité moratoire** : oui — l'implémentation de référence existe déjà
  (console `kernel_print_*` en place, ABI `SYS_PRINT_CHAR`/`SYS_PRINT_STRING`
  d'ADR-17 déjà agnostique de l'affichage). Cette ADR **formalise une règle d'or**
  pour border une dette identifiée, sans engager de nouvelle implémentation.

---

## Contexte

Le console kernel d'OricOS écrit aujourd'hui dans la **RAM écran texte de
l'Oric 1** (`$BB80-$BFE0`, 40×28, attributs sériels, fonte en `$B400`), via la
même ULA — un **choix de bootstrap** (Sprint 2.c) pour afficher du texte avant
l'existence du desktop GPU. Il hérite donc des limitations Oric 1 : 40×28,
attribute clash, 8 couleurs, cellule ligne0/col0 réservée à l'INK.

La cible (ADR-20 + ADR-21) est un texte rendu par le **GPU** (commande `TEXT`,
`kernel_gfx_text`) dans le framebuffer **XVGA 1024×768×4bpp** : pas d'attribute
clash, position pixel, couleur par caractère.

**Question** : rester sur le backend Oric 1 crée-t-il une dette lourde ?

### Analyse de dette (chiffrée)

- Code console couplé Oric 1 : **~140 lignes asm** (`kernel_print_char`,
  `print_string`, `console_init`, `clear_screen`, `scroll_up`, `print_hex8`,
  suivi curseur `CURSOR_ADDR`/`CURSOR_X`).
- **Fait protecteur** : l'ABI ADR-17 n'expose **que** `PRINT_CHAR`/`PRINT_STRING`
  (flux de caractères) — **aucun** syscall de positionnement curseur ni
  d'attribut couleur calqué Oric 1. L'ABI n'est donc pas contaminée.
- Migration vers GPU = réécriture **locale et bornée** du corps de ces routines
  derrière la même API ; les appelants (banner, panic, syscalls, apps) ne bougent pas.

La dette reste **légère** *si et seulement si* aucune spécificité Oric 1 ne fuit
hors de l'implémentation. Elle deviendrait **lourde** si la géométrie (40×28),
l'adressage écran (`$BB80+offset`), le modèle curseur linéaire, ou la sémantique
attribute-clash fuyaient dans l'ABI ou dans des apps userland.

---

## Décision

**Le console OricOS est défini comme un flux de caractères ; le backend
d'affichage est un détail d'implémentation interchangeable.**

### Règle d'or (contrat ABI)

L'interface publique du console — `kernel_print_char`, `kernel_print_string`,
`SYS_PRINT_CHAR` ($01), `SYS_PRINT_STRING` ($02) — **ne doit jamais exposer** :
- de géométrie d'écran (nombre de colonnes/lignes, ex. 40×28) ;
- d'adresse de RAM écran (`$BB80`, offsets directs) ;
- de modèle de curseur en adresse linéaire écran ;
- de sémantique d'attribut sériel Oric 1 (INK/PAPER occupant une cellule).

Ces éléments restent **privés** à l'implémentation du backend courant.

### Backends

| Backend | Statut | Usage |
|---|---|---|
| **Oric 1 text mode** (`$BB80`, ULA, 40×28) | v1 **bootstrap** | boot, debug, panic, avant desktop |
| **GPU `kernel_gfx_text`** (XVGA 1024×768×4bpp, ADR-20/21) | cible | desktop OricOS, console « riche » |

Le passage de l'un à l'autre est une **réimplémentation du corps** de
`kernel_print_*`, sans changement d'ABI.

### Contraintes de bordure (pour garder la dette légère)

1. **Ne pas ajouter** de syscall de positionnement curseur ou d'attribut
   couleur modelé sur l'Oric 1 (ex. pas de `SYS_SET_INK` style attribute-clash).
   Tout futur syscall de positionnement/couleur devra cibler le modèle XVGA
   (coordonnées pixel/cellule, palette 16, ADR-20) — via une révision d'ADR-17.
2. **Ne pas livrer d'app userland** supposant 40×28, l'adressage `$BB80`, ou les
   attributs Oric 1 **avant** que le backend GPU console existe. Fenêtre de
   risque = Sprint 4 (userland) : le backend GPU doit précéder toute app faisant
   du texte positionné/coloré.
3. **`$BB80`/40×28 = détail privé**. Les seules fuites tolérées sont les
   **assertions de test** (test-only, ex. `test_oricos_boot`).
4. **Séparation host/guest (ADR-02)** : à terme, le mode texte Oric 1 redevient
   exclusivement le chemin **guest** ; le texte de l'hôte OricOS passe par le GPU.

---

## Options envisagées

### (a) Figer le mode texte Oric 1 comme mode texte officiel Oric 2
- ❌ Attribute clash + 40×28 + 8 couleurs incompatibles avec XVGA 4bpp (ADR-20).
- ❌ Contamine l'ABI si on expose curseur/attributs → dette lourde, gel ABI.
- **Écartée**.

### (b) Basculer immédiatement le console sur le GPU
- ⚠️ Exige `kernel_gfx_text` + framebuffer XVGA opérationnels et un backend GPU
  mûr avant d'avoir un console de boot/debug fiable. Retarde les jalons OS en cours.
- **Écartée pour maintenant** (sera la cible, pas le présent).

### (c) Flux de caractères + backend interchangeable **[retenue]**
- ✅ Garde le console Oric 1 en bootstrap (coût nul, déjà en place).
- ✅ Borne la dette à « légère » via la règle d'or (ABI agnostique).
- ✅ Migration GPU = réécriture locale ~140 lignes derrière l'API, sans churn appelants.
- **Retenue**.

---

## Conséquences

- **Positives** : dette bornée et explicite ; boot/debug fonctionnels sans
  dépendre du GPU ; bascule GPU future à coût maîtrisé ; ABI ADR-17 préservée.
- **Négatives** : double chemin texte (Oric 1 bootstrap + GPU cible) à terme ;
  vigilance requise à chaque ajout de syscall texte et à l'ouverture du userland.
- **Déclencheur de migration** : avant tout texte positionné/coloré en userland
  (Sprint 4) ou à l'arrivée du desktop GUI (Sprint 3), implémenter le backend
  GPU `kernel_gfx_text` derrière `kernel_print_*`.

Alternatives écartées : cf. §Options (a) et (b).
