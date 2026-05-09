# ADR-17 — ABI kernel publique exposée à userland

- **Statut** : ratifiée 2026-05-09 (Phase 0 programme état-de-l'art)
- **Date** : 2026-05-09
- **Décideurs** : bmarty, Claude Code
- **Contexte** : ADR-13 (mécanisme syscall COP + table) ratifiée, Sprint 4 userland C bloqué sur ABI manquante, 1 syscall hardcoded en v0.1.
- **Conformité moratoire** : oui — dossier d'instruction écrit, 3 alternatives chiffrées, recommandation senior tracée.

## Contexte

OricOS expose ses services à l'userland via `cop #imm` (ADR-13 ratifié 2026-05-08). v0.1 actuelle dispatche un seul syscall hardcoded (SYS_PRINT_CHAR). Sprint 4 (userland C llvm-mos, TC-libc, TC-poc-hello-c) requiert une ABI complète et stable.

Contraintes :
- llvm-mos cible **mode N 8-bit native** (M=1, X=1) suite ADR-05 v2 et investigation TC-llvmmos. Pas de registres 16-bit ni banking 24-bit dans le compilateur C.
- Convention de retour doit être consommable par le code généré llvm-mos sans intrinsics.
- Versioning ABI long terme : apps v1 doivent rester fonctionnelles si une v2 incompatible apparaît.

## Options envisagées

### Liste syscalls v1

- **(A) Minimale 10 syscalls** : print/read/exit/yield/get_key/fat_open/read/close/panic. Couvre Sprint 4 strict.
- **(B) Étendue 18 syscalls** **[retenue]** : A + alloc_bank/free_bank/gfx_*/sleep_ms. Anticipe SP-3.e (window manager v1).
- **(C) Étendue 24+ avec audio/timer haute précision** : surdimensionné v1.

### Convention d'erreur

- **(α) Carry flag** style GS/OS : élégant en asm, mais llvm-mos ne propage pas le carry au C. Bloque agilité Sprint 4.
- **(β) Sentinelle A=$FF + errno bank 1** **[retenue]** : compat immédiate llvm-mos C (`if (sys_x() == 0xFF) handle_error()`).
- **(γ) Y=errno + A=value** : ABI calling convention non standard llvm-mos, intrinsics requis.

### Versioning

- **Versioning par opcode immediate** **[retenu]** : `cop #$AA` = ABI v1, `cop #$AB` = ABI v2 future. Dispatch séparé. Apps v1 préservées éternellement. Cohérent avec bundle header version (ADR-08).

## Décision

### Mécanisme

```
cop #$AA                    ; ABI v1 signature
  → vecteur $00FFE4 mode N
  → trampoline bank 0 ($0150 : JML $01:5700)
  → kernel_cop_handler (bank 1 segment COP_HANDLER)
       cmp #$40
       bcs sys_invalid
       asl A
       tax
       jsr (syscall_table,X)
       ; → routine kernel locale, RTI
```

### Convention d'appel canonique (ABI v1)

- **Entrée** : `A` = num syscall (0..63), `X` et `Y` = args 8-bit.
- **Sortie** : `A` = valeur retour 8-bit, `Y` = high byte si retour 16-bit.
- **Préservés par le kernel** : `X` (sauf retour multi-byte), `D` (DPR), `DBR`, `PBR`, pile.
- **Args > 2 bytes** : bloc d'args en zero-page kernel-réservée `$D0-$DF` (8 bytes).
- **Erreur** : `A=$FF` → erreur, code dans `errno` bank 1 `$5760` (lecture directe DBR=0 ou via SYS_GETERRNO futur).

### Liste syscalls v1 (18 syscalls, slots `$01-$12`)

| # | Nom | Args | Retour |
|---|---|---|---|
| `$01` | SYS_PRINT_CHAR | X=char | — |
| `$02` | SYS_PRINT_STRING | X/Y=str_ptr (DBR-rel) | — |
| `$03` | SYS_READ_CHAR | bloquant | A=char |
| `$04` | SYS_EXIT | X=exit_code | n/a |
| `$05` | SYS_YIELD | — | — |
| `$06` | SYS_GET_KEY | non-bloquant | A=keycode/0 |
| `$07` | SYS_FAT_OPEN | X/Y=name_ptr | A=fd, $FF=err |
| `$08` | SYS_FAT_READ | bloc zp | A=nbytes, $FF=err |
| `$09` | SYS_FAT_CLOSE | X=fd | — |
| `$0A` | SYS_PANIC | X/Y=code | n/a |
| `$0B` | SYS_ALLOC_BANK | — | A=bank, $FF=err |
| `$0C` | SYS_FREE_BANK | X=bank | — |
| `$0D` | SYS_GFX_CLEAR | bloc zp | A=$00/$FF |
| `$0E` | SYS_GFX_FILL_RECT | bloc zp | A=$00/$FF |
| `$0F` | SYS_GFX_BLIT | bloc zp | A=$00/$FF |
| `$10` | SYS_GFX_LINE | bloc zp | A=$00/$FF |
| `$11` | SYS_GFX_TEXT | bloc zp | A=$00/$FF |
| `$12` | SYS_SLEEP_MS | X/Y=ms16 | — |

Slots `$00`, `$13-$3F` réservés extensions v1+. `$40-$7F` réservés futur. `$80-$FF` réservés signaux/contrôle système.

### Dispatch v0.2

Table `syscall_table` à bank 1 `$5750`, 64 entrées × 2 octets = 128 B. Migration de la v0.1 cmp/bne hardcoded vers table en Phase 1 du programme (sprint OS-2.f.v2).

## Conséquences

### Positives

- Sprint 4 userland C débloqué : ABI stable et consommable directement par llvm-mos.
- Versioning par opcode immediate permet introduction v2 sans casser v1.
- Liste étendue 18 syscalls couvre window manager v1 (SP-3.e).
- Slot reservé permet croissance sans re-design.

### Négatives

- 128 B en bank 1 pour table dispatch (négligeable).
- Errno global non thread-safe v1 : accepté car scheduler préemptif sauvegarde context complet (TCB ADR-14), donc pas de race intra-task. Multi-task race possible si 2 tasks font syscall simultané sur erreur — à instruire en cas d'incident, mitigation triviale (errno par TCB).
- Args > 2 bytes via zero-page partagée : nécessite que les apps respectent la convention. Documenté dans libc et bundle.

## Références

- CLAUDE.md §2 (ADR-17)
- ADR-13 (mécanisme syscall), ADR-14 (TCB), ADR-05 v2 (langage), ADR-08 (bundle)
- `OricOS/kernel/kernel.s` segment COP_HANDLER (à étendre Phase 1)
- BACKLOG sprint OS-2.f.v2
