# ADR-27 (DRAFT) — Modèle de backing store fenêtre

- **Statut** : **en cours d'instruction — option (b) retenue par l'humain**
  (2026-05-27). DRAFT non ratifié (moratoire §10 : ratification à finaliser
  quand la migration kernel atteindra le seuil 50 % + cohérence ADR-21).
  Référence GPU (registre BPL configurable) **implémentée** (Phosphoric 1.22.88).
- **Date d'ouverture** : 2026-05-27
- **Décideurs** : bmarty (choix option b), Claude Code
- **Origine** : audit GPU toolbox senior (2026-05-27), Finding B. Découvert
  à l'occasion du fix BLIT v0.2 (byte_w/byte_h 16-bit, Phosphoric 1.22.87).

## 0. Décision et état d'avancement (2026-05-27)

L'humain a retenu l'**option (b)** : stride GPU (BPL) configurable + backing
store compact. Avancement :

- ✅ **Référence GPU implémentée** (Phosphoric 1.22.88) : registre `bpl`
  persistant (défaut 512) + opcode `GPU_OP_SET_BPL` ($08) ; BLIT utilise
  `bpl` pour la SOURCE et 512 (XVGA) pour la destination ; FILL_RECT*/LINE/
  TEXT* honorent `bpl`. Helper kernel `kernel_gfx_set_bpl`. 2 tests unitaires
  (`test_set_bpl_changes_fill_stride`, `test_blit_compact_source_stride`).
  **Rétro-compatible** : défaut 512 → comportement identique (594 tests verts).
- ⏳ **Reste (migration kernel, < 50 %)** : allocation backing store compacte
  et/ou multi-banques contiguë ; pose de `bpl = byte_w` par fenêtre dans
  `kernel_wm_compose` ; dessin app à la stride compacte ; clip aux dimensions
  réelles de la surface (pas XVGA) ; reset `bpl=512` avant tout dessin direct
  framebuffer (`kernel_wm_redraw`).

### Contrainte dure tranchée : pas de port I/O libre

Les 16 ports GPU `$0340-$034F` sont **tous** assignés (ARG1-4, STATUS, TRIGGER,
INT_CTRL) et `$0350` est le contrôleur KBD2 (ADR-22). Un registre BPL **dédié
par port** imposerait d'étendre l'allocation I/O GPU → révision ADR-21 + ADR-22
+ MEMORY_MAP. **Écarté.** Le BPL est donc exposé via un **opcode `SET_BPL`**
(mécanisme CMD_OP + ARG1 + TRIGGER existant) qui fixe un état `bpl` persistant.
Zéro nouveau port, zéro impact memory map.

### Hazard identifié : état global `bpl`

`bpl` est un état GPU **global** partagé par toutes les surfaces. Le kernel doit
le discipliner : `SET_BPL byte_w` avant de composer une fenêtre, `SET_BPL 0`
(→512) avant tout dessin direct dans le framebuffer XVGA. Un oubli corrompt la
stride. Documenté dans `kernel_gfx_set_bpl`. (Une alternative v0.3 = src_bpl/
dst_bpl par-commande encodés dans les octets hauts libres d'ARG3/ARG4.)

---

## 1. Contexte

Le modèle de composition GUI (« GrafPort-like », convention G.2 du Sprint 3.m)
attribue à chaque fenêtre un **backing store** : une zone SDRAM où l'app dessine
en coordonnées **locales** via les syscalls `SYS_GFX_*`. Le window manager
composite ensuite ces backing stores vers le framebuffer XVGA via
`kernel_wm_compose` (BLIT par fenêtre, respect du Z-order).

Convention d'implémentation actuelle (`wm.s`, `sys_win_create` §3005) :

> Backing store SDRAM implicite par slot : base = `($06+slot):$0000` — **1 banque
> de 64 KiB par slot**.

Le GPU utilise une **stride (BPL) figée à 512 octets/ligne** (= XVGA 1024 px / 2,
ADR-20), identique pour source et destination du BLIT.

## 2. Problème (Finding B)

64 KiB / 512 = **128 lignes maximum** par backing store.

`kernel_wm_compose` exécute `byte_h = h` (hauteur fenêtre en lignes). Pour une
fenêtre de hauteur `h > 128 px`, le BLIT source lit `h × 512` octets > 65536,
donc **déborde de la banque du slot dans les banques voisines** (`$06+slot+1`…),
qui appartiennent à d'autres backing stores :

| Hauteur fenêtre | Octets source (stride 512) | Banques sur-lues |
|---|---|---|
| 128 px | 65 536 | 1 (exactement la banque) |
| 200 px | 102 400 | 2 |
| 744 px (maximisée) | 380 928 | ~6 |

**Historique** : avant le fix v0.2, `byte_h` était tronqué à 8 bits
(`h=768 & 0xFF = 0` → 0 ligne → fenêtre invisible). La troncature **masquait**
le débordement. Le fix v0.2 (correct) **expose** cette limite d'architecture.

**Périmètre d'impact** : uniquement le chemin **compose** (apps dessinant en
backing store + `SYS_WIN_FLUSH`). Le chemin `kernel_wm_redraw` (FILL_RECT16
direct vers framebuffer, sans backing store) n'est **pas** concerné. Les apps
actuelles (clock, win_hello) créent de petites fenêtres → le bug n'est pas
encore déclenché en pratique, mais c'est un **piège latent** dès qu'une app
ouvre une fenêtre haute.

## 3. Contrainte de dimensionnement

Une fenêtre exploitable plein écran fait ~1024×744 (XVGA moins chrome). À
stride 512, le backing store correspondant pèse `744 × 512 = 380 928 octets`
≈ **6 banques de 64 KiB**. Pour 8 slots plein écran : jusqu'à **48 banques
(3 MiB)**. Le pool actuel « banks 4-15 » (768 KiB, MEMORY_MAP §6) est trop
petit pour ce cas ; les banks 16-127 (7 MiB) ou 128-191 (4 MiB) sont
candidates.

## 4. Options instruites (NON tranchées)

| Option | Principe | Coût SDRAM | Coût HDL/GPU | Complexité kernel | Risque |
|---|---|---|---|---|---|
| **(a) Backing store multi-banques contigu** | Allouer `ceil(h×512/65536)` banques contiguës par slot ; base recalculée au create/resize | jusqu'à 6 banques/slot | nul (stride 512 conservée) | allocateur contigu (vs LIFO actuel) ; `kernel_gfx_window_base` calcule base par slot→banque | fragmentation SDRAM ; resize = realloc |
| **(b) Stride backing store = largeur fenêtre** | Registre BPL GPU configurable (déjà annoncé ADR-21 v0.2 / gpu_device.h) ; backing store compact `ceil(w/2)×h` | idem ~6 banques si pleine largeur, moins si étroite | **registre BPL src/dst GPU** + recalcul offsets dans exec_blit/fill/text | GPU passe BPL au BLIT ; compose fournit 2 strides | change l'API GPU (ADR-21 v0.2) ; tests à étendre |
| **(c) Plafond 128 px + clamp** | `byte_h ≤ 128` dans compose ; fenêtres limitées 128 px | inchangé | nul | trivial | **UX cassée** : pas de grande fenêtre sur desktop 768 → non-viable |
| **(d) Dessin direct framebuffer + clipping** | Abandon backing store ; apps dessinent dans le FB, WM clippe au rect fenêtre | nul (pas de backing) | clipping rect GPU | gros refactor compose/redraw ; perte du redraw sans repaint app | rupture du modèle GrafPort ; scope élevé |

### Recommandation senior tracée (à valider, non décisionnelle)

L'option **(b)** s'aligne avec une intention **déjà documentée** :
- ADR-21 : « v0.1 sync ; **v0.2 : BPL et résolution configurables via registres
  dédiés** » ;
- `gpu_device.h` §73-75 : « v0.2 : BPL et résolution configurables ».

Elle résout Finding B *et* débloque les backing stores compacts (économie SDRAM
pour fenêtres étroites). Coût principal = un registre BPL GPU + propagation dans
les `exec_*`. (a) est plus simple côté GPU mais déplace la complexité vers
l'allocateur contigu et gaspille la SDRAM (stride pleine largeur toujours).
(c) est non-viable, (d) est hors-scope v1.

**Ce dossier ne ratifie pas (b).** Décision soumise aux 3 conditions du
moratoire (CLAUDE.md §10) : dossier chiffré (ce document, à compléter avec
mesures), ≥ 50 % d'implémentation de référence, cohérence ADR-19/20/21.

## 5. Garde défensive intermédiaire (optionnelle, sans décision d'archi)

En attendant l'instruction, une **garde défensive** dans `kernel_wm_compose`
(`byte_h = min(h, 128)`) éliminerait la sur-lecture (sûr) au prix d'une
troncature visuelle des fenêtres hautes. À arbitrer séparément — non inclus
dans ce DRAFT car c'est un changement de comportement.

## 6. Critères de réouverture / d'instruction

Instruire (passer DRAFT → dossier ratifiable) dès que **l'un** est atteint :
1. Une app cible nécessite une fenêtre > 128 px de haut composée via backing
   store (déclencheur fonctionnel réel).
2. ADR-21 v0.2 (BPL configurable) est mise en chantier → l'option (b) devient
   chiffrable et à moitié implémentée.
3. Date plancher : revue au prochain jalon GUI.

## 7. Conséquences du statut « ouverte »

- **Positif** : pas de ratification compulsive (moratoire respecté) ; le fix
  BLIT v0.2 reste correct et mergé ; les apps petites fenêtres fonctionnent.
- **Négatif** : piège latent non corrigé — toute app ouvrant une fenêtre haute
  + `SYS_WIN_FLUSH` produira une corruption visuelle (sur-lecture de banques
  voisines). Mitigation : documenter la limite « fenêtre ≤ 128 px en compose »
  pour les apps v1 ; garde défensive §5 disponible si un cas réel survient.

## 8. Références

- CLAUDE.md §3 (décisions ouvertes), §10 (moratoire ADR)
- ADR-19 (SDRAM unifiée), ADR-20 (framebuffer XVGA 1024×768×4bpp), ADR-21 (GPU
  blitter, BPL configurable v0.2 annoncée)
- docs/MEMORY_MAP.md §6-8 (pools de banques)
- `OricOS/kernel/modules/wm.s` : `kernel_wm_compose` (§1136), `sys_win_create`
  (§3001), `kernel_gfx_window_base` (gfx.s §13)
- Audit GPU toolbox 2026-05-27, Finding B (Phosphoric CHANGELOG 1.22.87)
