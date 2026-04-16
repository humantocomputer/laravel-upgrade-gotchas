# Pest 2 → 3 → 4 / PHPUnit 10 → 11 → 12

Les versions de Pest sont liées aux versions de PHPUnit, qui sont liées aux versions de Laravel. Pest 3 accompagne Laravel 11-12, Pest 4 accompagne Laravel 13.

## Pest 2 → 3 (PHPUnit 10 → 11)

**PHP 8.2 minimum requis.**

### Changements critiques

- **`tap()` supprimé** — remplacé par `defer()`.
- **`toHaveMethod()` / `toHaveMethods()`** — déplacés vers les expectations architecturales (`arch()`). Ne fonctionnent plus comme expectations classiques.
- **Collision** doit être mis à jour vers v8.
- **PHPUnit 10/11 : `withConsecutive()` supprimé** — cette méthode très utilisée pour mocker des appels séquentiels n'existe plus. Utiliser des closures chaînées avec `willReturnCallback()` ou `willReturnOnConsecutiveCalls()`.

### PHPUnit 11 spécifique

- **Annotations dépréciées** — `@test`, `@dataProvider`, `@depends`, `@group` etc. sont dépréciés en faveur des attributs PHP 8 :
  ```php
  // Ancien (déprécié)
  /** @test */
  public function it_works() {}
  
  // Nouveau
  #[Test]
  public function it_works() {}
  ```
  Note : en Pest, cela ne s'applique pas directement (pas d'annotations), mais les plugins PHPUnit sous-jacents peuvent être affectés.

- **Schema `phpunit.xml` changé** — le format XML a évolué. Exécuter `vendor/bin/phpunit --migrate-configuration` pour migrer automatiquement.

## Pest 3 → 4 (PHPUnit 11 → 12)

**PHP 8.3 minimum requis.**

### Changements critiques

- **PHPUnit 12 : annotations supprimées définitivement** — plus de `@test`, `@dataProvider`, `@depends` en docblock. Tout doit utiliser les attributs PHP 8. Les suites de tests qui utilisent encore les annotations en PHPUnit pur casseront.

- **`createStub()` ne peut plus configurer d'expectations** — utiliser `createMock()` à la place.

- **Mock objects pour classes abstraites/traits supprimés** — `getMockForAbstractClass()` et `getMockForTrait()` n'existent plus.

- **Noms de snapshots changés** — `toMatchSnapshot()` génère des noms différents. Il faut régénérer tous les snapshots avec `--update-snapshots`.

### Compatibilité Pest ↔ Laravel

| Pest | PHPUnit | PHP min | Laravel |
|---|---|---|---|
| 2.x | 10.x | 8.1 | 10-11 |
| 3.x | 11.x | 8.2 | 11-12 |
| 4.x | 12.x | 8.3 | 13 |

## Checklist de migration

- [ ] Vérifier la version PHP minimale
- [ ] Mettre à jour `phpunit.xml` avec `--migrate-configuration`
- [ ] Remplacer `tap()` par `defer()` (Pest 3+)
- [ ] Remplacer `withConsecutive()` (PHPUnit 10+)
- [ ] Migrer les annotations vers les attributs PHP 8 (PHPUnit 12)
- [ ] Remplacer `createStub()` par `createMock()` si des expectations sont utilisées (PHPUnit 12)
- [ ] Régénérer les snapshots si `toMatchSnapshot()` est utilisé (Pest 4)
- [ ] Mettre à jour Collision vers v8+ (Pest 3+)
