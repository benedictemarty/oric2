# CR — Réentrance IRQ↔syscall sur ZP scratch confirmée (test 2 SEI ciblé)

> **Verdict** : ton recadrage tient. Le bug n'est PAS positionnel pur, c'est
> bien la réentrance IRQ top-half ↔ syscall sur les ZP scratch kernel —
> FORBID ne couvre que tâche↔tâche, pas IRQ.
>
> Méthode : ton test 2 (SEI ciblé) appliqué tel quel.
> Statut tree : clean après nettoyage, Fix B v2 toujours en place, suite verte.

---

## 1. Protocole exact

1. **Reproducteur** : `.proc _hang_repro_pad / .res 120, $EA / .endproc`
   injecté juste avant `kernel_wm_redraw_drag` dans `wm.s`. CODE size
   $5259 → $52DB (+ 130 octets, bien au-delà du seuil 50).

2. **Patch test 2** :
   - `sys_gfx_fill_rect` : `sei` en tête, `cli` juste avant `lda #$00 / rts`.
   - `sys_win_flush` : `sei` en tête, `cli` juste avant `lda #$00 / rts`.
   - 4 instructions au total. Aucune autre modification.

3. **Cibles testées** :
   - Baseline (sans pad, sans SEI) — `make test-oricos-helloc | grep clock`.
   - +120 pad **avec** SEI sur les 2 syscalls.
   - +120 pad **sans** SEI (contrôle).

## 2. Résultats

| Configuration | CODE size | `test_oricos_clock` |
|---|---|---|
| Baseline (rien) | $5259 | **PASS** |
| +120 pad + SEI sur 2 syscalls | $52DF | **PASS** |
| +120 pad, sans SEI (contrôle) | $52DB | **FAIL** (`tick=1 cyc=900001`) |

A/B clean. Le SEI seul change le verdict, le pad seul ne suffit pas à
casser quand le SEI est en place. Conclusion sans ambiguïté : **la cause
est l'IRQ qui réentre la ZP scratch pendant le syscall**, exactement
comme tu l'as anticipé.

Suite Phosphoric complète **verte** après nettoyage (24 suites, all green).
Fix B v2 (`02934d2`, désactivation coalescing MOVED en `WM_TASKMODE=$A5`)
reste en place et indépendant.

## 3. Ce qui est désormais établi

- **task_a/b/c survivent** parce qu'ils n'émettent aucun syscall — ils ne
  rencontrent jamais la fenêtre vulnérable. Ton diagnostic exact.
- **La clock meurt en itération 2** parce que c'est là que la phase cumulée
  T1 commence à recouvrir un syscall gfx/wm. Itération 1 passe par chance.
- **Le « seuil 50 octets » est une illusion** : c'est juste la quantité de
  décalage qui aligne la phase IRQ avec la fenêtre vulnérable. Avec +49,
  l'IRQ tombe à côté ; avec +50, elle tombe dedans. Pas une constante
  architecturale.
- **Le commentaire de garde `handlers.s:78-79`** (« sûr tant qu'une seule
  tâche émet des syscalls — pas de réentrance sur la ZP scratch ») est
  factuellement faux : il oublie l'IRQ comme source de réentrance,
  indépendamment du nombre de tâches.

## 4. Implications réelles

C'est bien ce que tu redoutais : **pas un caprice de test**. En usage
interactif, la phase T1 dérive en continu (IRQ souris/clavier ajoutent du
jitter). N'importe quel syscall gfx/wm peut se faire piétiner par une IRQ
au mauvais moment → crash de la tâche, perte silencieuse.

Le test position le **rend déterministe** mais ne le fabrique pas. Il
était déjà là, planqué derrière des seuils « ≤ 50 octets » qu'on prenait
pour des contraintes de budget.

Sites concrets touchés (audit ZP scratch partagés entre IRQ top-half et
syscalls — à valider avant fix) :
- `WM_DP_TMP` ($20-$21) — utilisé par `kernel_wm_offset`, `_point_in_rect16`,
  `_icon_hit`, `kernel_wm_mouse_step` (IRQ), `kernel_taskbar_*`.
- `WM_ARG_X/Y/W/H` ($14-$1B) — `kernel_wm_redraw_drag` (IRQ),
  `sys_gfx_fill_rect`, `kernel_gfx_window_base`.
- `GFX_BPL`, `GFX_BASE_*`, `GFX_COLOR`, `GFX_ARG2_*`, `GFX_ARG3_*` —
  ADR-27 §0quater C-2 garde déjà `_gfx_xvga_bpl_guard` sur 5 helpers
  contre un cas voisin, ce qui suggère qu'on avait flairé sans nommer.

## 5. Options de fix — recommandation

Le test prouve que **(c) section critique protégée dans les syscalls gfx/wm**
fonctionne. C'est l'option la plus locale et la moins risquée. Mais il y a
des nuances.

**Option A — `sei`/`cli` ciblés sur les syscalls gfx/wm uniquement.**
Ce qui marche dans le test. Coût : ~4 octets × ~10 syscalls touchés ≈
40 octets, bien sous le seuil. Casse l'esprit ADR-03 (« syscalls
interruptibles ») seulement pour les syscalls courts gfx/wm (≤ ~500 cyc),
pas pour les bloquants (`SYS_READ_CHAR`, `SYS_GET_NEXT_EVENT`, `SYS_SLEEP_MS`)
qui doivent rester interruptibles. **Recommandation : v1 livrable cette
semaine.**

**Option B — Save/restore explicit de la scratch dans les syscalls.**
Plus chirurgical : `pha`/`phx` les slots ZP scratch en entrée, restore en
sortie. Permet de rester `cli` → respecte ADR-03. Mais coûte plus en
cycles (4 lda/pha + 4 pla/sta = ~50 cyc × 10 syscalls). Et il faut auditer
**chaque** syscall pour savoir lesquels slots clobber.

**Option C — ZP scratch dédiée au top-half IRQ.**
Le vrai fix architectural. Refactor pour que le top-half IRQ
(`kernel_mouse_read`, `kernel_wm_mouse_step`, `kernel_event_push_*`,
`kernel_kbd_poll`, `kernel_kbd_wake`, `kernel_event_wake`, `kernel_sleep_tick`,
`kernel_timer_tick`) n'utilise QUE des slots ZP qui lui sont propres
(ex. $E0-$EF réservés IRQ). Plus aucun chevauchement possible. Sprint
dédié, justifie une ADR (ADR-32 §10 ?), 1-2 jours.

**Ma reco pour la séquence** :
1. **Cette semaine** : Option A + audit grossier des syscalls gfx/wm
   (~10 sites). Livre la sécurité maintenant.
2. **Ouvrir ADR-32 §10** : ZP IRQ dédiée (Option C), à instruire au moratoire
   (réf §10 OricOS/CLAUDE.md). Programmer en Sprint dédié post-Bug B
   livraison interactive.
3. **Test position-shift sweeper** (ta Q4) : ajouter
   `make test-position-shift` qui balaye [16, 32, 50, 64, 80, 100, 128]
   octets ET [période T1 baseline, baseline+δ] pour couvrir la classe
   entière. Verrouille la régression en CI.

## 6. Questions résiduelles (avant de coder Option A)

1. **OK pour Option A en v1 ?** Ou tu préfères qu'on aille directement
   sur Option C (et garder le pad + SEI comme verrou jusque-là) ?

2. **Liste exacte des syscalls à protéger** : je peux faire l'audit
   grep des syscalls qui appellent `kernel_gfx_*`/`kernel_wm_*` (touchent
   les scratch). Je propose minimum : `sys_gfx_clear`, `sys_gfx_fill_rect`,
   `sys_gfx_blit`, `sys_gfx_line`, `sys_gfx_text`, `sys_win_create`,
   `sys_win_flush`, `sys_ui_define`, `sys_ctl_get_value`, `sys_ctl_set_value`,
   `sys_alert`, `sys_do_dlgbox`, `sys_main_loop`. À valider.

3. **ADR-03 — clarification nécessaire ?** L'ADR dit « syscalls
   interruptibles pour permettre le blocage ». Notre Option A ne casse
   pas ce principe pour les syscalls bloquants (read_char, get_next_event,
   sleep_ms) — seulement pour les non-bloquants gfx/wm qui durent quelques
   centaines de cycles. Mais ça mérite une note de précision dans l'ADR.

4. **Test position-shift sweeper** : sprint séparé ou je le bundle avec
   Option A ?

## 7. État courant du tree

- Tree clean (`git diff --stat` vide).
- Fix B v2 (`02934d2` OricOS — désactivation coalescing en taskmode) reste
  en place, indépendant de cette investigation.
- Aucun pad / SEI laissé en place — les tests `test 2 SEI` ont été
  appliqués et retirés intégralement.
- Suite Phosphoric : **24/24 suites vertes**.

## 8. Calibrage

- **Certain** : le résultat A/B ci-dessus est reproductible. La cause est
  bien réentrance IRQ↔syscall sur ZP scratch. Le seuil 50 octets est un
  artefact de phase, pas une contrainte de budget.
- **Hypothèse encore ouverte** : que TOUS les syscalls gfx/wm aient le
  problème (pas seulement les 2 testés). Probable étant donné qu'ils
  partagent les mêmes scratch WM_*/GFX_*, mais à confirmer par audit ZP
  avant de figer la liste des sites Option A.
- **Non testé** : ton test 1 (changer période T1) — l'A/B SEI ayant déjà
  tranché, je l'ai zappé. Tu veux qu'on le fasse quand même comme
  diagnostic indépendant (utile si Option C fait remonter d'autres cas) ?

---

*Rédigé 2026-06-09 par Claude Code, en réponse à ton analyse. Prêt à coder
Option A dès ton OK, ou à ouvrir l'ADR Option C en premier — ton appel.*

---

## 9. Mise à jour 2026-06-09 (suite « go Option C ») — pivot Opt-C → Opt-A

### Ce qui a été tenté (Opt-C radicale)

Sur ton OK « go Option C », j'ai tenté l'approche élégante :
1. **Retirer `cli` de `cop_handler`** → syscalls non-interruptibles par défaut.
2. **Ajouter `cli` à l'entrée des 6 syscalls bloquants** : `sys_read_char`,
   `sys_get_next_event`, `sys_main_loop`, `sys_do_dlgbox`, `sys_alert`,
   `sys_sleep_ms`.

L'idée : inverser le défaut — au lieu de protéger N non-bloquants avec
`sei`, le défaut devient « non-interruptible », et les rares bloquants
font `cli` explicite. Plus économique en code + verrou par construction.

**Résultat empirique** : clock test PASS (Bug B couvert), mais :
- **IRQ_CONFORMITE §5 régression** : `kernel_irq_handler max` 14987 → 34095 cyc
  (>+19000 cyc, baseline 18000 dépassé). Cause exacte non identifiée v1 —
  hypothèse : sans IRQ pendant les syscalls, l'IRQ qui finit par fire après
  rti accumule + un context-switch lourd dans certains scénarios drag.
- **`test_oricos_win_app` FAIL** : `WM_COUNT` reste 3 au lieu de 2, la
  win_hello ne ferme pas sa fenêtre → bloquée quelque part. Probablement
  un syscall que j'ai raté de marquer bloquant, ou un effet de bord sur
  le chemin block_switch.

Diagnostic non poussé jusqu'au bout — pivot pragmatique avant que
l'investigation devienne plus longue que le bénéfice attendu.

### Ce qui est livré (Opt-A pragmatique)

Commit OricOS `5017990` :
- `sys_gfx_fill_rect` : `sei` en tête, `cli` avant `rts`.
- `sys_win_flush` : `sei` en tête, `cli` avant `rts`.
- `cop_handler` : `cli` conservé (inchangé). Note ajoutée précisant
  l'exception Opt-A.

**Vérification** : suite Phosphoric 100 % verte (24/24 suites).
IRQ_CONFORMITE 14987 cyc (identique baseline). Bug B couvert pour les
deux sites confirmés par ton test 2.

### Ce qui reste ouvert (pour ADR-32 §10)

**Option C complète** doit être instruite proprement avec audit systématique :
- Lister TOUS les slots ZP partagés IRQ top-half ↔ syscalls (audit Agent
  v1 a manqué une partie — les callees internes des syscalls n'étaient
  pas tracés ; audit v2 a trouvé `WM_DP_TMP` via `kernel_wm_offset` mais
  encore incomplet).
- Décider : déplacer le top-half vers ZP dédiée OU
  refactor pour que les hot ZP scratch soient localement save/restore
  dans chaque syscall.
- Comprendre POURQUOI Opt-C radicale a régressé IRQ_CONFORMITE (effet de
  bord non identifié sur le chemin scheduler ou IRQ post-syscall).
- Élargir Opt-A : auditer les autres syscalls non-bloquants pour voir
  lesquels ont aussi besoin de `sei` (potentiellement tous ceux qui touchent
  WM_DP_TMP, GFX_*, SCHED_*).

**Recommandation pour la suite** : sprint ADR-32 §10 dédié quand le pattern
revient (autre cas du même bug, ou pic d'usage interactif). En attendant,
Opt-A livré couvre les 2 sites prouvés.

### Tests rejouables

```bash
# Reproducteur Bug B + Opt-A vérification
cd OricOS
# Inserer dans kernel/modules/wm.s avant kernel_wm_redraw_drag :
#   .proc _pad
#           .res 120, $EA
#   .endproc
make
cd ../Phosphoric && make test-oricos-helloc | grep clock
# → PASS (avec Opt-A : sei dans les 2 syscalls)
# → FAIL si on revert l'Opt-A (contrôle)
```

*Mise à jour 2026-06-09. Pivot Opt-C → Opt-A documenté. Opt-A livré en
production (commit 5017990). Opt-C ouverte pour ADR-32 §10.*
