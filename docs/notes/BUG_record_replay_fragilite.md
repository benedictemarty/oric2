# BUG — Fragilité de la chaîne record/replay (dossier ouvert 2026-06-11)

> Ouvert suite au retour interactif de Bénédicte (« ben non plein de
> traces », puis « c'est fragile ») sur la validation du sprint ADR-27 C2
> (--tallwin). La passe de vérification a produit UN fix réel (traces du
> compose-loop) et révélé DEUX bugs résiduels distincts de la chaîne
> record/replay. Ce dossier fige les repros et l'état des connaissances —
> à traiter dans un sprint de CONSOLIDATION dédié, pas par patchs à chaud.

## Fix livré pendant la passe (single-writer du geste)

**Symptôme** : « plein de traces » au drag interactif d'une fenêtre
composée (--tallwin) : blocs blancs fantômes le long du chemin de drag.

**Cause prouvée** : DEUX peintres concurrents sur la fenêtre draguée —
le compose-loop de l'app (task_tallwin : `kernel_wm_compose` en boucle)
BLIT le backing à des positions périmées pendant que task_wm
(redraw_drag) efface/repeint ; les copies périmées tombent hors de
l'union dirty-rect du move courant → jamais effacées.

**Fix** (wm.s) : pendant un geste (drag/resize armé), la boucle compose
SKIP la fenêtre focus (single-writer : task_wm seule peint la draguée) ;
`redraw_drag` blit lui-même le backing de la draguée
(`kernel_wm_compose_slot`, cœur extrait de la boucle) AVANT son chrome —
position lue dans la même passe que le move, zéro course. Mesuré : 0
résidu sur fast-drag 10 moves (vs 34+ avant), chrome visible pendant le
drag.

## Découverte pendant la passe — variables WM superposées au segment CHARSET

La suite complète a vu ROUGE le fix ci-dessus (`tallwin_multibank` :
compose toujours skippé). Cause prouvée (R1, xxd sur l'image kernel) :
le bloc de variables WM vit DANS la plage du segment **CHARSET
$5800-$5FFF** (fonte uploadée au GPU au boot, zone ensuite réutilisée
comme BSS) → le contenu image initial est constitué d'**octets de
glyphes, non nuls** ($08 à $015938 = `WM_DRAG_ARMED`). `WM_DRAG_ARMED`
n'était jamais initialisé — personne ne le lisait avant le 1er clic
(qui l'écrit). Le gate geste du compose-loop est le premier lecteur
à froid → geste fantôme permanent, fenêtre focus jamais composée.
Fix : init à 0 dans `kernel_wm_init` (à côté de `WM_RESIZE_ARMED`).

**Item d'audit R6 pour le sprint de consolidation** : recenser TOUTES
les variables placées dans $5800-$5FFF (et toute zone superposée à un
segment chargé : TCB $5C00, bitmap $5B00, WM_FOCUS $5B73…) et vérifier
qu'elles sont écrites avant première lecture — l'hypothèse implicite
« BSS = zéro » est FAUSSE sur toute cette plage.

## Découverte pendant la passe — test win_app couteau-sur-le-fil

Deuxième rouge de la suite sur le fix : `test_oricos_win_app`
(WM_COUNT=3, l'app ne sort jamais). Bissection empirique du diff :
le rouge survenait même avec la logique de skip MORTE (branche
commentée), et **16 NOPs exécutés dans la boucle compose suffisaient
à le déclencher**, tandis que 16 octets de code mort (pur décalage de
layout) passaient. Cause : le test livrait la touche 'Z' au cycle
EXACT où le compositing devient visible — instant où l'app n'est pas
forcément encore bloquée dans `read_char`, et le routage du waiter
clavier (option γ focus) dépend de qui attend à la livraison. ±32 cyc
dans compose suffisaient à changer le destinataire. Fix (R8, intention
préservée) : touche livrée avec une marge de +200k cyc après le
compositing visible. Même famille que « clic pendant 1er rendu perdu »
(tests rebasés clics ≥ 220k) — **3e instance : ériger l'invariant
(R6) au sprint de consolidation : aucun stimulus de test n'est livré
sur un flanc détecté, toujours flanc + marge.**

## SPRINT DE CONSOLIDATION — bilan (2026-06-11, « go » de Bénédicte)

### 1. Audit R6 « variables superposées au CHARSET » : FAIT

Scan systématique des 282 constantes $01xxxx contre l'image kernel :
97 variables ont un octet image initial NON NUL. Croisé avec les sites
d'écriture init vs événementiels. La plupart sont des scratch
écrits-avant-lus ou des structures bornées par un compteur initialisé
(rings via COUNT, ZORDER via ZORDER_N — sûres par protocole). **1 bug
latent réel trouvé et fixé : WM_OWNER** — les slots jamais créés
contenaient des octets image plausibles comme pids ($02/$04/$08…) ; au
sys_exit d'une tâche au pid correspondant, close_owner fermait une
fenêtre INEXISTANTE (WM_COUNT corrompu). De plus la sentinelle « pas
d'owner » était $00 alors que pid 0 existe, et kernel_wm_close (bouton
×) ne nettoyait pas l'owner (slot fermé non réutilisé re-fermable).
Fix : sentinelle WM_OWNER_NONE=$FF + init wm_init + clear au close.
Garde rouge-checkée : test_oricos_wm_owner_sentinel (ROUGE vu :
$1C/$00/$00/$3E/$00/$08, VERT après).

### 2. DÉCOUVERTE MAJEURE — oricrobot périmé : bugs A et B = artefacts

Le binaire `oricrobot` datait d'AVANT le travail GPU v4 (caps,
EXEC_LIST_XY) : `GPU_CAPS_KERNEL = $00` dans tout l'environnement robot
→ le kernel tournait en régime v1 sync/direct, SANS display-lists. Tous
les diagnostics robot de la veille (dont les « bugs résiduels » A et B)
testaient un régime qui n'est PAS celui de la validation interactive.
Après reconstruction (`make oricrobot`, caps $F4) :
- **Bug A (liste périmée tallwin) : NON REPRODUIT** — render correct.
  Reste l'artefact connu de l'app démo task_tallwin qui blitte son
  backing plein-rect par-dessus son propre chrome (question de design
  ADR-27 : le chrome devrait-il vivre dans le backing ? — à instruire).
- **Bug B (chrome non rejoué post-drag) : NON REPRODUIT** — état final
  correct (fenêtres complètes, z-order propre, 0 trace). Vérifié aussi
  par décodage de la display-list slot 0 : 2 FILL_RECT16 + 6 TEXT16,
  liste complète et correcte, dx=0, exec GPU propre.
- **Le fix single-writer (traces) TIENT dans le vrai régime** : 0 résidu.

**Fix harnais : `make tests` dépend désormais d'`oricrobot`** (le
binaire ne peut plus diverger silencieusement des devices émulés).
Leçon R6 : un harnais de mesure a la même exigence de fraîcheur que le
code mesuré — un robot stale a fabriqué 2 faux bugs et failli déclencher
2 faux fixes.

### 3. Bug borné NON fixé — labels d'icônes morts à la source

Les labels d'icônes desktop (« Files »/« Prefs ») sont $FF dès
ICON_TABLE (bank 1) : la copie source dans kernel_icon_add lit du $FF
(pointeur source erroné). Préexistant (les vieux screenshots montraient
« ddddddd » en régime direct ; blanc en régime listé). Fausse piste
écartée avec preuve : TB_CLK_SDRAM ($011200) ne collisionne PAS avec les
labels ($011200 bank SDRAM $01) — l'upload horloge écrit ADDR_HI=$00 →
$001200. À fixer dans une passe dédiée (chercher le caller de
kernel_icon_add au boot et son pointeur label).

### 4. Reste du sprint (à dérouler)

- Oracle « framebuffer listé == rendu direct » (test de transparence).
- Matrice d'interaction {drag court/long/sur fenêtre, resize,
  release-pendant-skip} × {listée/composée/modale}.
- Design ADR-27 : chrome dans le backing vs blit client-only (artefact
  tallwin §2).

## Bug résiduel A — liste rejouée périmée/tronquée (scénario tallwin)

**Repro** (oricrobot) : `--tallwin` (fenêtre 200×200 créée à (100,100),
PAR-DESSUS OricOS), clic titlebar tallwin, drag loin, release.
**Observé** : à (100,100), la fenêtre affichée a une titlebar LIGHTBLUE
(couleur focus) alors que le focus est tallwin, et NI titre NI widgets —
ressemble à une liste de 2 fills seulement, ou une liste d'un état
antérieur. À déterminer : QUELLE fenêtre est rendue là (OricOS slot 0 ?)
et pourquoi sa liste n'est ni invalidée ni complète.
**Scripts** : /tmp/rob_bisect.txt (b0..b3) — à versionner dans le sprint.

## Bug résiduel B — chrome non rejoué après long drag (config par défaut)

**Repro** (oricrobot, AUCUN flag à part nostp) : drag d'Editor (350,310)
par sa titlebar jusqu'à ~(130,120) (3 moves de ~70 px), release.
**Observé** : le TITRE « Editor » + boutons O/× sont rendus à la position
finale, mais PAS les fills (titlebar bleue, corps gris) → titre flottant
sur le desktop. Les entrées TEXT16 de la liste sont rejouées, pas les
FILL_RECT16 — ou bien les fills sont effacés après coup (ordre erase/
replay du catch-up coalescing ?). Piste : interaction WM_RD_SKIPPED
(rattrapage fin de geste) × replay translaté × culling.

## Constat d'architecture (le fond de « c'est fragile »)

La chaîne de rendu accumule 8 mécanismes interdépendants livrés en
2 jours : display-lists record/replay (C2a), arène de chaînes + widgets
listés + EXEC_LIST_XY translaté (C2b), culling dirty-rect, coalescing de
frames (WM_RD_SKIPPED), NOCLEAR, compose-skip geste (ce fix), backing
multi-banques (C2). Chaque mécanisme est testé isolément (rouge→vert),
mais les INTERACTIONS (drag long × invalidation × catch-up × compose ×
z-order × clamp) ne sont couvertes par aucune matrice systématique — les
bugs ci-dessus sont tous des bugs d'interaction.

## Proposition : sprint de CONSOLIDATION (à arbitrer par Bénédicte)

1. **Geler les features GUI** le temps du sprint.
2. **Matrice de tests d'interaction** déterministe (harnais headless) :
   {drag court, drag long, drag sur fenêtre, resize, release-pendant-
   skip} × {fenêtre listée, fenêtre composée, modale} × {focus change}.
   Chaque case = un test pixel.
3. **Invariants vérifiables** à logger/asserter en debug : « une liste
   valide rejouée == le rendu direct du même état » (test de
   transparence par comparaison framebuffer listé vs direct, le vrai
   oracle), « ORG == position au record », « arène d'un slot intacte
   tant que sa liste est valide ».
4. **Simplifier si nécessaire** : si la matrice révèle que le culling ou
   le coalescing créent plus de bugs qu'ils n'apportent (les gains perf
   sont déjà acquis ailleurs), les retirer — le critère −90 % ADR-34
   était atteint avant ces 2 mécanismes (1 307 cyc mesurés au C2b).
