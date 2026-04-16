# Laravel 11 → 12 : Changements

La v12 introduit Carbon 3, des changements dans le container, la validation, et le routing.

**PHP 8.2 minimum requis.**

## Changements critiques (comportement silencieux)

### Carbon 3 obligatoire

C'est le changement le plus impactant de cette migration. Voir `carbon-3-migration.md` pour le détail complet.

**Résumé des pièges** :
- `diffInDays()` etc. retournent `float` au lieu de `int`
- `createFromTimestamp()` utilise UTC par défaut
- `addMinutes('30')` avec une string ne fonctionne plus

### Container : nullable defaults respectés

**Avant** : `public function __construct(public ?Carbon $date = null)` → le container injectait une instance Carbon.

**Après** : le container respecte le `= null` et n'injecte rien.

**Piège** : tout service avec `?SomeClass $param = null` recevra `null` au lieu d'une instance résolue. Peut provoquer des `Call to a member function on null` inattendus.

**Fix** : soit retirer le `= null` si le paramètre est requis, soit gérer le cas `null` dans le code.

### `HasUuids` → UUIDv7

**Avant** : UUIDv4 (aléatoires).

**Après** : UUIDv7 (ordonnés chronologiquement, commencent par un timestamp).

**Piège** : si du code valide le format UUID strict ou compare des UUIDs, le nouveau format peut casser. Les UUIDv7 sont aussi plus longs dans certaines représentations.

**Fix** : pour garder l'ancien comportement, utiliser `use HasVersion4Uuids as HasUuids;`.

### Précédence des routes unifiée

**Avant** : en mode non-caché, la dernière route avec un nom donné gagnait. En mode caché, la première.

**Après** : la première route enregistrée gagne toujours (caché ou non).

**Piège** : si un package redéfinit une route avec le même nom que l'application, le comportement change entre dev (non-caché) et prod (caché) avant L12, mais est maintenant cohérent. Les routes définies dans `web.php` avant les packages gagneront toujours.

### `mergeIfMissing` avec dot notation

**Avant** : `$request->mergeIfMissing(['user.name' => 'X'])` créait une clé littérale `user.name`.

**Après** : merge dans un tableau imbriqué `['user' => ['name' => 'X']]`.

**Piège** : si du code dépend de la clé littérale avec un point, il ne la trouvera plus.

## Breaking changes explicites

### Validation `image` exclut les SVG

**Avant** : la règle `image` acceptait les SVG.

**Après** : SVG rejeté. Utiliser `image:allow_svg` pour les accepter.

**Piège** : les uploads d'images SVG seront silencieusement rejetés par la validation.

### Concurrency driver Redis par défaut

**Avant** : le driver de concurrence par défaut était `file`.

**Après** : `redis` par défaut.

**Piège** : si Redis n'est pas configuré, les opérations de concurrence échouent.

### Storage disk local

**Avant** : `storage/app` comme racine.

**Après** : `storage/app/private` comme racine.

**Piège** : `Storage::disk('local')->path('file.txt')` pointe vers un chemin différent. Les fichiers existants dans `storage/app/` ne seront plus trouvés sans migration.

**Fix** : définir explicitement `'root' => storage_path('app')` dans `config/filesystems.php` si nécessaire.

### Constructeurs Database bas niveau

`Blueprint`, `Grammar`, et d'autres classes DB requièrent maintenant une instance `Connection` en premier argument. `setConnection()` supprimé. Impacte les packages custom et le code qui instancie `Blueprint` manuellement.

### `Schema::getTables()`, `Schema::getViews()`, `Schema::getTypes()`

Retournent maintenant les résultats de **tous les schémas** par défaut (multi-schema). Impacte les projets PostgreSQL multi-schema.

## Dépréciations

| Élément | Remplacé par | Notes |
|---|---|---|
| `HasUuids` (v4 implicite) | `HasVersion4Uuids` / `HasVersion7Uuids` explicite | UUIDv7 par défaut |
| `$casts` propriété | `casts()` méthode | Toujours fonctionnel |
| Ancien format `Storage::disk('local')` | Nouveau chemin `storage/app/private` | Config explicite recommandée |
