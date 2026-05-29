# ADR-29 — Sémantique des messages pendant un drag de contrôle valeur

- **Statut** : **ratifiée** (2026-05-30) — option **C** retenue (hint
  déclaratif aligné GeoWorks, default `HINT_VALUE_DELAYED_DRAG_NOTIFICATION`,
  override IMMEDIATE via flag global `WM_DRAG_NOTIFY_HINT`). Implémentation
  Étape 1 livrée, **validation interactive utilisateur positive**
  (`--ctl-demo` : scrollbar fluide, value 1:1 visuellement, app reçoit 1
  `MSG_CONTROL` à la release au lieu de 60+/sec, FORBID_COUNT se libère
  normalement, curseur propre après fix `_wm_redraw_ctl`). Conforme moratoire
  CLAUDE.md §10 (audit §8 ci-dessous).
- **Date d'ouverture** : 2026-05-30
- **Date de ratification** : 2026-05-30
- **Décideurs** : bmarty (validation interactive), Claude Code (instruction +
  implémentation)
- **Contexte technique** : suite directe de la **rétractation §1.2ter et §6.7**
  d'ADR-28 (la « famine » imputée à une saturation du ring n'est en réalité
  pas le bon diagnostic). Bug interactif observé sur `--ctl-demo` : pendant
  un drag de scrollbar, la `value` se fige et la GUI devient non-réactive,
  même après release du bouton. ADR liées : ADR-25 (concurrence,
  `FORBID_COUNT`), ADR-26 (modèle GUI déclaratif GenUI/SpecUI), ADR-28
  (threading WM, ratifié mais §6.6/§6.7/Étape 3 partiellement revertés).
- **Conformité moratoire** : oui — audit §8 ci-dessous (3 conditions
  CLAUDE.md §10 vérifiées).

## 0. Résumé exécutif

Le bug interactif « fin de course d'ascenseur » observé sur `--ctl-demo` n'est
**pas** un drop de button-UP (réfuté par `mtrace3.log`, l'UP est posté dans
`EVENT_RING`), **pas** un overflow du ring, **pas** un problème de coût IRQ.

C'est un **bottleneck applicatif** localisé avec précision dans `mtrace4.log` :
- L'app `ctl_demo` reçoit `MSG_CONTROL` à chaque `EV_MOUSE_MOVED` pendant le
  drag (60 events/seconde via SDL).
- Pour chaque message, elle imprime « `ctl: v=XX\r\n` » (10 syscalls COP).
- Une fois l'écran texte plein, chaque `\n` déclenche `kernel_scroll_up`
  (1080 octets copiés, ≈ 6 500 cyc).
- `FORBID_COUNT` reste **bloqué à 1** car `sml_loop` ne libère le forbid qu'au
  `sml_block` (atteint uniquement si la queue est vide).
- L'app monopolise le CPU à 100 %, la queue se remplit, la value cesse d'être
  mise à jour. **Mesure mtrace4 : CPU à `01:11EE` (`kernel_scroll_up.bcc
  scrl_copy`) stable pendant 200+ frames**.

Le diagnostic révèle un **manque de contrat sémantique** sur ce que l'app
reçoit pendant un drag : aujourd'hui elle reçoit *tout*. Les OS de référence
(GeoWorks/PC GEOS, SymbOS, Intuition/Amiga) ont tous résolu ce problème par
le même pattern : **ne notifier l'app qu'à la release**, sauf opt-in explicite
au mode temps réel.

L'option retenue (C, alignement GeoWorks) introduit un **hint déclaratif** sur
les widgets `GenValue` (scrollbars, views, futurs sliders), avec **default
`HINT_DELAYED_DRAG_NOTIFICATION`** : l'app reçoit `MSG_CONTROL` **une seule
fois** à la release, avec la value finale. Le visuel du widget reste mis à
jour à chaque MOVED par le kernel (`_wm_scroll_update` → `_wm_redraw_ctl`,
déjà en place). Apps qui veulent un feedback live (préview de scroll, etc.)
opt-in explicitement.

## 1. Contexte chiffré (mtrace4)

Reproduction : `./oric1-emu --kernel ../OricOS/build/kernel.bin --xvga --ctl-demo --mouse-trace`.
Scénario utilisateur : clic en haut du scrollbar, drag vers le bas en
dépassant le bord de la fenêtre, release en dehors, tentative de re-clic.

**Trace `/tmp/mtrace4.log` (extrait représentatif)** :

```
[mtrace f=293] value(scrollbar id=03) 33 → 34   [mouse2 x=355 y=248 btn=01]
[mtrace f=293]   scheduler : TASK_CUR=2 FORBID=0 EVENT_WAITER=0
[mtrace f=294] value(scrollbar id=03) 34 → 36   [mouse2 x=355 y=250 btn=01]
[mtrace f=294] FORBID_COUNT 0 → 1 (CPU at 01:4203)
[mtrace f=295] EVENT_RING count=1 tail=14   ← queue cesse de se vider
[mtrace f=434] FORBID stuck 5 frames | CPU at 01:11EE (was 01:11EB)
[mtrace f=479] FORBID stuck 50 frames | CPU at 01:11EE (was 01:11EB)
[mtrace f=629] FORBID stuck 200 frames | CPU at 01:11EE (was 01:11EB)
```

**Lecture** :
1. Jusqu'à f=294, l'app traite chaque event à temps : value augmente
   linéairement de 1 à 36, queue reste vide (`count=0`).
2. À **f=294**, `FORBID_COUNT 0 → 1` se déclenche (sortie de `sml_block` via
   `sml_resume`). À partir de là, FORBID **reste à 1 pendant tout le reste de
   la session**.
3. Le PC CPU oscille entre `$01:11EB` (`cpx #1080`) et `$01:11EE`
   (`bcc scrl_copy`) — la **boucle interne de `kernel_scroll_up`** dans
   `console.s`. Stable pendant 200 + frames, soit ≈ 4 secondes.
4. Pendant ce temps, `EVENT_RING_COUNT` monte régulièrement (1, 2 à f=396 UP
   posté, 3 à f=719, …, 11 en fin de session). Aucun event n'est popé. L'app
   n'avance pas.
5. `SCROLL_DRAG_ID` reste à `03` (jamais cleared par `mlc_up`, qui n'est
   jamais atteint).

**Cause racine** : l'app `ctl_demo` exécute 10 syscalls `SYS_PRINT_CHAR` à
chaque `MSG_CONTROL`. Après que l'écran texte est plein, chaque `\n` provoque
un `kernel_scroll_up` (1080 octets, ≈ 6 500 cyc). À 60 events/sec, ça coûte
≈ 390 000 cyc/sec, sur un budget de 1 MHz — soit 39 % CPU sur le print seul,
sans compter le `scroll_update` + `redraw_ctl`. Cumulé, l'app ne tient pas la
cadence, queue se remplit, `sml_block` jamais atteint, `FORBID` jamais
relâché.

**Le bug n'est ni un drop d'event, ni un coût IRQ excessif**. C'est un
**contrat sémantique mal calibré** : l'app est noyée par un flux qu'elle
n'avait pas demandé et que **rien ne l'oblige à recevoir**.

## 2. Problème

### 2.1. Formulation

L'API actuelle d'OricOS notifie l'app par `MSG_CONTROL` à **chaque
`EV_MOUSE_MOVED`** dès lors qu'un widget scrollbar/view est en cours de drag.
Aucun moyen pour l'app de déclarer « notifie-moi seulement à la fin du drag »
— pourtant ce mode est :
- Le **default** historique des OS de référence (GeoWorks ≥ 1.0, SymbOS,
  Intuition Boopsi).
- **Suffisant** pour la majorité des apps (qui ne font rien d'utile sur
  chaque update intermédiaire — ctl_demo en est un exemple direct, elle
  imprime juste « ctl: v=XX »).
- **Plus sûr** sous toute combinaison `print → scroll_up` ou
  `redraw → fill_rect` lente : pas de famine.

### 2.2. Pourquoi ce n'est pas un simple bug d'app

`ctl_demo` est dans son droit de print à chaque `MSG_CONTROL` — l'API ne
documente aucune contrainte. Si l'app était corrigée (ne pas print pendant le
drag), le bug existerait encore pour la prochaine app naïve. Le **fix
structurel** est dans l'API, pas dans l'app.

C'est exactement le raisonnement appliqué par GeoWorks il y a 30 ans sur un
8088 à 8 MHz : on ne peut pas demander à chaque dev d'app de connaître les
subtilités de l'event flow et d'éviter les hangs sous flux dense. Le système
doit fournir un default sûr.

## 3. Options envisagées

### Option A — Statu quo (ne rien faire)

Le bug persiste. Documenter dans la doc apps que les handlers `MSG_CONTROL`
doivent être « rapides » (mais sans définition de « rapide »).

- **Coût** : 0.
- **Bénéfice** : 0. Le bug reste reproductible sur la majorité des apps qui
  font du logging/feedback dans `MSG_CONTROL`.
- **Risque** : crédibilité de l'OS (« le scrollbar marche pas »).
- **Réversibilité** : trivial.

**Écartée.**

### Option B — Pattern SymbOS (notify-on-release fixe)

Toujours notifier l'app uniquement à la release du bouton. Pas d'opt-in pour
le mode live. Implementation triviale : `mlc_moved` retourne toujours
`MSG_NULL` quand `SCROLL_DRAG_ID` est armé, `mlc_up` retourne `MSG_CONTROL`
avec la value finale.

- **Coût** : ≈ 10 lignes asm dans `_ml_classify`.
- **Bénéfice** : bug éliminé pour 100 % des apps. Simple.
- **Risque** : les apps qui voudraient un feedback temps réel (preview de
  scroll, déplacement d'objet en sync) sont impossibles. Pas d'échappatoire.
- **Réversibilité** : facile (revert localisé).

**Écartée** : trop rigide pour une plateforme qui se veut polyvalente. Et
incohérente avec ADR-26 qui parle de « hints déclaratifs ».

### Option C — Pattern GeoWorks (hint déclaratif, default DELAYED) — **recommandée**

Aligné sur `Include/Objects/gValueC.def` de PC/GEOS :

```
HINT_VALUE_IMMEDIATE_DRAG_NOTIFICATION   vardata
    ;Send out status and/or apply messages constantly during dragging,
    ;if available under the specific UI.

HINT_VALUE_DELAYED_DRAG_NOTIFICATION     vardata
    ;Delay status and/or apply messages until the user releases the mouse
    ;after dragging, if available under the specific UI.
```
(source : [bluewaysw/pcgeos](https://github.com/bluewaysw/pcgeos/blob/master/Include/Objects/gValueC.def) lignes 800-806)

**Pour OricOS** :
1. Ajouter un champ `WG_HINT_DRAG_NOTIFY` (1 octet) à chaque entrée
   `WIDGET_TABLE`. Valeurs : `0 = HINT_DELAYED` (default), `1 = HINT_IMMEDIATE`.
2. Ajouter un tag GenUI optionnel : si l'app veut le mode IMMEDIATE, elle
   pose `GU_HINT_DRAG_NOTIFY_IMMEDIATE` immédiatement avant `GU_SCROLL_V`
   (ou `GU_VIEW`). Sinon, le default `HINT_DELAYED` s'applique.
3. Dans `_ml_classify` (`mlc_moved`) :
   - Lit `WG_HINT_DRAG_NOTIFY` du widget armé.
   - Si `HINT_DELAYED` (0) → renvoie `MSG_NULL` (visuel toujours mis à jour
     par `_wm_scroll_update` côté kernel).
   - Si `HINT_IMMEDIATE` (1) → renvoie `MSG_CONTROL` (comportement actuel).
4. Dans `_ml_classify` (`mlc_up`) :
   - Si `SCROLL_DRAG_ID != $FF` au moment de l'UP → renvoie `MSG_CONTROL` une
     fois avec la value finale (quel que soit le hint). C'est la
     « notification de release » garantie.

- **Coût** : ≈ 30 lignes asm, 1 octet/widget RAM, 1 tag GenUI optionnel.
- **Bénéfice** :
  - Default safe → bug éliminé pour 95 % des apps (celles qui n'opt-in pas).
  - Apps qui veulent du temps réel ont une voie propre (opt-in IMMEDIATE).
  - **Cohérence native** avec ADR-26 (modèle GenUI/SpecUI déclaratif,
    GeoWorks-like).
  - Pattern documenté, validé par 30 ans d'usage sur PC GEOS / GeoWorks
    Ensemble (référence vivante au pattern *« if available under the specific
    UI »*).
- **Risque** : changement de contrat de l'API pour les apps qui supposaient
  recevoir du temps réel par défaut. Mais à ce jour, **aucune app
  d'OricOS** ne le suppose (audit : `ctl_demo`, `gui_demo`, `view_demo`,
  `clock` se limitent à imprimer/redessiner sur reception ; aucune ne fait de
  preview temps réel). Migration triviale.
- **Réversibilité** : élevée (rollback en flippant le default à `IMMEDIATE`).

## 4. Recommandation senior (tracée)

**Viser l'option C** (GeoWorks-aligned, hint déclaratif, default DELAYED).

Justification :
1. **Cause racine traitée** : le bug n'est pas un détail à patcher mais un
   contrat sémantique manquant. C donne ce contrat.
2. **Validation externe** : 30 ans d'usage en production sur GeoWorks/PC
   GEOS, validé sur 8088 à 8 MHz (machines plus contraintes qu'OricOS).
3. **Cohérence intrinsèque** : ADR-26 a déjà choisi le modèle Gen/Spec
   déclaratif et a explicitement référencé GeoWorks. C en est le
   prolongement naturel.
4. **Risque minimum** : aucune app actuelle ne dépend du comportement
   IMMEDIATE par défaut → migration sans casse.
5. **Réversibilité** : flag-gated en Étape 1 pour test interactif
   préalable. Default flippable.

## 5. Conséquences

### 5.1. Positives

- Bug « fin de course ascenseur » éliminé pour le default (et donc pour
  toutes les apps non opt-in).
- API plus claire : `HINT_DRAG_NOTIFY` documenté côté contrat.
- Alignement explicite avec une référence externe vivante et reconnue.
- Décharge l'app du travail défensif d'éviter `MSG_CONTROL` rapides.
- Permet aux futurs apps d'opt-in proprement à du temps réel si besoin.
- `FORBID_COUNT` se libère normalement (la queue se vide rapidement, l'app
  ne traite que 1 message par drag).

### 5.2. Négatives / coûts

- Léger changement de contrat API (`MSG_CONTROL` plus rare pendant un drag).
- Nécessite mise à jour de la doc apps + des exemples.
- 1 octet de RAM par widget supplémentaire.
- Audit nécessaire des apps existantes pour s'assurer qu'aucune ne dépend du
  comportement actuel (audit rapide : aucune ne dépend).
- Le hint n'est pas garanti d'être honoré par tous les futurs specUI (cf.
  *« if available under the specific UI »* dans la source GeoWorks) — c'est
  un hint, pas un contrat dur. À documenter.

### 5.3. Sur ADR-26 (modèle GUI déclaratif)

ADR-29 **renforce** ADR-26 : elle ajoute un hint cohérent avec le pattern
déjà ratifié. Pas de modification d'ADR-26.

### 5.4. Sur ADR-28 (threading WM)

ADR-29 est **orthogonale** à ADR-28. Elle fonctionne en mode legacy
(`TC_WM_FLAG=0`) comme en mode serveur (Étape 2 passe-plat). Si Étape 3 est
reprise un jour proprement, le default DELAYED restera valable (même
sémantique côté app).

Note : la **rétractation de §6.7** d'ADR-28 (quota anti-drop button-UP, qui
fixait un drop qui n'a jamais lieu) est tracée séparément. ADR-29 traite la
**vraie** cause du bug que §6.7 visait incorrectement.

## 6. Points ouverts à instruire avant ratification

1. **Audit interactif des apps** : confirmer qu'aucune app actuelle ne dépend
   du comportement IMMEDIATE actuel (audit rapide pour
   `ctl_demo`/`gui_demo`/`view_demo`/`clock`/`hello_c` : aucune ne le fait,
   mais valider).
2. **Choix du tag GenUI** : `GU_HINT_DRAG_NOTIFY_IMMEDIATE` comme tag
   indépendant **avant** un widget, ou flag inline dans `GU_SCROLL_V` /
   `GU_VIEW` (octet supplémentaire) ? Le premier est plus cohérent avec
   GeoWorks (« hints » comme objets séparés). Le second est plus compact.
3. **Notification finale UP** : doit-elle être garantie même si la value n'a
   pas changé pendant le drag (l'utilisateur a cliqué + release sans
   bouger) ? GeoWorks la garantit. Choix recommandé : oui (cohérence).
4. **Behaviour cross-widget** : le hint doit-il s'étendre aux futurs
   `GenRange` (sliders bornés bas+haut) ? Probablement oui (GenRange hérite
   de GenValue dans GeoWorks, cf. `gRangeC.def`).
5. **Default flippable** : exposer un flag kernel global
   (`WM_DEFAULT_DRAG_HINT`) pour permettre de changer le default à
   `IMMEDIATE` au runtime ? Utile pour expérimentation A/B mais ajoute de la
   surface. **Recommandation** : non, garder le default codé en dur, opt-in
   au widget.

## 7. Plan d'implémentation de référence (incrémental, testable)

### 7.0. Étape 0 — Audit apps + reproduction headless

- Audit grep apps/ pour identifier toute dépendance au pattern IMMEDIATE
  actuel. À ce jour : aucune.
- **Étendre `test-oricos-scroll-burst`** pour vérifier que le pattern
  DELAYED produit exactement **1 `MSG_CONTROL` par drag** (vs N aujourd'hui)
  et que la value finale est correcte à la release.
- Gate : `make tests` vert. Réversible (test seul).

### 7.1. Étape 1 — Implémentation behind a flag (`GUI_DRAG_DELAYED=$A5`)

- Ajout du champ `WG_HINT_DRAG_NOTIFY` à `WIDGET_TABLE` (1 octet, default 0
  = HINT_DELAYED).
- Modification de `mlc_moved` et `mlc_up` dans `_ml_classify` (wm.s).
- Tag GenUI `GU_HINT_DRAG_NOTIFY_IMMEDIATE` ajouté à `oricos.h` et au parseur
  `SYS_UI_DEFINE`.
- **Gated par un flag global** `WM_GUI_DRAG_HINT_DEFAULT` ($01EE70 par ex.),
  posé à `$A5` par défaut au boot (= DELAYED actif), à `$00` pour repli
  IMMEDIATE.
- **Test interactif chez l'utilisateur** avec `./oric1-emu --kernel … --ctl-demo`
  : vérifier que le drag fonctionne fluidement, value se met à jour
  visuellement, app reçoit `ctl: v=XX` une fois à la release.
- Gate : `make tests` vert + retour interactif positif.

### 7.2. Étape 2 — Granularité par widget (alignement GeoWorks complet) — **FAIT 2026-05-30**

Refinement post-ratification (le moratoire §10 est respecté car l'ADR a été
ratifiée sur l'Étape 1 + validation interactive, l'Étape 2 enrichit le
pattern sans modifier la décision d'architecture). Le flag global
`WM_DRAG_NOTIFY_HINT` est **conservé** comme kill-switch debug (override
forcé IMMEDIATE pour tous les widgets), pas retiré.

**Livraison** :
- ✅ **Constantes** (`kernel.s`) :
  - `WIDGET_HINTS = $016320` (8 × 1B, hint par widget,
    0 = `HINT_DRAG_DELAYED` default, 1 = `HINT_DRAG_IMMEDIATE`),
  - `UI_PENDING_HINT = $016328` (1B scratch parser GenUI),
  - `GU_HINT_IMMEDIATE_DRAG_NOTIFY = $0A` (tag GenUI),
  - `HINT_DRAG_DELAYED = $00`, `HINT_DRAG_IMMEDIATE = $01`,
  - `.assert UI_PENDING_HINT < $016400` (garde anti-overlap RAW_RING).
- ✅ **Parser GenUI** (`wm.s sud_loop`) : cas `GU_HINT_IMMEDIATE_DRAG_NOTIFY`
  ajouté entre `sud_n2g` et `sud_n3`. Tag seul (sans data) qui pose
  `UI_PENDING_HINT = HINT_DRAG_IMMEDIATE` puis revient à `sud_loop`.
- ✅ **`kernel_wm_add_widget`** (`tk.s`, _waw_count) : copie
  `UI_PENDING_HINT → f:WIDGET_HINTS,X` (X = id du widget créé), puis reset
  `UI_PENDING_HINT = 0`. Tout widget créé sans tag explicite hérite donc du
  default DELAYED (sûr).
- ✅ **`mlc_moved_go`** (`wm.s`) : consulte `f:WIDGET_HINTS,X` au lieu du
  flag global seul. Override global `WM_DRAG_NOTIFY_HINT=$A5` reste
  prioritaire. Hiérarchie : override global > widget hint > default.
- ✅ **`mlc_up`** (`wm.s`) : même logique. DELAYED → MSG_CONTROL final ;
  IMMEDIATE (widget ou override) → MSG_NULL (app déjà notifiée).
- ✅ **SDK** (`oricos.h`) : `#define GU_HINT_IMMEDIATE_DRAG_NOTIFY 10` exposé
  aux apps userland C avec commentaire d'usage.
- ✅ **Gate** : `make tests` vert (suite complète + scroll-cost passe avec
  value progression 1:1). **Validation interactive utilisateur positive**
  (2026-05-30) : `--ctl-demo` (sans tag → default DELAYED) reste fluide,
  aucune régression par rapport à Étape 1.

**Usage côté app C** :

```c
static const unsigned char ui[] = {
    GU_WINDOW, /*...*/,

    /* Scrollbar A : default DELAYED (sans hint) — sûr pour 95% des apps */
    GU_SCROLL_V, 140, 0, 14, 0, 12, 0, 100, 0, 40,

    /* Scrollbar B : opt-in IMMEDIATE (feedback temps réel) */
    GU_HINT_IMMEDIATE_DRAG_NOTIFY,
    GU_SCROLL_V, 160, 0, 14, 0, 12, 0, 100, 0, 40,

    GU_END
};
```

**Alignement GeoWorks atteint** :

| Élément OricOS | Équivalent PC/GEOS `gValueC.def` |
|---|---|
| Tag `GU_HINT_IMMEDIATE_DRAG_NOTIFY` | `HINT_VALUE_IMMEDIATE_DRAG_NOTIFICATION` |
| Default (sans tag) = DELAYED | `HINT_VALUE_DELAYED_DRAG_NOTIFICATION` |
| Hint placé AVANT un widget value | Hint attaché à l'objet (vardata) |
| `WIDGET_HINTS[id]` byte par widget | Instance vardata par `GenValueClass` |

**Différences assumées** :
- 1 octet de hint vs vardata complète : simplification OricOS (un seul bit
  utilisé v1, extensible à 8 si futurs hints).
- `WIDGET_HINTS` tableau séparé vs champ inline dans le widget : évite
  d'agrandir `WIDGET_ENTSZ` (16 → 18) qui briserait `asl × 4`. Trade-off
  acceptable.

### 7.3. Seuil moratoire

Le seuil 50 % a été atteint à la fin de l'**Étape 1** (implémentation flag
global, suite verte, validation interactive utilisateur positive). Ratification
le 2026-05-30 sur ce retour. **Étape 2** (granularité par widget +
tag GenUI) livrée immédiatement après comme refinement post-ratification
non-bloquant (le moratoire §10 est respecté : aucune modification de la
décision d'architecture ratifiée).

## Références

### Internes OricOS
- CLAUDE.md §2 (ADR-25 concurrence, ADR-26 GenUI/SpecUI), §3 (ADR-15/27/28
  en cours), §10 (moratoire).
- `docs/adr/0026-modele-gui-declaratif.md` (GeoWorks-like, MainLoop messages).
- `docs/adr/0028-threading-window-manager.md` §1.2ter (rétractation),
  §6.7 (rétractation).
- `kernel/modules/wm.s` (`_ml_classify`, `mlc_moved`, `mlc_up`,
  `_wm_scroll_update`).
- `kernel/modules/event.s` (`kernel_event_push_mouse` — pas concerné).
- `apps/ctl_demo/ctl.c` (app type qui reproduit le bug).

### Externes (PC/GEOS = FreeGEOS, sources officielles)
- [bluewaysw/pcgeos](https://github.com/bluewaysw/pcgeos) — repo officiel
  source code PC/GEOS (sous BlueWaySW depuis ouverture par Breadbox).
- [Include/Objects/gValueC.def](https://github.com/bluewaysw/pcgeos/blob/master/Include/Objects/gValueC.def) —
  définition de `GenValueClass`, hints `HINT_VALUE_IMMEDIATE_DRAG_NOTIFICATION`
  (lignes 800-802) et `HINT_VALUE_DELAYED_DRAG_NOTIFICATION` (lignes 804-806).
- [bluewaysw.github.io/pcgeos](https://bluewaysw.github.io/pcgeos/) — doc
  browsable FreeGEOS.

### Externes (références supplémentaires non lues, mais conceptuellement
similaires)
- SymbOS (référencé par ADR-03 et ADR-06) : pattern fixe « notify on release
  » sans opt-in.
- Intuition (Amiga) — Boopsi `GadgetClass` : `GACT_FOLLOWMOUSE` comme opt-in
  au mode temps réel ; absence du flag = notification à la release seulement
  (`IDCMP_GADGETUP`).

### Mesures
- `/tmp/mtrace3.log` (1 704 lignes) : trace SDL+IRQ+SCROLL_DRAG_ID+EVENT_RING
  qui réfute l'hypothèse du drop d'UP et identifie le bottleneck.
- `/tmp/mtrace4.log` (1 520 lignes) : trace enrichie avec TCB+FORBID+CPU PC
  qui localise précisément la boucle infinie kernel_scroll_up
  (`PC=$01:11EE`).

## 8. Audit de ratification (CLAUDE.md §10)

Ratification le **2026-05-30**, conformément aux 3 conditions du moratoire
(CLAUDE.md §10) :

**Condition 1 — Dossier d'instruction écrit** ✅
- Contexte chiffré : §1 cite `mtrace4.log` (PC stuck `$01:11EE`,
  `kernel_scroll_up.bcc scrl_copy` pendant 200+ frames, `FORBID_COUNT=1`
  bloqué, ~390 k cyc/sec consommés par le print). Outil reproductible
  (`--mouse-trace`).
- Alternatives chiffrées : §3 (A statu quo / B SymbOS dur / C GeoWorks
  hint+default), coût-bénéfice tracé pour chacune.
- Recommandation senior tracée : §4 « Viser l'option C » avec 5 justifications
  numérotées.
- Référence externe vérifiée par WebFetch : `gValueC.def` lignes 800-806
  citées verbatim avec URL stable.

**Condition 2 — Implémentation prête (≥ 50 %)** ✅
- **Étape 1 livrée** : flag global `WM_DRAG_NOTIFY_HINT` ($01EE70, default 0
  = DELAYED, $A5 = override IMMEDIATE). Modification `_ml_classify` :
  `mlc_moved_go` retourne MSG_NULL en mode DELAYED ; `mlc_up` notifie une
  fois à la release en mode DELAYED.
- **Fix `_wm_redraw_ctl`** : `kernel_wm_cursor_blit` (restore+save+draw) au
  lieu de `kernel_wm_draw_cursor` (invalidate+save+draw). Bug pré-existant
  §6.6 révélé par le changement de timing, corrigé en même temps.
- **`make tests` vert** à chaque palier (suite complète + `test-oricos-scroll-cost`
  qui mesure la progression value 1:1 préservée).
- **Validation interactive utilisateur positive** (2026-05-30) sur
  `--ctl-demo` : scrollbar fluide, value 1:1, FORBID se libère, curseur propre.

**Condition 3 — Cohérence ADR existantes** ✅
- ADR-26 (GenUI/SpecUI déclaratif GeoWorks-like) : **renforcée** par ADR-29
  (le hint déclaratif est l'extension naturelle du pattern Gen/Spec).
- ADR-25 (concurrence Exec-classique) : aucune modification de FORBID/block/
  wake. ADR-29 résout le symptôme (FORBID stuck à 1) sans modifier la
  primitive de concurrence.
- ADR-28 (threading WM) : ADR-29 est orthogonale (fonctionne en mode legacy
  comme en mode serveur Étape 2 passe-plat). Rétractation explicite des §1.2ter
  « famine réfutée » et §6.7 « quota anti-drop button-UP » d'ADR-28 (mal
  ciblés, tracées dans le fichier source d'ADR-28). Le design option C
  d'ADR-28 reste valable.
- Aucune contradiction non-explicite avec une ADR ratifiée.

**Refinements post-ratification (statut au 2026-05-30)** :
1. ~~**Étape 2** (§7.2 du plan) : granularité par widget~~ — **FAIT
   2026-05-30** : tableau `WIDGET_HINTS` (8 × 1B) + tag
   `GU_HINT_IMMEDIATE_DRAG_NOTIFY` (val 10) implémentés. Validation
   interactive utilisateur positive. Détails livrés en §7.2.
2. **Étendre l'opt-in IMMEDIATE aux futurs GenRange** (sliders bornés
   bas+haut, héritent de GenValue chez GeoWorks, cf. `gRangeC.def`).
3. **Option CLI Phosphoric `--drag-hint-immediate`** : pour A/B systématique
   en cours de dev/debug. Pose `WM_DRAG_NOTIFY_HINT=$A5` au boot.
4. **Audit doc apps** : documenter le contrat `MSG_CONTROL` (1 fois à la
   release par défaut, opt-in IMMEDIATE possible) dans le SDK `oricos.h`.
   Partiellement fait : commentaire ajouté à `GU_HINT_IMMEDIATE_DRAG_NOTIFY`
   dans `oricos.h`.
5. **Spec UI** (cf. ADR-26) : selon le specUI futur (e.g., look CGA vs look
   XVGA), honorer ou pas le hint comme prévu par GeoWorks (*« if available
   under the specific UI »*).
6. **Démo `ctl_immediate_demo`** : nouvelle app SDK qui utilise
   `GU_HINT_IMMEDIATE_DRAG_NOTIFY` sur un scrollbar et montre du feedback
   temps réel (par exemple, met à jour une preview rectangle). Permet aux
   futurs devs de comprendre le pattern par exemple.

Refinements 1 (Étape 2) **livré**. 2-6 tracés et suivis, aucun n'est bloquant
pour la décision d'architecture désormais figée par cette ratification.
