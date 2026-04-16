# Filament v3 → v4 → v5

Les versions de Filament sont liées aux versions de Laravel et Livewire. Filament v4 accompagne Laravel 12, Filament v5 accompagne Laravel 13 / Livewire 4.

## Filament v3 → v4 (Laravel 12)

### Changements critiques (comportement silencieux)

#### Grid/Section/Fieldset : 1 colonne par défaut

**Avant** : les formulaires Grid, Section, Fieldset occupaient toute la largeur (full-width).

**Après** : 1 colonne par défaut.

**Piège** : **tous les layouts multi-colonnes des formulaires admin sont cassés** sans aucune erreur. Les champs s'empilent verticalement au lieu de s'afficher côte à côte.

**Fix** : ajouter `->columns(2)` (ou le nombre voulu) explicitement sur chaque Grid/Section/Fieldset.

#### Table filters deferred par défaut

**Avant** : les filtres de table s'appliquaient immédiatement au changement.

**Après** : un bouton "Apply" est requis (deferred).

**Piège** : l'UX change — les utilisateurs doivent cliquer "Apply" pour voir les résultats filtrés.

**Fix** : ajouter `->deferFilters(false)` sur la table pour restaurer l'ancien comportement.

#### `unique()` validation ignore le record courant

**Avant** : la règle `unique()` dans les formulaires ne savait pas ignorer le record en cours d'édition par défaut.

**Après** : ignore automatiquement le record courant.

**Piège** : si du code ajoutait manuellement `->ignorable($this->record)`, il est maintenant redondant (pas cassant, mais à nettoyer).

#### File visibility : `private` par défaut sur S3

**Avant** : `public` par défaut pour les fichiers uploadés.

**Après** : `private` par défaut sur les disques non-local (S3, etc.).

**Piège** : les fichiers uploadés via Filament sur S3 ne seront plus accessibles publiquement.

**Fix** : ajouter `->visibility('public')` sur les champs FileUpload.

### Breaking changes explicites

- **Nouvelle architecture Schema** — Forms et Infolists ont leurs propres classes Schema. La structure des resources change.
- **Nouvelle structure de répertoires** — les resources et clusters sont réorganisés.
- **Suppression de `doctrine/dbal`** — comme Laravel 11+.

## Filament v4 → v5 (Laravel 13 / Livewire 4)

### Changement principal

Le bump v4 → v5 existe **principalement pour le support de Livewire 4**. Il n'y a pas de breaking changes majeurs sur les forms/tables/actions.

### Prérequis critique : Tailwind CSS v4

**Filament v5 requiert Tailwind v4.** C'est le plus gros obstacle de la migration. Voir `tailwind-migration.md` pour les détails.

### Compatibilité plugins

Les plugins Filament tiers peuvent ne pas être compatibles immédiatement avec v5. Vérifier chaque plugin avant de migrer.

### Compatibilité Filament ↔ Laravel

| Filament | Laravel | Livewire | Tailwind |
|---|---|---|---|
| v3 | 10-11 | 3 | v3 |
| v4 | 12 | 3 | v3 |
| v5 | 13 | 4 | v4 |

## Checklist de migration

### v3 → v4
- [ ] Ajouter `->columns()` explicite sur tous les Grid/Section/Fieldset
- [ ] Tester les filtres de table (comportement deferred)
- [ ] Vérifier la visibility des fichiers uploadés sur S3
- [ ] Nettoyer les `->ignorable()` redondants sur les règles unique
- [ ] Mettre à jour la structure des resources si nécessaire

### v4 → v5
- [ ] Migrer Tailwind v3 → v4 (voir `tailwind-migration.md`)
- [ ] Vérifier la compatibilité de tous les plugins Filament tiers
- [ ] Tester l'ensemble de l'admin après la migration Livewire 4
