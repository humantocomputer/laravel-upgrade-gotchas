# Laravel 10 → 11 : Changements

La v11 est la plus grosse restructuration depuis Laravel 5. L'application est simplifiée : moins de fichiers, configuration centralisée dans `bootstrap/app.php`.

**PHP 8.2 minimum requis.**

## Changements critiques (comportement silencieux)

### Rate Limiting : minutes → secondes

**Avant** : `RateLimiter::for('api', fn () => Limit::perMinute(60))` — le `decayMinutes` contrôlait la fenêtre.

**Après** : `decayMinutes` renommé en `decaySeconds`. `ThrottlesExceptions` et `ThrottlesExceptionsWithRedis` acceptent aussi des secondes.

**Piège** : si du code custom utilise `decayMinutes` ou définit des rate limiters avec des valeurs en minutes, les limiteurs seront **60x plus restrictifs** (1 minute au lieu de 60).

**Fix** : multiplier les valeurs par 60 ou utiliser les nouveaux noms de paramètres.

### Suppression des Service Providers par défaut

**Avant (L10)** : 5 providers — `AppServiceProvider`, `AuthServiceProvider`, `BroadcastServiceProvider`, `EventServiceProvider`, `RouteServiceProvider`

**Après (L11)** : seul `AppServiceProvider` subsiste. Le framework gère le reste automatiquement.

**Piège** : si un package tiers ou du code custom référence `RouteServiceProvider::HOME` ou étend `EventServiceProvider`, il cassera silencieusement ou ignorera la config.

**Fix** : migrer toute logique custom vers `AppServiceProvider::boot()` ou `bootstrap/app.php`.

### Auto-discovery des Events/Listeners

**Avant** : `EventServiceProvider::$listen` mappait events → listeners manuellement.

**Après** : Laravel scanne `app/Listeners/` et détecte les events via le type-hint de `handle()` ou `__invoke()`.

**Piège** : si un listener a un type-hint incorrect ou ambigu dans `handle()`, il ne sera pas découvert — sans aucune erreur.

**Fix** : chaque listener doit avoir exactement un paramètre typé dans `handle(OrderPlacedEvent $event)`.

### Auto-discovery des Policies

**Avant** : `AuthServiceProvider::$policies` mappait modèles → policies manuellement.

**Après** : Laravel découvre automatiquement les policies si elles suivent la convention `App\Policies\{Model}Policy`.

**Piège** : une policy dans un sous-dossier non standard ne sera pas découverte.

### Suppression du HTTP Kernel

**Avant** : `app/Http/Kernel.php` avec `$middleware`, `$middlewareGroups`, `$routeMiddleware`.

**Après** : tout dans `bootstrap/app.php` via `->withMiddleware()`.

**Piège** : les alias de middleware custom (ex: `'admin' => AdminMiddleware::class`) doivent être re-déclarés dans `bootstrap/app.php`. Un alias oublié donnera une erreur 404 ou un middleware non appliqué.

**Fix** :
```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'admin' => AdminMiddleware::class,
    ]);
})
```

### Routes : `bootstrap/app.php` remplace `RouteServiceProvider`

**Avant** : `RouteServiceProvider::boot()` avec `Route::middleware('web')->group(...)`.

**Après** : `->withRouting(web: ..., api: ...)` dans `bootstrap/app.php`.

**Piège** : le préfixe `api/` n'est plus appliqué automatiquement aux routes API. Il faut le définir explicitement avec `apiPrefix: 'api'`.

## Breaking changes explicites

### Casts : introduction de `casts()`

**Avant** : `protected $casts = ['status' => StatusEnum::class]` (propriété).

**Après** : la méthode `casts()` est introduite comme alternative. La propriété fonctionne encore mais sera dépréciée.

**Action** : pas urgent, mais planifier la migration vers `casts()` progressivement.

```php
// Nouveau style (L11+)
protected function casts(): array
{
    return [
        'status' => StatusEnum::class,
        'password' => 'hashed',
    ];
}
```

### `once()` helper global

Nouveau helper pour mémoiser une valeur par instance d'objet. Peut entrer en conflit si un helper `once()` custom existe déjà.

### Health check route

Laravel 11 ajoute une route `/up` par défaut. Peut entrer en conflit avec une route existante du même nom.

### Artisan commandes auto-discovered

Les commandes dans `app/Console/Commands/` sont auto-discovered. Le fichier `app/Console/Kernel.php` n'est plus nécessaire. Les commandes enregistrées manuellement via `$commands` seront exécutées en double si elles sont aussi dans le dossier standard.

### Suppression de la dépendance `doctrine/dbal`

**Avant** : `doctrine/dbal` était inclus pour les modifications de colonnes dans les migrations (`change()`).

**Après** : Laravel utilise ses propres méthodes. Si le projet dépendait de `doctrine/dbal` pour des introspections custom, il faut le réinstaller explicitement.

### Contrat `Authenticatable` : nouvelle méthode `getAuthPasswordName()`

Si un modèle implémente le contrat `Authenticatable` directement (sans le trait `AuthenticatableTrait`), il doit ajouter cette méthode.

### Conflit potentiel : relation nommée `casts`

Le modèle Eloquent définit maintenant une méthode `casts()`. Si un modèle a une **relation** nommée `casts`, il y a conflit de noms.

### Sanctum : middleware references

Dans `config/sanctum.php`, les middleware `authenticate_session`, `encrypt_cookies`, `validate_csrf_token` doivent utiliser les noms de classe pleinement qualifiés (plus d'alias string).

### Migrations des packages first-party

Les migrations de Cashier Stripe (≥15.0), Passport (≥12.0), Sanctum, Spark Stripe (≥5.0), Telescope ne se chargent plus automatiquement. Il faut les publier manuellement via `php artisan vendor:publish`.

## Dépréciations

| Élément | Remplacé par | Supprimé en |
|---|---|---|
| `EventServiceProvider` | Auto-discovery | L12 (fichier absent du skeleton) |
| `RouteServiceProvider` | `bootstrap/app.php` | L12 |
| `AuthServiceProvider` | Auto-discovery policies | L12 |
| `app/Http/Kernel.php` | `->withMiddleware()` | L12 |
| `$casts` propriété | `casts()` méthode | L14 (prévu) |
| `app/Console/Kernel.php` | `->withSchedule()` | L12 |
