# Revue critique senior — kernel OricOS (2026-06-10)

> Regard d'ingénieur senior kernel/65C816 sur l'état d'OricOS, demandé par
> Bénédicte Marty. Basé sur : audit factuel du code (14 724 lignes asm,
> exploration dédiée), connaissance directe des chantiers récents (ADR-32
> §10.9-§10.13, SP-3.p), et l'historique des incidents. Honnête, sans
> complaisance — c'est le but.

---

## 0. Verdict d'ensemble

OricOS est un kernel **remarquablement avancé pour son âge** (un mois de
sprints) : multitâche préemptif réel, GUI fenêtrée opérationnelle,
syscalls, FS, apps C cross-compilées. La **culture d'ingénierie** autour
(ADR, rouge→vert, golden model, invariants gardés mécaniquement) est très
au-dessus de ce qu'on voit dans les projets rétro — c'est elle qui a permis
de trouver et fermer des bugs que la plupart des kernels 8/16-bit amateurs
gardent à vie.

Mais le code lui-même porte les cicatrices de sa vitesse : **trois
décisions structurelles prises tôt coûtent maintenant à chaque sprint**,
et une partie de l'énergie récente (toute la saga ADR-32) a été dépensée à
compenser la première d'entre elles au lieu de la corriger. Les §1-§3
ci-dessous sont les vrais sujets ; le reste est de la dette normale.

---

## 1. Les trois problèmes structurels

### 1.1 Le window manager vit dans l'IRQ handler — le péché capital

`kernel_irq_handler` → `kernel_wm_mouse_step` → hit-test, focus, drag,
**redraw incrémental du desktop, compose, callbacks** — tout cela en
contexte IRQ. Les chiffres du projet lui-même : `redraw_drag` ≈ 53 % du
budget frame (ADR-28 §1.2bis), un drag = >10 000 cycles **interruptions
masquées de fait** pendant le rendu GPU sous `php/sei`.

Conséquences déjà payées :

- Toute la classe de courses ZP IRQ↔tâche (ADR-32) **n'existe que parce
  que** du code de politique (WM) tourne en contexte IRQ. L'enveloppe
  `IRQ_ZP_SAVE` (~2×750 cyc par IRQ souris) est un palliatif correct mais
  c'est un impôt permanent qui paie les intérêts de cette dette, pas le
  principal.
- La latence IRQ pire-cas est non bornée (dépend du nombre de fenêtres,
  d'icônes, de widgets à redessiner). Sur la cible HDL avec de vraies
  sources d'interruption (SD, série, vsync), c'est intenable.
- Chaque nouvelle feature WM (resize, taskbar, hotzones…) ré-élargit la
  surface IRQ et ré-exerce la classe.

Le projet le SAIT — ADR-28 (task_wm serveur) est la bonne direction, mais
elle est bloquée en « expérimental » depuis le 2026-06-01 par le bug
`task_wm_starve` **non résolu**. Mon point de vue : ce bug est le
verrou le plus important du projet. Tant qu'il n'est pas levé,
l'architecture cible (IRQ = top-half minimal qui poste des events, tâche
serveur = toute la politique) reste un vœu, et chaque sprint GUI aggrave
le monolithe IRQ.

### 1.2 Le layout mémoire bank 1 est artisanal — et il a déjà mordu quatre fois

**114 régions de données** définies par des `= $01xxxx` écrits à la main,
protégées par des `.assert` ajoutés après coup. Le linker n'alloue
rien : il ne voit que les segments de code. Historique des collisions
**réelles** de cette approche :

1. `SENTINEL/VERSION` ($5000) écrasés par la croissance de CODE
   (SP-3.o S.4c).
2. `TICK_COUNTER` = adresse du `rti` NMI ($5500) — chaque tick écrasait
   un opcode (fixé §10.13).
3. `WM_RH_TMP`/`TC_ENTRY_LO` partagent $32-$33 (collision potentielle
   resize-IRQ ↔ task_create, documentée, non fermée).
4. `GFX_ARG4_LO` vs `EVT_TMP` ($6E) — évitée de justesse, par
   commentaire.

Quatre incidents de la même famille = R6 : c'est un problème de classe,
pas une série d'accidents. **ld65 sait faire ce travail** : des segments
`BSS`-like (`type = bss`, `define = yes`) par module, les variables
déclarées en `.res` dedans, le linker garantit la non-superposition et
le `kernel.map` devient la source de vérité. Le chantier est mécanique
(gros sed + vérification), éliminerait la classe entière, et rendrait
les `.assert` manuels obsolètes.

À noter : le même défaut vient de mordre côté code — le piège
far-addressing GUICODE (labels de segment sans bank, SP-3.p) est un
cousin direct.

### 1.3 wm.s est un fourre-tout de 5 578 lignes

38 % du kernel dans un fichier qui contient : le WM proprement dit, la
souris, le curseur, la taskbar, les icônes, les hotzones, **les syscalls
GFX**, les dialogues modaux, `sys_yield`/`sys_sleep_ms` (du scheduler !),
`kernel_timer_tick`, et `kernel_gfx_fill_rect16` (du GPU). 272 `jsr`,
40+ exports. Quand on a passé la session ADR-32 à chercher « qui écrit ce
slot », le coût de cette non-séparation était très concret.

Découpage naturel, sans rien réécrire : `wm_core.s` (table, focus,
z-order), `wm_input.s` (souris, drag, hit-test), `wm_render.s` (redraw,
compose, curseur), `syscalls_gfx.s`, `dialog.s`, `taskbar.s`. Le timer et
yield/sleep retournent dans `sched.s`. Coût : une demi-journée de
mécanique, bénéfice permanent en navigabilité et en revue.

---

## 2. Problèmes sérieux (mais de second rang)

### 2.1 Gestion d'erreur majoritairement silencieuse

~10 `rts` nus sur chemin d'échec contre 2 panics. `kernel_free_bank`
droppe silencieusement si la free-list est pleine (fuite définitive de
banque), les rings droppent sans compteur, `WIDGET_TABLE` pleine = rts.
La règle R5 (« échec d'alloc kernel = BRUYANT ») existe mais n'est pas
appliquée rétroactivement. Recommandation : un compteur de drops par
ressource (4 octets de RAM chacun) + un panic optionnel en build debug —
les hangs silencieux du futur viendront de là.

### 2.2 Le boot de production exécute 500 lignes de tests

~34 % de `boot.s` : self-tests VRAM/GPU/TEXT non gatés qui écrivent le
framebuffer réel, fenêtres démo (« OricOS », « Editor »), icônes démo,
tâches A-F. C'était le bon choix en sprint 1-2 (validation visuelle
immédiate) ; aujourd'hui ça allonge le boot, pollue l'état initial que
TOUS les tests intègrent (les coordonnées en dur des tests pixel
dépendent des fenêtres démo !), et brouille la frontière kernel/démo.
Gating minimal : un flag `BOOT_SELFTEST` (posé par les harness de test,
absent en `--kernel` interactif), et à terme une « app » de démo séparée.

### 2.3 Trois rings d'événements en cohabitation

`KBD_RING` (legacy), `EVENT_RING` (unifié), `RAW_RING` (taskmode,
quasi-mort tant qu'ADR-28 est gelée). La « migration progressive » est
à mi-gué depuis trois semaines : les pushers alimentent en double
(kbd → KBD_RING **et** EVENT_RING), le routage clavier (waiter eligible)
vit sur le legacy. Chaque correction de course doit être pensée trois
fois. Finir la migration (tout sur EVENT_RING, KBD_RING en façade ou
supprimé) est moins coûteux que la maintenir.

### 2.4 L'ABI syscall aliasse $D0-$D9 pour trois usages

Bloc d'args COP, record d'événement poppé, et scratch de copie
(dialogues). C'est documenté, mais c'est exactement le genre d'aliasing
qui a produit l'incident « oricos_alert lit des résidus » cette session.
Une v2 de l'ABI devrait séparer : args entrants ($D0-$D7), record event
($E0-$E9 par ex. — la zone $E0 est libre), et interdire le scratch dans
le bloc COP. Petit chantier, à faire avant que le nombre d'apps grossisse.

### 2.5 GPU synchrone, busy-wait sous sei

Chaque primitive attend `GPU_STATUS_BUSY` sous interruptions masquées.
En émulation le GPU est instantané, donc invisible ; en HDL un blit
SDRAM réel prendra des microsecondes → latence IRQ massive, et le
busy-wait gaspille le CPU. L'ADR-21 prévoit « v0.2 async » — il faudra
y passer avant le port : file de commandes + IRQ de complétion, et le
modèle « compose en tâche » (ADR-28) devient obligatoire, pas optionnel.

### 2.6 Le scheduler stocke des priorités qu'il n'utilise pas

`TCB_PRIO` est rempli par `task_create` et jamais consommé par
`find_next` (round-robin pur). Soit l'assumer (retirer le champ — ADR-03
dit round-robin), soit l'implémenter ; l'état actuel est un piège pour
le lecteur qui croit avoir des priorités.

---

## 3. Risques spécifiques à la cible HDL/ULX3S

1. **Latence IRQ non bornée** (§1.1 + §2.5) — un kernel HDL avec vsync,
   SD et série a besoin d'un top-half borné. C'est le critère qui devrait
   décider du « quand » d'ADR-28, pas le confort.
2. **`GFX_BPL_SHADOW`** : un état GPU shadowé par le kernel parce que le
   GPU n'est pas lisible. En HDL, soit rendre le registre lisible (1 mux),
   soit accepter la fragilité pour toujours. À trancher dans le contrat
   HW-1.
3. **Le modèle cli-dans-COP** (syscalls interruptibles) impose à tout le
   kernel la discipline P1/P2/P3. Elle est désormais gardée par tests,
   mais chaque nouveau contributeur (ou session) peut l'ignorer. Pour le
   HDL, documenter ça dans le DAT comme invariant de plateforme.
4. **Pas d'assertions runtime** : les piles bank 0 bump-allouées par page
   débordent silencieusement dans la page voisine. Un canary par page de
   pile (1 octet, vérifié au switch) coûterait ~20 cycles par context
   switch et attraperait la classe.

---

## 4. Ce qui est bon — et qu'il ne faut PAS toucher

- **Le modèle déclaratif GenUI/DoDlgBox** (ADR-26) : choix juste, validé
  en pratique, économe en ABI. Continuer à cueillir GeoWorks brique par
  brique comme prévu.
- **La discipline golden model** : 4 gardes de classe rouge→vert dans
  `make tests` en une session, les harness versionnés (R9), oricrobot.
  C'est l'actif le plus précieux du projet — c'est lui qui rend les
  refactors §1 envisageables sans peur.
- **Le processus ADR + moratoire** : la ratification ADR-32 avec
  périmètre « non ratifié » explicite est exemplaire.
- **La doc inline** : les commentaires de wm.s/event.s qui tracent bugs,
  fixes et références croisées valent de l'or. (Ils compensent — un peu —
  le monolithe.)
- **sched.s/event.s/kbd.s** : petits, lisibles, corrects. La taille
  modeste de sched.s (378 L) pour un préemptif 16 tâches est une qualité.

---

## 5. Recommandations priorisées

| Prio | Chantier | Coût estimé | Ce que ça achète |
|---|---|---|---|
| **P0-a** | Résoudre `task_wm_starve` puis basculer `WM_TASKMODE` par défaut (finir ADR-28) | instruction du bug : 1-2 j ; bascule : sprint dédié | Ferme §1.1 — top-half minimal, latence bornée, l'enveloppe IRQ_ZP_SAVE devient retirée ou triviale |
| **P0-b** | Layout données par le linker (segments bss ld65, fin des `= $01xxxx` manuels) | 1-2 j mécanique + suite verte | Élimine la classe « collision layout » (4 incidents) |
| **P1-a** | Splitter wm.s (6 fichiers thématiques) | ½ j mécanique | Navigabilité, revues, baisse du coût des futures chasses |
| **P1-b** | Politique d'erreur : compteurs de drop + panic debug sur les ~10 chemins silencieux | ½ j | Les hangs silencieux de demain deviennent des diagnostics |
| **P1-c** | Gater les self-tests/démos du boot | ½ j (attention : recaler les tests pixel) | Boot propre, frontière kernel/démo |
| **P2-a** | Finir la migration EVENT_RING (retirer KBD_RING du chemin chaud) | 1 j | Une seule vérité événements |
| **P2-b** | ABI v2 : séparer args COP / record event / scratch | 1 j + maj SDK | Ferme la classe aliasing $D0 |
| **P2-c** | GPU async (file + IRQ complétion) — avant le port HDL | sprint dédié, avec ADR | Latence IRQ HDL viable |
| **P2-d** | Canaries de pile + trancher TCB_PRIO | ½ j | Robustesse, honnêteté du code |

Un mot d'ordre transversal : **arrêter d'ajouter des features GUI dans le
modèle actuel au-delà du sprint en cours**. Chaque widget ajouté pendant
que le WM vit en IRQ et que le layout est manuel augmente le coût des
deux chantiers P0. La suite GUI (item survolé, centrage, fontes
multiples) est du plaisir légitime — mais P0-a/P0-b d'abord la rendront
deux fois moins chère.

---

*Revue rédigée le 2026-06-10. Matériau : exploration factuelle dédiée
(14 724 L, métriques kernel.map, comptages d'erreurs), sessions ADR-32
§10.9-§10.13 et SP-3.p. Destinataire : Bénédicte Marty.*
