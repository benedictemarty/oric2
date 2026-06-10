# BUG task_wm_starve — CLOS (cause prouvée 2026-06-10)

> Clôture du dossier `BUG_task_wm_starve(1).md` (2026-06-01, resté « NON
> RÉSOLU »). Versionné ici conformément à R9. Travail P0-a de la revue
> senior (`REVIEW_senior_OricOS_2026-06-10.md`).

## Symptôme (rappel)

Avec `--ctl-demo --wm-taskmode` : task_wm (pid 8) jamais ordonnancée,
curseur figé. État contradictoire : `RAW_WAITER=0` + `TCB[8].STATE=BLOCKED`,
`RAW_RING_COUNT=1` stable. Sans l'un OU l'autre flag : OK.

## Verdict : bug ÉMULATEUR (§5septies), fixé par Phosphoric `08a9bf` — dossier jamais refermé

Le fix (`asl mem $06/$16/$0E/$1E` M-aware, 2026-06-01) portait déjà la
mention « résout bug §10 task_wm starve » dans son message de commit, mais
la causalité n'avait pas été prouvée et le dossier + la note `event.s`
sont restés « NON RÉSOLU » pendant 9 jours, gelant ADR-28.

## Preuve (R1 — exclut les alternatives)

1. **Kernel d'époque (`e74d714`) + émulateur actuel → PASS** (10 moves /
   0 STALE, harness `test_oricos_ctl_taskmode_starve`). Le bug disparaît
   sans toucher au kernel ⇒ émulateur.
2. **Kernel d'époque + bug CPU re-simulé** (ASL mem forcé 8-bit-fixe sous
   M=16, build `-DSIMULATE_RMW_M16_BUG` temporaire) → **10/10 STALE**,
   signature exacte (`RAW=1/W=0`). Les sites kernel exécutant ASL mem en
   M=16 pendant le run, loggés : `01:4E24` (×34) = `kernel_task_create`
   (scan bitmap `asl SCHED_TMP`), `01:4F7B` (×4) = `kernel_bitmap_clear`.
3. Émulateur restauré, kernel HEAD → PASS, suite complète verte.

## Mécanisme (reconstruit, cohérent avec chaque observation d'époque)

Avec ASL mem 8-bit-fixe sous M=16, le masque 16-bit du scan bitmap ne
propage jamais dans l'octet HAUT :

1. À la création de **task_wm (pid 8)** — le premier pid dont le bit vit
   dans le high byte ($0100) — le masque devient $0000 au 8e shift.
   `ora` avec $0000 : **le bit 8 n'est jamais posé dans TCB_BITMAP.**
   task_wm est créée (TCB valide) mais la bitmap la déclare absente.
2. La **création suivante** — l'app **ctl_demo** — re-scanne : pour le
   slot 8, `and` masque $0000 = 0 → « libre » → `kernel_task_create`
   **écrase le TCB de task_wm** (pid, STATE, S, PC réinitialisés).
3. D'où la double condition exacte (`--wm-taskmode` crée task_wm,
   `--ctl-demo` fournit la création qui l'écrase), le `TASK_CUR` qui
   n'est « jamais 8 » (le slot appartient désormais à ctl_demo), et
   l'état RAW_WAITER/BLOCKED incohérent (TCB réécrit sous le waiter).
4. `kernel_bitmap_clear` avait le même défaut (pids ≥ 8 jamais libérés,
   fuite de slots) — secondaire ici.

Toutes les hypothèses réfutées du dossier d'époque (§1.a-1.f) restent
réfutées ; le « suspect n°1 » (do_switch READY inconditionnel) était un
faux coupable — l'écrivain fantôme de `tcb[8]` était `task_create`
lui-même, via la bitmap aveugle au high byte.

## Gardes en place

- `test_oricos_ctl_taskmode_starve` : dans `make tests` (end-to-end).
- Klaus 65C816 étendu : pivots RMW M=16 (`ASL/INC/DEC` mem) — R7, la
  défense de classe contre cette famille de divergences émulateur.
- La re-simulation est rejouable : patch `-DSIMULATE_RMW_M16_BUG` décrit
  ci-dessus (non conservé dans le code — ce document suffit).

## Conséquences

- **Le verrou d'ADR-28 est levé** : `WM_TASKMODE` n'a plus de bug connu.
  La bascule par défaut (P0-a, 2e moitié) reste un sprint dédié avec
  validation interactive utilisateur (leçon ADR-29) — non incluse ici.
- Note `event.s` mise à jour (résolution tracée), références ADR-33/
  CLAUDE.md à rafraîchir à la bascule.
- Leçon process : un fix dont le message de commit affirme « résout X »
  doit refermer le dossier de X dans le même commit — 9 jours de verrou
  fantôme sur ADR-28 pour un bug déjà mort.
