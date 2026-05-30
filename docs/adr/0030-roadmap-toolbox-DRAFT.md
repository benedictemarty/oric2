# ADR-30 (DRAFT) — Roadmap toolbox (alignement GeoWorks)

- **Statut** : **ouverte — dossier d'instruction** (2026-05-30). DRAFT non
  ratifié (moratoire §10 : implémentations de référence à fournir par étapes,
  ratification globale ou par tranches après livraisons).
- **Date d'ouverture** : 2026-05-30
- **Décideurs** : bmarty (arbitrage), Claude Code (instruction)
- **Contexte technique** : suite directe de l'audit factuel de couverture
  toolbox OricOS vs PC/GEOS (WebFetch 2026-05-30 du dossier
  `Include/Objects/` du repo `bluewaysw/pcgeos`). ADR liées : ADR-06 (GUI
  SymbOS-like), ADR-26 (GUI déclaratif GenUI/SpecUI, GeoWorks-like, source
  d'inspiration explicite), ADR-29 (drag notification hint, premier
  alignement GeoWorks `gValueC.def`).
- **Conformité moratoire** : oui — ce document est un **dossier d'instruction**
  (CLAUDE.md §10 cond. 1). Aucune ADR ratifiée n'est modifiée. Référence
  externe vérifiée par WebFetch (audit factuel des 64 `.def` du repo
  pcgeos).

## 0. Résumé exécutif

L'audit factuel du **2026-05-30** établit que la toolbox OricOS expose
**9 widgets via GenUI** (`GU_BUTTON`, `GU_VIEW`, `GU_CHECK`, `GU_SCROLL_V/H`,
`GU_RADIO`, `GU_TEXT`, plus `GU_HINT_IMMEDIATE_DRAG_NOTIFY` qui est un hint),
contre **40 classes `Gen*`** côté GeoWorks (PC/GEOS), pour une couverture
réelle de **~22 %**. En élargissant aux classes Vis* et subsystems
(clipboard, help, styles, table) la couverture descend à **~14 %**.

OricOS couvre les **widgets fondamentaux** correctement (button, checkbox,
radio, scroll, text, view) mais manque :
- **Widgets spécialisés** : spinner (`gSpinGC`), liste exposée (`gDListC`,
  déjà implémentée en interne mais sans tag GenUI), slider borné min/max
  (`gRangeC`), champ formaté (`gFieldC`).
- **Composants structurels haut-niveau** : application (`gAppC`), document
  (`gDocC`, `gDocCtrl`), display (`gDispC`), page (`gPageCC`).
- **Menus déclaratifs** : `menu_defs` actuel est en code natif, pas exposé
  via GenUI.
- **Hints structurels** : 1 hint sur ~80 chez GeoWorks (`HINT_*`).
- **Subsystems** : clipboard, help, styles, table — aucun.

Cette ADR propose une **roadmap incrémentale priorisée** pour amener la
couverture des widgets de ~55 % (widgets fondamentaux actuels) vers **~85 %**
(widgets fondamentaux + 5 ajouts haut-impact), sans engager le projet à
implémenter tout l'écosystème GeoWorks (qui serait disproportionné pour un
65C816 mono-app).

**Recommandation senior** : 5 widgets prioritaires à livrer en autant
d'étapes, chacune ratifiable indépendamment, dans cet ordre de valeur
décroissante :

1. **`GU_LIST`** (expose le `WG_TYPE_LIST` interne existant) — *quick win*.
2. **`GU_MENU` + `GU_MENU_ITEM`** (refactor `menu_defs` en déclaratif).
3. **`GU_RANGE`** (slider borné min+max).
4. **`GU_SPIN`** (incrémenteur numérique).
5. **`GU_FIELD`** (champ formaté date/heure/nombre).

Couverture cible post-Étape 5 : **~14 widgets exposés**, soit ~85 % des
widgets d'interaction directe de GeoWorks. Au-delà (Vis*, subsystems,
framework Document/Application/Page), c'est un **deuxième chantier** hors
scope ADR-30, à instruire séparément si la roadmap toolbox aboutit.

## 1. Contexte chiffré (audit factuel WebFetch 2026-05-30)

### 1.1. Inventaire PC/GEOS officiel

Source : `https://api.github.com/repos/bluewaysw/pcgeos/contents/Include/Objects`
(consultée 2026-05-30). **64 fichiers** au total.

**40 classes `Gen*` détectées** (préfixe `g*C.def`) :

| Catégorie | Classes |
|---|---|
| **Containers / structure** (8) | `gAppC`, `gAppDocC`, `gDocC`, `gDocCtrl`, `gDocGrpC`, `gInterC`, `gPrimC`, `gSysC` |
| **Contrôles d'interaction directe** (14) | `gTrigC` (button), `gBoolC` (checkbox), `gBoolGC` (group), `gItemC` (radio), `gItemGC` (radio group), `gValueC` (scrollbar), `gRangeC` (slider borné), `gSpinGC` (spinner), `gTextC` (text), `gFieldC` (field), `gEditCC` (edit ctrl), `gListC` (list), `gDListC` (dynamic list), `gLEntryC` (list entry) |
| **Views / display** (6) | `gViewC`, `gViewCC` (control), `gDispC`, `gContC` (content), `gPageCC`, `gScreenC` |
| **Menu / toolbar** (4) | `gToolCC`, `gToolGC`, `gCtrlC`, `gDCtrlC` |
| **OS / dialog** (3) | `gFSelC` (file selector), `gPenICC` (pen input), `gProcC` (process) |
| **Icônes / glyphs** (2) | `gGlyphC`, `gGlyphDC` |
| **Frameworks** (3) | `gAListC` (active list), `gGadgetC`, `gUIDocCC` |

**7 classes `Vis*` (renderer hierarchy)** : `visC`, `vCompC`, `vCntC`,
`vTextC`, `vLTextC`, `gadgets.def`, `winC`.

**~18 subsystems / composants standalones** : `clipbrd`, `colorC`, `coverpg`,
`eMenuC`, `emTrigC`, `emomC`, `helpCC`, `iDialCC`, `inkfix`, `inputC`,
`keyControl`, `metaC`, `processC`, `spline`, `styles`, `table`, `uiInputC`,
`metaC`.

**+ 2 sous-dossiers** : `SSheet` (Spreadsheet objects), `Text` (composants
texte avancés).

### 1.2. Inventaire OricOS actuel

Code source (`oricos.h` + `kernel.s` + `wm.s` + `tk.s`) :

| Tag GenUI | ID | Équivalent GeoWorks |
|---|---|---|
| `GU_WINDOW` | 1 | `gPrimC` |
| `GU_TITLE` | 2 | attribut de `gPrimC` |
| `GU_BUTTON` | 3 | `gTrigC` |
| `GU_VIEW` | 4 | `gViewC` |
| `GU_CHECK` | 5 | `gBoolC` |
| `GU_SCROLL_V` | 6 | `gValueC` (mode V) |
| `GU_SCROLL_H` | 7 | `gValueC` (mode H) |
| `GU_RADIO` | 8 | `gItemC` + `gItemGC` (combinés) |
| `GU_TEXT` | 9 | `gTextC` (ligne unique) |
| `GU_HINT_IMMEDIATE_DRAG_NOTIFY` | 10 | hint `HINT_VALUE_IMMEDIATE_DRAG_NOTIFICATION` |

Plus **3 composants non-déclaratifs** (présents en code mais pas exposés via
GenUI) :
- `WG_TYPE_LIST` (`kernel_ctl_list_select`) — *quick win* : implémentation
  existante, juste à exposer.
- `menu_defs` (table multi-menu System/View) — code natif, à refactorer en
  déclaratif.
- `kernel_icon_add` (icônes desktop) — hors-fenêtre, modèle séparé.

Plus **1 système séparé** : `SYS_DO_DLGBOX` (dialogs modaux via table DB_*)
— pas intégré dans GenUI mais cohabite.

### 1.3. Couverture chiffrée

| Périmètre | Compte | Couverture |
|---|---|---|
| Classes `Gen*` totales | 40 | **9 / 40 = 22 %** |
| Widgets d'interaction directe `Gen*` | 14 | **8 / 14 = 57 %** |
| Tous `.def` (Gen + Vis + subsystems) | 64 | **9 / 64 = 14 %** |
| Hints structurels | ~80 (de mémoire, à confirmer) | **1 / 80 = 1 %** |

**Note méthodologique** : ces chiffres reflètent l'audit factuel WebFetch
du 2026-05-30, et révisent à la baisse l'estimé précédent (« ~40 % » dans la
synthèse mémoire qui précédait l'audit). Leçon ADR-29 appliquée : ne jamais
chiffrer une couverture par souvenir, toujours fetcher la source.

## 2. Problème

### 2.1. Quel est le manque concret ?

Pour qu'une app utilisateur typique tienne sur OricOS sans devoir bricoler
des widgets manuellement, **5 widgets supplémentaires** sont nécessaires :

1. **Liste sélectionnable** (`GU_LIST`) — quasi tout outil utilisateur a
   besoin d'une liste (fichiers, items, options). Implémentation existante,
   juste à exposer.
2. **Menus déclaratifs** — actuellement `menu_defs` est en code asm natif.
   Une app C n'a aucun moyen propre de déclarer ses menus.
3. **Slider borné** (`GU_RANGE`) — scrollbar va de 0 à max ; un volume,
   luminosité, ou settings ont une plage min/max. `gRangeC` chez GeoWorks.
4. **Spinner** (`GU_SPIN`) — incrémenteur numérique compact (touch ▲▼).
   `gSpinGC` chez GeoWorks.
5. **Champ formaté** (`GU_FIELD`) — date, heure, nombre, masque. `gFieldC`
   chez GeoWorks.

### 2.2. Pourquoi pas tout implémenter ?

L'écosystème GeoWorks complet (40 classes Gen + Vis + subsystems + Document
+ Application + Page framework) suppose une machine bien plus puissante que
le Oric 2 (8088 à 8 MHz minimum, 640 Ko RAM). Sur 65C816 / 256 Ko, viser la
parité serait disproportionné.

OricOS ne vise **pas** à reproduire PC/GEOS complet, mais à offrir un
**modèle déclaratif GeoWorks-like** (ADR-26) pour les apps utilisateur,
suffisant pour des UIs riches mais sobres. La cible réaliste : couvrir les
**widgets d'interaction directe** (~14 chez GeoWorks) à ~85 % et laisser
les composants haut-niveau (Document, Application, Vis hierarchy) au niveau
implicite ou à instruire séparément si besoin.

## 3. Options envisagées

### Option A — Statu quo (rien faire)

- **Coût** : 0.
- **Bénéfice** : 0.
- **Limite** : la toolbox reste à ~57 % des widgets d'interaction
  fondamentaux. Apps utilisateur typiques (file manager, settings panel,
  even un client mail simple) doivent réinventer LIST, MENU, etc. en code
  natif. ADR-26 reste partiellement appliquée.

**Écartée.**

### Option B — Big-bang (toolbox complète d'un coup)

Livrer en une PR les 5 widgets prioritaires + refactor menu + tag GU_LIST
+ tests.

- **Coût** : élevé (estimé ~1500-2000 lignes asm, ~10-15 jours pleins).
  Demande de toucher beaucoup de fichiers en parallèle (wm.s, tk.s,
  kernel.s, sud_loop parser, SDK).
- **Bénéfice** : couverture instantanée ~85 %.
- **Risque** : régressions multiples, debug difficile à isoler, batch trop
  gros à reviewer.
- **Réversibilité** : faible.

**Écartée** : incompatible avec la culture de petits commits atomiques du
projet et avec la leçon ADR-28/29 (validation interactive par incrément).

### Option C — Quick wins seulement (`GU_LIST` + ce qu'on a)

Juste exposer `GU_LIST` (la liste interne) et s'arrêter là.

- **Coût** : ~30 lignes asm + tag.
- **Bénéfice** : couverture passe de 57 % à 64 % (1 widget de plus).
- **Limite** : ne traite pas les manques structurels (menu, range, spin,
  field). Roadmap timide.

**Écartée** comme option principale** mais retenue comme **Étape 1** du plan
incrémental ci-dessous.

### Option D — Roadmap incrémentale priorisée — **recommandée**

5 widgets, chacun livré dans une étape indépendante (1 PR, 1 commit, 1 test,
1 validation interactive). Chaque étape est ratifiable en autonomie, sans
forcer la suivante.

- **Coût** : étalé (~5 étapes × ~5-10 jours = ~25-50 jours selon
  granularité).
- **Bénéfice** : couverture progresse linéairement de 57 % à ~85 %, avec
  garantie de stabilité à chaque palier.
- **Risque** : faible par construction (chaque étape isolée).
- **Réversibilité** : élevée à chaque étape.

## 4. Recommandation senior (tracée)

**Viser l'option D** (roadmap incrémentale priorisée) avec l'ordre suivant,
arbitré par **valeur/coût** :

| Étape | Widget | Classe GW | Coût estimé | Valeur | Justification de l'ordre |
|---|---|---|---|---|---|
| 1 | `GU_LIST` | `gDListC` / `gListC` | **bas** (~30 LOC) | élevée | *Quick win* : impl interne déjà faite, juste exposer + parser |
| 2 | `GU_MENU` + `GU_MENU_ITEM` | `gToolCC` / `eMenuC` | **moyen** (~200 LOC) | élevée | Refactor `menu_defs` → déclaratif. Tout app utilisateur a un menu |
| 3 | `GU_RANGE` | `gRangeC` | **bas** (~100 LOC) | moyenne | Hérite de `GU_SCROLL_V` (ADR-29). Slider borné = +1 attr (min) |
| 4 | `GU_SPIN` | `gSpinGC` | **moyen** (~150 LOC) | moyenne | Nouveau widget : 2 boutons +/- + champ. Composant fréquent |
| 5 | `GU_FIELD` | `gFieldC` | **élevé** (~250 LOC) | moyenne | Formatage + validation. Plus complexe ; utile dans forms |

**Justification du choix de priorité** :
- **#1 et #2** ont le plus de valeur immédiate (toute app a une liste et/ou
  menu) et le coût le plus faible (`GU_LIST` est trivial, menus à
  refactorer mais code existant).
- **#3** est un *follow-up* naturel à ADR-29 (`GU_RANGE` hérite de
  `gValueC`, mode notification déjà géré).
- **#4** et **#5** sont moins urgents et plus coûteux ; à différer si le
  budget bug-fix doit primer.

Chaque étape applique la **leçon ADR-29** : implémentation gated par
tag/flag, suite headless verte, validation interactive utilisateur AVANT
ratification individuelle.

## 5. Conséquences

### 5.1. Positives

- Couverture toolbox passe de 22 % (Gen total) à ~35 %, et de 57 % à 85 %
  sur les widgets d'interaction directe.
- Apps utilisateur peuvent déclarer des UIs riches en GenUI seul, sans
  bricolage natif.
- ADR-26 (modèle déclaratif) est plus complètement appliquée.
- Refactor menus en déclaratif → meilleur typing/sûreté pour les futures
  apps.

### 5.2. Négatives / coûts

- **RAM** : chaque widget ajoute en moyenne 0-1 octet d'attribut + entrée
  `WIDGET_TABLE` (16 octets/widget existants). Estimé total +1-5 octets pour
  `WIDGET_HINTS`-style extensions.
- **Code** : ~750-1500 LOC asm (toutes étapes), réparti.
- **Doc** : SDK `oricos.h` + exemples apps à mettre à jour.
- **Tests** : un test intégration par étape (~5 nouveaux tests).
- **Audit apps existantes** : à chaque ajout, vérifier qu'aucune app actuelle
  ne dépend du comportement antérieur (par exemple `menu_defs` actuel ne
  doit pas casser quand refactoré en déclaratif).

### 5.3. Sur ADR-26 (modèle GUI déclaratif)

ADR-30 **renforce** ADR-26 en complétant la couverture déclarative. Pas de
modification d'ADR-26.

### 5.4. Sur ADR-29 (drag notification hint)

L'Étape 3 (`GU_RANGE`) hérite de `gValueC` chez GeoWorks → bénéficie
automatiquement du hint `DELAYED/IMMEDIATE` (déjà implémenté pour
`GU_SCROLL_*` par ADR-29). Cohérence naturelle.

## 6. Points ouverts à instruire avant ratification (par étape)

1. **Numérotation des tags GenUI** : actuel s'arrête à 10
   (`GU_HINT_IMMEDIATE_DRAG_NOTIFY`). Convention pour étapes 1-5 : tags
   11-15 ? Ou réserver 11-19 pour widgets et 20+ pour hints ?
2. **Refactor `menu_defs` en déclaratif** : breaking change pour le code
   kernel existant. Migration progressive ou *swap* atomique ? Audit
   nécessaire de toutes les références.
3. **`GU_LIST` API d'items** : items inline dans la table GenUI (chaînes
   null-terminées concaténées) ou pointeur vers table externe ? GeoWorks
   utilise un message `MSG_GEN_DYNAMIC_LIST_QUERY` (callback) — trop
   complexe pour 65C816 ; pattern OricOS = inline plus simple.
4. **`GU_FIELD` complexité** : à étudier — supporter quel formatage v1 ?
   (nombre seul ? ou date/heure aussi ?) Risque de scope creep.
5. **Tags vs attributs inline** : convention ADR-29 = tag séparé AVANT le
   widget. Pour des attributs cumulés (range = min+max), peut-être plus
   propre en inline (`GU_RANGE relx rely w h min max` au lieu de
   `GU_HINT_MIN val GU_RANGE ...`). À trancher au moment de l'étape 3.

## 7. Plan d'implémentation (5 étapes, indépendantes)

### 7.0. Étape 0 — Audit final + standard d'étape

- **Audit** : grep apps/ pour s'assurer qu'aucune app actuelle ne dépend
  d'un comportement qui serait cassé par les ajouts (probable : aucune).
- **Standard d'étape** : chaque étape suit la séquence ADR-29 éprouvée :
  1. Implémentation gated par tag/flag.
  2. `make tests` vert + nouveau test intégration headless.
  3. Validation interactive utilisateur (`./oric1-emu --kernel ... --xvga --<demo>`).
  4. Documentation SDK + exemple app.
  5. Ratification individuelle après validation positive.
- Gate : audit publié dans le ChangeLog, standard d'étape suivi pour toutes
  les suivantes.

### 7.1. Étape 1 — `GU_LIST` (expose liste interne) — **RATIFIÉE 2026-05-30**

**Quick win livré et validé.** L'audit factuel pré-implémentation a comparé
`Include/Objects/gListC.def` (21 lignes, `GenList` static) et
`Include/Objects/gDListC.def` (387 lignes, `GenDynamicList` avec callbacks).
**Choix d'alignement** : `GenList` static, items inline (text monikers
`NULL_TERM_TEXT_FPTR`). `GenDynamicList` (mutation runtime via callbacks) à
instruire séparément si besoin.

**Livraison** :
- ✅ **`kernel.s`** : `GU_LIST = $0B` + `UI_LIST_BUF = $016330` (128 octets
  buffer items en bank 1) + `.assert UI_LIST_BUF + 128 <= $016400`
  (anti-overlap RAW_RING).
- ✅ **`wm.s sud_loop`** : cas `GU_LIST` ajouté entre `sud_n2g` et `sud_n2h`
  (chaîne d'`if-cascading` du parser GenUI).
- ✅ **`wm.s sud_list`** : nouveau handler — `_sud_rect` (lit relx16, rely16,
  relw16, relh16) + lit `count8` + boucle copie inline strings null-term
  depuis `[$D0],Y` (bank app) → `f:UI_LIST_BUF` (bank 1) avec protection
  débordement à 128 octets, `count` scratch dans `DP_TMP`. Configure
  `WG_TYPE = WG_TYPE_LIST`, `DP_PCPTR → UI_LIST_BUF`, `WG_CB = 0`
  (selected init), `WG_CB+1 = count`, `GFX_COLOR = $07` (lightgray cohérent
  task_list_entry), puis `_sud_attach` (`kernel_wm_add_widget`).
- ✅ **SDK `oricos.h`** : `#define GU_LIST 11` exposé aux apps userland C
  avec doc d'usage (format inline, équivalence text monikers GeoWorks,
  référence à `gListC.def`, mention que `GenDynamicList` n'est pas v1).
- ✅ **Démo `apps/ctl_demo/ctl.c`** : window agrandie à h=170, GU_LIST ajouté
  à rel `(12,72,120,48)` avec 3 items `Item A`/`B`/`C`. Recompilée
  (1621 octets bundle), kernel rebuilt, suite verte.
- ✅ **Validation interactive utilisateur positive** (2026-05-30) :
  `./oric1-emu --xvga --ctl-demo` affiche la liste, items cliquables, app
  reçoit `MSG_CONTROL` avec le bon index.
- ✅ **`make tests` complet vert** (suite intégrale + tests scroll-cost +
  genui).

**Limite assumée (tracée vers ADR-31)** : un widget dont `rel.y + h >
window.h` (par exemple après resize-down de la fenêtre) reste peint en
dehors du rect window. Pas spécifique à `GU_LIST` — bug architectural
pré-existant (OricOS n'a pas de clip-list ni backing par fenêtre).
Plus visible avec `GU_LIST` qui prend ~48 px. **ADR-31** ouvert le
2026-05-30 pour instruire le pattern de clip widget hors rect parent.

### 7.2. Étape 2 — `GU_MENU` + `GU_MENU_ITEM` (déclaratif)

Refactor `menu_defs` (actuel table en asm) en déclaratif GenUI. Approche
recommandée : *parallèle* (les deux coexistent) v1, puis migration apps,
puis suppression `menu_defs` v2.

- Tags `GU_MENU = 12` + `GU_MENU_ITEM = 13`.
- `GU_MENU` : nom du menu (chaîne inline) + suivi par 0+ `GU_MENU_ITEM`
  jusqu'au prochain `GU_MENU` ou `GU_END`.
- `GU_MENU_ITEM` : label inline + callback offset16 (comme `GU_BUTTON`).
- Stockage interne : continue à utiliser `menu_defs` (juste construit par
  parser au lieu d'asm hardcodé).
- Test : `test_oricos_genui_menu`.

Coût estimé : ~250 LOC asm + ~30 LOC test C. Valeur : haute (apps peuvent
enfin déclarer leurs menus en C pur).

### 7.3. Étape 3 — `GU_HINT_MIN_VALUE` (attribut min sur GenValue) — **RATIFIÉE 2026-05-30**

**Pivot d'instruction** : audit factuel WebFetch a révélé que `gRangeC.def`
contient juste « *soon-to-be-dead GenRangeClass... Nuked. 7/7/92 cbh* »
— GeoWorks a **supprimé `GenRangeClass`** en juillet 1992 car
`GenValueClass` a déjà `ATTR_GEN_VALUE_MINIMUM` et `ATTR_GEN_VALUE_MAXIMUM`
(`gValueC.def` lignes 93-148). Pas de classe Range séparée nécessaire.

**Décision** : OricOS suit le design final GeoWorks — pas de nouveau widget
`WG_TYPE_RANGE`, juste un **hint déclaratif `GU_HINT_MIN_VALUE`** sur les
`GU_SCROLL_V/H` existants. Cohérent avec ADR-29 pattern hints, extensible
(futur `GU_HINT_MAX_VALUE`, `GU_HINT_INCREMENT`).

**Implémentation livrée** :
- `GU_HINT_MIN_VALUE = $0E` (tag + 1 byte payload).
- `WIDGET_MIN_VALUES = $0163B0` (8 × 1B, un par widget).
- `UI_PENDING_MIN_VALUE = $0163B8` (1B, posé par parser sur prochain widget).
- `sud_hint_min_value` dans `wm.s sud_loop`.
- `_waw_count` (tk.s) copie `UI_PENDING_MIN_VALUE → WIDGET_MIN_VALUES[id]`.
- `sys_ctl_get_value` retourne `WIDGET_VALUE + WIDGET_MIN_VALUES[id]`.
- SDK : `#define GU_HINT_MIN_VALUE 14` (`oricos.h`).
- Démo : `ctl_demo` configuré avec `GU_HINT_MIN_VALUE 20` → range 20..60.

**Coût réel** : ~25 LOC asm + 4 LOC SDK + 2 LOC démo (vs ~150 LOC estimés).
Le pivot d'instruction a divisé le coût par 5.

**Validation interactive utilisateur 2026-05-30** : `v=20..60` selon
position du scrollbar (au lieu de `v=00..40`). Confirmé positif.

**Conformité moratoire CLAUDE.md §10** :
1. ✅ Dossier d'instruction : audit factuel `gRangeC.def` + `gValueC.def`,
   pivot documenté (GenRange nuked, GenValue MINIMUM utilisé seul).
2. ✅ Implémentation prête : 100 % du code livré (parser + storage +
   syscall + SDK + démo).
3. ✅ Cohérence ADR : extension naturelle du pattern hints ADR-29, pas de
   contradiction. Aligné design final GeoWorks 1992+.

### 7.4. Étape 4 — `GU_SPIN` (incrémenteur numérique)

Composant compact : champ texte affichant un nombre + 2 mini-boutons ▲▼
incrément/décrément. `gSpinGC` chez GeoWorks.

- Tag `GU_SPIN = 15` avec `relx16 rely16 w16 h16 min8 max8 step8`.
- Widget type `WG_TYPE_SPIN`, rendu : nombre encadré + ▲▼ à droite.
- Hit-test ▲ → value += step, ▼ → value -= step. Clamp `[min, max]`.
- Test : `test_oricos_genui_spin`.

Coût estimé : ~200 LOC asm + ~30 LOC test C. Valeur : moyenne (composant
fréquent dans settings, prefs).

### 7.5. Étape 5 — `GU_FIELD` (champ formaté)

Champ avec masque de formatage (date, heure, nombre, masque alphanumérique).
`gFieldC` chez GeoWorks. **Plus complexe** : validation à la frappe, masque
visuel, parsing.

- Tag `GU_FIELD = 16` avec `relx16 rely16 w16 h16 mask_offset16 maxlen8`.
- Mask : table de chars (`'D'` = digit, `'A'` = alpha, `':'`, `'-'`, etc.
  littéraux).
- Logique de validation à la saisie (intercepter clavier).
- Test : `test_oricos_genui_field`.

Coût estimé : ~300 LOC asm + ~50 LOC test C. Valeur : utile pour forms
mais peut être différé si autres priorités.

### 7.6. Seuil moratoire (par étape)

Chaque étape atteint son seuil 50 % à la fin de son implémentation gated +
suite verte. Ratification individuelle après validation interactive
utilisateur. Pas de seuil global pour ADR-30 — c'est une roadmap, pas une
décision unique.

## Références

### Internes OricOS
- CLAUDE.md §2 (ADR-26 GUI déclaratif, ADR-29 drag notification hint), §10
  (moratoire).
- `docs/adr/0026-modele-gui-declaratif.md`.
- `docs/adr/0029-drag-notification-hint.md` (le pattern qui inspire ADR-30 :
  alignement GeoWorks vérifié par WebFetch).
- `kernel/modules/wm.s` (`sud_loop`, `_ml_classify`, `mlc_*`).
- `kernel/modules/tk.s` (`kernel_wm_add_widget`, `kernel_tk_*`).
- `tools/oricos-sdk/include/oricos.h` (déclarations GU_*).

### Externes (PC/GEOS = FreeGEOS, sources officielles)
- [bluewaysw/pcgeos](https://github.com/bluewaysw/pcgeos) — repo officiel.
- [Include/Objects/](https://github.com/bluewaysw/pcgeos/tree/master/Include/Objects) —
  64 fichiers `.def` (audit factuel WebFetch 2026-05-30).
- [gValueC.def](https://github.com/bluewaysw/pcgeos/blob/master/Include/Objects/gValueC.def) —
  base de référence ADR-29, sert de modèle pour les autres widgets.
- [gListC.def](https://github.com/bluewaysw/pcgeos/blob/master/Include/Objects/gListC.def),
  [gDListC.def](https://github.com/bluewaysw/pcgeos/blob/master/Include/Objects/gDListC.def),
  [gRangeC.def](https://github.com/bluewaysw/pcgeos/blob/master/Include/Objects/gRangeC.def),
  [gSpinGC.def](https://github.com/bluewaysw/pcgeos/blob/master/Include/Objects/gSpinGC.def),
  [gFieldC.def](https://github.com/bluewaysw/pcgeos/blob/master/Include/Objects/gFieldC.def) —
  références spécifiques pour chaque étape (à WebFetch au moment de
  l'implémentation pour vérifier les sémantiques exactes).

### Audit
- WebFetch `https://api.github.com/repos/bluewaysw/pcgeos/contents/Include/Objects`
  (2026-05-30) : 64 entrées, dont 40 classes Gen*, 7 classes Vis*, 2
  sous-dossiers (SSheet, Text), ~18 subsystems.

## Méta — Itérations possibles post-ratification (étape par étape)

Cette ADR n'est pas un engagement à livrer les 5 étapes. C'est une
**roadmap** qui peut être arrêtée à n'importe quelle étape selon les
priorités du moment. Par exemple, livrer seulement Étapes 1+2 (LIST + MENU)
amène déjà la couverture à ~70 %, ce qui peut suffire pendant longtemps.

Si en pratique on s'arrête après l'étape N, la roadmap reste un document
de référence ouvert (DRAFT permanent ou ratifié partiellement) — pas une
décision figée comme une ADR architecturale classique.
