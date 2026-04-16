# Laravel 12 → 13 : Changements

La v13 apporte l'auto-discovery des events, un nouveau CSRF, et des changements dans la sérialisation du cache. L'auto-discovery des events est le piège le plus fréquent lors de la migration — les listeners s'exécutent en double si l'ancien `EventServiceProvider` est encore présent.

**PHP 8.3 minimum requis.**

## Changements critiques (comportement silencieux)

### Pagination : 25 au lieu de 15 par défaut

**Avant** : `Model::paginate()` retournait 15 items par page.

**Après** : 25 items par page par défaut.

**Piège** : toutes les listes paginées afficheront plus d'éléments sans aucune erreur. Peut impacter la performance, le layout, et l'UX.

**Fix** : définir `$perPage` explicitement sur les modèles ou passer le paramètre à `paginate(15)`.

### Job queue : format de sérialisation incompatible

**Avant (L12)** : format de sérialisation des payloads de jobs.

**Après (L13)** : format plus compact et différent.

**Piège CRITIQUE pour le déploiement** : les jobs mis en queue par Laravel 12 **échoueront** sur un worker Laravel 13. Il faut **drainer les queues avant le déploiement**.

**Fix** : `php artisan queue:work --stop-when-empty` sur toutes les queues avant de déployer la nouvelle version.

### Events auto-discovery complète (LE piège classique)

**Avant** : `EventServiceProvider::$listen` était le moyen standard d'enregistrer events → listeners.

**Après** : Laravel auto-discover tous les listeners dans `app/Listeners/`. Si `EventServiceProvider` existe encore avec un tableau `$listen`, les listeners seront enregistrés DEUX FOIS — une fois par auto-discovery, une fois par le provider.

**Conséquences** : emails envoyés en double, stock décrémenté deux fois, sessions corrompues, panier incohérent.

**Fix** :
- Supprimer le tableau `$listen` de `EventServiceProvider` (ou supprimer le fichier entier)
- S'assurer que chaque listener a un `handle(SpecificEvent $event)` correctement typé
- Pour exclure un listener : `#[ExcludeFromAutoDiscovery]` ou config dans `bootstrap/app.php`

### Préfixes cache/session Redis

**Avant** : `Str::slug(APP_NAME, '_') . '_cache_'` (underscore : `budo_fight_cache_`)

**Après** : `Str::slug(APP_NAME) . '-cache-'` (hyphen : `budo-fight-cache-`)

**Piège** : si on supprime les fichiers `config/cache.php` ou `config/session.php` custom pour "simplifier" et utiliser les defaults, TOUS les caches sont invalidés et TOUS les utilisateurs déconnectés.

**Fix** : conserver les fichiers de config existants avec les préfixes explicites. Ou définir `CACHE_PREFIX` et `SESSION_COOKIE` dans `.env`.

### `serializable_classes => false` par défaut

**Avant** : tout objet PHP pouvait être sérialisé/désérialisé depuis le cache.

**Après** : seuls les scalaires et tableaux sont autorisés par défaut. Les objets Eloquent, DTOs, collections stockés via `Cache::put()` ou `Cache::remember()` ne pourront plus être lus.

**Piège** : `unserialize(): Allowed classes option should be array or true` en production.

**Fix** : soit lister les classes dans `config/cache.php` :
```php
'serializable_classes' => [
    App\Models\Product::class,
    App\DTOs\CartItem::class,
],
```
Soit (recommandé) ne cacher que des tableaux/scalaires et convertir les objets avec `->toArray()`.

### CSRF : `PreventRequestForgery` + `Sec-Fetch-Site`

**Avant** : `VerifyCsrfToken` / `ValidateCsrfToken` vérifie uniquement le token CSRF.

**Après** : `PreventRequestForgery` vérifie aussi le header `Sec-Fetch-Site`. Les requêtes cross-origin sans ce header sont rejetées même avec un token valide.

**Piège** : webhooks externes (Stripe, PayPal, etc.), intégrations AJAX cross-origin, et APIs partenaires qui postent vers des routes web seront bloqués.

**Fix** : exclure les routes concernées via `$middleware->preventRequestForgery(except: [...])` dans `bootstrap/app.php`.

### Instantiation de modèle interdite dans `boot()`

**Avant** : `(new static)->getTable()` dans `boot()` ou `boot*()` fonctionnait.

**Après** : lance une `LogicException`.

**Piège** : traits custom qui font `new static()` dans `boot()` pour accéder à des propriétés du modèle.

**Fix** : utiliser `$this->getTable()` dans `booted()`, ou `static::resolveRelationUsing()` avec un callback.

### Serialisation des collections — relations restaurées

**Avant** : les relations eager-loaded étaient perdues après sérialisation (ex: dans un job queue).

**Après** : les relations sont restaurées après désérialisation.

**Piège** : si du code dépendait du fait que les relations étaient absentes après passage en queue.

### Pivot polymorphique pluralisé

**Avant** : nom singulier inféré pour les tables pivot morph.

**Après** : nom pluralisé.

**Piège** : si des morph pivot models custom n'ont pas de `$table` explicite.

**Fix** : définir `protected $table = 'nom_exact'` sur les pivot models.

## Breaking changes explicites

### `JobAttempted` event

**Avant** : `$event->exceptionOccurred` (boolean).

**Après** : `$event->exception` (objet Exception ou null).

### `QueueBusy` event

**Avant** : `$event->connection`.

**Après** : `$event->connectionName`.

### `Js::from()` unicode

**Avant** : les caractères non-ASCII étaient échappés (`\u00e8`).

**Après** : caractères directs (`è`).

**Piège** : tests frontend qui comparent des strings contenant des séquences échappées.

### Sujet du mail de reset password

**Avant** : "Reset Password Notification"

**Après** : "Reset your password"

**Piège** : tests ou traductions custom qui vérifient le sujet exact.

### `Manager::extend()` binding

**Avant** : `$this` dans le callback = service provider.

**Après** : `$this` = instance du manager.

**Piège** : les custom drivers enregistrés via `extend()` qui utilisaient `$this->app` doivent utiliser `app()` à la place.

### `withScheduling` différé

Les schedules enregistrés via `ApplicationBuilder::withScheduling()` sont différés jusqu'à la résolution du `Schedule`. Du code qui dépend du timing d'enregistrement au boot peut casser.

### Polyfill PHP 8.5 : `array_first()`, `array_last()`

Symfony polyfill-php85 introduit ces fonctions en global. Conflit possible avec le package `laravel/helpers` ou tout code custom définissant ces fonctions.

### Méthodes supprimées

| Méthode | Remplacement |
|---|---|
| `Route::controller()` | `Route::resource()` ou routes explicites |
| `$request->has()` avec array | `$request->hasAny()` |
| `Str::slug()` avec séparateur en 2e arg positionnel | Argument nommé `separator:` |

### Contrat `Cache\Store` : nouvelle méthode `touch()`

Les custom cache store drivers doivent implémenter cette méthode.

### Noms de vues de pagination renommés

`pagination::default` → `pagination::bootstrap-3`, `pagination::simple-default` → `pagination::simple-bootstrap-3`. Si le code référence les anciens noms, il cassera.

## Dépréciations

| Élément | Remplacé par | Notes |
|---|---|---|
| `$casts` propriété | `casts()` méthode | Suppression prévue L14 |
| `VerifyCsrfToken` / `ValidateCsrfToken` | `PreventRequestForgery` | Alias dépréciés encore fonctionnels |
| `EventServiceProvider::$listen` | Auto-discovery | Le fichier est ignoré du skeleton |
| `$event->exceptionOccurred` | `$event->exception` | Sur `JobAttempted` |
