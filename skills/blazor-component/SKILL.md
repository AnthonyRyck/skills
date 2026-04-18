---
name: blazor-component
description: Use when creating or editing a Blazor component (.razor file). Covers autonomy rules, code-behind structure, scoped styles, attribute indentation, and the page vs component distinction.
---

# Blazor Component

## Vue d'ensemble

Un composant Blazor est un fichier `.razor` **sans** directive `@page`. Il est autonome, ne connaît que ses paramètres, et ne parle jamais directement aux services.

> **Règle clé :** si le fichier contient `@page`, c'est une **Page**, pas un composant — les règles `[Inject]` et accès aux services ne s'appliquent PAS (voir section Page vs Composant ci-dessous).

---

## Structure des fichiers

```
{Feature}Component.razor       ← markup uniquement
{Feature}Component.razor.cs    ← toute la logique (code-behind)
{Feature}Component.razor.css   ← styles scoped au composant
```

- Le `.razor` ne contient que du HTML/markup et les directives Blazor (`@if`, `@foreach`, etc.).
- Le `.razor.cs` contient la classe partielle avec les paramètres, les callbacks et la logique.
- Le `.razor.css` est **obligatoire** pour les styles — jamais de styles inline sauf exception justifiée.

---

## Composants autonomes — règles fondamentales

| Règle | Détail |
|---|---|
| Pas de `[Inject]` | Les composants ne s'injectent aucun service |
| Données via `[Parameter]` | Toutes les données viennent du parent |
| Communication montante via `EventCallback<T>` | Le composant notifie, le parent décide |

```csharp
// ✅ BON — composant autonome
[Parameter]
public Favorite Favorite { get; set; } = default!;

[Parameter]
public EventCallback<Favorite> OnRemove { get; set; }

// ❌ MAUVAIS — accès direct à un service
[Inject]
private IFavoriteService FavoriteService { get; set; }
```

---

## Page vs Composant

```
Fichier .razor
    │
    ├─ contient @page ?
    │       │
    │       ├─ OUI → Page
    │       │          ✅ Peut utiliser [Inject]
    │       │          ✅ Peut accéder aux services
    │       │          Nommage : suffixe "Page"  (FavoritesPage.razor)
    │       │
    │       └─ NON → Composant
    │                  ❌ Pas de [Inject]
    │                  ❌ Pas d'accès aux services
    │                  Nommage : suffixe "Component"  (FavoriteCardComponent.razor)
```

---

## Nommage

```
FavoritesPage.razor          // Page — accès aux services autorisé
FavoriteCardComponent.razor  // Composant — paramètres uniquement

// ❌ MAUVAIS
Favorites.razor              // Ambigu : page ou composant ?
FavoriteCard.razor           // Pas de suffixe
```

---

## Indentation des attributs

Chaque attribut Blazor/HTML doit être **sur sa propre ligne**, aligné verticalement avec le premier attribut.

```razor
// ✅ BON — un attribut par ligne, alignés
<FavoriteCardComponent Favorite="favorite"
                       OnRemove="HandleRemove"
                       IsReadOnly="true" />

// ✅ BON — composant simple sans attribut ou un seul : inline acceptable
<FavoriteCardComponent Favorite="favorite" />

// ❌ MAUVAIS — attributs mélangés sur plusieurs lignes sans alignement
<FavoriteCardComponent Favorite="favorite" OnRemove="HandleRemove"
    IsReadOnly="true" />
```

La règle s'applique aussi aux éléments HTML natifs dès qu'ils ont plusieurs attributs :

```razor
// ✅ BON
<input type="text"
       id="search"
       class="form-control"
       @bind="SearchTerm" />

// ❌ MAUVAIS
<input type="text" id="search"
       class="form-control" @bind="SearchTerm" />
```

---

## Styles

- **Toujours** créer un fichier `.razor.css` pour les styles du composant.
- Les styles sont scopés automatiquement par Blazor — pas de fuite vers d'autres composants.
- Styles inline (`style="..."`) : uniquement si valeur calculée dynamiquement, sinon interdit.

---

## Erreurs courantes

| Erreur | Correction |
|---|---|
| `[Inject]` dans un composant (sans `@page`) | Passer les données via `[Parameter]` |
| Logique métier dans le `.razor` | Déplacer dans le `.razor.cs` |
| Styles dans le `.razor` | Créer un `.razor.css` |
| Attributs sur la même ligne | Un attribut par ligne, alignés |
| Nom sans suffixe | Ajouter `Component` ou `Page` selon la nature |
