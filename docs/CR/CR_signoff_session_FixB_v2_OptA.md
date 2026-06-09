# CR — Sign-off session Fix B v2 + Opt-A + invalidation Opt-C

> **Verdict senior** : push validé. Tree clean, 24/24 vert, Opt-A sain et
> synergie cli/FORBID préservée, Opt-C correctement *non* shippé, Fix B v2
> cohérent. Outils oricrobot `cpu`/`key` réutilisables.
>
> Date : 2026-06-09. Auteur sign-off : inge senior (revue externe).
> Statut : **CONDITIONS DE SUIVI** tracées ci-dessous (non bloquantes pour
> le push, à tracker).

---

## 1. Livrables validés

| Livrable | Commit | Statut |
|---|---|---|
| Fix B v2 — désactivation coalescing MOVED en `WM_TASKMODE=$A5` | `02934d2` | ✅ shippé |
| Opt-A — `sei`/`cli` ciblé sur `sys_gfx_fill_rect` + `sys_win_flush` | `5017990` | ✅ shippé |
| Invalidation Opt-C (retrait `cli` cop_handler) | — | ✅ **non** shippé, trace A/B dans `CR_reentrance_irq_syscall_confirmed.md` §10 |
| Outils oricrobot `cpu` / `key` | (oricrobot tree) | ✅ shippé, réutilisables |
| ADR-32 §10 — squelette ZP top-half IRQ + audit ordonné par risque | `8047f7e` | ✅ dossier d'instruction posé |

## 2. Points de validation senior

### 2.1 Fix B v2 — OK
Delta OLD→NEW dépasse fenêtre + erase-OLD-seul ne couvre pas la
trajectoire ; pousser events individuels remet le delta petit. Risque
assumé `RAW_RING` 16 slots → drop silencieux sous burst : acceptable v1
(une position sautée se corrige à l'event suivant).

### 2.2 Opt-A — OK comme v1
Vérifié : `kernel_forbid` posé dans `cop_handler` **avant** dispatch ⇒
`sei` syscall sous FORBID. `cli` avant `rts` dans body, **avant**
`kernel_permit` ⇒ IRQ différée fire avec FORBID=1 ⇒ `do_switch` voit
FORBID≠0 ⇒ pas de switch. **Synergie cli/FORBID intacte**, ZP scratch
protégée pendant body, VIA T1 IFR latche (pas de tick perdu), latence
bornée.

### 2.3 Invalidation Opt-C — raisonnement validé
La trace A/B (`CR_reentrance_irq_syscall_confirmed.md` §10) montre que
retirer le `cli` cop_handler casse une primitive d'ordonnancement réelle :
l'IRQ fait son bookkeeping mid-syscall, FORBID diffère le switch, et
l'IRQ déjà traitée ne préempte pas juste après `permit`. En retirant le
`cli`, l'IRQ s'accumule et fire **après** `permit` avec FORBID=0 ⇒
préemption immédiate ⇒ task_e vole 'Z'. Bug logique d'ordonnancement,
**pas** effet de bord. Le bon shape était SEI ciblé (= Opt-A).

## 3. Critiques calibrées (non bloquantes)

### 3.1 Routage clavier — critique générale **retirée**
Vérification `kernel_kbd_waiter_eligible` (`sched.s:305`) : le routage
focus **existe** (waiter avec fenêtre non-focus retenu C=0, possesseur
focus ou tâche sans fenêtre éligible). Les tâches GUI sont routées
correctement par focus.

### 3.2 Course exempt↔focus — défendable, à acter
Chemin **exempt** (task_e sans fenêtre) + `KBD_WAITER` slot unique ⇒
entre une tâche exempte et la tâche focus, la touche va à qui est
`KBD_WAITER` au moment où elle arrive. Résolu par timing, et la synergie
cli/FORBID le résout en faveur de la tâche focus active (garantie
« ne pas préempter au milieu d'une chaîne flush→print→read_char »).
**Défendable comme primitive v1**, pas un bug — dépend du rôle voulu de
task_e (lecteur clavier non-GUI légitime ?). **Action : ligne ADR-32
pour acter si cette course exempt↔focus est intentionnelle.**

### 3.3 Opt-A = fix de point, pas de classe
Hypothèse résiduelle ADR-32 §8 confirmée importante : tous les syscalls
non-bloquants qui touchent `WM_DP_TMP`/`GFX_*`/`EVT_TMP`/`WM_ARG_*` ont
le **même** hazard ; ils n'ont juste pas été frappés par ce test
précis. Danger : tests verts ⇒ pression §10 disparaît ⇒ futur syscall
tape collision ZP non protégée ⇒ rejouer toute la session.

## 4. Conditions de suivi (tracées vers ADR-32 §10.7)

> Pas bloquantes pour le push. À tracker comme items réels.

- [ ] **§10 comme item réel** (pas « un jour ») — ADR-32 §10 est
      l'instrument. Reprise active dès qu'un syscall non-Opt-A déclenche
      la classe, ou trimestre Q3 2026 plancher.
- [ ] **Invariant ZP IRQ acté en ADR ratifié** (ADR-32 §10.3) — partie
      intégrante de la ratification §10. Énoncé :
      > « Aucune routine en contexte IRQ top-half n'écrit hors de
      > `$E0–$EF` en zone page directe. »
- [x] **`make test-position-shift` landé maintenant** (ADR-32 §10.5) —
      **v1 livré 2026-06-09**. Sweep injection-phase MOU2 × syscall,
      canary `clock: done`. Baseline 2 cibles PASS, intégré `make tests`,
      24/24 vert. Détecte régression Opt-A. v2 (multi-syscall élargi,
      phase range 0-256, sweep T1 period) optionnel selon besoin.
- [ ] **Ligne ADR sur course exempt↔focus** (cf. §3.2 ci-dessus) — à
      acter avant ratification §10.
- [ ] **`coalesce-on-overflow` RAW_RING** — optionnel : quand ring
      plein, fusionner MOVED les plus anciens (delta plus large
      ponctuellement) au lieu de drop. Reprise sur observation cas
      burst réel.
- [ ] **Retrait Opt-A post-§10** — bonus une fois ZP IRQ dédiée
      stabilisée (le SEI ciblé devient redondant).

## 5. Référents

- `docs/CR/CR_reentrance_irq_syscall_confirmed.md` — trace A/B test 2
  SEI ciblé + invalidation Opt-C.
- `docs/adr/0032-zp-race-irq-task-DRAFT.md` §10 — squelette ZP top-half
  IRQ dédiée, audit ordonné par risque, plan d'attaque, conditions de
  suivi (§10.7 = miroir de §4 ci-dessus).
- `docs/notes/HANDOFF_inge_senior_Bug_B_session.md` — synthèse session
  complète pré-sign-off.
- Commits : `02934d2` (Fix B v2), `5017990` (Opt-A), `8047f7e` (ADR-32
  §10).

---

*Sign-off senior tracé 2026-06-09. Push autorisé. Suite : §10 (ZP
top-half IRQ dédiée) en sprint dédié, pas dans le scope session
courante.*
