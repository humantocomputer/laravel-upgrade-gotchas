# Laravel Upgrade Gotchas

Skill [Claude Code](https://claude.ai/code) qui documente tous les pièges et changements de comportement entre versions de l'écosystème Laravel.

## Pourquoi ce skill ?

Les bugs les plus dangereux lors d'une migration Laravel sont les **changements de comportement silencieux** : le code ne génère aucune erreur, mais ne fait plus ce qu'on attend. Exemple : les events/listeners exécutés en double en Laravel 13 à cause de l'auto-discovery.

Ce skill permet à Claude de connaître ces pièges **avant** d'écrire du code, pas après.

## Couverture

| Package | Versions couvertes | Fichier de référence |
|---|---|---|
| Laravel | 10 → 11 → 12 → 13 | `laravel-10-to-11.md`, `laravel-11-to-12.md`, `laravel-12-to-13.md` |
| Carbon | 2 → 3 | `carbon-3-migration.md` |
| Livewire | 3 → 4 | `livewire-3-to-4.md` |
| Pest / PHPUnit | 2→4 / 10→12 | `pest-phpunit-migration.md` |
| Filament | 3 → 4 → 5 | `filament-migration.md` |
| Tailwind CSS | 3 → 4 | `tailwind-migration.md` |

### Tableau de compatibilité

| Laravel | PHP min | Carbon | Livewire | Pest | PHPUnit | Filament | Tailwind |
|---|---|---|---|---|---|---|---|
| 10 | 8.1 | 2 | 3 | 2 | 10 | 3 | 3 |
| 11 | 8.2 | 2/3 | 3 | 2/3 | 10/11 | 3/4 | 3 |
| 12 | 8.2 | 3 | 3 | 3 | 11 | 4 | 3 |
| 13 | 8.3 | 3 | 4 | 4 | 12 | 5 | 4 |

## Installation

Cloner dans le dossier skills de Claude Code :

```bash
# Global (tous les projets)
git clone git@github.com:humantocomputer/laravel-upgrade-gotchas.git ~/.claude/skills/laravel-upgrade-gotchas

# Projet spécifique
git clone git@github.com:humantocomputer/laravel-upgrade-gotchas.git .claude/skills/laravel-upgrade-gotchas
```

Le skill est automatiquement détecté par Claude Code au prochain lancement.

## Audit de projet

Quand Claude arrive sur un nouveau projet Laravel, il ne connaît pas l'historique de migration. Le skill inclut un **audit automatique** (`references/project-audit.md`) qui scanne le code pour détecter les vestiges d'anciennes versions :

- `EventServiceProvider` avec `$listen` encore présent en L13 ? Events dupliqués.
- `diffInDays()` sans cast `(int)` ? Comparaisons float cassées.
- `@tailwind base` dans les CSS ? Directive supprimée en TW v4.
- `withConsecutive` dans les tests ? Méthode supprimée PHPUnit 10+.

Chaque vestige est classé par risque (CRITIQUE / HAUT / MOYEN) avec les commandes grep pour le détecter et le fichier de référence à consulter.

## Déclenchement

Le skill s'active automatiquement quand Claude :

- **Arrive sur un nouveau projet Laravel** (audit des vestiges)
- Touche à :

- Events, listeners, service providers
- Middleware, CSRF
- Sessions, cache, cookies, config Redis
- Eloquent (casts, boot, UUIDs, sérialisation)
- Carbon (dates, diff, timestamps)
- Validation, routing, queues/jobs
- Livewire (wire:model, wire:navigate, transitions)
- Tests Pest/PHPUnit
- Filament forms/tables
- Tailwind CSS (opacité, gradients, dark mode, config)
- Migration entre versions

## Exemples de pièges couverts

**Events dupliqués (Laravel 13)** — L'auto-discovery enregistre les listeners automatiquement. Si un `EventServiceProvider` les enregistre aussi, chaque listener s'exécute deux fois.

**`diffInDays()` retourne un float (Carbon 3)** — `$order->created_at->diffInDays(now()) === 0` ne fonctionne plus car le résultat est `0.75` au lieu de `0`.

**Pagination 25 au lieu de 15 (Laravel 13)** — `paginate()` retourne 25 éléments par défaut, silencieusement.

**Classes d'opacité supprimées (Tailwind v4)** — `bg-opacity-25` n'existe plus. Le codemod automatique ne les migre pas.

**Grid 1 colonne par défaut (Filament v4)** — Tous les formulaires multi-colonnes s'empilent verticalement.

## Contribuer

PR bienvenues pour ajouter des pièges manquants ou couvrir de nouvelles versions.

## Licence

MIT
