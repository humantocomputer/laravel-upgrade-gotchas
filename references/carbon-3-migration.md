# Carbon 2 → 3 : Migration

Carbon 3 est obligatoire depuis Laravel 12. Les changements sont subtils mais omniprésents — ils touchent tout code qui manipule des dates.

## Changements critiques (comportement silencieux)

### `diffIn*()` retourne des float

**Avant** : `$start->diffInDays($end)` retournait `int` (ex: `5`).

**Après** : retourne `float` (ex: `5.23`).

**Piège** :
```php
// CASSE silencieusement en Carbon 3
if ($order->created_at->diffInDays(now()) === 0) { ... }
// Retourne 0.75 au lieu de 0 → la condition est toujours fausse

// CASSE aussi
$prices[$date->diffInDays($start)] = $price;
// L'index est un float → warning ou comportement inattendu
```

**Fix** : toujours caster en int :
```php
if ((int) $order->created_at->diffInDays(now()) === 0) { ... }
// Ou utiliser les méthodes fluent
if ($order->created_at->isToday()) { ... }
```

### `createFromTimestamp()` timezone UTC

**Avant** : utilisait `date_default_timezone_get()` (Europe/Paris dans la plupart des projets FR).

**Après** : UTC par défaut.

**Piège** : tous les timestamps seront interprétés avec +0 au lieu de +1/+2 (selon l'heure d'été). Les dates affichées seront décalées de 1 ou 2 heures.

**Fix** : toujours passer le timezone explicitement :
```php
Carbon::createFromTimestamp($timestamp, 'Europe/Paris');
// Ou
Carbon::createFromTimestamp($timestamp, config('app.timezone'));
```

### `addMinutes()` etc. n'acceptent plus de strings

**Avant** : `->addMinutes('30')` fonctionnait (cast implicite).

**Après** : `TypeError` — seuls `int` et `float` sont acceptés.

**Piège** : les valeurs provenant de config, base de données, ou formulaires arrivent souvent en string.

**Fix** :
```php
->addMinutes((int) $config->get('delay_minutes'));
```

### `setTestNow()` copie l'objet

**Avant** : modifier `$date` après `Carbon::setTestNow($date)` modifiait aussi le mock.

**Après** : l'objet est copié, les modifications ultérieures n'affectent plus le mock.

**Piège** : tests qui modifient la date mockée après `setTestNow()` pour simuler le passage du temps.

**Fix** : appeler `setTestNow()` à nouveau avec la nouvelle date :
```php
Carbon::setTestNow(now());
// ... faire des choses ...
Carbon::setTestNow(now()->addHour()); // Nouveau mock, pas de modification en place
```

### Constructeur strict

**Avant** : `new Carbon('invalid')` retournait la date courante ou une approximation.

**Après** : lance une exception pour les formats invalides.

**Piège** : du code qui passait des valeurs potentiellement vides ou malformées au constructeur.

## Méthodes affectées — référence rapide

| Méthode | Avant (Carbon 2) | Après (Carbon 3) | Fix |
|---|---|---|---|
| `diffInDays()` | `int` | `float` | `(int) $a->diffInDays($b)` |
| `diffInHours()` | `int` | `float` | `(int) $a->diffInHours($b)` |
| `diffInMinutes()` | `int` | `float` | `(int) $a->diffInMinutes($b)` |
| `diffInSeconds()` | `int` | `float` | `(int) $a->diffInSeconds($b)` |
| `diffInWeeks()` | `int` | `float` | `(int) $a->diffInWeeks($b)` |
| `diffInMonths()` | `int` | `float` | `(int) $a->diffInMonths($b)` |
| `diffInYears()` | `int` | `float` | `(int) $a->diffInYears($b)` |
| `createFromTimestamp()` | timezone local | UTC | Passer le timezone |
| `add*()` / `sub*()` | accepte string | int/float only | Caster en int |
| `setTestNow()` | référence | copie | Re-appeler setTestNow |
| `new Carbon('bad')` | fallback | exception | Valider l'input |

## CarbonInterval et CarbonPeriod

### `CarbonInterval` : propriétés `total*` retournent des float

**Avant** : `CarbonInterval::hours(2)->totalMinutes` retournait `120` (int).

**Après** : retourne `120.0` (float), avec plus de précision pour les intervalles non-ronds.

**Piège** : comparaisons strictes `=== 120` échoueront.

### `CarbonPeriod` : itération modifiée

Le comportement d'itération a changé pour les périodes avec des bornes inclusives/exclusives. Vérifier les boucles `foreach` sur des `CarbonPeriod`.

### Compatibilité des macros

Les macros Carbon 2 (`Carbon::macro()`) ne sont pas toutes compatibles avec Carbon 3. Si le projet utilise des macros custom, les re-tester.
