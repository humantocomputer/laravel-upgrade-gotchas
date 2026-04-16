---
name: laravel-upgrade-gotchas
description: "Pièges et changements de comportement entre versions Laravel (10→11→12→13), Carbon 2→3, Livewire 3→4, Pest 2→4, Filament 3→5, Tailwind 3→4. TOUJOURS invoquer ce skill quand on : arrive sur un nouveau projet Laravel (audit des vestiges), touche à events/listeners, service providers, middleware, sessions/cache/cookies, Eloquent casts/boot/UUID, Carbon dates (diffIn*, createFromTimestamp), validation rules, routing, queues/jobs, Livewire wire:model, tests Pest/PHPUnit, Filament forms/tables, classes Tailwind CSS (opacité, gradients, dark mode, config), supprime ou modifie des fichiers de config (cache.php, session.php, tailwind.config.js), ou migre entre versions. Invoquer même en cas de doute — mieux vaut vérifier que de casser silencieusement."
license: MIT
metadata:
  author: humantocomputer
---

# Laravel Upgrade Gotchas

Ce skill existe parce que les changements de comportement silencieux entre versions Laravel sont les bugs les plus dangereux : aucune erreur, mais le code ne fait plus ce qu'on attend. Exemple vécu : les events/listeners dupliqués en Laravel 13 à cause de l'auto-discovery.

## Matrice de décision

Avant de modifier du code dans ces domaines, lis le fichier de référence correspondant :

| Tu touches à... | Lis ce fichier |
|---|---|
| Events, listeners, `EventServiceProvider` | `references/laravel-12-to-13.md` (auto-discovery) |
| Service providers (App, Auth, Route, Broadcast) | `references/laravel-10-to-11.md` (consolidation) |
| HTTP Kernel, middleware global | `references/laravel-10-to-11.md` (suppression Kernel) |
| CSRF, `VerifyCsrfToken`, `PreventRequestForgery` | `references/laravel-12-to-13.md` (Sec-Fetch-Site) |
| `config/cache.php`, `config/session.php`, préfixes Redis | `references/laravel-12-to-13.md` (préfixes) |
| `Cache::put()`, `Cache::remember()` avec objets | `references/laravel-12-to-13.md` (serializable_classes) |
| `$casts`, `casts()`, attributs Eloquent | `references/laravel-10-to-11.md` + `references/laravel-12-to-13.md` |
| `boot()`, `booted()` dans les modèles | `references/laravel-12-to-13.md` (instantiation interdite) |
| UUIDs, `HasUuids` | `references/laravel-11-to-12.md` (UUIDv7) |
| Carbon : `diffIn*`, `createFromTimestamp`, `add*` | `references/carbon-3-migration.md` |
| Validation : `image`, `mergeIfMissing` | `references/laravel-11-to-12.md` |
| Routes : nommage, précédence, domaines | `references/laravel-11-to-12.md` + `references/laravel-12-to-13.md` |
| Queues/jobs : events, scheduling | `references/laravel-12-to-13.md` |
| `wire:model`, `wire:transition`, tags Livewire | `references/livewire-3-to-4.md` |
| Container injection, nullable defaults | `references/laravel-11-to-12.md` |
| `Js::from()`, helpers globaux | `references/laravel-12-to-13.md` |
| Tests Pest, PHPUnit, `withConsecutive`, mocks | `references/pest-phpunit-migration.md` |
| Filament forms, tables, Grid/Section layout | `references/filament-migration.md` |
| Tailwind classes, opacité, gradients, dark mode | `references/tailwind-migration.md` |
| `tailwind.config.js`, `@tailwind` directives | `references/tailwind-migration.md` |
| Rate limiting, `decayMinutes` | `references/laravel-10-to-11.md` |
| Pagination (`paginate()` default) | `references/laravel-12-to-13.md` |
| Queue jobs, déploiement, sérialisation | `references/laravel-12-to-13.md` |
| Migration entre versions Laravel | Tous les fichiers pertinents selon les versions source→cible |

## Audit de projet — première chose à faire sur un nouveau projet

Quand tu arrives sur un projet Laravel dont tu ne connais pas l'historique, **ne te fie pas uniquement à la version dans `composer.json`**. Un projet en Laravel 13 peut encore contenir des vestiges de L10 qui causent des bugs silencieux (events dupliqués, rate limiting cassé, etc.).

**Lis `references/project-audit.md`** qui contient la liste complète des patterns à scanner, classés par niveau de risque (CRITIQUE, HAUT, MOYEN). Chaque pattern indique :
- Ce qu'il faut chercher (fichier, grep)
- De quelle version c'est un vestige
- Le risque concret
- Le fichier de référence à lire

Les vestiges les plus dangereux à détecter en priorité :
1. `EventServiceProvider` avec `$listen` → events dupliqués
2. `new static()` dans `boot()` → LogicException en L13
3. `decayMinutes` dans rate limiters → 60x trop restrictif
4. `diffInDays()` sans `(int)` → comparaisons float cassées
5. `@tailwind base` dans CSS → directive supprimée TW v4
6. `withConsecutive` dans tests → supprimé PHPUnit 10+

## Règles universelles

Ces règles s'appliquent toujours, quelle que soit la version :

1. **Ne jamais enregistrer manuellement ce que Laravel auto-discover.** Depuis Laravel 11, les events/listeners, les policies, et les commandes Artisan sont auto-discovered. Un enregistrement manuel en plus = exécution en double.

2. **Ne jamais supprimer `config/cache.php` ou `config/session.php` sans vérifier les préfixes.** Les defaults du framework ont changé (underscore → hyphen en L13). Supprimer ces fichiers invalide tous les caches Redis et déconnecte tous les utilisateurs.

3. **Toujours caster `(int)` les résultats de `diffIn*()`** (Carbon 3). Ces méthodes retournent des `float` depuis Carbon 3. Toute comparaison stricte (`=== 0`) cassera silencieusement.

4. **Toujours passer le timezone explicitement à `createFromTimestamp()`** (Carbon 3). Le défaut est passé de `date_default_timezone_get()` à UTC.

5. **Toujours fermer les tags Livewire** : `<livewire:component />` (Livewire 4). Sans `/`, le contenu après le tag est interprété comme slot.

6. **Ne jamais instancier un modèle dans `boot()`/`booted()`** (Laravel 13). `new static()` dans ces méthodes lance une `LogicException`.

7. **Drainer les queues avant un déploiement de version majeure Laravel.** Le format de sérialisation des jobs peut changer entre versions (ex: L12→L13). Les jobs en queue de l'ancienne version échoueront sur le nouveau worker.

8. **Vérifier les classes d'opacité Tailwind après migration v4.** `bg-opacity-*`, `text-opacity-*` etc. sont supprimées en Tailwind v4. Le codemod automatique ne les migre PAS.

## Compatibilité des versions — tableau de référence

| Laravel | PHP min | Carbon | Livewire | Pest | PHPUnit | Filament | Tailwind |
|---|---|---|---|---|---|---|---|
| 10 | 8.1 | 2 | 3 | 2 | 10 | 3 | 3 |
| 11 | 8.2 | 2/3 | 3 | 2/3 | 10/11 | 3/4 | 3 |
| 12 | 8.2 | 3 | 3 | 3 | 11 | 4 | 3 |
| 13 | 8.3 | 3 | 4 | 4 | 12 | 5 | 4 |

## Checklist pré-modification

Avant de modifier events, providers, middleware, ou config :

- [ ] Vérifier la version Laravel du projet (`composer.json` → `laravel/framework`)
- [ ] Lire le fichier de référence correspondant dans la matrice ci-dessus
- [ ] Vérifier si le pattern qu'on s'apprête à utiliser est obsolète dans cette version
- [ ] Si on touche à la config cache/session : vérifier les préfixes existants et ne pas les changer involontairement
