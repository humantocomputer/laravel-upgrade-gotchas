# Audit de projet — Détection des vestiges de migration

Quand tu arrives sur un projet Laravel, tu ne connais pas son historique de migration. Ce fichier te permet de **scanner le code pour détecter les vestiges** d'anciennes versions et savoir quels gotchas sont réellement applicables.

## Procédure

1. Lire `composer.json` → version de `laravel/framework`, `livewire/livewire`, `pestphp/pest`, `filament/*`
2. Lire `package.json` → version de `tailwindcss`
3. Scanner les patterns ci-dessous
4. Pour chaque vestige trouvé, lire le fichier de référence et évaluer le risque

## Vestiges à scanner

### Service Providers & Structure (vestiges L10)

| Quoi scanner | Comment | Risque si trouvé | Réf. |
|---|---|---|---|
| `EventServiceProvider` avec `$listen` | Fichier `app/Providers/EventServiceProvider.php` existe ET contient `protected $listen` avec des entrées | **CRITIQUE** — listeners dupliqués en L11+ (auto-discovery) | `laravel-12-to-13.md` |
| `RouteServiceProvider` custom | Fichier `app/Providers/RouteServiceProvider.php` existe | Moyen — config ignorée si `bootstrap/app.php` gère le routing | `laravel-10-to-11.md` |
| `AuthServiceProvider` avec `$policies` | Fichier `app/Providers/AuthServiceProvider.php` avec tableau `$policies` | Moyen — policies dupliquées (auto-discovery L11+) | `laravel-10-to-11.md` |
| `BroadcastServiceProvider` | Fichier `app/Providers/BroadcastServiceProvider.php` existe | Faible — redondant mais pas cassant | `laravel-10-to-11.md` |
| `app/Http/Kernel.php` | Fichier existe | Moyen — middleware ignoré si `bootstrap/app.php` gère | `laravel-10-to-11.md` |
| `app/Console/Kernel.php` | Fichier existe | Moyen — schedules et commandes potentiellement dupliqués | `laravel-10-to-11.md` |

```bash
# Scan rapide
ls app/Providers/EventServiceProvider.php app/Providers/RouteServiceProvider.php app/Providers/AuthServiceProvider.php app/Http/Kernel.php app/Console/Kernel.php 2>/dev/null
grep -l 'protected \$listen' app/Providers/EventServiceProvider.php 2>/dev/null
grep -l 'protected \$policies' app/Providers/AuthServiceProvider.php 2>/dev/null
```

### Eloquent (vestiges L10-L12)

| Quoi scanner | Comment | Risque si trouvé | Réf. |
|---|---|---|---|
| `protected $casts =` (propriété) | Grep dans `app/Models/` | Faible en L13 (déprécié mais fonctionnel), sera supprimé en L14 | `laravel-10-to-11.md` |
| `new static()` dans `boot()`/`booted()` | Grep dans `app/Models/` et traits | **CRITIQUE** en L13 — lance `LogicException` | `laravel-12-to-13.md` |
| Relation nommée `casts` | Grep `function casts()` qui retourne une `Relation` | **CRITIQUE** — conflit avec la méthode `casts()` du framework L11+ | `laravel-10-to-11.md` |

```bash
# Scan rapide
grep -rn 'protected \$casts' app/Models/ --include="*.php" | head -5
grep -rn 'new static' app/Models/ --include="*.php" | grep -i 'boot'
```

### Carbon (vestiges Carbon 2)

| Quoi scanner | Comment | Risque si trouvé | Réf. |
|---|---|---|---|
| `diffInDays()` sans `(int)` cast | Grep `diffInDays\|diffInHours\|diffInMinutes\|diffInSeconds` | **HAUT** — retourne float en C3, comparaisons strictes cassées | `carbon-3-migration.md` |
| `createFromTimestamp()` sans timezone | Grep `createFromTimestamp(` sans 2e argument | **HAUT** — UTC au lieu de timezone local | `carbon-3-migration.md` |
| `addMinutes('` avec string | Grep `add(Minutes\|Hours\|Days)\(['"]` | **CRITIQUE** — TypeError en C3 | `carbon-3-migration.md` |

```bash
# Scan rapide
grep -rn 'diffInDays\|diffInHours\|diffInMinutes\|diffInSeconds\|diffInWeeks' app/ --include="*.php" | head -10
grep -rn 'createFromTimestamp(' app/ --include="*.php" | head -5
```

### Rate Limiting (vestige L10)

| Quoi scanner | Comment | Risque si trouvé | Réf. |
|---|---|---|---|
| `decayMinutes` | Grep dans providers et middleware | **CRITIQUE** — 60x trop restrictif en L11+ (secondes au lieu de minutes) | `laravel-10-to-11.md` |

```bash
grep -rn 'decayMinutes' app/ --include="*.php"
```

### Cache & Sessions (vestige L12)

| Quoi scanner | Comment | Risque si trouvé | Réf. |
|---|---|---|---|
| Préfixe underscore dans `config/cache.php` | Grep `'_cache'` ou `Str::slug.*'_'` | Pas cassant si le fichier est conservé, **CRITIQUE** si on le supprime | `laravel-12-to-13.md` |
| Préfixe underscore dans `config/session.php` | Grep `'_session'` | Idem | `laravel-12-to-13.md` |
| Objets Eloquent dans `Cache::remember` | Grep `Cache::remember\|Cache::put` avec des modèles | **HAUT** en L13 — `serializable_classes => false` par défaut | `laravel-12-to-13.md` |

```bash
grep -n "slug.*'_'" config/cache.php config/session.php 2>/dev/null
```

### CSRF (vestige L12)

| Quoi scanner | Comment | Risque si trouvé | Réf. |
|---|---|---|---|
| `VerifyCsrfToken` ou `ValidateCsrfToken` | Grep dans les providers et middleware | Faible — alias dépréciés mais fonctionnels en L13 | `laravel-12-to-13.md` |

### Livewire (vestiges LW3)

| Quoi scanner | Comment | Risque si trouvé | Réf. |
|---|---|---|---|
| Tags `<livewire:` non fermés | Grep `<livewire:[a-z]` sans `/>` dans les Blade | **HAUT** en LW4 — contenu après = slot inattendu | `livewire-3-to-4.md` |
| `wire:model.blur` sans `.live` | Grep `wire:model.blur` (sans `.live.blur`) | Moyen — sync client différent en LW4 | `livewire-3-to-4.md` |
| `wire:transition` avec modifiers | Grep `wire:transition\.` | **HAUT** — modifiers supprimés en LW4, ignorés silencieusement | `livewire-3-to-4.md` |
| Firewall/CSP avec `/livewire/update` | Grep dans config nginx, CSP headers | Moyen — URL changée en `/livewire-{hash}/update` | `livewire-3-to-4.md` |

```bash
# Scan rapide
grep -rn 'wire:model\.blur' resources/views/ --include="*.blade.php" | grep -v '\.live\.blur' | head -5
grep -rn 'wire:transition\.' resources/views/ --include="*.blade.php" | head -5
```

### Tests (vestiges Pest 2 / PHPUnit 9-10)

| Quoi scanner | Comment | Risque si trouvé | Réf. |
|---|---|---|---|
| `withConsecutive` | Grep dans `tests/` | **CRITIQUE** — supprimé PHPUnit 10+ | `pest-phpunit-migration.md` |
| `@test`, `@dataProvider` annotations | Grep dans `tests/` | **CRITIQUE** en PHPUnit 12 — annotations supprimées | `pest-phpunit-migration.md` |
| `createStub()` avec expectations | Grep `createStub` + `expects` | **CRITIQUE** en PHPUnit 12 | `pest-phpunit-migration.md` |
| `getMockForAbstractClass` | Grep dans `tests/` | **CRITIQUE** en PHPUnit 12 — supprimé | `pest-phpunit-migration.md` |
| `tap()` dans Pest | Grep `->tap(` dans `tests/` | Moyen — supprimé Pest 3+, remplacé par `defer()` | `pest-phpunit-migration.md` |

```bash
grep -rn 'withConsecutive\|@test\|@dataProvider\|getMockForAbstractClass' tests/ --include="*.php" | head -10
```

### Filament (vestiges v3)

| Quoi scanner | Comment | Risque si trouvé | Réf. |
|---|---|---|---|
| `Grid::make()` / `Section::make()` sans `->columns()` | Grep dans les resources Filament | **HAUT** en v4+ — 1 colonne par défaut, layouts cassés | `filament-migration.md` |
| Absence de `->deferFilters(false)` | Grep tables Filament, vérifier les filtres | Moyen — UX changée (bouton Apply requis en v4) | `filament-migration.md` |
| FileUpload sans `->visibility('public')` sur S3 | Grep FileUpload dans les resources | **HAUT** en v4+ — private par défaut sur S3 | `filament-migration.md` |

```bash
grep -rn 'Grid::make\|Section::make' app/Filament/ --include="*.php" | grep -v 'columns' | head -5
```

### Tailwind CSS (vestiges v3)

| Quoi scanner | Comment | Risque si trouvé | Réf. |
|---|---|---|---|
| `tailwind.config.js` existe | Fichier à la racine | Config potentiellement ignorée en TW v4 | `tailwind-migration.md` |
| `@tailwind base` / `@tailwind components` / `@tailwind utilities` | Grep dans les fichiers CSS/LESS | **CRITIQUE** en TW v4 — directives supprimées | `tailwind-migration.md` |
| `bg-opacity-*`, `text-opacity-*`, etc. | Grep dans les Blade et CSS | **CRITIQUE** en TW v4 — classes supprimées | `tailwind-migration.md` |
| `bg-gradient-to-*` (ancien nom) | Grep dans les Blade | Moyen en TW v4 — renommé `bg-linear-to-*` | `tailwind-migration.md` |
| `flex-shrink-0` / `flex-grow` | Grep dans les Blade | Faible — renommés `shrink-0` / `grow`, anciens noms parfois alias | `tailwind-migration.md` |
| `darkMode: 'class'` dans config | Grep dans `tailwind.config.js` | **HAUT** en TW v4 — default changé à media query | `tailwind-migration.md` |

```bash
# Scan rapide
grep -rn 'opacity-[0-9]' resources/views/ --include="*.blade.php" | grep -E '(bg|text|border|ring)-opacity' | head -10
grep -rn '@tailwind' resources/ --include="*.css" --include="*.less" | head -5
ls tailwind.config.js tailwind.config.ts 2>/dev/null
```

### Pagination (vestige L12)

| Quoi scanner | Comment | Risque si trouvé | Réf. |
|---|---|---|---|
| `paginate()` sans argument | Grep appels `->paginate()` sans nombre | Moyen en L13 — 25 au lieu de 15, impacte UX et perf | `laravel-12-to-13.md` |

```bash
grep -rn '->paginate()' app/ --include="*.php" | head -10
```

## Résumé par niveau de risque

### CRITIQUE (casse le code ou le comportement immédiatement)

1. `EventServiceProvider::$listen` en L11+ → events dupliqués
2. `new static()` dans `boot()` en L13 → LogicException
3. `addMinutes('30')` string en Carbon 3 → TypeError
4. `decayMinutes` en L11+ → rate limiting 60x trop restrictif
5. `withConsecutive` en PHPUnit 10+ → méthode inexistante
6. `@test`/`@dataProvider` annotations en PHPUnit 12 → ignorées
7. `@tailwind base` en TW v4 → directive inexistante

### HAUT (comportement silencieusement cassé)

1. `diffInDays()` sans `(int)` → comparaisons float fausses
2. `createFromTimestamp()` sans timezone → décalage horaire
3. Tags Livewire non fermés → slots inattendus
4. `wire:transition` avec modifiers → transitions ignorées
5. `bg-opacity-*` en TW v4 → classes ignorées
6. Grid/Section Filament sans `->columns()` → layout 1 colonne
7. Cache d'objets Eloquent en L13 → désérialisation impossible
8. FileUpload Filament sur S3 → fichiers privés par défaut

### MOYEN (fonctionne mais déprécié ou UX changée)

1. `protected $casts` propriété → déprécié, sera supprimé
2. `wire:model.blur` sans `.live` → sync client différent
3. `paginate()` sans argument → 25 au lieu de 15
4. Filament filters deferred → UX différente
5. `VerifyCsrfToken` → alias déprécié
