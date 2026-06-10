# ADR-32 (DRAFT) — Course ZP IRQ↔tâche : propriétaire unique WM + migration mouse_step hors IRQ

- **Statut** : **ouverte — dossier d'instruction** (2026-05-31). DRAFT
  non ratifié (moratoire CLAUDE.md §10 : implémentation de référence à
  produire par étapes ; ratification après validation interactive et
  passage du harnais d'injection event-async).
- **Date d'ouverture** : 2026-05-31
- **Décideurs** : bmarty (arbitrage), Claude Code (instruction)
- **Origine** : audit ingénieur 6502/65C816 du kernel
  (`AUDIT_65C816_REMEDIATION.md` §3.3a + axe 8.1). Identifie la course
  ZP entre contexte IRQ (`kernel_wm_mouse_step`, `_cursor_draw`) et
  contexte tâche (`sys_win_create`, `sys_gfx_*`, MainLoop) comme
  **suspect n°1 du revert ADR-28 Étape 3** (curseur figé, widgets
  non-réactifs ; cf. fichier `0028-threading-window-manager.md` §6.7).
- **ADR liées** :
  - **ADR-25** Exec-classique (Forbid/Permit) — `forbid` ne masque PAS
    les IRQ : sa portée est tâche↔tâche uniquement. C'est la racine du
    bug (les sites tâche ne se protègent pas des IRQ MOU2).
  - **ADR-28** Threading WM (option C : politique + rendu en tâche
    serveur, curseur IRQ). Étape 3 « skip mouse_step IRQ » revertée
    suite test interactif (curseur figé). Le présent ADR finalise
    l'option C de manière atomique sans reproduire le revert.
  - **ADR-29** Drag notification hint — révèle le pattern app
    bottleneck (vs coût IRQ) en fin de drag ; non lié à la course ZP
    proprement dite, mais alignement intent.
  - **Audit §3.3a / §3.3b / axes 8.1, 8.5** — diagnostic structurel.
- **Conformité moratoire** : oui — **dossier d'instruction** (CLAUDE.md
  §10 cond. 1). ≥ 2 alternatives chiffrées. Recommandation senior
  tracée. Pas de modification d'ADR ratifiée tant que la ratification
  ADR-32 n'est pas votée (point de bascule ADR-28 documenté en §6).

---

## 0. Résumé exécutif

**Le bug.** `kernel_wm_mouse_step` tourne en contexte IRQ MOU2 et écrit
des slots ZP **partagés avec le contexte tâche** : `WM_ARG_*` ($14-$1F),
`WM_DP_TMP` ($20-$21), `GFX_*` ($70-$93). Les sites tâche
(`sys_win_create`, `sys_gfx_*`, MainLoop, hit-testers) lisent et
écrivent ces mêmes slots **sous `forbid`** — ce qui ne masque PAS les
IRQ (ADR-25). Conséquence : un event MOU2 (cursor MOVE, click) arrivant
entre l'écriture et la lecture de `WM_ARG_*` par un syscall écrase
silencieusement les args.

**Symptômes observés.** Fenêtres qui « sautent » à la position souris,
BLIT GPU avec args corrompus, et — plus pernicieux — le revert ADR-28
Étape 3 où sortir un sous-ensemble de mouse_step vers une tâche serveur
a exposé la course que l'exécution tout-en-IRQ masquait par sérialisation
implicite.

**Pourquoi les 541 tests ne l'attrapent pas.** Tests headless,
séquentiels, pas d'injection d'event device asynchrone *pendant* un
syscall. C'est l'angle mort identifié par l'axe 8.5 de l'audit.

**Décision proposée (option B, recommandée).** Finaliser l'option C
d'ADR-28 de manière **atomique** : tout `kernel_wm_mouse_step` (focus,
drag, resize, redraw, compose, callbacks) migre vers la **tâche serveur
WM** (qui tourne sous `forbid`, donc sérialisée avec les autres
syscalls). L'IRQ MOU2 ne fait plus QUE : lire le device, pousser un
record dans `RAW_RING` (déjà câblé ADR-28 Étape 0), poser un flag, `rti`.
Plus aucune écriture WM_ARG_*, GFX_*, WM_DP_TMP en contexte IRQ. Le
curseur — actuellement dessiné en IRQ via `_cursor_draw` — migre lui
aussi vers la tâche WM (cf. §6.6 du diagnostic).

**Pourquoi pas un autre fix.** L'audit considère trois options ; la
seule sans race résiduelle est la migration complète. Patchwork `sei`
(option A) = dette qui grossit ; partition stricte ZP IRQ-only
(option C) = ne supprime pas le coût IRQ (§3.3b — redraw 393 KiB en
IRQ) ni le code arbitraire en ISR (callbacks invoqués depuis
mouse_step), et laisse `_cursor_draw` qui écrit GFX_* en IRQ comme un
producteur résiduel.

**Risque n°1 — le revert.** ADR-28 Étape 3 a déjà tenté une migration
partielle et a été revertée pour cause de curseur figé / widgets
non-réactifs. La cause probable (post-mortem audit §3.3a) : transition
non-atomique = pendant la fenêtre où l'IRQ ne fait plus mouse_step mais
la tâche serveur ne l'a pas encore consommé, le RAW_RING se remplit
plus vite qu'il est drainé, le curseur n'est plus repeint, et les
callbacks n'arrivent jamais. **Le plan ci-dessous (§5) impose une
transition atomique par flag de bascule + validation incrémentale.**

---

## 1. Contexte technique chiffré

### 1.1 — Sites de la course (mesure audit)

Grep sur `wm.s` pour les écritures dans la zone partagée :

```
grep -nE 'sta (GFX|WM_ARG|WM_DP)' kernel/modules/wm.s | wc -l
→ 206 écritures
```

206 sites d'écriture. Tous ne sont pas problématiques (certains sont en
contexte tâche unique), mais l'analyse audit identifie au minimum :

- **`kernel_wm_mouse_step`** (wm.s ~l.1925, contexte IRQ) :
  - pose `WM_ARG_X/Y` depuis `MOUSE_X/Y` avant `kernel_wm_hit_test`
  - appelle `kernel_wm_offset` → clobbe `WM_DP_TMP`
  - cascade vers `kernel_wm_redraw`/`compose` → clobbe `GFX_*`, `WCMP_*`
  - invoque les callbacks icône/widget via `jsr (vec,X)` — **code app
    arbitraire en ISR**.

- **`_cursor_draw`** (appelé par `kernel_wm_cursor_blit` depuis
  `mouse_step`, IRQ) : écrit `WM_ARG_X/Y/W/H` ET `GFX_*` à **chaque
  événement de mouvement souris** (pas seulement au clic). La fenêtre
  de corruption est *quasi permanente* pendant un déplacement continu
  de la souris.

- **`sys_win_create`** (wm.s ~l.3335, contexte tâche) : écrit
  `WM_ARG_X/Y/W/H` depuis $D0-$D7. Commentaire de l'autrice :
  « WM_ARG_* partagé avec l'IRQ souris → clobber possible… rare ;
  partition ZP = polish ». **Ce n'est pas du polish — c'est ce bug.**

- **`sys_gfx_*`** : posent `GFX_BASE/ARG2/ARG3/ARG4/COLOR/FONT/STR`
  ($70-$93). Partagés avec le compositing IRQ.

- **`_ml_classify`** (wm.s ~l.3416), `_wm_widget_hit` (l.~4423),
  `_wm_redraw_ctl` : **encadrent déjà** leur accès `WM_ARG_*` par
  `php ; sei … plp`. Patchwork ad-hoc — **le `sei` épars est lui-même
  le smell**, pas la solution.

### 1.2 — Pourquoi `forbid` ne suffit pas

ADR-25 / `kernel_forbid` :
- incrémente `FORBID_COUNT` (testé dans `do_switch` du scheduler)
- **n'écrit PAS** au flag I du processeur, **n'exécute PAS** `sei`
- conséquence : pendant un syscall sous `forbid`, les IRQ
  (T1/KBD2/MOU2/VIA) continuent de s'exécuter. C'est explicitement le
  contrat — Exec-classique sépare préemption tâche↔tâche (Forbid) et
  masquage I/O (sei/cli). La course IRQ↔tâche n'est *pas* dans le
  contrat Forbid.

### 1.3 — Pourquoi les tests ne la voient pas

Les 541 tests d'intégration Phosphoric sont :
1. **Headless** — pas de souris continue qui MOVE
2. **Séquentiels** — un event device est posé AVANT un syscall, jamais
   *pendant* (pas d'horloge cycle-précise pour injecter au cycle N+k)
3. **Sans cadence MOU2 réaliste** — quelques events par scenario, pas le
   flux continu (>100/s) d'un déplacement de souris réel

Donc 541 verts + bug interactif visible = pas une régression de test,
c'est un trou de couverture. **Comblé par axe 8.5** (harnais
d'injection event-async via hook PC).

### 1.4 — Coût latence sous-jacent (§3.3b)

Même course résolue, `kernel_wm_mouse_step` en IRQ appelle
`kernel_wm_redraw` dans 4 branches (menu, close, resize, end-of-drag).
`kernel_wm_redraw` fait `gfx_clear` de **393 KiB** + dessin de toutes
les fenêtres/widgets/titres/taskbar + compose (BLITs + spin sur
`GPU_STATUS_BUSY`). À période T1 = 4096 cycles, tout clic fait
largement déborder l'IRQ → IRQ back-to-back, famine, drops T1/KBD2/MOU2.

Le présent ADR ne traite **pas le coût** (axe 8.4 dirty-rect compositor)
mais le résout *de fait* en sortant le redraw du contexte IRQ. La
latence post-migration sera *celle de la tâche serveur* (dont
l'ordonnancement n'est pas contraint au T1).

---

## 2. Options chiffrées

### Option A — Statu quo + multiplication des `sei` ad-hoc

**Description.** Chaque site tâche qui touche `WM_ARG_*`/`GFX_*`/
`WM_DP_TMP` ajoute `php ; sei … plp` autour de la séquence. Patchwork
étendu, modèle Exec-classique conservé.

**Effort.** ~20-30 sites à instrumenter. Pas de refactor architectural.
~1 jour de travail + validation interactive.

**Bénéfice.** Plus de course (chaque section critique est elle-même
masquée IRQ).

**Coût.**
- Dette qui grossit à chaque nouveau syscall touchant la ZP partagée
  (oubli humain = bug silencieux qui réapparaîtra)
- Le coût §3.3b (redraw 393 KiB en IRQ) n'est PAS résolu — latence
  inchangée
- Le callbacks en ISR (`jsr (vec,X)` depuis mouse_step) restent — code
  arbitraire en ISR, contraire à toutes les conventions OS
- Le smell « patchwork sei » signalé par l'audit reste

**Verdict.** Recommandation senior : **non**. Ferme un symptôme, laisse
la maladie structurelle. Compatible avec axe 8.1 « propriétaire
unique » uniquement comme étape transitoire avant option B.

### Option C — Partition stricte ZP IRQ-only / task-only

**Description.** Allouer un set disjoint de slots ZP pour l'IRQ
(`WM_ARG_IRQ_*`, `GFX_IRQ_*`, `WM_DP_IRQ_TMP`). L'IRQ utilise
exclusivement ses slots privés ; la tâche ses slots historiques. Le
modèle `EVT_TMP = $6E` (déjà désigné « IRQ-only » dans la doc ZP) est
le précédent.

**Effort.** ~16-24 octets ZP supplémentaires alloués + modification de
toutes les écritures IRQ pour rediriger vers les nouveaux slots.
~1.5 jours.

**Bénéfice.** Plus de course (slots disjoints, IRQ et tâche n'écrivent
plus jamais aux mêmes adresses).

**Coût.**
- 16-24 octets ZP consommés. Zone $94-$FF (108 B libres) absorbe sans
  douleur, mais la ZP kernel devient sensiblement plus fragmentée.
- Le coût §3.3b (redraw 393 KiB en IRQ) n'est PAS résolu
- Les callbacks en ISR restent
- Le redraw IRQ qui écrit GFX_IRQ_* doit composer SUR le même
  framebuffer XVGA partagé avec les BLITs tâche → on déplace la course
  d'un niveau (ZP → VRAM / GPU FIFO). Le GPU peut être lui-même
  préempté par un IRQ qui programme un nouveau BLIT pendant qu'un BLIT
  tâche est en vol — `GPU_STATUS_BUSY` masque cela partiellement, mais
  c'est un placebo.

**Verdict.** Recommandation senior : **non**. Ferme la course ZP
proprement mais ne résout aucun des autres problèmes structurels
(latence, callbacks en ISR, contention GPU). Compromise architecturale
qui rendra la prochaine évolution (multi-tâches concurrentes UI,
animations) impossible sans refactor majeur.

### Option B — Migration complète mouse_step → tâche serveur WM **(recommandée)**

**Description.** Finalise l'option C d'ADR-28 (déjà ratifiée — politique
fenêtre + rendu en tâche serveur). L'IRQ MOU2 et T1 deviennent strictement
**producteurs** :

```
IRQ MOU2 :
  - kernel_mouse_read  → lit + clear MOU2_STATUS
  - kernel_event_push_mouse → push record dans RAW_RING (déjà câblé)
  - poser flag WM_INPUT_PENDING
  - rti
  (durée IRQ : ~50-80 cycles, plafonnée et bornée)

Tâche serveur WM (déjà créée Étape 0 ADR-28) :
  while (forever) {
    SYS_YIELD jusqu'à WM_INPUT_PENDING
    pop RAW_RING
    kernel_wm_mouse_step (tout : focus, drag, resize, redraw, compose,
                          callbacks invoqués ici en contexte tâche)
  }
```

`sys_win_create` / `sys_gfx_*` / `_wm_mouse_step` sérialisés par
construction : ils sont tous sous `forbid`, **et** plus aucun
producteur IRQ ne touche leurs ZP. La course disparaît
**par construction**, pas par patch.

**Le curseur** — actuellement dessiné en IRQ (`_cursor_draw` via
`kernel_wm_cursor_blit`) — migre aussi vers la tâche WM. C'est le point
qui a fait reverter ADR-28 Étape 3 (curseur figé). La solution :
**transition atomique** par flag (§5 ci-dessous), pas migration
incrémentale.

**Effort.** ~3-5 jours répartis :
- Étape 1 : harnais test injection MOU2 async (axe 8.5, prérequis)
- Étape 2 : flag de bascule `WM_TASKMODE = $A5` (par défaut OFF →
  comportement actuel IRQ ; ON → tâche WM consomme RAW_RING)
- Étape 3 : migration `mouse_step` complet sous flag, IRQ ne fait plus
  que producer + flag
- Étape 4 : migration `_cursor_draw` (le plus risqué — curseur doit
  rester fluide)
- Étape 5 : nettoyage `php ; sei … plp` épars devenus inutiles
- Étape 6 : ratification, retrait du flag (le comportement devient le
  défaut)

**Bénéfice.**
- Ferme la course ZP par construction (cible audit §3.3a)
- Ferme le coût latence IRQ (cible audit §3.3b) — redraw 393 KiB en
  tâche, plus en ISR
- Plus de callbacks en ISR (cible audit §5 curseur owner)
- Plus de patchwork `sei` (axe 8.1 « propriétaire unique » atteint)
- Forme aboutie d'ADR-28 (lève les rétractations 2026-05-30 sur
  Étape 3)

**Coût.**
- Refactor structurel — touche le hot path (mouse_step appelé sur
  chaque event MOU2)
- Risque de reproduire le revert si la transition n'est pas atomique
  (mitigation : §5)
- Latence souris théoriquement plus longue (event MOU2 → tâche WM
  scheduling → traitement). En pratique : tâche WM préemptée à chaque
  tick T1, donc < 4096 cycles, sous le seuil de perception (4 ms à
  1 MHz).

**Verdict.** Recommandation senior : **oui**.

### Synthèse comparative

| Option | Course ZP | Coût latence | Callbacks ISR | Effort | Dette future |
| --- | --- | --- | --- | --- | --- |
| A — sei épars | ✓ | ✗ | ✗ | faible | élevée (récurrente) |
| C — partition ZP | ✓ | ✗ | ✗ | moyen | moyenne (contention GPU) |
| **B — migration WM-task** | **✓** | **✓** | **✓** | **élevé** | **faible** |

---

## 3. Plan d'atomicité (critique — anti-revert)

Le revert d'ADR-28 Étape 3 est la leçon principale. Sa cause probable
(diagnostic audit §3.3a) : on a sorti **une partie** de mouse_step de
l'IRQ vers le serveur, laissant l'autre partie (le curseur, les
callbacks) en IRQ. Pendant les fenêtres où l'IRQ n'arme plus
mouse_step et où le serveur n'a pas encore réagi, le curseur se fige.
Les callbacks ne sont jamais invoqués.

**Règle d'or pour ADR-32.** Aucun état intermédiaire « moitié IRQ /
moitié tâche » ne doit exister à un instant T. Bascule via flag
**unique** :

```
WM_TASKMODE = $01EE60   ; nouveau, sentinelle libre ($EE60 zone TC_*)
  $00 → comportement actuel (mouse_step en IRQ, curseur en IRQ)
  $A5 → option B (mouse_step en tâche serveur, curseur en tâche)
```

L'IRQ MOU2 lit le flag à son entrée :
```asm
mouse_irq_handler:
        lda WM_TASKMODE
        cmp #$A5
        bne _mouse_legacy        ; ancien chemin IRQ inchangé
        ; chemin tâche
        jsr kernel_mouse_read
        jsr kernel_event_push_mouse   ; pousse dans RAW_RING
        lda #$A5
        sta WM_INPUT_PENDING          ; flag pour la tâche WM
        rti
_mouse_legacy:
        jsr kernel_wm_mouse_step      ; ancien comportement
        rti
```

Conséquences :
1. **Rollback instantané** — si le mode tâche cause un bug, écrire
   `$00` à `WM_TASKMODE` au runtime restaure l'ancien comportement
   sans rebuild. Possible depuis un test, depuis le shell, depuis
   un syscall debug.
2. **Test parallèle** — la suite tests Phosphoric tourne par défaut
   en mode legacy (= comportement actuel, 541 verts garantis). Le
   nouveau test `test_oricos_zp_race_win_create` active explicitement
   `WM_TASKMODE = $A5` pour exercer le nouveau chemin.
3. **Ratification graduelle** — Étape 6 ne flip le défaut qu'après
   validation interactive utilisateur sur scenarios drag continu,
   resize fenêtre, clic taskbar, double-clic icône — TOUS les
   scenarios qui ont fait reverter Étape 3.

**Anti-pattern interdit.** Pas de « partial mode » où certains
événements (e.g. MOVE) vont au serveur et d'autres (e.g. DOWN) restent
en IRQ. Tout ou rien.

---

## 4. Harnais de test (prérequis Étape 1 — axe 8.5)

Sans harnais d'injection event-async, on ne peut pas écrire de test
verrouillant la fix. Sans test, on ne peut pas garantir l'absence du
revert. **Donc Étape 1 = harnais, AVANT toute migration**.

Spec minimale du harnais :

```c
// Phosphoric: hook PC dans la boucle d'émulation principale.
// Quand cpu.PBR == bank && cpu.PC == addr, exécute callback.
typedef void (*pc_hook_t)(cpu65c816_t* cpu, void* userdata);

void cpu816_set_pc_hook(cpu65c816_t* cpu,
                        uint8_t bank, uint16_t addr,
                        pc_hook_t hook, void* ud);
void cpu816_clear_pc_hook(cpu65c816_t* cpu, uint8_t bank, uint16_t addr);
```

Test type pour §3.3a :

```c
TEST(test_zp_race_win_create_moved) {
    // 1. Boot OricOS + spawn task_winrace qui appelle SYS_WIN_CREATE
    //    avec args FIXES (x=100, y=120, w, h).
    // 2. Hook PC sur l'adresse JUSTE APRÈS le `sta WM_ARG_X` de
    //    sys_win_create (résolue depuis build/kernel.lbl, axe 8.6).
    // 3. Dans le hook : armer un event MOU2 MOVED à position POISON
    //    (x=$0BAD, y=$0BAD). L'IRQ MOU2 doit consommer ce poison.
    // 4. Laisser tourner jusqu'à ce que sys_win_create complete.
    // 5. Lire WM_TABLE[handle].x — doit valoir 100, JAMAIS $0BAD.

    // En mode WM_TASKMODE=$00 (legacy) : test ROUGE attendu (la course
    //   existe, le poison passe).
    // En mode WM_TASKMODE=$A5 (option B) : test VERT attendu (l'IRQ
    //   MOU2 ne touche plus WM_ARG_*, la tâche serveur consomme
    //   l'event APRÈS sys_win_create).
}
```

Le test prouve à la fois :
- Que la course existe en mode legacy (réfute le « rare ; polish » de
  sys_win_create)
- Que l'option B la ferme

Variante MOVED (mouvement continu) à compléter avec un scenario
multi-events injectés à intervalles serrés — sinon on ne couvre que la
fenêtre étroite du DOWN, pas le déplacement de souris continu qui est
le cas pire interactif.

---

## 5. Plan d'implémentation (post-ratification)

### Étape 1 — Harnais injection PC-hook (Phosphoric)
- Ajouter `cpu816_set_pc_hook` à `src/cpu/cpu65c816.c` (vérifier coût
  cycle ; idéalement intégré au step déjà existant `cpu816_step`).
- Tester en isolation : hook fictif qui logge → vérification cycle
  précis.
- **Critère** : hook fire au cycle exact où PC = (bank, addr).
- **Pas de modification kernel.**

### Étape 2 — Flag WM_TASKMODE + chemin legacy intact
- Allouer `WM_TASKMODE = $01EE60` (zone TC_* sentinelles).
- Modifier `mouse_irq_handler` (handlers.s) : test flag, dispatch.
- Par défaut $00 → comportement actuel.
- **Critère** : suite tests Phosphoric verte, comportement utilisateur
  identique.
- **Risque** : nul (chemin legacy inchangé).

### Étape 3 — Tâche WM consomme RAW_RING (option B path actif sous flag)
- Boucle serveur WM : `while (1) { yield until pending; pop RAW_RING;
  invoke mouse_step }`.
- mouse_step et tout son sous-arbre (focus, drag, resize, redraw,
  compose, callbacks) sont appelés ICI, en contexte tâche, sous
  `forbid`.
- Test `test_zp_race_win_create_moved` avec `WM_TASKMODE = $A5`
  doit passer (utilise harnais Étape 1).
- **Critère** : nouveau test vert + suite Phosphoric verte (mode
  legacy par défaut).
- **Risque** : nul tant que le flag reste à $00 par défaut.

### Étape 4 — Curseur sous propriétaire unique tâche WM
- `_cursor_draw` quitte l'IRQ pour la tâche WM.
- Backing store curseur : seul possesseur = tâche WM.
- **Critère** : test interactif utilisateur validé sur :
  - mouvement souris continu (curseur fluide)
  - clic + drag fenêtre (curseur suit, fenêtre suit)
  - resize fenêtre par bord (curseur reste visible pendant resize)
  - clic icône desktop (callback s'exécute)
  - clic bouton taskbar (focus change)
  - double-clic chrome × (fermeture)
- **Risque** : MOYEN — c'est le point qui a fait reverter Étape 3
  d'ADR-28. Mitigation : flag de bascule + tests interactifs
  exhaustifs avant tout commit. Si KO → rollback flag.

### Étape 5 — Nettoyage `sei`/`php`/`plp` épars
- `_ml_classify`, `_wm_widget_hit`, `_wm_redraw_ctl` etc. : retirer
  les `sei`/`plp` devenus inutiles (la course ZP n'existe plus).
- **Critère** : suite tests verte + relecture audit (plus aucun
  `sta WM_ARG_*` ou `sta GFX_*` en contexte IRQ).
- **Risque** : faible (cleanup additif inverse).

### Étape 6 — Flip du défaut + ratification ADR-32
- `WM_TASKMODE = $A5` par défaut.
- Validation interactive finale (re-tester scenarios Étape 4).
- Ratification ADR-32 ; mise à jour CLAUDE.md §2 ; suppression du
  flag (devient implicite) ou conservation pour debug futur.

---

## 6. Impact sur ADR-28 (point de bascule)

ADR-28 reste ratifiée. Le présent ADR-32 :
- **Finalise** l'option C d'ADR-28 (politique + rendu en tâche
  serveur) sans modifier sa décision.
- **Lève** les rétractations 2026-05-30 sur Étape 3 d'ADR-28 — la
  migration mouse_step hors IRQ est *correcte* dans son intention, le
  revert venait du manque d'atomicité, traité par §3 ci-dessus.
- **N'invalide pas** §6.7 d'ADR-28 (quota anti-drop button-UP) qui
  était mal-ciblé (le bug fin-de-course est un app bottleneck cf.
  ADR-29, pas un drop IRQ).

À la ratification d'ADR-32, ajouter une révision mineure d'ADR-28 :
« Étape 3 réhabilitée et complétée par ADR-32 (transition atomique
via WM_TASKMODE) ».

---

## 7. Critères de réouverture / d'arrêt

**Réouverture** : si le harnais Étape 1 révèle des courses additionnelles
(GFX, WCMP, autres) non couvertes par l'option B, instruction
complémentaire requise.

**Arrêt anticipé** : si Étape 4 (curseur sous tâche WM) échoue
interactivement après 2 tentatives, **arrêter l'ADR-32** et tomber sur
l'option C (partition ZP) comme repli — coûteux mais sans risque
interactif. Documenter le motif dans la révision ADR-32.

---

## 8. Décision attendue

À ratifier après production du harnais (Étape 1) et de la maquette
Étape 3 sous flag (sans toucher le défaut). À ce stade :
- option B validée fonctionnellement par le nouveau test
- coût latence quantifié (cycles tâche vs IRQ)
- pas de régression suite tests Phosphoric

**Si OK → ratification + poursuite Étapes 4-6.**
**Si KO → instruction option C (partition ZP) comme repli.**

---

*Rédigé 2026-05-31 — dossier d'instruction Claude Code à partir de
`AUDIT_65C816_REMEDIATION.md` §3.3a / §3.3b / §5 curseur / axes 8.1, 8.5.*

---

## 9. Rétractation Étape 4 — validation interactive KO (2026-05-31)

**Constat** : test interactif utilisateur avec `--wm-taskmode` (=
`TC_WM_FLAG=$A5 + WM_TASKMODE=$A5`) → **curseur invisible**. Test
contrôle `--wm-server` seul (TC_WM_FLAG=$A5, WM_TASKMODE=$00) →
curseur OK.

**Conclusion factuelle** : la migration de `cursor_blit` du contexte
IRQ vers le contexte tâche (via `task_wm`) ne tient PAS sans plan
d'atomicité supplémentaire. Test headless (`test_oricos_taskmode_full`)
mesurait uniquement `CURSOR_OLD_X/Y` (état logique post-blit) — pas
le pixel persisté. Headless vert ≠ interactif vert.

**Hypothèses techniques candidates** (non instruites) :
1. **Préemption T1 mid-blit** : en IRQ, `cursor_blit` était atomique
   (I=1). En task, T1 peut couper save→draw→restore ; un autre code
   écrit dans la zone curseur → backing store corrompu.
2. **Race ZP `WM_ARG_*` / `GFX_*`** : exactement le sujet §3.3a.
   `sys_gfx_*` appelé depuis MainLoop d'une autre task peut stomper
   les ZP que `cursor_blit` utilise.
3. **Backing store re-save avant restore** : events souris en rafale →
   save(A) puis save(B) sans restore intermédiaire → corruption
   permanente du backing.

**Décision** : conformément au §7 « arrêt anticipé » (1 tentative
échouée sur 2 autorisées), **Étape 4 = NON ratifiée**. L'instruction
Étape 4 v2 doit livrer :
- (a) plan d'atomicité explicite (Forbid/Permit insuffisant car ne
  masque pas IRQ — ADR-25) : soit `sei`/`cli` autour de cursor_blit en
  task, soit migration de l'horloge `cursor_blit` vers une chaîne
  d'événements task-only avec collapsing.
- (b) test interactif headless qui mesure le **pixel** (pas l'état
  logique) — e.g. inspection VRAM à un offset précis du curseur après
  séquence d'events, ou snapshot PPM comparé.
- (c) option C (partition ZP) instruite comme repli.

**Pas de revert kernel** : `WM_TASKMODE` reste défini, default `$00`,
flag CLI `--wm-taskmode` conservé comme **harness de debug** pour
investiguer les hypothèses (1)/(2)/(3).

**Coût immédiat** : harnais d'audit du curseur en contexte task à
construire avant toute nouvelle tentative Étape 4.

**Référence implémentation** : ce que GEOS / SymbOS / Intuition font
pour le curseur — à étudier avant Étape 4 v2.

---

## 10. §10 — ZP top-half IRQ dédiée (préparation, attaque ordonnée par risque)

> **Statut §10** : dossier d'instruction ouvert 2026-06-09 suite session
> Bug B + Opt-A + investigation Opt-C bouclée. Élargit la portée du §1
> initial (mouse_step) à **toutes** les routines IRQ top-half. Conditions
> de suivi du sign-off senior intégrées (cf. CR §10).
> Implémentation différée — sprint dédié à programmer.

### 10.1 Pourquoi §10 (et pas juste mouse_step)

Le §1 originel ciblait `kernel_wm_mouse_step` comme suspect n°1.
L'investigation 2026-06-09 montre que **la classe est plus large** :
toute routine appelée en contexte IRQ top-half qui écrit des slots ZP
partagés avec un syscall body est sujette à la même classe de bug.

La synergie `cop_handler cli` + `kernel_forbid` (cf. CR §10 du dossier
`CR_reentrance_irq_syscall_confirmed.md`) **n'évite PAS la collision
ZP** — elle évite la préemption tâche↔tâche. La collision survient
quand l'IRQ écrit dans la ZP scratch puis revient au syscall body
qui s'attendait à trouver autre chose.

Opt-A (sei ciblé) couvre les 2 sites empiriquement déclencheurs
(`sys_gfx_fill_rect`, `sys_win_flush`). C'est un **fix de point, pas
de classe** (sign-off senior). Le danger : tests verts → pression sur
§10 disparaît → un futur syscall touchant la même ZP rejoue toute
la session.

**Item §10 = réel, pas « un jour ».**

### 10.2 Audit ZP partagés — ordonné par risque (à valider)

**Légende risque** : 🔴 = collision empiriquement observée ou très
probable ; 🟠 = collision théorique forte ; 🟡 = à vérifier ; 🟢 = OK
(slot non partagé ou pattern save/restore documenté).

| Slot | Adresse | IRQ-side writers | Syscall-side writers | Risque | Notes |
|---|---|---|---|---|---|
| `WM_DP_TMP` | $20–$21 | `kernel_wm_offset` (callee de mouse_step IRQ si taskmode=$00, de chrome_hit, etc.) | `kernel_wm_offset` (callee de `kernel_gfx_window_base` ← `sys_gfx_fill_rect`) | 🔴 | Collision empiriquement la plus probable. Identifié par audit Agent. |
| `WM_ARG_X/Y/W/H` | $14–$1B | `kernel_wm_redraw_drag` IRQ (taskmode=$00), `_wm_capture_focused_rect` | `sys_win_create`, `sys_gfx_fill_rect` via `kernel_gfx_*` | 🔴 | Sites tâche n'ont pas le `sei` Opt-A → bug latent si tests futurs. |
| `GFX_BASE_*` / `GFX_ARG2_*` / `GFX_ARG3_*` / `GFX_COLOR` | $70–$78 | `kernel_wm_redraw_drag`, `kernel_wm_compose` (potentiellement appelés en IRQ via mouse_step) | `sys_gfx_*` syscalls (tous) | 🟠 | ADR-27 §0quater C-2 `_gfx_xvga_bpl_guard` est un précédent reconnu mais partiel. |
| `GFX_BPL` / `GFX_ARG4_*` | $90–$93 | `kernel_gfx_finish`, `kernel_gfx_window_base` (callees IRQ taskmode=$00 + syscall) | `sys_gfx_fill_rect` (save/restore explicite documenté) | 🟡 | save/restore via stack dans le syscall = mitigation locale. À auditer si IRQ écrit dedans. |
| `EVT_TMP` | $6E | `_evt_tail_offset` (callee de `kernel_event_push_*` IRQ) | `_evt_tail_offset` (callee de syscalls postant des events) | 🟡 | Commentaire historique `kernel.s` ligne `GFX_ARG4_LO=$92 — PAS $6E (=EVT_TMP scratch IRQ)` → consciousness pré-existante mais peut-être incomplète. |
| `SCHED_PTR` | $2C–$2E | `kernel_tcb_ptr` (callee de `do_switch`, `kbd_wake`, `event_wake`, `raw_wake`, `sleep_tick`) | `kernel_tcb_ptr` (callee de syscalls touchant TCB : `sys_exit`, `sys_main_loop`, `sys_alert`, etc.) | 🟠 | `kernel_raw_wake` (event.s:625-638) save/restore SCHED_PTR explicite → précédent reconnu, à étendre aux autres. |
| `SCHED_CAND` / `SCHED_TMP` | $2F / $30–$31 | `kernel_sched_find_next`, `kernel_tcb_ptr`, `kernel_bitmap_*` | idem callees côté syscalls | 🟡 | Idem SCHED_PTR. À auditer pareil. |
| `DP_KBD_TMP` | $12 | `kernel_kbd_ring_push` (callee IRQ) | `kernel_kbd_ring_pop` (callee `sys_read_char`) | 🟢 | **Précédent traité** : `kbd_ring_pop` a son propre `php/sei…plp` (cf. `kbd.s:101-107`). Modèle de mitigation acceptable mais par site. |
| `DP_PTR` | $08–$0A | (a priori non touché en IRQ) | `kernel_app_spawn`, `kernel_print_*`, etc. | 🟢 | À reconfirmer par grep. |
| `DP_SYS_ARG_X` | (à retrouver) | (non IRQ) | cop_handler stocke arg1 ici | 🟢 | Posé par cop avant dispatch, consommé par syscall. Sous FORBID. |

**Audit à compléter avant attaque** : pour chaque ligne 🟠 ou 🟡 ci-dessus,
grep exhaustif `sta <SLOT>`/`stx`/`sty`/`stz` côté IRQ-callable
(routines appelées dans `kernel_irq_handler` ou ses callees récursifs)
et côté syscall-callable (routines appelées par `cop_handler` ou
callees récursifs). Lister, croiser, classifier 🔴/🟠/🟡/🟢.

### 10.2bis Audit invalidé 2026-06-09 (deux faux candidats consécutifs)

⚠️ **Le tableau §10.2 ci-dessus était basé sur lecture statique.** Audit
contradictoire 2026-06-09 a invalidé deux entrées clés :

- **GFX_ARG2_LO 🔴** : marqué comme collision empiriquement la plus
  probable. **Faux pour la classe Opt-A/clock**. Vérification senior
  (`handlers.s:178-188`) : `kernel_irq_handler` skip `mouse_step` si
  pas d'event MOU2 (`lda MOU2_STATUS / and #$80 / beq irq_no_mou`).
  Or `test_oricos_clock` *n'injecte aucune souris*. Donc `redraw_drag`
  jamais exécuté, `GFX_ARG2_LO` jamais écrit côté IRQ dans le bug
  historique. **C'est une collision réelle d'une CLASSE DIFFÉRENTE**
  (instance souris-drag), pas le slot Opt-A.

- **EVT_TMP 🟡** : commentaire « consciousness pré-existante mais
  peut-être incomplète ». **Faux : déjà gardé**. Vérification senior
  (`event.s:340-348`) : `kernel_event_pop` fait `sei` explicite autour
  de son usage d'EVT_TMP (commentaire « section critique vs IRQ
  producteur »). Bouger `$6E→$E2` serait **un no-op pour le bug clock**.

**Leçon** : un slot inféré par grep est un candidat, pas une collision.
La vraie collision Opt-A/clock est **mesurée** et reste à identifier
empiriquement (pad-shift + log ZP-IRQ pendant `sys_win_flush`).

### 10.3 Squelette ZP layout cible (proposé — à re-cibler après mesure)

```
$00–$7F  user/kernel scratch existant (à conserver)
  $08–$0A  DP_PTR (kernel, syscall context)
  $12      DP_KBD_TMP (legacy — peut bouger en $Exx)
  $14–$1B  WM_ARG_X/Y/W/H (syscall context)
  $20–$21  WM_DP_TMP (syscall context)
  $25–$2A  WM_CRH_TMP (syscall context)
  $2C–$31  SCHED_* (syscall context — à déplacer vers IRQ-only ?)
  $6E      EVT_TMP — ⚠ INVALIDÉ 2026-06-09 : déjà gardé par sei dans
           kernel_event_pop (event.s:340). Pas un slot Opt-A.
  $70–$78  GFX_* (syscall context)
  $90–$93  GFX_BPL/ARG4 (syscall context)
$80–$DF  zone libre / userland app imag-regs llvm-mos
$E0–$EF  ★ NOUVEAU : ZP réservée top-half IRQ ★
  $E0–$E1  IRQ_WM_DP_TMP (clone usage de WM_DP_TMP côté IRQ)
  $E2      IRQ_EVT_TMP (anciennement $6E)
  $E3      IRQ_KBD_TMP (clone DP_KBD_TMP côté IRQ)
  $E4–$E6  IRQ_SCHED_PTR (clone SCHED_PTR côté IRQ ; ou tout déplacer ici)
  $E7      IRQ_SCHED_CAND
  $E8–$E9  IRQ_SCHED_TMP
  $EA–$EF  réserve / extension future
$F0–$FF  (existant ?)
```

**Invariant à acter (cf. sign-off senior, item « invariant ZP IRQ »)** :

> **« Aucune routine en contexte IRQ top-half (`kernel_irq_handler` ou
> callee récursif) n'écrit hors de $E0–$EF en zone page directe. »**

Audit-smart à étendre pour catcher les violations futures
(`tools/audit-irq-zp.py` — nouveau script CI).

### 10.3bis Plan v2 — promotion pad-shift comme instrument principal (2026-06-09)

Senior recadre l'ordre :

1. **Mesurer d'abord, deviner jamais**. Le pad-shift method
   (`inject-pad-shift.py` initialement classé « référence nightly » au
   §10.5) **devient l'instrument principal**, parce que c'est le seul
   qui reproduit la condition historique du bug Opt-A/clock :
   - aucune injection MOU2,
   - shift CODE +N octets devant un point stable (e.g.
     `kernel_wm_redraw_drag` ou autre).
2. **Instrumenter les écritures ZP par contexte IRQ pendant `sys_win_flush`**
   (et `sys_gfx_fill_rect`) : snapshot ZP $00-$FF avant/après chaque
   step CPU sous IRQ, log changes avec PC. Croiser avec les LECTURES
   des syscalls. Slot vu écrit-par-IRQ-puis-relu-par-syscall = vraie
   collision.
3. **Sentinel sur CE slot mesuré** — pas un slot déduit. Critère ROUGE
   = `sys_win_flush` sans SEI + pad-shift contradictoire ⇒ détection.
4. **§10 premier move sur CE slot** — rouge → move → vert end-to-end.

L'instance souris/`GFX_ARG2_LO` reste un *second* test (classe valable)
mais n'est pas le garde du bug Opt-A historique.

### 10.4 Plan d'attaque ordonné (à re-cibler après mesure §10.3bis)

**Étape 1 — Audit complet (1 jour)**
- Grep exhaustif IRQ-callable vs syscall-callable pour tous les slots
  ZP du kernel. Produire un tableau `ZP_AUDIT_IRQ.md` qui consolide
  10.2 ci-dessus avec données vérifiées.
- Output : décision par slot — déplacer en $Exx, ou save/restore par
  site, ou laisser (slot non partagé).

**Étape 2 — Test-position-shift land (½ jour — voir 10.5)**
- À landger **AVANT** Étape 3 pour avoir le filet en place pendant
  les refactors. Sweep tailles × périodes T1.

**Étape 3 — Déplacement physique (1 jour)**
- Pour chaque slot classé « migration » à l'Étape 1 : ajouter
  symbole IRQ_xxx dans `kernel.s`, remplacer les usages côté IRQ-callable.
- Compiler après chaque slot, run `make test-position-shift` + suite
  complète. Petits commits atomiques.

**Étape 4 — Retrait Opt-A (bonus)**
- Une fois §10 livrée, le `sei`/`cli` sur `sys_gfx_fill_rect` +
  `sys_win_flush` (commit `5017990`) devient redondant. Retirer pour
  économiser cyc d'IRQ latency.

**Étape 5 — Ratification + invariant acté**
- ADR ratification (sortie DRAFT). Doc invariant ZP IRQ ajoutée à
  `OricOS/CLAUDE.md`. Script `audit-irq-zp.py` ajouté à `make
  audit-smart`.

### 10.5 `make test-position-shift` — **v1 SCAFFOLDING (détection non démontrée)** ⚠️

**Statut** : v1 = scaffolding seul, **capacité de détection non démontrée**.
Validé 2026-06-09 par run contradictoire : `sei`/`cli` retirés de
`sys_gfx_fill_rect` (collision Opt-A *prouvée*), kernel rebuild, test
relancé ⇒ **toujours vert** sur les 18 runs (2 cibles × 9 phases). Aucune
pression CI sur §10 actuellement.

Cause racine (audit senior `wm.s:2124+ _wm_mouse_step_body`) : le stimulus
v1 `mouse2_move_abs(POISON)` est un **move nu sans bouton** ⇒ branche
`motion-seule` ⇒ `cursor_blit` seul ⇒ **aucune écriture `WM_ARG_*`**. Les
écritures en collision (`sta WM_ARG_X/Y`) sont sur le chemin clic/drag
(bouton tenu, `wm_step_pressed`). Le bug d'origine était un **drag**, pas
un move.

Cible Makefile et source en place (`test-position-shift` / `test-oricos-position-shift`,
`Phosphoric/tests/integration/test_oricos_position_shift.c`) ; intégré
dans `make tests`. La structure reste exploitable pour v2.

**Mécanisme v1** (différent du draft initial — pas de pad ca65 ni de
rebuild kernel) : sweep injection-phase MOU2 × cible syscall.
PC-hook one-shot sur entrée `kernel_X` armé seulement après le
premier tick clock (système stable). Delivery MOU2
(`mouse2_move_abs(POISON, POISON)`) différée de `phase` cycles dans
la run loop. Canary = présence de `clock: done` dans le text buffer
Oric 1 ($BB80-$BFFF).

**Pourquoi MOU2 et pas T1** : T1 alimente le tick scheduler ; le
manipuler casse la canary uniformément ⇒ faux négatif. MOU2 est
asynchrone, non critique pour boot, et c'est l'IRQ qui dans le bug
réel (audit §3.3a) écrit dans `WM_ARG_*`, `GFX_*`, `EVT_TMP` pendant
le body syscall.

**Run de validation contradictoire (2026-06-09)** :

| Run | Cible | Attendu | Observé | Verdict |
|---|---|---|---|---|
| Opt-A en place | `kernel_gfx_fill_rect` | PASS-all | PASS-all | trivial (SEI masque) |
| **Opt-A retiré (sei/cli commentés)** | `kernel_gfx_fill_rect` | **FAIL ≥1 phase** | **PASS-all** | ⚠️ **stimulus n'exerce pas la collision** |

Le sweep arme `mouse2_move_abs` (move nu), mais la collision `WM_ARG_*` est
sur le chemin clic/drag de `_wm_mouse_step_body`. v1 ne touche jamais la
branche en collision.

**Corrections requises pour v2** (consigne senior, 3 points) :
1. **Stimulus = drag** (button-down + 2-3 moves espacés), pas move nu.
   Force la branche `wm_step_pressed` ⇒ écritures `WM_ARG_X/Y` ⇒ collision
   exerçable.
2. **Cible de validation = `kernel_gfx_fill_rect` *sans* SEI**, pas
   `kernel_wm_add`. Collision Opt-A *prouvée* — critère « v2 marche » =
   le drag injecté la fait rougir. Reproduire d'abord la collision connue,
   appliquer à wm_add ensuite.
3. **Détecteur ≠ canary `clock: done`**. La canary détecte un hang ; une
   corruption `WM_ARG_*` peut produire une fenêtre mal placée sans planter.
   Lire `WM_DP_TMP` / rect post-syscall et comparer à l'attendu (état
   mémoire direct, plus sensible).

**Raffinements secondaires v2** :
- Plage phase 0-256 step 4 pour aligner la fenêtre une fois la bonne
  branche exercée.
- Confirmation que l'IRQ est *réellement* prise dans le body (cpu.irq
  asserté + I=0 au moment voulu).
- Extension cibles selon §10.2 *après* validation gfx_fill_rect.

**Précédent existant à noter (gain pour §10)** : `kernel_wm_mouse_step` fait
déjà save/restore de `GFX_BPL_LO/HI` autour du body. §10 généralise ce
principe à `WM_ARG_*`/`WM_DP_TMP` — pattern existant, pas mécanique
inventée.

**Garde de progression** : ne RIEN pousser de v2 tant que
`kernel_gfx_fill_rect` *sans SEI* n'a pas été observé ROUGE. Tant que ce
rouge n'est pas obtenu, le test ne garde rien.

**Référence nightly** : `inject-pad-shift.py` (forme draft initial) reste
pertinente pour répondre à la question complémentaire « la croissance
réaliste du kernel reproduit-elle le bug ? » — ground-truth occasionnel,
pas par-run-CI.

**Sprint v1** : scaffolding livré.
**Sprint v2** : à faire — pas livré tant que stimulus drag n'a pas
démontré ROUGE sur gfx_fill_rect-sans-SEI.

**Pourquoi NOW (sign-off senior insistance)** : tests verts aujourd'hui
sans CI dédiée → pression sur §10 disparaît → futur syscall ou layout
CODE rouvre la classe. CI = visibilité = pression maintenue.

**Forme proposée** :
```bash
# Nouveau target Makefile Phosphoric
test-position-shift:
        @for PAD in 16 32 50 64 80 100 128 200; do \
          for T1_PERIOD in DEFAULT DEFAULT+13 DEFAULT-7; do \
            ./tools/inject-pad-shift.py $(PAD) $(T1_PERIOD); \
            make -C ../OricOS clean all; \
            ./test_oricos_helloc; \
            ./test_oricos_wm_cost; \
          done; \
        done
        @echo "Position-shift sweep OK : $$(NCOMBOS) combos passent"
```

`inject-pad-shift.py` insère temporairement `.proc _pad_test / .res N,
$EA / .endproc` à un point stable de `wm.s` (avant `kernel_wm_redraw_drag`)
puis le retire. Idempotent.

**Extension à prévoir** : ne pas se limiter aux 2 syscalls corrigés
Opt-A. Le sweep doit aussi exercer `sys_gfx_blit`, `sys_gfx_line`,
`sys_gfx_text`, `sys_win_create`, `sys_ctl_*`, etc. — pour révéler
quel autre syscall casse en premier sous shift.

**Sprint** : ½ jour, indépendant de §10 Étape 1. À landger en parallèle.

### 10.6 Items collatéraux à acter

**(a) Course `KBD_WAITER` exempt↔focus** (sign-off senior, ligne ADR)

Avec un seul slot `KBD_WAITER`, une tâche exempte (no-window, e.g.
`task_e`) et la tâche focus se disputent les touches selon le timing.
La synergie cli/FORBID résout la course en faveur du focus dans le
cas v1 grâce à la non-préemption pendant un chain
`flush→print→read_char`. **Est-ce intentionnel ?**

Options :
- **(α)** Intentionnel : task_e est un lecteur kbd non-GUI légitime,
  prioritaire quand il y est avant le focus. Documenter.
- **(β)** Bug : remplacer `KBD_WAITER` slot unique par une **liste**
  de waiters, route systématiquement vers focus quand un GUI waiter
  existe, exempt en fallback. Plus complexe.
- **(γ)** Hybride : `KBD_WAITER` reste unique mais `kbd_wake` filtre
  : si un GUI waiter est en attente (à tracker via tampon),
  prioriser ; sinon délivrer à l'exempt.

**Décision attendue** : ligne dans ADR-32 §10 actant l'intention.
Pas bloquant pour §10 ZP, mais à acter dans le même sprint pour
clarté.

**(b) RAW_RING coalesce-on-overflow** (sign-off senior, suggestion non-bloquante)

Quand RAW_RING est plein (16 slots), au lieu de drop silencieux :
fusionner les MOVED les plus anciens (delta cumulé, accepter un saut
ponctuel). Préserve l'absence de fragments en régime normal
(Fix B v2) ET pas de perte de mouvement sous burst.

Coût : ~15-20 octets ajoutés à `kernel_raw_push_mouse`. Risque :
re-introduit potentiellement le pattern de gros delta sous burst,
qui était la cause originale du Bug B en taskmode. À évaluer.

**Décision attendue** : option ouverte, pas dans §10 v1. Reprise
quand un cas burst réel est observé.

### 10.7 Conditions de suivi (du sign-off senior, à tracker)

- [ ] **§10 inscrit comme item réel** (pas « un jour ») — ce document
      est l'instrument. Suivi : reprise active dès qu'un syscall
      non-Opt-A déclenche la classe, ou trimestre Q3 2026 plancher.
- [ ] **Invariant ZP IRQ acté en ADR ratifié** (cf. 10.3) — partie
      intégrante de la ratification §10.
- [ ] **`make test-position-shift` landé maintenant** (cf. 10.5) —
      **v1 = scaffolding seul (2026-06-09)**, capacité de détection
      **non démontrée** (run contradictoire : gfx_fill_rect-sans-SEI reste
      vert). v2 requis avec stimulus drag + détecteur mémoire direct +
      cible de validation gfx_fill_rect-sans-SEI ⇒ ROUGE avant push.
- [ ] **Ligne ADR sur course exempt↔focus** (cf. 10.6.a) — à acter
      avant ratification §10.
- [ ] **`coalesce-on-overflow` RAW_RING** (cf. 10.6.b) — optionnel,
      reprise sur observation cas burst.
- [ ] **Retrait Opt-A post-§10** (cf. Étape 4) — bonus.

### 10.8 Réf croisée

- `docs/CR/CR_reentrance_irq_syscall_confirmed.md` §10 : trace A/B
  invalidation Opt-C, mécanisme cli/FORBID synergique.
- `docs/notes/HANDOFF_inge_senior_Bug_B_session.md` : synthèse session
  complète.
- OricOS `02934d2` (Fix B v2), `5017990` (Opt-A).
- Audits Agent v1+v2 (collisions ZP) : reproductibles via `make
  audit-smart` extended.

### 10.9 RÉSOLUTION de la chasse au slot (2026-06-10) — la collision n'est PAS en ZP

> **Statut : cause racine du bug clock Opt-A MESURÉE et PROUVÉE.**
> Le « slot » de collision n'est pas un slot mémoire : c'est le
> **registre B (octet haut de l'accumulateur A) à travers la frame
> IRQ 8-bit de `kernel_irq_handler`**.

**Méthode (§10.3bis appliqué — mesurer, jamais deviner)** :

1. Repro de la condition historique : `inject-pad-shift.py --apply 120`
   + `sei`/`cli` Opt-A commentés ⇒ `test_oricos_clock` FAIL avec la
   signature exacte (`tick=1 cyc=900001`). ✓
2. Logger ZP-IRQ (`test_oricos_zp_log`, Phosphoric) : snapshot
   ZP $00-$FF + stack bank 0 + bank 1 à chaque step pendant les bodies
   `sys_win_flush`/`sys_gfx_fill_rect`, diff post-step, classement par
   contexte (IRQ vs tâche). **Aucune écriture ZP par l'IRQ pendant les
   bodies** — l'hypothèse « collision slot ZP » est invalidée pour CE
   bug (elle reste valable comme classe pour l'instance souris/drag).
3. Watchpoints différentiels (TCB pid 8, `CURSOR_ADDR` $019090) +
   trace registres fine. **Preuve** :
   - cyc=207832 : tâche clock (pid 8) dans `kernel_print_char` pc_lf,
     `PC=01:11AD`, **C=$BC98, M=16** (P=$91, fenêtre `rep #$20 …
     lda/adc/sta CURSOR_ADDR`).
   - cyc=207839 : IRQ T1 prise (vecteur $FF00), C=$BC98 intact.
   - cyc=208408 : RTI ramène au même PC=01:11AD avec **C=$5C98** —
     l'octet haut de A vaut la valeur que le code IRQ a laissée dans B.
   - Conséquence : `sta CURSOR_ADDR` écrit $5C99 → tous les prints
     suivants partent hors écran ($5C99+ au lieu de $BC99+) →
     « clock: done » jamais visible. **L'app NE hang PAS** : elle
     termine ses 16 ticks et exit proprement (TCB pid 8 DEAD à
     cyc=251927 par `sys_exit`) — c'est l'écran qui ne reçoit plus rien.

**Mécanisme** : `kernel_irq_handler` ouvre par `sep #$30` puis
`pha/phx/phy` **8-bit** (handlers.s). Le `pla` final ne restaure que
A.low ; B conserve ce que le code IRQ y a laissé. Toute tâche ou
syscall interrompu **en M=16 entre un load et un store** reprend avec
A.high corrompu. Le risque était documenté en tête de `handlers.s`
(invariant IRQ_CONFORMITE §3.3 A) mais l'audit ne couvrait que
**X/Y 16-bit** — le cas **A/M=16** n'était ni listé ni couvert par
l'invariant « X=1 aux points préemptibles ». Le site déclencheur
(`kernel_print_char` avance/LF, console.s) n'était pas dans le tableau
d'audit.

**Pourquoi position-dépendant (pad +120)** : la corruption exige que
l'IRQ T1 tombe dans une fenêtre de quelques cycles (`rep #$20` …
`sta`). Le pad décale la phase relative boot↔T1 et aligne la fenêtre.
**Pourquoi Opt-A « réparait »** : les `sei` sur fill_rect/flush
décalaient le timing — l'IRQ ne tombait plus dans la fenêtre du print.
Fix de point par accident, exactement ce que le sign-off senior
craignait (§10.1).

**Découvertes collatérales (à traiter, hors bug principal)** :
- **(a) `TICK_COUNTER = $015500` ⟂ segment `NMI_HANDLER` ($5500)** :
  la variable et le `rti` du handler NMI occupent la MÊME adresse —
  chaque tick écrase l'opcode `rti`. Bénin v1 (aucune source NMI
  câblée) mais bombe à retardement : tout NMI exécuterait la valeur du
  compteur comme opcode. Réassigner TICK_COUNTER ou déplacer le
  segment.
- **(b) Harness Phosphoric : lire $0100-$0FFF via `memory_read` route
  $0300-$03FF vers l'I/O** — la lecture de $0304 (T1C-L) ACQUITTE
  l'IRQ T1. Le logger v1 mangeait une IRQ sur deux et faisait
  disparaître le bug qu'il mesurait. Leçon harness : tout snapshot
  mémoire doit exclure la fenêtre I/O. (Corrigé dans
  `test_oricos_zp_log.c`.)
- **(c) `TCB_BITMAP` ($015B20)=$00EF en fin de run cassé** : slot 4
  (pid 5, BLOCKED) marqué libre. Non investigué — à vérifier après le
  fix principal (peut être une conséquence du même mécanisme).

**Options de fix (à arbitrer — refactor structurel, confirmation
humaine requise par CLAUDE.md OricOS §5)** :
- **(A) Option B des sources** (pré-tracée en tête de handlers.s) :
  `kernel_irq_handler` save/restore **16-bit** (`rep #$30` +
  `pha/phx/phy`). Fix de CLASSE (couvre A, X, Y d'un coup, ferme aussi
  les 3 sites « RÉEL » de l'audit X/Y). Coût : le format de frame
  change → adaptation atomique de TOUS les forgeurs/consommateurs de
  frame (`kernel_task_create`, `sys_yield`, `do_switch`,
  `kernel_block_switch`). Multi-fichiers, ~1 jour + tests.
- **(B) Sauver B seul dans la frame** (`xba`/`pha` x2) : +1 octet de
  frame, même contrainte d'atomicité forgeurs, ne couvre pas X/Y
  (couverts par l'invariant X=1 existant — mais invariant fragile).
- **(C) Étendre l'invariant aux régions M=16** : wrapper `sei`/`cli`
  chaque séquence `rep #$20` préemptible (console.s print_char ×3,
  scroll_up, + audit complet wm.s/gfx.s). C'est le « patchwork sei
  ad-hoc » que ce dossier (§1) qualifie d'insoutenable — déconseillé.

**Recommandation senior tracée : (A)** — c'est l'option que l'auteur
du handler avait lui-même pré-identifiée, elle ferme la classe entière
(A+X+Y) au lieu du point, et elle rend caducs les wrappers sei/cli
(c) ET l'invariant X=1 (simplification nette).

**Test de verrouillage** : `test_oricos_irq_frame_m16` (Phosphoric) —
détecte la corruption de C à travers une IRQ prise en M=16 (snapshot C
à la prise, comparaison au retour au PC interrompu). **ROUGE démontré
sur le kernel actuel** (condition mémoire
`feedback_test_definition_of_done` satisfaite) ; passera VERT avec le
fix. Hors `make tests` tant que le fix n'est pas landé.

**Impact sur ce dossier** : §10.2/§10.3 (layout ZP $E0-$EF) restent
pertinents pour la classe ZP souris/drag, mais ne sont PLUS le chemin
du bug clock Opt-A. Le retrait d'Opt-A (Étape 4) devient possible dès
que le fix (A) est landé et que `test_oricos_irq_frame_m16` +
pad-shift ground-truth passent.

---

*Section §10 ajoutée 2026-06-09 suite sign-off senior. Conditions de
suivi tracées. §10.9 (résolution chasse au slot) ajoutée 2026-06-10 :
cause racine mesurée = frame IRQ 8-bit vs contexte M=16.*
