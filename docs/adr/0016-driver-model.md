# ADR-16 — Driver model OricOS

- **Statut** : ratifiée 2026-05-09 (Phase 0 programme état-de-l'art)
- **Date** : 2026-05-09
- **Décideurs** : bmarty, Claude Code
- **Contexte** : Sprints OS-2.d (clavier), OS-4.a (audio AY) à venir. ADR-21 GPU async v2 à modéliser. Pattern hybride déjà tacite dans le kernel.s actuel (clavier IRQ-driven via VIA T1, FAT32/console sync).
- **Conformité moratoire** : oui — ratification d'un pattern majoritairement déjà implémenté (kbd init/scan, console init/print, fat_*, gpu_* en place). Formalisation pour drivers à venir.

## Contexte

OricOS doit définir un modèle de driver standard avant de multiplier les sources matérielles (clavier OS-2.d, audio AY OS-4.a, GPU async ADR-21 v2, timer ms). Le kernel.s actuel a déjà adopté tacitement un modèle hybride non-formalisé : kernel_kbd_scan invoqué par IRQ VIA T1, kernel_fat_* en sync bloquant, etc.

Sans formalisation explicite, les drivers à venir risquent de diverger (audio en polling, autre en callback, etc.) sans cohérence d'API.

## Options envisagées

### (a) Tout IRQ-driven

Tous drivers exposent un handler IRQ + queue d'événements. Apps lisent en non-blocking.

- ✅ Uniformité, séparation claire.
- ⚠️ FAT32/console/bank alloc actuels sync devraient être refactorés en async — surcoût sans bénéfice. Contredit pattern existant.
- **Écartée**.

### (b) Hybride event-driven + sync **[retenue]**

- IRQ-driven event queue pour drivers à latence faible critique (clavier, audio, GPU async).
- Sync/blocking pour drivers naturellement bloquants (FAT32, console, GPU sync v1, bank alloc).
- Polling idle réservé futur.

- ✅ Ratifie pattern existant.
- ✅ Pragmatique, pas de refactor inutile.

### (c) Hybride + struct ops formelle dès v1

Vtable formelle pour chaque driver, même mono-instance. Prépare v2 modules dynamiques.

- Surcoût indirection asm 65C816 : `jsr (vtable,X)` au lieu de `jsr label`.
- Surcoût RAM ~10-20 B/driver.
- Justifié seulement si modules dynamiques anticipés v1 — non.
- **Écartée v1** ; ouverte v2 si besoin réel module dynamique ou multi-OS host.

## Décision

### Modèle

**Hybride event-driven + sync**, sans struct ops formelle v1. Convention de nommage `kernel_<drv>_<op>`.

### Classes de drivers

**Classe 1 — IRQ-driven event queue** :

| Driver | Source IRQ | Event queue | Wakeup userland |
|---|---|---|---|
| Clavier (OS-2.d) | IRQ contrôleur KBD2 `$0350-$035F` (révisé ADR-22, 2026-05-23 ; était VIA T1 scan) | ring buffer 16 keycodes en bank 1 `$5860` | SYS_GET_KEY (non-bloquant), SYS_READ_CHAR (bloquant) |
| Audio AY (OS-4.a) | VIA T2 ou tick NMI | feed AY registers depuis buffer | non exposé v1 |
| GPU async (ADR-21 v2) | GPU IRQ done | flag bit + callback | (futur) |
| Timer ms | NMI tick scheduler | TCB blocked list | wake on counter |

**Classe 2 — Sync/blocking** : FAT32 (`kernel_fat_*`), console (`kernel_print_*`), GPU sync v1 (`kernel_gfx_*`), bank alloc (`kernel_alloc_bank`).

**Classe 3 — Polling idle** : aucun en v1.

### Mécanisme IRQ formalisé

```
IRQ matériel mode N → vecteur $00FFEE
  → trampoline bank 0 ($0140 : JML $01:5600)
  → kernel_irq_dispatch (bank 1 $5600)
       lit VIA_IFR
       cas T1 (timer)  → kernel_kbd_scan + kernel_sched_tick
       cas T2 (audio)  → kernel_audio_tick (futur)
       cas autre       → ignore + ack
       RTI
```

Table dispatch IRQ à **`$01:5680`** (8 entrées × 2B = 16B), indexée par bit IFR. Drivers s'enregistrent dynamiquement via `driver_init`.

### Convention ring buffer events

- Slot fixe en bank 1 (~16 entrées × N bytes selon driver).
- Head/tail 8-bit en zero-page kernel.
- Sentinelle pop=0 cohérente avec convention syscall ADR-17.

### Spécifique clavier

- Ring buffer 16 keycodes × 1 byte = 16 B en bank 1 `$5860`.
- SYS_GET_KEY pop tail, return A=keycode (0=empty).

## Conséquences

### Positives

- Ratifie pattern existant, formalise pour drivers à venir.
- Pas de refactor douloureux du code actuel.
- Compatible compacité asm 65C816.

### Négatives

- Pas de modules dynamiques v1 (acceptable, non prévu avant v2).
- Convention nommage `kernel_<drv>_<op>` à respecter rigoureusement (discipline humaine, pas vérifiée automatiquement).

## v2 ouvertures (parquées)

- Struct ops (vtable) pour modules dynamiques chargeables.
- Driver discoverability runtime (table `__drivers_start`/`__drivers_end` via linker).
- Hot-reload drivers (debug only).

→ Réouverture déclenchée par : besoin réel de driver dynamique, ou multi-OS host.

## Références

- CLAUDE.md §2 (ADR-16)
- ADR-03 (préemptif strict), ADR-13 (syscall), ADR-14 (TCB), ADR-17 (ABI), ADR-21 (GPU async v2)
- `OricOS/kernel/kernel.s` (kernel_kbd_*, kernel_fat_*, kernel_gfx_*)
- BACKLOG OS-2.d, OS-4.a
