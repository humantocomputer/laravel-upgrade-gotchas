# Tailwind CSS v3 → v4

Tailwind v4 est une réécriture complète (moteur Oxide en Rust). La configuration et plusieurs classes utilitaires changent.

**Node.js 20 minimum requis. Safari 16.4+, Chrome 111+, Firefox 128+.**

## Changements critiques (comportement silencieux)

### Configuration : `tailwind.config.js` → `@theme` en CSS

**Avant** : configuration dans `tailwind.config.js` (JavaScript).

**Après** : configuration dans le CSS via la directive `@theme`.

**Piège** : l'ancien fichier de config est ignoré par défaut en v4. Toutes les couleurs custom, fonts, spacings définis dans `tailwind.config.js` disparaissent.

**Fix** : migrer la config vers `@theme` dans le CSS, ou utiliser le mode de compatibilité avec `@config "../../tailwind.config.js"`.

### Directives d'import

**Avant** :
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

**Après** :
```css
@import "tailwindcss";
```

### Classes d'opacité séparées supprimées (CRITIQUE)

**Avant** : `bg-black bg-opacity-25`, `text-white text-opacity-50`, `border-gray-200 border-opacity-75`.

**Après** : syntaxe slash uniquement : `bg-black/25`, `text-white/50`, `border-gray-200/75`.

**Piège** : les classes `bg-opacity-*`, `text-opacity-*`, `border-opacity-*`, `ring-opacity-*`, `placeholder-opacity-*`, `divide-opacity-*` sont **toutes supprimées**. **Le codemod automatique NE migre PAS ces classes.**

**Fix** : rechercher et remplacer manuellement. Regex utile :
```
(bg|text|border|ring|placeholder|divide)-opacity-(\d+)
```
→ ajouter `/<valeur>` à la classe de couleur correspondante.

### `border` default changé

**Avant** : `border` appliquait `border-color: gray-200`.

**Après** : `border` applique `border-color: currentColor`.

**Piège** : tous les éléments avec juste `border` auront une bordure de la couleur du texte au lieu de gris clair.

**Fix** : ajouter `border-gray-200` explicitement là où le gris par défaut était attendu.

### Dark mode : default changé

**Avant** : dark mode activé par la classe CSS (`dark:` fonctionne avec `.dark` sur un parent).

**Après** : dark mode activé par `@media (prefers-color-scheme: dark)` par défaut.

**Piège** : les projets qui utilisent un toggle dark mode via classe JS ne fonctionnent plus — le dark mode suit la préférence système.

**Fix** : configurer explicitement dans le CSS :
```css
@variant dark (&:where(.dark, .dark *));
```
Ou dans le mode compat : `darkMode: 'selector'` dans `tailwind.config.js`.

## Renommages de classes utilitaires

| Ancien (v3) | Nouveau (v4) |
|---|---|
| `bg-gradient-to-r` | `bg-linear-to-r` |
| `bg-gradient-to-l` | `bg-linear-to-l` |
| `bg-gradient-to-t` | `bg-linear-to-t` |
| `bg-gradient-to-b` | `bg-linear-to-b` |
| `bg-gradient-to-tr` | `bg-linear-to-tr` |
| `bg-gradient-to-tl` | `bg-linear-to-tl` |
| `bg-gradient-to-br` | `bg-linear-to-br` |
| `bg-gradient-to-bl` | `bg-linear-to-bl` |
| `flex-shrink-0` | `shrink-0` |
| `flex-shrink` | `shrink` |
| `flex-grow` | `grow` |
| `flex-grow-0` | `grow-0` |
| `overflow-ellipsis` | `text-ellipsis` |
| `decoration-slice` | `box-decoration-slice` |
| `decoration-clone` | `box-decoration-clone` |

Note : les anciens noms fonctionnent encore comme alias dans certains cas, mais il vaut mieux migrer.

## Migration automatique

Tailwind fournit un codemod :
```bash
npx @tailwindcss/upgrade
```

**Limitations du codemod** :
- Ne migre PAS les classes d'opacité séparées (`bg-opacity-*` etc.)
- Ne migre PAS les configs JavaScript complexes avec des plugins custom
- Ne détecte pas les classes dynamiques construites en PHP/JS (`"bg-{$color}-500"`)

## Checklist de migration

- [ ] Mettre à jour Node.js vers v20+
- [ ] Exécuter `npx @tailwindcss/upgrade`
- [ ] Migrer `tailwind.config.js` vers `@theme` ou utiliser `@config` en mode compat
- [ ] Remplacer `@tailwind base/components/utilities` par `@import "tailwindcss"`
- [ ] Rechercher et remplacer toutes les classes `*-opacity-*` par la syntaxe slash
- [ ] Ajouter `border-gray-200` là où `border` seul était utilisé avec l'ancien default
- [ ] Configurer le dark mode si un toggle JS est utilisé
- [ ] Remplacer les classes renommées (gradients, flex-shrink/grow)
- [ ] Vérifier les classes dynamiques construites en PHP/JS
- [ ] Tester visuellement l'ensemble du site
