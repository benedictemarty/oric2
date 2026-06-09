# Mot pour l'expert — hang `test_oricos_clock` position-dépendant (~50 octets CODE shift)

> HEAD : OricOS `02934d2` (Fix B v2 livré), Phosphoric `f883567` (oricrobot
> `cpu` cmd ajoutée). Cause racine non isolée. Document de transmission.

---

## 1. Contexte — pourquoi je suis tombé dessus

Travail en cours : Fix B `BUG_drag_v2_fragments` (fragments visuels en
`WM_TASKMODE=$A5` sous drag rapide). Approche v1 = **rect englobant**
dans `kernel_wm_redraw_drag` (union OLD ∪ NEW), recommandée par mot expert
précédent.

Implémentation v1 livrée en local : +40 octets unconditionnel, ou +65 octets
gaté sur `WM_TASKMODE=$A5`. Les deux versions cassaient `test_oricos_clock`
au-delà d'un certain seuil de taille — pas par budget cycles, par **hang**
réel.

Pivot v2 livré : désactiver le **coalescing MOVED** dans `kernel_raw_push_mouse`
en taskmode (+10 octets, sous le seuil, IRQ_CONFORMITE préservé). Bug B
résolu côté symptôme. Mais le **seuil 50 octets** lui-même reste ouvert.

---

## 2. Symptôme isolé

**Reproducteur minimal, indépendant du contenu** :

```asm
; Inséré n'importe où dans CODE (testé avant kernel_wm_redraw_drag)
.proc _pad
        .res 50, $EA      ; 50 octets de NOP
.endproc
```

- `make` OricOS, `make test-oricos-helloc` côté Phosphoric.
- Résultat : `test_oricos_clock FAIL — "clock: done" absent (start=1 tick=1 cyc=900001)`.

Seuil net : **+49 octets PASS, +50 octets FAIL** (mesuré par dichotomie).

Symptôme côté screen RAM ($BB80) après 5 M cyc en oricrobot :

```
clock: start
clock: tick                      ← itération 1 complète
c                                ← itération 2, print stuck après 1 char
[reste vide]
```

Le budget cycles n'est pas le problème : test à 2 M cyc → toujours stuck à
`tick=1`. **C'est un hang, pas un slowdown.**

---

## 3. Ce que j'ai observé en investiguant

### a) La tâche clock meurt

Snapshots TCB pid 8 (clock task) via `mem $15C8D` :
- t=100k cyc : STATE=$01 READY (en attente de scheduling)
- t=200k cyc : STATE=$02 RUNNING ou READY (alterne)
- t=300k cyc : STATE=$00 **DEAD**

Idem en baseline (sans pad) : DEAD à 300k cyc. Différence : en baseline,
clock a terminé proprement (5 prints complets dont "clock: done") avant
de SYS_EXIT. En +50 pad, elle est morte après 1 iteration complète + 1
char de la 2ᵉ.

### b) Scheduler en boucle réduite après la mort

PC bouncing entre `task_a_entry` ($1407), `task_b_entry` ($1412),
`task_c_entry` ($141C) — les 3 tâches démo busy-loop. Les pids 5/7
restent BLOCKED (task_e/task_f attendent kbd/sleep). Idle (pid 6) skipped.
**Aucune tâche app n'est plus runnable** → clock app perdue.

`TICK_COUNTER` continue d'avancer normalement ($56 → $C4 → ...). Donc l'IRQ
T1 fait son travail. Le bug n'est pas dans l'IRQ — il est dans **la chaîne
qui mène à la mort de la tâche clock**.

### c) Log ring identique baseline vs pad

```
$190E0 = $02   ; LOG_WARN
$190E1 = $02   ; ERR_BAD_SYSCALL
```

Présent dans les **deux** cas → pas la cause directe (probablement une tâche
démo qui tape un mauvais num syscall, pré-existant).

### d) Layout binaire propre

- CODE end +50 pad = $54D1 (plafond $5500, marge 47 octets).
- NMI_HANDLER $5500 (1 octet), IRQ_HANDLER $5600, COP_HANDLER $5700, SYSCALL_TABLE $5750, CHARSET $5800, BUNDLES $7000 — tous **fixes** (start= dans kernel.cfg).
- `bundle_clock` à $818D fixe, jamais shifté.
- Pas d'overflow d'assertion, `make audit-smart` propre.

Seul CODE shifte. Toutes les références JSR/JMP/SYSCALL_TABLE.word sont
résolues par ld65 à link-time → pas un problème d'adresse mal calculée.

### e) Page crossing — non, je ne pense pas

Sur 65C816, le seul effet de layout sur les cycles est :
- `lda abs,X/Y` : +1 cyc si (base+index) cross page
- Quelques autres modes indexés idem

Ce serait +1 cyc par instruction concernée, donc max ~quelques k cycles
par tick — pas un hang. Et un hang n'a pas de relation continue avec
"+1 cycle quelque part".

Je suspecte plutôt **un effet d'alignement plus subtil**, p.ex. un
opcode/operand qui rentre dans une page de manière différente et déclenche
un bug d'émulateur cycle-précis, OU une corruption mémoire kernel
déclenchée par un cas d'usage inexploré.

---

## 4. Ce que je n'ai PAS pu déterminer

- **PC exact au moment où la tâche clock crashe** : entre 230k et 235k cyc,
  PC passe de COP_HANDLER ($5702) à `kernel_event_wait`/`kernel_sched_find_next`
  puis à `task_b_entry`. La transition est rapide. J'aurais besoin d'un
  watchpoint sur `TCB_TABLE_BASE+7*20+1` (write de pid 8 STATE) pour
  capturer **qui** écrit DEAD et avec **quelle pile**.
- **Le syscall qui crashe** : c'est probablement SYS_GFX_FILL_RECT ou
  SYS_WIN_FLUSH (entre print "clock: tick" iter 1 et print iter 2 il y a
  3 syscalls + le while-yield-loop). Mais je n'ai pas trace fine.
- **Si c'est kernel ou émulateur** (`CLAUDE.md §5septies`) : je n'ai pas
  pu construire l'argument décisif. Le « le bug disparaît SANS modifier
  le kernel » serait probant mais demanderait de modifier l'émulateur,
  ce que je n'ai pas tenté.

---

## 5. Hypothèses ouvertes

Par ordre décroissant de plausibilité, à mon avis :

### H1 — Corruption stack côté clock task (kernel)
La tâche clock a une page de pile dérivée du pid (pid 8 → page $09 il me
semble, à vérifier). Si une routine kernel sous IRQ déborde et écrit dans
la pile de la tâche interrompue, ça corrompt la frame de retour → RTI
saute n'importe où → tâche crashe.
**Test** : watchpoint sur $0900-$09FF pendant les iter 1→2.

### H2 — Bug cycle-précis dans l'émulateur 65C816
Une instruction RMW mémoire en mode M=16, ou MVN/MVP, ou abs,X page-cross,
qui aurait un comportement faux dans certaines positions/contextes. Le
`5quater` warning de CLAUDE.md mentionne des sites RMW-mem M=16 récemment
audités. Si une nouvelle config alignement déclenche un site non couvert…
**Test** : trace CPU dans la fenêtre 230k-300k cyc, comparer instructions
exécutées baseline vs +50 pad.

### H3 — Self-modifying code sensible
`app_exec_call` (`fat.s` ~723) patche son bank-byte dynamiquement. Si
`app_exec_jsl_bank` se retrouve à une adresse particulière, le patch
pourrait foirer (impossible en théorie : sta [$20] résout correctement)
mais difficile à exclure sans trace.
**Test** : peek `app_exec_call` 4 octets avant/après pour voir si patché OK.

### H4 — Effet d'allocation pile dynamique (`STACK_NEXT_PAGE`)
Si une tâche démo (task_e, task_f) alloue sa pile et que `STACK_NEXT_PAGE`
prend une valeur qui collide avec la pile de la clock task...
**Test** : peek `STACK_NEXT_PAGE` après chaque task_create boot.

---

## 6. Ce que j'aurais besoin de demander

1. **Cette fragilité est-elle déjà connue ?** Le CHANGELOG `2026-06-02c`
   mentionne « test_oricos_clock (budget cyc serré) » et le mot expert v1
   parlait de « viser ≤ 50 octets » — soit le seuil est anciennement observé
   et accepté comme caprice, soit personne ne l'a vu plus loin.

2. **Quelle est la bonne hypothèse à creuser en premier ?** Mes 4 ci-dessus,
   ou autre angle que je n'ai pas vu ?

3. **Faut-il un harnais de trace CPU windowed (start/end cycles) côté
   Phosphoric** pour capturer les ~5k cycles autour de la mort de la tâche
   clock ? Si oui, je le code (probablement Sprint dédié `cpu816_trace_window`).

4. **Est-ce que ça vaut le coup de durcir aussi le test** :
   - Asserter `TCB[clock].STATE != DEAD before "clock: done"` (échoue tôt
     avec un message clair au lieu d'un timeout vague).
   - Ajouter un `make test-position-shift` qui builds le kernel avec
     `.res 80, $EA` injecté dans wm.s et vérifie que `clock` passe TOUJOURS —
     **rendrait visible ce mode de fragilité dans le CI** au lieu de
     l'attendre passivement.

5. **Faut-il prioriser ça maintenant**, ou laisser dormir tant que :
   - Pas d'autre fix > 49 octets en file.
   - Bug B v2 (livré) tient.
   - Pas d'observation utilisateur de crash en cours d'usage.

---

## 7. Annexe — comment reproduire en 30 secondes

```bash
cd OricOS
# Insérer dans kernel/modules/wm.s juste avant kernel_wm_redraw_drag :
#   .proc _pad
#           .res 50, $EA
#   .endproc
make
cd ../Phosphoric
./test_oricos_helloc 2>&1 | grep clock
# → FAIL "clock: done" absent

# Trace screen + CPU pour observer :
cat > /tmp/probe.txt << EOF
nostp
app 1EE50
run 300000
cpu
mem 15C8D            # TCB pid 8 STATE — devrait être DEAD
mem 19032            # TASK_CUR
EOF
./oricrobot ../OricOS/build/kernel.bin /tmp/probe.txt
```

Le scénario tient avec OU sans Fix B v2 (le seuil est sur la taille shiftée,
pas sur le contenu).

---

*Rédigé 2026-06-09 par Claude Code. Fix B v2 livré et stable ; cette note
documente l'investigation collatérale du seuil position-dépendant que je
n'ai pas pu fermer.*
