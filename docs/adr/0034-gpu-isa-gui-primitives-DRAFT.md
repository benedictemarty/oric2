# ADR-34 (DRAFT) — ISA graphique stable : primitives GUI, GPU async, display-lists

- **Statut** : **ouverte — dossier d'instruction** (2026-06-10). DRAFT non
  ratifié (moratoire CLAUDE.md §10 : implémentation 0 %, ratification par
  étapes après code testé).
- **Décideurs** : Bénédicte Marty (arbitrage), Claude Code (instruction).
- **Origine** : double impulsion — (1) revue senior 2026-06-10 §2.5 (GPU
  synchrone busy-wait sous `sei` : intenable pour le port HDL) et §1.1
  (rendu fenêtres = 53 % du budget frame, brûlé par le CPU) ; (2) vision
  exprimée par Bénédicte : *« pour moi les fenêtres GUI sont des
  primitives du GPU »*, précisée par deux exigences directrices :
  **« si je change de carte, je ne change pas les primitives »**
  (stabilité du contrat) et **« l'objectif est la réduction de la charge
  IRQ »** (la finalité première — le rendu sort du chemin d'interruption ;
  la portabilité est la forme du contrat qui la sert durablement).
- **ADR liées** : ADR-21 (GPU blitter — la base à étendre), ADR-19 (SDRAM
  unifiée, accès GPU direct), ADR-02 (compositor), ADR-26 (GenUI/SpecUI —
  le look n'est pas figé), ADR-27 (SET_BPL, backing stores), ADR-28
  (rendu en tâche — verrou levé 2026-06-10), ADR-15 (isolation, parquée —
  réveillée partiellement par l'option C, cf. §6), ADR-33 (sprite HW).

---

## 0. L'exigence directrice : les primitives sont un CONTRAT, la carte est une implémentation

La phrase de Bénédicte définit le critère d'arbitrage de tout ce dossier :
le jeu de primitives graphiques de l'Oric 2 doit être une **ISA stable et
versionnée** — au même titre que le jeu d'instructions du 65C816 — telle
que l'on puisse remplacer l'implémentation (Phosphoric logiciel
aujourd'hui, ULX3S/ECP5 demain, une autre carte FPGA après-demain, voire
un ASIC) **sans changer ni le kernel ni les applications**.

Ce contrat existe déjà en germe : ADR-21 fixe l'interface I/O `$0340` et
9 opcodes (`NOP`, `CLEAR`, `FILL_RECT`, `BLIT`, `LINE`, `TEXT`,
`FILL_RECT16`, `TEXT16`, `SET_BPL`), et Phosphoric en est le golden model
comportemental. Ce dossier propose de **promouvoir ce germe au rang
d'ISA graphique formelle** (« GPU-ISA »), avec règles de compatibilité,
et de décider à quel NIVEAU d'abstraction placer les primitives GUI.

Le choix du niveau est tout l'enjeu :
- **Trop bas** (pixels, timings, stride implicite) → changer de carte
  casse le logiciel. C'est l'anti-modèle ULA Oric 1 : le software qui
  tape le matériel.
- **Trop haut** (une primitive `DRAW_WINDOW` au chrome figé) → changer de
  *look* exige de changer le silicium, et chaque évolution GUI devient
  une révision matérielle. Contraire à ADR-26 (SpecUI : le look est
  remplaçable sans recompiler).
- **Le bon niveau** : primitives géométriques génériques + composition +
  **listes de commandes** — stables à travers les cartes ET les looks.

## 1. Leçons des références (demandées : Apple IIgs, SymbOS)

### 1.1 Apple IIgs — notre jumeau CPU, et le contre-exemple de performance

Le IIgs est la machine la plus proche de l'Oric 2 : **même 65C816**
(2,8 MHz), GUI fenêtrée complète (Toolbox). Deux leçons opposées :

- **Leçon d'architecture (à imiter)** : la stratification QuickDraw II →
  Window Manager → Control Manager → Event Manager. Les applications ne
  connaissent ni le framebuffer ni le VGC : elles parlent à QuickDraw
  (primitives : rects, regions, pixmaps, fontes) et au Window Manager
  (politique). **L'API était le contrat** — les apps ont survécu aux
  évolutions d'écran. C'est exactement « changer de carte sans changer
  les primitives », réalisé côté logiciel. Notre GPU-ISA doit être le
  QuickDraw *matériel* de l'Oric 2 : même rôle de contrat, un étage plus
  bas.
- **Leçon de performance (à ne pas répéter)** : le IIgs n'a **aucun
  blitter** — QuickDraw II dessine tout au CPU. Résultat historique : la
  GUI la plus lente de sa génération, un Finder célèbre pour sa
  lourdeur, et un marché de cartes accélératrices (TransWarp, ZipGS) qui
  ne soignait que le symptôme. **Un 65C816 qui dessine ses fenêtres
  lui-même sature** — c'est démontré par l'histoire, et nos propres
  mesures le confirment (redraw ≈ 53 % du budget frame, ADR-28 §1.2bis).
  Le IIgs valide a posteriori l'ADR-21 : le blitter n'est pas un luxe,
  c'est la condition d'existence de la GUI.

### 1.2 SymbOS — le précédent « même OS, cartes différentes »

SymbOS (référence ADR-03/06) tourne sur CPC, MSX, PCW, Enterprise — des
machines aux systèmes vidéo **radicalement différents** :

- Sur **CPC** : pas d'accélération, le desktop manager dessine au Z80.
- Sur **MSX avec V9938/V9958** : le VDP possède un **command engine**
  matériel (HMMM/LMMM/HMMV : copies et fills rectangulaires exécutés par
  le VDP, **asynchrones**, le CPU consulte le bit `CE` — command
  executing — ou continue son travail). SymbOS route ses opérations de
  rendu vers ces commandes quand elles existent.

Deux leçons directes :

- **La portabilité par le driver** : SymbOS définit ses opérations de
  rendu à un niveau qui se mappe aussi bien sur du dessin CPU que sur le
  command engine V9938. C'est le précédent vivant de l'exigence
  directrice — même OS, primitives stables, implémentations par carte.
- **Le modèle async du V9938 est le modèle cible de notre v2** : des
  commandes rectangulaires postées, une exécution matérielle pendant que
  le CPU continue, un bit busy (`CE` ≙ notre `GPU_STATUS_BUSY`) — il ne
  nous manque que l'**IRQ de complétion** (le V9938 l'a aussi : IE1) et
  la **file** pour ne jamais attendre.
- Accessoirement : SymbOS rend depuis un **processus** desktop, pas
  depuis les interruptions — la validation historique d'ADR-28.

### 1.3 Clin d'œil Amiga (déjà référencé par le projet)

L'Amiga sépare Blitter (opérations mémoire génériques) et **Copper** (une
*display list* : le matériel exécute une liste d'instructions par frame).
Intuition (les fenêtres) est purement logiciel au-dessus. Personne, dans
toute cette classe de machines — IIgs, MSX, Amiga, ST — n'a jamais mis la
*fenêtre* dans le silicium : tous ont mis des **primitives génériques +
éventuellement des listes**, et la fenêtre est restée une structure de
données logicielle. C'est un signal historique fort pour le choix du
niveau d'abstraction.

## 2. État actuel (GPU-ISA v1, de facto)

| Élément | État |
|---|---|
| Interface | registres `$0340-$034F` : ARG1-4 (24/32-bit), OP, TRIGGER, STATUS |
| Opcodes | 9 ($00-$08), sémantique stable depuis ADR-21/27 |
| Exécution | **synchrone** — busy-wait `GPU_STATUS_BUSY` sous `php/sei` ; instantanée dans Phosphoric (pas de parallélisme réel modélisé) |
| Notification | aucune (ni IRQ complétion, ni vsync) |
| Découverte | aucune (pas de registre version/capabilities) |
| Fenêtre | orchestrée par wm.s : ~12-30 commandes par fenêtre, ~60-100 cyc CPU de setup chacune ; redraw complet ≈ 53 % du budget frame |
| Composition | BLIT des backing stores (compositor, ADR-02/27) + sprite HW (ADR-33) |

## 3. Options instruites

### Option A — Statu quo (v1 sync, orchestration CPU)
- **Coût** : 0.
- **Bénéfice** : aucun. Le busy-wait sous `sei` devient une latence IRQ
  de plusieurs µs sur HDL réel (revue §2.5/§3.1) ; le CPU reste le
  goulot du rendu. **Non viable au-delà du prototype.**

### Option B — GPU-ISA v2 : exécution asynchrone (file + IRQ de complétion)
- **Contenu** : FIFO de commandes (profondeur 8-16), le TRIGGER empile au
  lieu de bloquer ; `GPU_STATUS` expose busy + fifo-full ; nouvelle ligne
  IRQ « complétion / fifo-vide » (+ optionnel : IRQ vsync du scan-out,
  qui donne au système graphique son propre « timer », au sens de la
  question de Bénédicte du 2026-06-10) ; registre `GPU_CAPS/VERSION`
  (lecture seule) pour la découverte — c'est lui qui matérialise
  « changer de carte » : l'OS lit les capacités au boot.
- **Modèle de référence** : V9938 command engine + IE1.
- **Coût estimé** : Phosphoric ~2-3 j (modéliser la file + cycles
  d'exécution + IRQ — fin de l'exécution « instantanée ») ; kernel ~1-2 j
  (remplacer les polls par post-and-continue là où c'est sûr) ; HDL
  (estimation à valider en Phase HDL) : 1-2 BRAM pour la FIFO, une FSM
  d'exécution déjà nécessaire de toute façon.
- **Bénéfice** : latence IRQ bornée, CPU libéré pendant les blits,
  prérequis absolu du port ULX3S. **Sans regret quelle que soit la
  suite.**

> **Lecture des options sous l'angle « réduction de la charge IRQ »**
> (l'objectif premier) : A laisse ~53 % du budget frame en rendu CPU dont
> une partie en IRQ ; B borne la latence IRQ (plus de busy-wait sous sei)
> mais le CPU construit toujours chaque commande ; **C vide le chemin IRQ
> de tout rendu** — l'IRQ souris se réduit à poster un événement, le GPU
> rejoue les listes. C réalise l'objectif, B en est le prérequis.

### Option C — GPU-ISA v3 : display-lists (`EXEC_LIST`)
- **Contenu** : un opcode `EXEC_LIST addr24[, len]` : le GPU **fetch et
  exécute une liste de commandes depuis la SDRAM** (format = la même
  encodage que les registres : op + args, terminateur `END`). Une fenêtre
  devient *sa* display-list : le kernel la **construit une fois** (à la
  création/au resize/au changement de contenu), et le redraw entier —
  drag compris — se réduit à : poster `EXEC_LIST` par fenêtre dans le
  z-order, puis composer. **C'est la vision « les fenêtres sont des
  primitives du GPU », réalisée sans figer le look** : le contenu de la
  liste reste défini par le logiciel (SpecUI peut changer le chrome sans
  toucher au silicium), mais son *exécution* est matérielle.
- **Modèle de référence** : Copper Amiga (esprit), command buffers des
  GPU modernes (le concept a 40 ans d'avenir devant lui — gage de
  stabilité du contrat).
- **Coût estimé** : Phosphoric ~2-3 j au-dessus de B (fetch unit
  modélisée) ; kernel : refactor de `kernel_wm_redraw`/`_wm_draw_*` en
  *constructeurs de listes* (~3-5 j, migration progressive possible — les
  helpers actuels restent valides, une fenêtre peut être « immediate
  mode » ou « retained list ») ; HDL : FSM de fetch SDRAM (+~200-400 LUT
  estimés, à valider).
- **Bénéfice** : le coût CPU du redraw passe de ~53 % du budget frame à
  ~une commande par fenêtre (~100 cyc) ; l'IRQ souris n'a plus AUCUN
  rendu à faire (achève P0-a par construction) ; le drag devient fluide
  par nature.
- **Risque propre** : le GPU exécute des listes en SDRAM **sans MMU** —
  une liste corrompue écrit n'importe où. Mitigations v1 : terminateur
  obligatoire + borne `len`, listes construites exclusivement par le
  kernel (OS de confiance, ADR-04). Noté comme **critère de réouverture
  supplémentaire d'ADR-15** (isolation v2).

### Option D — Primitive `DRAW_WINDOW` (chrome câblé) — instruite pour être écartée
- Le chrome (cadre, titlebar, boutons, ombres demain) serait figé dans le
  matériel : chaque évolution de look = révision HDL, contraire à ADR-26
  et à l'exigence directrice elle-même (le « changement de carte »
  inclut l'évolution de la même carte). Aucun précédent dans la classe
  (cf. §1.3). **Recommandation : écarter.**

## 4. Recommandation senior tracée

**B puis C, en deux étapes ratifiables séparément** (leçon ADR-30) :

1. **GPU-ISA v2 (option B)** d'abord — async + IRQ complétion + vsync +
   `GPU_CAPS`. Prérequis HDL sans regret, et la FIFO est l'infrastructure
   du fetch de listes.
2. **GPU-ISA v3 (option C)** ensuite — `EXEC_LIST`, et la migration du
   WM en constructeur de listes. C'est l'étape qui réalise la vision
   « fenêtres = primitives GPU » au bon niveau d'abstraction.

**Règles de contrat à graver dès la ratification de B** (le cœur de
« changer de carte sans changer les primitives ») :
- La sémantique d'un opcode publié ne change **jamais** ; les évolutions
  sont **additives** (nouveaux opcodes, nouveaux bits CAPS).
- `GPU_CAPS` est obligatoire sur toute implémentation ; l'OS s'adapte
  aux capacités, jamais à l'identité de la carte.
- Phosphoric reste le **golden model normatif** : une carte est conforme
  si elle passe la suite de tests GPU de Phosphoric (les tests device
  deviennent la suite de conformance de l'ISA).
- Le document d'ISA (opcodes, encodages, registres, sémantique mémoire)
  devient une annexe du DAT (track A, IEEE 42010).

## 5. Critères de ratification (moratoire §10)

- **Étape B** : ratifiable quand la file+IRQ est implémentée dans
  Phosphoric avec tests device (rouge→vert sur « le CPU continue pendant
  un blit long ») ET qu'un chemin kernel (ex. `kernel_wm_compose`) tourne
  en post-and-continue mesuré.

### 5bis. ÉTAPE B LIVRÉE (2026-06-10) — critères remplis, ratification à arbitrer

- **Device (Phosphoric)** : FIFO 16 (snapshot des latchs au TRIGGER),
  mode timé opt-in (`gpu_set_timed`/`gpu_tick` — les harness legacy sont
  inchangés, l'émulateur vivant tourne timé), modèle de durée (8 cyc +
  octets/4 — non contractuel), IRQ de complétion (`IRQF_GPU`, assertée
  FIFO→vide, ack write-1-to-clear STATUS, enable INT_CTRL),
  **CAPS/VERSION en lecture TRIGGER** (= $32 : FIFO+IRQ, v2 ; additif —
  v1 lisait 0), QFULL/OVF sticky. **4 tests device, rouge-check
  démontré** (exec immédiate simulée → 3 FAIL ; restaurée → 25/25).
- **Kernel (OricOS)** : `GPU_CAPS_KERNEL` ($019094) lu au boot ;
  `kernel_gfx_blit_post` (wait seulement si QFULL) + `kernel_gfx_drain`
  (barrière pour les futurs lecteurs CPU) ; `kernel_wm_compose` routé
  par capacités (carte v1 → chemin sync intact), **sans drain final**
  (audit des 3 callers : aucun ne relit la SDRAM — l'affichage rattrape
  à la frame suivante, sémantique async).
- **Gain mesuré** (`test-oricos-gpu-async`, dans `make tests`) :
  compose **22 534 → 10 591 cycles CPU (−52 %)** vs carte v1 simulée
  (caps pokés à 0) ; transparence fonctionnelle prouvée (même
  framebuffer composé) ; découverte caps prouvée.
- **Leçon collatérale (pour l'étape C et ADR-28)** : le recalibrage du
  test `evt-push-atomic` a montré qu'à 400 Hz d'événements souris, le
  chemin IRQ legacy (mouse_step + enveloppe P1 ≈ 5 000 cyc/event)
  **sature le CPU** — la marge de task_wm dépendait de ±30 octets de
  position de code. Cadence de test ramenée à 125 Hz (réaliste). C'est
  la mesure la plus directe à ce jour du coût du rendu-en-IRQ.
- **Ratification de l'étape B** : **RATIFIÉE 2026-06-10 par Bénédicte
  Marty** (« c'est validé, ratifie B ») après validation interactive.
  Conformité moratoire : dossier d'instruction ✓ (§0-§4), implémentation
  100 % testée ✓ (rouge-check device, −52 % mesuré, suite verte),
  cohérence ADR ✓ (étend ADR-21 de manière additive, sert ADR-27/28).
  **Les règles de contrat du §4 sont GRAVÉES à compter de cette
  ratification** : sémantique d'opcode immuable, évolutions additives,
  GPU_CAPS obligatoire, Phosphoric = suite de conformance de l'ISA.
- **Étape C** : ratifiable quand `EXEC_LIST` est implémentée + une
  fenêtre réelle rendue par liste avec gain mesuré sur `redraw_drag`
  (objectif : −90 % de cycles CPU) + validation interactive utilisateur
  (drag fluide).

### 5ter. ÉTAPE C lancée — C1 LIVRÉE (2026-06-10), C2 = sprint suivant

**C1 (livrée)** :
- **Device** : `GPU_OP_EXEC_LIST` ($09) — le GPU fetch et exécute une
  display-list en SDRAM (entrées de 13 octets [op][arg1-4×3], terminateur
  $FF). Gardes : récursion interdite (ERR), borne 64 entrées (liste sans
  terminateur → ERR, le GPU ne court pas dans la SDRAM). Coût modélisé =
  fetch + somme des entrées. **ISA v3** : CAPS_BYTE = $73 (FIFO + IRQ +
  LIST, version 3 — extension additive, le bit FIFO v2 reste lisible par
  un OS v2). 2 tests device (liste 3 commandes avec SET_BPL intra-liste :
  ordre prouvé ; gardes récursion/borne).
- **Premier consommateur kernel** : `kernel_tk_label_prop` (textes des
  dialogues) — si cap LIST : construit la display-list des N entrées
  TEXT16 en SDRAM (positions proportionnelles précalculées, écriture via
  VRAM port auto-inc) et poste UNE commande EXEC_LIST ; drain en TÊTE
  (protège les scratch d'une liste encore en vol), pas de drain final.
  Carte sans LIST : boucle sync v1 intacte (routage par capacités).
- **Gains mesurés** (`test-oricos-gpu-async`, make tests) : label
  « Save changes? » (13 chars) : **5 549 → 2 854 cycles CPU (−48 %)**
  vs carte FIFO-sans-LIST simulée. Suite complète verte.

**C2 (sprint suivant — le cœur de la vision)** : kernel_wm_redraw /
_wm_draw_windows / redraw_drag en CONSTRUCTEURS de listes (une fenêtre =
sa display-list, reconstruite à la création/resize/contenu, REJOUÉE au
drag). Critère de ratification C inchangé : −90 % cycles CPU sur
redraw_drag + validation interactive drag fluide. Pré-requis techniques
en place (EXEC_LIST, caps, pattern constructeur démontré sur label_prop).

## 6. Impacts croisés

- **ADR-28** : synergie totale — task_wm poste des `EXEC_LIST` au lieu de
  dessiner ; le débat « rendu en IRQ vs tâche » devient sans objet pour
  le gros du coût.
- **ADR-15 (parquée)** : l'option C ajoute un critère de réouverture
  (GPU consommateur de pointeurs SDRAM).
- **ADR-33** : inchangée (le sprite curseur reste hors listes).
- **DAT/track A** : le document GPU-ISA est le premier livrable de
  contrat HW formel du projet — un gabarit pour les contrats suivants
  (KBD2, MOU2, VRAM).

---

*Dossier ouvert le 2026-06-10 à la demande de Bénédicte Marty (vision
« fenêtres = primitives GPU » + exigence « changer de carte sans changer
les primitives »), instruit avec les références demandées (Apple IIgs,
SymbOS) + Amiga. Implémentation 0 % — non ratifiable en l'état
(moratoire). Première étape proposée : GPU-ISA v2 (option B).*
