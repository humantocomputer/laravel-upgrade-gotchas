# Livewire 3 → 4 : Changements

Livewire 4 accompagne Laravel 13. Les changements touchent principalement le comportement de `wire:model`, la fermeture des tags, et les transitions.

## Changements critiques (comportement silencieux)

### `wire:model.blur` contrôle le sync client

**Avant** : `.blur` et `.change` ne contrôlaient que le moment de l'envoi réseau. La valeur était immédiatement synchronisée côté client (dans Alpine/le DOM).

**Après** : `.blur` et `.change` contrôlent aussi la synchronisation côté client. La valeur dans le DOM ne change qu'au blur/change.

**Piège** : les formulaires qui utilisaient `wire:model.blur` avec de la logique Alpine qui lisait la valeur entre les frappes ne fonctionnent plus — la valeur Alpine reste à l'ancienne valeur jusqu'au blur.

**Fix** : utiliser `wire:model.live.blur` pour restaurer l'ancien comportement (sync client immédiat, envoi réseau au blur).

### Tags composants doivent être fermés

**Avant** : `<livewire:counter>` sans fermeture fonctionnait et rendait le composant.

**Après** : le tag doit être fermé : `<livewire:counter />` (auto-fermé) ou `<livewire:counter></livewire:counter>`.

**Piège** : sans la fermeture, tout le HTML qui suit le tag sera interprété comme un slot du composant. Le rendu sera cassé mais sans erreur PHP.

**Fix** : rechercher tous les `<livewire:` sans `/>` et les fermer. Regex utile :
```
<livewire:[a-z-]+(?:\s[^>]*)?>(?!\s*<\/livewire:)
```

### URL Livewire hashée

**Avant** : les requêtes Livewire allaient vers `/livewire/update`.

**Après** : `/livewire-{hash}/update` (hash basé sur la config de l'app).

**Piège** : les règles de firewall, configs CDN, headers CSP, et reverse proxies qui whitelistaient `/livewire/*` devront être mis à jour.

**Fix** : utiliser un pattern plus large comme `/livewire*/update` ou configurer le préfixe explicitement.

### `wire:transition` réécrit

**Avant** : basé sur Alpine `x-transition` avec modifiers (`.opacity`, `.scale`, `.duration.500ms`).

**Après** : basé sur la View Transitions API native du navigateur. Tous les anciens modifiers sont supprimés.

**Piège** : les transitions custom avec `.opacity.scale.duration.500ms` ne fonctionnent plus du tout — elles sont ignorées silencieusement.

**Fix** : utiliser les CSS View Transitions ou `x-transition` directement sur l'élément Alpine.

### Morph markers changés

**Avant** : Livewire utilisait des commentaires HTML comme markers pour le DOM morphing.

**Après** : le format des markers a changé.

**Piège** : du JavaScript custom qui parsait les commentaires HTML de Livewire pour détecter des composants ne fonctionnera plus.

## Breaking changes explicites

### `$wire` dans Alpine

Les accès à `$wire` dans les expressions Alpine sont plus stricts. `$wire.entangle()` doit utiliser la syntaxe correcte du modèle.

### Lazy loading par défaut

Certains composants lourds peuvent maintenant être lazy-loaded par défaut. Si un composant apparaît avec un délai inattendu, vérifier si `lazy` est activé.

### `wire:navigate` et scripts JS

Le SPA mode via `wire:navigate` a été amélioré. **Les `<script>` tags dans les vues Blade ne sont plus ré-exécutés** lors de la navigation SPA. Les scripts inline qui initialisent des librairies tierces (charts, maps, etc.) ne fonctionneront plus après la première navigation.

**Fix** : utiliser `@script` / `@endscript` de Livewire, ou écouter l'event `livewire:navigated` pour ré-initialiser.

### `wire:confirm` modifié

Le comportement du dialog de confirmation a changé. Tester les composants qui utilisent `wire:confirm`.

### `Livewire::test()` assertions renommées

Certaines assertions de test ont été renommées ou supprimées. Vérifier la compatibilité de la suite de tests.

## Checklist de migration Livewire 3 → 4

- [ ] Fermer tous les tags `<livewire:...>` avec `/>` 
- [ ] Remplacer `wire:model.blur` par `wire:model.live.blur` si la valeur Alpine est lue entre les frappes
- [ ] Mettre à jour les règles firewall/CDN pour le nouveau path `/livewire-{hash}/update`
- [ ] Remplacer les `wire:transition` avec modifiers par des View Transitions CSS
- [ ] Tester les formulaires avec `wire:model.blur` et `wire:model.change`
