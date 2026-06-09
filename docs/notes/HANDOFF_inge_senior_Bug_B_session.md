# Handoff inge senior — session Bug B + réentrance IRQ↔syscall

> **TL;DR** : Bug B drag fragments résolu (commit OricOS `02934d2`).
> Bug architectural d'ordonnancement découvert et fermé par Opt-A (commit
> OricOS `5017990`). Opt-C investigation conclue : retirer `cli` de cop_handler
> casse la synergie cli/FORBID, **invalidé**. ZP top-half dédiée reste candidate
> pour ADR-32 §10. Tree clean, 24/24 suites Phosphoric vertes.

---

## 1. Ce qui a été livré (production)

### OricOS `02934d2` — Fix B v2 : désactiver coalescing MOVED en taskmode

Cause racine BUG_drag_v2_fragments : `kernel_raw_push_mouse` coalesce les
MOVED événements dans RAW_RING. En `WM_TASKMODE=$A5`, le delta dérivé
(`WHERE − WM_LAST`) du record poppé dépasse la largeur fenêtre, et l'erase
OLD seul de `kernel_wm_redraw_drag` ne couvre pas la trajectoire OLD→NEW.
**Fix** : gate `WM_TASKMODE=$A5` au début de `kernel_raw_push_mouse` qui
saute le bloc de coalescing. Chaque IRQ pousse son event individuel,
task_wm les drain un à un, delta petit, fragments disparus.

Coût : +10 octets CODE. EVENT_RING (côté app) garde son coalescing dans
`kernel_event_push_mouse` (UI app reçoit toujours groupé, pas de
régression). Risque accepté : RAW_RING (16 slots) peut overflow sous
burst — drop silencieux.

### OricOS `5017990` — Opt-A : sei ciblé sur 2 syscalls

Découverte collatérale durant l'investigation Bug B : la discipline ADR-03
(cop_handler `cli` → syscalls interruptibles) ouvrait une fenêtre de
réentrance IRQ top-half ↔ syscall sur les ZP scratch kernel. Test
position-shift (`+50 octets` de pad neutre dans `wm.s` cassait
`test_oricos_clock`) confirme la classe de bug en A/B clean :
- baseline : PASS
- +120 pad + `sei`/`cli` sur sys_gfx_fill_rect + sys_win_flush : PASS
- +120 pad sans protection : FAIL (clock task crashe mi-iter 2)

**Fix** : `sei` en tête de `sys_gfx_fill_rect` et `sys_win_flush`, `cli`
avant `rts`. Ces 2 sites sont ceux que ton test 2 SEI ciblé a prouvés
suffisants. Coût : ~300 cyc IRQ latency max par appel.
IRQ_CONFORMITE §5 inchangé (14987 cyc max, identique baseline).

---

## 2. Le mécanisme exact (instruction principale)

Le `cli` dans `cop_handler` n'est PAS qu'une discipline ADR-03 « syscalls
interruptibles » floue. C'est une **synergie deliberate** avec
`kernel_forbid`. Sous cette synergie, la séquence dans un syscall non-bloquant :

1. cop_handler entre → `cli` (I=0) + `jsr kernel_forbid` (FORBID=1)
2. IRQ peut fire pendant le syscall body
3. IRQ top-half exécute : kbd_poll → kbd_wake (marque task_e READY si
   waiter eligible), event_wake, T1 tick, sleep_tick, timer_tick
4. `do_switch` vérifie FORBID, voit ≠ 0, `jmp restore_and_return` —
   **PAS de switch**. La tâche en cours garde le CPU.
5. IRQ termine, retour au syscall body, syscall finit
6. `kernel_permit` (FORBID=0), `rti` au caller

Ce gating FORBID/do_switch est la **primitive d'ordonnancement v1** du
kernel. C'est ce qui fait que win_app garde le CPU entre `sys_win_flush`,
`sys_print_string`, `sys_read_char` même si une IRQ a marqué task_e READY
en route. win_app pop 'Z' du ring avant que task_e n'ait l'occasion de
s'exécuter.

**Mon Opt-C radicale** (retirer le `cli` de cop_handler, ajouter `cli`
explicite aux 6 syscalls bloquants) **casse cette synergie** :

1. Syscall non-bloquant tourne avec I=1 (hérité COP hw, plus de cli)
2. IRQ masquée pendant le body — `kbd_wake` ne s'exécute pas
3. Syscall finit, `kernel_permit` (FORBID=0), `rti` au caller (I=0)
4. **Maintenant** l'IRQ accumulée fire avec FORBID=0
5. `do_switch` préempte → task_e (récemment READY par kbd_wake) vole
   le CPU
6. task_e pop 'Z' avant que win_app n'atteigne sys_read_char

**Conséquence pratique** : `test_oricos_win_app` faillait avec
WM_COUNT=3 (win_app stuck en sys_read_char après que task_e ait
consommé sa touche). Trace A/B instrumentée dans
`docs/CR/CR_reentrance_irq_syscall_confirmed.md` §10.

**Conclusion** : Opt-C radicale est **invalidée** comme architecture.
Pas un effet de bord obscur scheduler/dispatch, c'est un vrai bug
logique d'ordonnancement.

---

## 3. Ce qui reste ouvert — ADR-32 §10

### « Vraie » Option C — ZP top-half IRQ dédiée

L'analyse ZP partagés tient :
- `WM_DP_TMP` ($20-$21) écrit par `kernel_wm_offset` (callee de
  `kernel_gfx_window_base`, lui-même callee de `sys_gfx_fill_rect`)
  ET potentiellement par `kernel_wm_mouse_step` quand IRQ a un event
  mouse (gated WM_TASKMODE=$00).
- D'autres slots probables non encore audités à fond : `EVT_TMP` ($6E),
  `WM_ARG_*` ($14-$1B), `GFX_*` ($70-$78), `SCHED_*` ($2C-$31).

L'approche correcte : **allouer une scratch ZP dédiée au top-half IRQ
($E0-$EF par exemple)** et déplacer les écritures IRQ-context vers
cette zone. **GARDER `cop_handler cli` + `kernel_forbid` intactes** —
la synergie est essentielle.

Routines à auditer pour leur usage ZP en contexte IRQ :
- `kernel_mouse_read`, `kernel_wm_mouse_step` (taskmode-dépendant)
- `kernel_event_push_mouse`, `kernel_event_push_key`,
  `kernel_event_push_verbatim`
- `kernel_kbd_poll`, `kernel_kbd_wake`, `kernel_event_wake`,
  `kernel_raw_wake`
- `kernel_sleep_tick`, `kernel_timer_tick`
- `do_switch`, `kernel_sched_find_next`, `kernel_tcb_ptr`,
  `kernel_block_switch`, `restore_and_return`

Une fois la ZP IRQ dédiée, le `sei` ciblé d'Opt-A devient redondant
(la collision ne peut plus arriver par construction). On peut le
retirer comme bonus.

### Test de non-régression livrable

`make test-position-shift` : sweeper de shifts CODE [16, 32, 50, 64, 80,
100, 128, 200] octets injectés dans `wm.s`. Verrouille la classe de
fragilité « réentrance IRQ↔syscall » en CI. Couplé à plusieurs périodes
T1 pour couvrir la phase IRQ.

Pas implémenté v1 — à bundler avec ADR-32 §10 ou en sprint séparé court.

---

## 4. Outils debug ajoutés (utiles pour futures investigations)

### `Phosphoric/tools/oricrobot.c`

Deux commandes ajoutées (commits `f883567` et `922e2a8`) :

**`cpu`** (sans argument) — dumpe l'état complet du CPU 65C816 :
```
cpu  PBR=01 PC=4DF8 DBR=08 D=0000 S=09EB A=5D10 X=0009 Y=0001 P=35 E=0
```
Permet de localiser où la machine est stuck dans un scénario headless.

**`key XX`** — push d'un caractère ASCII (a1 en hex) dans la FIFO KBD2.
Assert l'IRQ si IRQ_EN. Permet de scripter des scénarios kbd headless
(release de touche après run X cyc).

Exemple complet utilisé pour reproduire `test_oricos_win_app` en
oricrobot pendant l'investigation :
```
nostp
app 1EF50
run 800000      # attend que compose visible
key 5A          # push 'Z'
run 2000000     # attend que app exit
mem 15B72       # WM_COUNT — doit être 2 si succès
```

### Test instrumentation patterns

Pour debug d'un état stuck, l'approche productive a été d'ajouter
temporairement dans le test (avant l'assert qui échoue) un dump :
- TCB[N].STATE pour chaque pid en jeu
- KBD_WAITER / EVENT_WAITER / RAW_WAITER
- KBD_RING + HEAD + TAIL (contents bruts, pas que le count)
- WM_FOCUS + WM_OWNER[focus]
- cpu.P (pour voir le bit I), cpu.PC, cpu.irq
- Device state (kbd2.count, kbd2.head, kbd2.tail, kbd2.ctrl)

Le diff a été reverté du test avant commit. À ressortir si même
investigation à refaire.

---

## 5. Documents et commits

**Commits dans l'ordre temporel** :

| Repo | Hash | Description |
|---|---|---|
| OricOS | `02934d2` | Fix B v2 — coalescing MOVED off en taskmode |
| OricOS | `5017990` | Opt-A — sei sur 2 syscalls gfx/wm |
| OricOS | `fa29963` | doc CHANGELOG Opt-A |
| Phosphoric | `f883567` | oricrobot cmd `cpu` |
| Phosphoric | `922e2a8` | oricrobot cmd `key XX` |
| workspace | `c050b95` | mot expert hang clock position-dép |
| workspace | `b0bf1ec` | CR test 2 SEI confirmation A/B |
| workspace | `f659ce3` | CR §9 pivot Opt-C → Opt-A |
| workspace | `4c94614` | CHANGELOG Opt-A workspace |
| workspace | `0d79c25` | CR §10 investigation Opt-C bouclée |

**Documents dans le repo** :
- `docs/notes/MOT_EXPERT_drag_v2_fragments.md` — initial Bug B
- `docs/notes/MOT_EXPERT_clock_hang_position_dependent.md` — mot expert
  ~50 octets hang clock
- `docs/CR/CR_reentrance_irq_syscall_confirmed.md` — CR complet 10
  sections, analyse + A/B + pivot Opt-C→A + investigation finale
- `docs/notes/HANDOFF_inge_senior_Bug_B_session.md` — ce document

---

## 6. État du tree

- **OricOS** : 3 commits ahead origin/main, tree clean.
- **Phosphoric** : 2 commits ahead gitea/main (origin GitHub à jour ou
  pas selon push), tree clean.
- **Workspace** : 6 commits ahead origin/main, tree clean.
- Suite Phosphoric : **24/24 suites vertes**.
- Aucun pad, SEI test, ou debug résiduel.

---

## 7. Si tu veux rejouer / vérifier

```bash
# Vérifier Bug B fix (taskmode)
cd Phosphoric && make tests | grep -iE "fail"
# → vide (24/24 verts)

# Reproduire le hang clock (avant Opt-A)
cd ../OricOS
# Insérer dans kernel/modules/wm.s avant kernel_wm_redraw_drag :
#   .proc _pad
#           .res 120, $EA
#   .endproc
make
cd ../Phosphoric && make test-oricos-helloc | grep clock
# → PASS avec Opt-A en place, FAIL si tu revert Opt-A

# Reproduire le mécanisme Opt-C-radicale qui casse win_app
# Dans kernel/modules/handlers.s : commenter le `cli` ligne 83
# Dans kernel/modules/wm.s :
#   - retirer sei/cli de sys_gfx_fill_rect, sys_win_flush
#   - ajouter cli en tête de sys_read_char, sys_get_next_event,
#     sys_main_loop, sys_do_dlgbox, sys_alert, sys_sleep_ms
make && cd ../Phosphoric && make test-oricos-helloc
# → test_oricos_win_app FAIL : WM_COUNT=3 (task_e a volé 'Z')
```

---

## 8. Calibrage et confiance

- **Mécanisme cli/FORBID synergique** : tracé empiriquement par A/B
  instrumenté. Certitude haute.
- **Opt-A comme v1** : suite 24/24 verte, IRQ_CONFORMITE préservé.
  Certitude haute.
- **Opt-C-radicale invalide** : test_oricos_win_app reproductible,
  cause identifiée (perte de touche par préemption post-permit).
  Certitude haute.
- **Hypothèse résiduelle** : que TOUS les autres syscalls non-bloquants
  partagent le même risque (réentrance IRQ↔syscall sur d'autres ZP).
  Probable étant donné le partage WM_DP_TMP/GFX_*. **À auditer** au
  moment d'attaquer ADR-32 §10. Pas urgent (Opt-A couvre les 2 sites
  empiriquement déclencheurs).

---

*Rédigé 2026-06-09 — fin de session. Bug B livré, mécanisme
réentrance compris et fermé v1, ADR-32 §10 préparée. Push imminent
si tu valides.*
