# ADR-25 — Modèle de concurrence du kernel OricOS

- **Statut** : **ratifiée 2026-05-25** (option (b) Exec-classique). L'humain a
  approuvé le modèle (« go pour Exec-classique », 2026-05-25) ; les 3 conditions
  du moratoire §10 sont remplies (cf. section dédiée). Implémentation de référence
  OS-2.g v2.b validée par tests : g.6 (Forbid/Permit + atomicité syscall) + g.5
  (block/wake + SYS_READ_CHAR bloquant), 563 tests verts. Généralisations
  restantes (masque de signaux multi-bits, primitives `Disable`/`Enable`
  formelles) = polish v2.b, sans remettre en cause le modèle.
- **Date** : 2026-05-25
- **Décideurs** : bmarty (humain — approuve Exec-classique, 2026-05-25), Claude Code
- **Jalon déclencheur** : **OS-2.g v2** (cycle de vie des tâches :
  `task_create`/`destroy`/`block`/`wake`) — débloque le multitâche préemptif
  réel promis par ADR-03.
- **ADR liées** : **ADR-03** (multitâche préemptif strict — ce dossier en
  *précise le contrat d'atomicité*, jusqu'ici implicite), **ADR-14** (table TCB
  16 slots + bitmap + état `BLOCKED` — fournit le substrat), **ADR-16** (driver
  model — le « wakeup userland » des classes IRQ-driven *est* le `task_wake`),
  **ADR-04** (OS de confiance, sans MMU — justifie l'acceptabilité des
  primitives à gel global).

---

## Contexte

Sur un **mono-cœur 65C816 sans primitive atomique** (hors masquage d'IRQ), deux
besoins du kernel tirent en sens opposés :

1. **Atomicité** — l'état kernel partagé (ZP scratch globale, rings, curseur)
   doit être protégé contre l'entrelacement (IRQ + context-switch).
2. **Blocage** — un syscall qui attend un évènement (`SYS_READ_CHAR`) doit
   *laisser les IRQ tourner* pour que l'évènement (touche KBD2) arrive.

Masquer l'IRQ pendant tout un syscall donne l'atomicité mais **interdit le
blocage** (deadlock) ; ne rien masquer permet le blocage mais expose des
**races** sur l'état partagé. C'est le dilemme central, encore non arbitré
formellement dans OricOS.

### Chiffrage — état actuel du kernel

| Élément | Réalité (mai 2026) |
|---|---|
| Scheduler | **figé 2 tâches** (`handlers.s` `do_switch`, `CUR∈{1,2}`, `NEXT=3-CUR`) |
| TCB exploité | **6/20 octets** (`saved_S`, `STATE`) ; PID/PRIO/PARENT/PC/PB/DB/FLAGS/NAME morts |
| `TCB_BITMAP` | initialisé, **jamais scanné** (pas de `task_create`) |
| `TASK_STATE_BLOCKED` | défini, **jamais utilisé** |
| `SYS_EXIT` | `stp` → **halte toute la machine** |
| `SYS_YIELD` | `rts` (no-op) |
| `sys_read_char` | spin-poll + `wai` ; débloqué via **stopgaps empilés** : pré-injection clavier → `cli` dans le handler COP → section critique dans `ring_pop` |
| ZP scratch | **globale partagée** (`DP_PTR`, `DP_PCPTR`, `DP_TMP`, `WM_ARG_*`, `GFX_ARG*`, `DP_KBD_TMP`) — pas de copie par tâche |

### Dettes résolues par ce modèle

- **#2** — réentrance de la ZP scratch sous préemption (latent : se déclenche
  dès que ≥ 2 tâches émettent des syscalls).
- **Deadlock `SYS_READ_CHAR`** — spin masqué dans le kernel.
- **#5** — écart ADR-14 (16 TCB round-robin priorité) vs scheduler figé 2-tâches.
- **`SYS_EXIT`=STP** — une app qui sort fige la machine.

---

## Décision drivers

- **ADR-03 (préemptif) est non négociable** → le modèle GS/OS (mono-tâche +
  polling d'évènements en espace utilisateur, qui *esquive* le dilemme) est
  **exclu**.
- **ADR-04 (OS de confiance, pas de MMU)** → on tolère qu'une tâche
  non-coopérative puisse figer le système (aucune protection — assumé en v1/v2).
- **Coût** mémoire/CPU minimal (ZP rare, pas de pile profonde).
- Doit s'articuler avec le **block/wake** piloté par les drivers (ADR-16).

---

## Options envisagées

### (a) Statu quo — `cli` global + sections critiques ad hoc
- ✅ Déjà en place ; le deadlock est levé (`cli`), une race patchée (`ring_pop`).
- ❌ La réentrance ZP (#2) **n'est pas traitée** ; chaque ressource partagée
  exige une section critique manuelle qu'on **oublie** (on en a déjà ajouté une
  en urgence). Rustines non systématiques, fragiles.
- **Écartée** — ne tient pas à l'échelle de N tâches.

### (b) Exec-classique — `Forbid`/`Disable` + signaux `Wait`/`Signal` *(RECOMMANDÉE)*
Modèle d'AmigaOS Exec (microkernel préemptif **sans MMU**) et de **SymbOS**
(son incarnation Z80, déjà référence d'ADR-03/06) :

- **`Forbid`/`Permit`** (compteur de nesting global) : suspend le
  **context-switch** ; **les IRQ restent vivantes**. Enveloppe chaque syscall →
  atomicité tâche-contre-tâche **sans masquer les IRQ** (donc pas de deadlock).
- **`Disable`/`Enable`** : masque l'IRQ pour **une poignée d'instructions
  seulement** — réservé aux micro-RMW partagés avec un handler (ex. le
  `KBD_RING_COUNT`). *(Réf. : un `Disable` prolongé casse l'I/O temps-réel —
  Exec borne à ~250 µs ; principe = « jamais une boucle ».)*
- **Signaux** : **1 octet par TCB** (8 bits). `task_wait(mask)` → `BLOCKED` +
  yield ; le driver fait `task_signal(pid, bit)`. **`Wait()` lève
  automatiquement le `Forbid`/`Disable`** le temps du blocage, puis le restaure
  au réveil → résout proprement « atomicité + blocage ».
- **Partition ZP** : scratch des handlers d'IRQ **disjoint** de celui des
  syscalls (aujourd'hui `DP_KBD_TMP`/`WM_ARG_*` sont partagés → source des
  races). `Forbid` suffit alors pour le reste.

**Coût chiffré** : ~1 octet signaux/TCB + 2 compteurs de nesting
(`forbid_count`, `disable_count`) ; refactor ZP (déplacer le scratch IRQ) ;
réécriture de `do_switch` (saut des `BLOCKED`) et de `sys_read_char`. Pas de
structure lourde, pas de copie de contexte étendue.

- ✅ Résout **les 4 dettes** ; mappe pile-poil ADR-03/04/06/14/16 ; éprouvé
  (Amiga 1985, SymbOS 2006) ; implémentable sur 65816.
- ❌ Une tâche qui `Forbid` sans `Permit` fige le système (défaut connu d'Exec)
  — **acceptable sous ADR-04** (OS de confiance). Inversion de priorité possible
  (cf. Conséquences).

### (c) Mutexes / sémaphores
État de l'art moderne (AmigaOS récent **déprécie** `Forbid`/`Disable` au profit
des mutexes ; RTOS Cortex-M).
- ✅ Pas de gel global : seules les tâches en conflit bloquent.
- ❌ Amène l'**inversion de priorité** → impose l'héritage de priorité
  (machinerie supplémentaire) ; surdimensionné pour 2-3 tâches v2.
- **Reportée v3** (critères de réouverture ci-dessous).

### (d) Message-passing intégral (SymbOS complet)
- ✅ Le plus composable pour une GUI ; isole totalement l'état (pas de scratch
  partagée). SymbOS le justifie *« pour éviter les problèmes avec la pile, les
  variables globales et les ressources partagées »* — notre problème exact.
- ❌ Ports + mailboxes + copies de messages : lourd à amorcer. Se pose **au
  dessus** des signaux (b), plus tard.
- **Reportée v3+** (IPC GUI).

### (e) GS/OS-like — coopératif, pas de blocage kernel
- ❌ **Contredit ADR-03** (suppose le mono-tâche). **Écartée.**

---

## Décision proposée (à ratifier)

**Adopter l'option (b) Exec-classique pour OS-2.g v2.** Primitives :

```
task_create(entry, prio, bank) → pid     ; scan bitmap, init TCB, forge pile
task_destroy(pid) / SYS_EXIT              ; STATE=DEAD, free bank+pile, reschedule
task_wait(sigmask) → sigs                 ; BLOCKED + yield (lève Forbid)
task_signal(pid, bit)                     ; arme un signal, réveille si attendu
Forbid()/Permit()                         ; nesting, suspend le switch
Disable()/Enable()                        ; nesting, masque IRQ (micro-RMW only)
```

Mapping OS-2.g : **g.5** = `task_wait`/`task_signal` (signaux) ; **g.6** =
`Forbid`/`Disable` + partition ZP ; `sys_read_char` se bloque sur le signal
clavier, réveillé par l'IRQ KBD2.

---

## Conséquences

**Positives** — résout #2, deadlock, #5, `SYS_EXIT`. **Supersede** le `cli`
ajouté dans le handler COP (redevient `Forbid` ; `Disable` ponctuel pour le
ring) et **déprécie** la pré-injection clavier. Trace le contrat d'atomicité
manquant d'ADR-03.

**Négatives / risques**
- Tâche non-coopérative (`Forbid`/`Disable` sans relâche) → gel. Acceptable
  ADR-04 ; à revoir si apps non-trusted (cf. ADR-15).
- **Inversion de priorité** (tâche haute prio attend une basse en section
  critique) — connue ; mitigation (héritage de prio) reportée v3.
- `Disable` doit rester **bref** (jamais de boucle) — discipline à documenter.

**Sur les ADR existantes**
- **Révise ADR-03** : ajoute le contrat « syscalls sous `Forbid`, blocage via
  `Wait`, `Disable` borné aux RMW ».
- **Précise ADR-16** : le « wakeup userland » = `task_signal` depuis le handler.
- Compatible ADR-04/14 sans changement.

---

## Conformité moratoire (CLAUDE.md §10)

| Condition | Statut |
|---|---|
| 1 — dossier d'instruction écrit (contexte chiffré, ≥2 alternatives chiffrées, reco) | ✅ ce document |
| 2 — ≥ 50 % implémentation testée **OU** jalon dur ≤ 4 sem | ✅ **g.6 (Forbid/atomicité) + g.5 (block/wake) implémentés et testés** (563 verts) |
| 3 — cohérence ADR §2 (pas de contradiction non-explicite) | ✅ implémente ADR-03/14, précise ADR-16, compatible ADR-04 |

→ **Ratifiée 2026-05-25.** Les 3 conditions sont remplies et l'humain a approuvé
l'option (b). Le `cli` du handler COP est désormais encadré par `Forbid` ; le
spin `WAI` de `SYS_READ_CHAR` n'est conservé que pour le fallback boot-context
(hello_c). Restent en polish v2.b : signaux multi-bits génériques (vs `KBD_WAITER`
dégénéré) et primitives `Disable`/`Enable` formelles (vs `sei/php/plp` ad hoc).

## Critères de réouverture vers (c)/(d) — v3

- Apps **non-trusted** ratifiées (cf. ADR-15) → le gel global devient
  inacceptable → mutexes/sémaphores.
- **Inversion de priorité** observée en pratique → héritage de priorité.
- **> 4-6 tâches** concurrentes ou IPC GUI riche → message-passing (d).

---

## Sources

- AmigaOS Exec — exclusion & synchronisation (`Forbid`/`Disable`/`Wait`/`Signal`,
  danger du `Disable` prolongé, dépréciation au profit des mutexes) :
  <https://wiki.amigaos.net/wiki/Exec_Tasks>,
  <https://wiki.amigaos.net/wiki/Exec_Signals>,
  <http://amigadev.elowar.com/read/ADCD_2.1/Includes_and_Autodocs_2._guide/node0353.html> (Forbid),
  <http://amigadev.elowar.com/read/ADCD_2.1/Includes_and_Autodocs_3._guide/node0203.html> (Disable),
  <https://en.wikipedia.org/wiki/Exec_(Amiga)>.
- SymbOS — microkernel préemptif Z80, message-passing motivé par
  l'évitement des ressources partagées : <https://en.wikipedia.org/wiki/SymbOS>,
  <https://www.cpcwiki.eu/index.php/SymbOS>.
- État de l'art RTOS (masquage par seuil `BASEPRI`, sections critiques brèves,
  inversion de priorité) :
  <https://mcuoneclipse.com/2016/08/28/arm-cortex-m-interrupts-and-freertos-part-3/>,
  <https://en.wikipedia.org/wiki/Real-time_operating_system>.
