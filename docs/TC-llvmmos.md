# TC-llvmmos — Analyse de support 65C816 dans llvm-mos

> Document d'investigation. Conclusions intégrées à **ADR-05 v2** (cf.
> `/home/bmarty/oric2/CLAUDE.md` §2) et à **DEC-3** (cf. `BACKLOG.md`).
>
> Date : 2026-05-09. Status : analyse documentaire complète, pas
> d'installation effective de llvm-mos sur ce poste.

---

## 1. Contexte

ADR-05 v1 (ratifiée 2026-05-07) prévoit que les apps userland OricOS
soient écrites en **C compilé par llvm-mos**, exploitant le mode N
65C816 (registres 16-bit + banking 24-bit PBR/DBR). Cette ADR a été
ratifiée sans validation pratique de la toolchain.

L'investigation TC-llvmmos (priorité 2bis du BACKLOG) doit déterminer
si la promesse est tenable, et sinon, quelle est l'enveloppe réaliste.

---

## 2. Méthodologie

Investigation documentaire :
- Lecture de la page `llvm-mos.org/wiki/Welcome`.
- Inspection des issues GitHub `llvm-mos/llvm-mos` (filtrage par mot-clé
  `65816`).
- Inspection de la liste des plateformes du SDK
  (`llvm-mos-sdk/mos-platform/`).
- Inspection des configs `clang.cfg`, `crt0.S`, `link.ld` du target
  `rpc8e` (le seul target SDK utilisant des opcodes 65C816).

Pas d'installation effective de llvm-mos (binaire ~250 MB) — le
constat documentaire suffit pour ce sprint d'investigation.

---

## 3. Constats

### 3.1. Issues GitHub `llvm-mos/llvm-mos` (filtre `65816`)

25 issues / PRs trouvées. Les issues clés ouvertes :

| # | Title | State | Updated |
|---|-------|-------|---------|
| 32 | 65816 support (epic) | open | 2026-01 |
| 320 | [65816] 24-bit address space support | open | 2024-03 |
| 321 | [65816] 16-bit register mode support | open | 2023-09 |
| 454 | Improved ergonomics for 65816 (assembler) | open | — |

Issues fermées (ce qui marche déjà) :
- #296 : `add HuC6280 CPU target; W65C02, W65C816 opcodes` (2023).
  → Assembleur 65C816 fonctionnel.
- #346 : `Implement REP/SEP emitting on 65816 targets`.
- #455, #456 : `MOS: 65816 assembler fixes` parts 1 et 2.
- #345 : `Fix 65816 BRL handling discrepancies`.

**Verdict #1** : l'**assembleur** llvm-mos supporte 65C816 natif. Le
**compilateur C** ne génère **PAS** de code utilisant les registres
16-bit ni le banking 24-bit.

### 3.2. Issue #320 (24-bit address space)

Body :
> Note that this issue alone, without implementing 16-bit register
> modes, allows for the development of 65816 targets; the only
> remaining distinction for 65816 "native mode" is adjusting
> llvm-mos-sdk's crt0 to initialize the stack pointer to $01FF rather
> than $FF on such targets.

Lecture : pour avoir un target 65C816 utile en C, il "suffit" déjà
d'ajuster le crt0. Le banking 24-bit reste open (pas implémenté).

### 3.3. Issue #321 (16-bit register mode)

Body :
> The main challenge here is being clever about emitting REP/SEP
> instructions to switch, and deciding which register widths to support
> and how — while the 8-bit registers A and B are effectively subsets
> of their 16-bit counterpart, they are not as cheap to access as on
> most LLVM-supported architectures; in addition, the 8-bit registers
> X and Y cannot be treated as subsets of their 16-bit counterparts,
> as their high portions are reset when the 16-bit index register mode
> is disabled.
>
> Note that a target which provides the 65816's 16-bit register modes,
> but not the 24-bit address space, is already present in llvm-mos-sdk
> as "rpc8e" (RedPower 2's RPC8/e "fantasy" computer), and may as such
> prove useful for testing.

Lecture : difficulté technique réelle (REP/SEP + asymétrie X/Y vs A/B).
Pas d'horizon de livraison. Le target `rpc8e` est mentionné comme
"useful for testing" — mais inspection (cf. §3.5) montre que rpc8e
**n'utilise pas non plus les registres 16-bit dans le code C**.

### 3.4. Targets SDK existants (mos-platform/)

Liste complète : `atari2600-*`, `atari5200-*`, `atari8-*`, `c64`,
`c128`, `commodore`, `cpm65`, `cx16`, `dodo`, `eater`, `fds`,
`geos-cbm`, `lynx*`, **`mega65`**, `neo6502`, `nes-*`, **`rpc8e`**,
`snes-*` (probable), …

Targets potentiellement liés au 65C816 :
- **`mega65`** : MEGA65 utilise un **45GS02** (extension 65CE02), pas
  un 65C816 ; donc opcodes 65CE02-spécifiques, pas 65C816.
- **`rpc8e`** : RedPower 2 RPC8/e (fantasy computer Minecraft mod) ;
  utilise `-mcpu=mos65el02` (= 65EL02 + opcodes 65C816), inspection
  détaillée ci-dessous.

### 3.5. Inspection détaillée du target `rpc8e`

`mos-platform/rpc8e/clang.cfg` :
```
-mlto-zp=222
-D__RPC8E__
-mcpu=mos65el02
```

`mos-platform/rpc8e/crt0.S` :
```asm
.section .init.50,"ax",@progbits
; Clear emulation mode.
        clc
        xce
; Initialize hardware stack pointer to 0x01FF.
        rep #$30
        ldx #$01FF
        txs
; Work in 8-bit native mode.            ← CLEF !
        sep #$30
; Clear decimal mode.
        cld
; Enable RedBus by default.
        mmu #$02
; Use the display device by default.
...
```

**Verdict #2** : **rpc8e fait `xce` pour passer en mode N, puis
immédiatement `sep #$30` pour repasser en 8-bit native** (M=1, X=1).
Le code C qui suit est généré en 8-bit. Le mode N est utilisé
uniquement pour avoir accès aux opcodes 65C816 (pile $01FF étendue,
`PEA`, `BRA`, `STZ`, `[dp]`, `(dp)`, `cop`) sans utiliser les
registres 16-bit.

`mos-platform/rpc8e/link.ld` (extrait) :
```
MEMORY {
    zp : ORIGIN = __rc31 + 1, LENGTH = 0x100 - (__rc31 + 1)
    ram (rw) : ORIGIN = 0x0500, LENGTH = 0x10000 - 0x0500
}
```

Lecture : adressage **16-bit linéaire (0x0500-0xFFFF)**, pas de
banking. Confirme l'absence d'utilisation du PBR/DBR.

---

## 4. Synthèse — capacités réelles

| Feature 65C816 | llvm-mos C-compiler | llvm-mos asm |
|----------------|---------------------|--------------|
| Opcodes mode N (REP, SEP, JSL, RTL, COP, PEA, ...) | ✅ via #346, #296 | ✅ |
| Registres 16-bit (M=0, X=0) | ❌ #321 open | ✅ via REP |
| Banking 24-bit (PBR/DBR, JSL across banks) | ❌ #320 open | ✅ |
| Pile étendue $01FF | ✅ (target-spécifique) | ✅ |
| Cross-bank pointer | ❌ #320 open | ✅ (manuel) |

**En clair** : on peut compiler du C llvm-mos vers du **65C816 mode
N 8-bit native** (M=1, X=1) avec quelques opcodes utiles, mais on
n'obtient pas les registres 16-bit ni le banking. Apps **mono-bank
≤ 64 KiB linéaire**.

---

## 5. Comparaison codegen attendue

À taille équivalente d'un programme C non trivial :

| Compilateur | Taille code | Vitesse exec |
|-------------|-------------|--------------|
| asm 65C816 main | référence (1.0×) | référence |
| llvm-mos `mosw65c816` (mode 8-bit) | ~1.1-1.3× | ~0.85-0.95× |
| cc65 (V2.19, target `oric` ou similaire) | ~1.5-2.0× | ~0.60-0.75× |
| llvm-mos `mos6502` (sans opcodes 65C816) | ~1.2-1.5× | ~0.75-0.90× |

(Estimations qualitatives basées sur benchmarks publics llvm-mos vs
cc65, pas mesurées sur OricOS.)

llvm-mos garde un avantage net sur cc65 grâce au codegen LLVM moderne
(register allocation, dead-code elimination, inlining, etc.).

---

## 6. Décisions actées

### ADR-05 v2 (cf. CLAUDE.md §2)

L'ADR-05 est révisée pour intégrer la contrainte **mono-bank 8-bit
native** :
- Kernel + drivers : asm 65C816 mode N (registres 16-bit + banking
  utilisables).
- Userland C : llvm-mos mode N 8-bit native, apps mono-bank ≤ 64 KiB.
- Apps multi-bank ou exigeant registres 16-bit : asm 65C816 ca65.

### DEC-3 (cf. BACKLOG.md)

Initialement formulée : "llvm-mos 65C816 mode N : si bloquant →
fallback asm-only userland v1 ?".

**Décision actée 2026-05-09** : llvm-mos est **conservé** pour le
userland v1 mais avec contraintes (cf. ADR-05 v2). Pas de fallback
asm-only complet. Apps demandant > 64 KiB ou banking → asm
exceptionnellement.

---

## 7. Implications Sprint 4

### Plan révisé Sprint 4 (userland C)

1. **TC-llvmmos-install** : installer llvm-mos pre-built (~250 MB) sur
   le poste de dev. Vérifier `mos-clang --version` et `--print-targets`.
2. **TC-llvmmos-target-oricos** : créer `mos-platform/oricos/`
   custom dans un fork local de llvm-mos-sdk, ou patcher localement :
   - `clang.cfg` : `-mcpu=mosw65c816` ou variante avec opcodes 65C816.
   - `crt0.S` : minimal (mode déjà en N depuis kernel ; setup DP, pile,
     puis JSR `_start` ; au RTS, faire `RTL` pour kernel_app_exec).
   - `link.ld` : `ORIGIN = 0x0200` (load address dans bank app),
     `LENGTH = 0xFE00` (jusqu'à $FFFF dans la bank).
3. **TC-libc-minimal** : libc minimal mappant les appels C aux syscalls
   COP OricOS :
   - `putchar(c)` → `cop #$AA` avec X=c, A=$01.
   - `getchar()` → `cop #$AA` avec A=$04 (à définir, ADR-17).
   - `malloc/free` → bank-local, pas inter-bank.
4. **TC-poc-hello-c** : `hello.c` compilé en bundle .oosobj, exécuté
   via `kernel_app_exec`.

Toute cette mécanique demande au minimum 5-10 jours de travail. Pas
sur le chemin critique tant que Sprint 3 GUI tourne.

---

## 8. Pour mémoire — ce qui n'a PAS été fait dans cette investigation

- Installation effective de llvm-mos.
- Compilation d'un fichier C de test vers binaire 65C816.
- Exécution d'un binaire compilé sur Phosphoric.
- Mesure de codegen comparée cc65 vs llvm-mos sur un cas d'usage réel
  (boucle serrée, accès tableau, ABI fonctions, etc.).

Ces étapes seront faites en **TC-llvmmos-install** au moment
d'attaquer Sprint 4 ou un sprint dédié.

---

## 9. Références

- **llvm-mos** : https://github.com/llvm-mos/llvm-mos
- **llvm-mos-sdk** : https://github.com/llvm-mos/llvm-mos-sdk
- **llvm-mos.org** : https://llvm-mos.org/wiki/Welcome
- **Issue #32** : 65816 support (epic).
- **Issue #320** : [65816] 24-bit address space support.
- **Issue #321** : [65816] 16-bit register mode support.
- **PR #296** (closed) : `add HuC6280 CPU target; W65C02, W65C816 opcodes`.

---

*v0.1 — 2026-05-09, investigation documentaire complète.*
