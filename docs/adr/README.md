# Architecture Decision Records (ADR) — projet Oric 2

Format **MADR** (Markdown Any Decision Records). Un fichier par ADR.

> **Source de vérité** : `CLAUDE.md` §2 (ratifiées) et §3 (ouvertes / parquées).
> Les fichiers de ce répertoire sont une vue indexée et structurée pour
> référence externe (site mkdocs Phase 4 du programme état-de-l'art).
>
> En cas de divergence entre `CLAUDE.md` et un fichier ADR, **`CLAUDE.md`
> fait foi**. Toute mise à jour est répliquée dans les deux.

## Index

### Ratifiées (CLAUDE.md §2)

| # | Titre | Date | Sujet |
|---|---|---|---|
| 01 | CPU 65C816 | 2026-05-07 | cf. CLAUDE.md §2 |
| 02 | Compositor matériel double ULA | 2026-05-07 | cf. CLAUDE.md §2 |
| 03 | Multitâche préemptif strict | 2026-05-07 | cf. CLAUDE.md §2 |
| 04 | Isolation mémoire bank-based v1 | 2026-05-07 | cf. CLAUDE.md §2 |
| 05 | Langage OricOS asm + C llvm-mos (v2) | 2026-05-07/09 | cf. CLAUDE.md §2 |
| 06 | Modèle GUI SymbOS-like | 2026-05-07 | cf. CLAUDE.md §2 |
| 07 | Système de fichiers FAT32 SD | 2026-05-07 | cf. CLAUDE.md §2 |
| 08 | Packaging apps natives bundle | 2026-05-07 | cf. CLAUDE.md §2 |
| 09 | Audio AY-3-8912 + SID-like | 2026-05-07 | cf. CLAUDE.md §2 |
| 10 | Compatibilité ascendante Oric 1 | 2026-05-07 | cf. CLAUDE.md §2 |
| 11 | Sémantique mode E vs NMOS 6502 | 2026-05-07 | cf. CLAUDE.md §2 |
| 12 | Mode HIRES Oric 2 | 2026-05-08 | cf. CLAUDE.md §2 |
| 13 | Mécanisme syscall COP + table | 2026-05-08 | cf. CLAUDE.md §2 |
| 14 | Format TCB et table tâches | 2026-05-08 | cf. CLAUDE.md §2 |
| 16 | [Driver model](0016-driver-model.md) | 2026-05-09 | hybride event-driven + sync |
| 17 | [ABI syscall userland](0017-abi-syscall-userland.md) | 2026-05-09 | 18 syscalls, sentinelle, COP versionné |
| 18 | [Retrait du 6502](0018-retrait-6502.md) | 2026-05-09 | retrait net post-validation |
| 19 | VRAM SDRAM unifiée (v2) | 2026-05-09 | cf. CLAUDE.md §2 |
| 20 | XVGA 1024×768×4bpp (v3) | 2026-05-09 | cf. CLAUDE.md §2 |
| 21 | GPU Blitter HW autonome | 2026-05-09 | cf. CLAUDE.md §2 |
| 22 | [Clavier Oric 2 paravirtualisé](0022-clavier-oric2-paravirt.md) | 2026-05-23 | hybride : contrôleur KBD2 hôte + matrice virtuelle guest |
| 23 | [Console flux de caractères](0023-console-flux-caracteres.md) | 2026-05-24 | backend interchangeable Oric1↔GPU, ABI sans géométrie/attribut |
| 24 | [Souris Oric 2](0024-souris-oric2.md) | 2026-05-24 | contrôleur $0360-$036F, hybride absolu+delta, IRQ MOU2 |
| 25 | [Modèle de concurrence kernel](0025-modele-concurrence-kernel.md) | 2026-05-25 | Exec-classique : Forbid/Permit + block/wake (signaux), atomicité syscall ; réf AmigaOS Exec / SymbOS |

### Ouvertes / parquées (CLAUDE.md §3)

| # | Titre | Statut | Réouverture |
|---|---|---|---|
| 15 | [Isolation mémoire post-v1](0015-isolation-memoire-post-v1.md) | parquée v2 (2026-05-09) | apps non-trusted OU HW-2 mûr OU 2026-12-31 |
| 26 | [Modèle GUI déclaratif GenUI/SpecUI (DRAFT)](0026-modele-gui-declaratif-DRAFT.md) | **draft / à instruire** (2026-05-26) | ≥ 50 % impl arc SP-3.n (étapes G.1→G.4) |

## Migration progressive

Phase 4 du programme état-de-l'art (S6-S8) prévoit la migration de toutes les ADR ratifiées de §2 vers ce répertoire au format MADR strict. En attendant :
- ADR ratifiées avant 2026-05-09 : référence dans CLAUDE.md §2 uniquement.
- ADR ratifiées le 2026-05-09 et après : fichier MADR dédié + référence CLAUDE.md §2.

## Format MADR utilisé

```markdown
# ADR-NN — Titre

- **Statut** : ratifiée / ouverte / parquée vN / superseded by ADR-MM
- **Date** : YYYY-MM-DD
- **Décideurs** : nom(s)
- **Contexte technique** : référence sprint, ADR liée
- **Conformité moratoire** : oui / dérogation justifiée (cf. CLAUDE.md §10)

## Contexte
...

## Options envisagées
...

## Décision
...

## Conséquences

### Positives
...

### Négatives
...

## Références
- CLAUDE.md §2/§3
- BACKLOG.md
- Sprints/PR
```
