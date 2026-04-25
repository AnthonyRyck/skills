---
name: blazor-vertical-slice
description: >
  Utiliser quand on crée une nouvelle feature Blazor Server, qu'on ajoute une page, qu'on organise du code existant,
  ou qu'on analyse si un slice respecte les conventions. Couvre la structure des dossiers, le pattern ViewService/ViewModel,
  les conventions de nommage CQRS (Command/Query/Result), les repositories, et l'enregistrement DI.
  Déclencher aussi quand quelqu'un demande "comment ajouter une feature X", "comment structurer ce code",
  ou "est-ce que ce code respecte les conventions Vertical Slice".
---

# Blazor Vertical Slice

## Vue d'ensemble

Une **feature** représente un domaine de responsabilité bien délimité. Elle correspond *très souvent* à une page Blazor, mais pas nécessairement : une feature peut être purement technique (migrations de base de données, synchronisation, envoi d'e-mails, etc.) et ne contenir aucune vue.

Ce qui est constant :
- Chaque feature est contenue dans son propre dossier et est **complètement indépendante** des autres features.
- Elle ne connaît qu'elle-même et, éventuellement, le dossier `Shared`.
- Si elle a une vue : **zéro logique dans le composant**. La page affiche, le `ViewService` traite.

---

## Structure d'une feature

Il existe deux variantes selon la nature de la feature.

### Feature avec vue (page Blazor)

```
Features/
  {FeatureName}Feature/
    Configurations/           ← enregistrement DI propre à ce slice
    Components/               ← composants Blazor propres à cette feature (optionnel)
    Models/                   ← tous les modèles du slice (ViewModel, Command, Query, Result, entités)
    Repositories/             ← accès aux données (interface + implémentation)
    Services/                 ← ViewService (interface + implémentation)
  {PageName}Page.razor        ← markup uniquement
  {PageName}Page.razor.cs     ← injection + appels lifecycle, rien d'autre
  {PageName}Page.razor.css    ← styles scoped
```

Le dossier `Components/` est **optionnel** : il n'apparaît que si la page utilise des composants Blazor qui lui sont propres (cards, overlays, formulaires réutilisables dans la feature, etc.). Ces composants suivent les règles du skill `blazor-component` : pas de `[Inject]`, données via `[Parameter]`, communication montante via `EventCallback<T>`.

### Feature sans vue (technique ou métier pur)

Une feature peut n'avoir aucune page Blazor. Elle regroupe alors uniquement la logique, les données et la configuration dont elle est responsable.

```
Features/
  {FeatureName}Feature/
    Configurations/           ← enregistrement DI
    Models/                   ← modèles métier, Command, Query, Result (si besoin)
    Repositories/             ← accès aux données (optionnel)
    Services/                 ← service(s) métier (interface + implémentation)
    Scripts/                  ← ressources statiques liées à la feature (ex: scripts SQL)
```

**Exemple — feature de migrations DbUp :**

```
Features/
  MigrationFeature/
    Configurations/
      MigrationConfiguration.cs
    Scripts/
      001_CreateTableUsers.sql
      002_AddColumnEmail.sql
    Services/
      IMigrationService.cs
      MigrationService.cs
```

Pas de page, pas de ViewModel — juste un service qui exécute les migrations et ses scripts. Le dossier `Scripts/` est un exemple de ressource propre à la feature ; il peut être nommé autrement selon le besoin (`Templates/`, `Seeds/`, etc.).

**Exemple concret — feature de recherche avec composants :**

```
Features/
  RechercheFeature/
    Configurations/
      RechercheConfiguration.cs
    Components/
      MovieCardComponent.razor
      MovieCardComponent.razor.cs
      MovieCardComponent.razor.css
      MovieDetailOverlayComponent.razor
      MovieDetailOverlayComponent.razor.cs
      MovieDetailOverlayComponent.razor.css
    Models/
      SearchViewModel.cs
      GetMoviesQuery.cs
      GetMoviesResult.cs
    Repositories/
      IMediaRepository.cs
      MediaRepository.cs
    Services/
      ISearchViewService.cs
      SearchViewService.cs
  SearchPage.razor
  SearchPage.razor.cs
  SearchPage.razor.css
```

---

## Nommage

| Élément | Convention | Exemple |
|---|---|---|
| Dossier de feature | `{Nom}Feature` | `RechercheFeature` |
| Page | `{Nom}Page` | `SearchPage.razor` |
| ViewModel | `{Nom}ViewModel` | `SearchViewModel` |
| Interface ViewService | `I{Nom}ViewService` | `ISearchViewService` |
| Implémentation ViewService | `{Nom}ViewService` | `SearchViewService` |
| Interface Repository | `I{Entité}Repository` | `IMediaRepository` |
| Implémentation Repository | `{Entité}Repository` | `MediaRepository` |
| Query (lecture) | `{Action}{Entité}Query` | `GetMoviesQuery` |
| Command (écriture) | `{Action}{Entité}Command` | `UpdateUserCommand` |
| Résultat | `{Action}{Entité}Result` | `GetMoviesResult` |
| Configuration DI | `{Nom}Configuration` | `RechercheConfiguration` |

---

## Responsabilités de chaque couche

### Page (`.razor` + `.razor.cs`)

La page **n'effectue aucun traitement**. Elle :
- Injecte le `IViewService`
- Appelle le ViewService dans les handlers du cycle de vie Blazor (`OnInitializedAsync`, etc.)
- Passe les données du ViewModel aux composants enfants

```csharp
// ✅ BON — SearchPage.razor.cs
public partial class SearchPage
{
    [Inject]
    private ISearchViewService SearchViewService { get; set; } = default!;

    protected override async Task OnInitializedAsync()
    {
        await SearchViewService.InitialiserAsync();
    }

    private async Task HandleSearch(string terme)
    {
        await SearchViewService.RechercherAsync(terme);
    }
}
```

```razor
// ✅ BON — SearchPage.razor (markup uniquement)
@page "/search"
@inherits SearchPage

<SearchFormComponent OnSearch="HandleSearch" />
<ResultListComponent Items="SearchViewService.ViewModel.Resultats" />
```

```csharp
// ❌ MAUVAIS — logique métier dans la page
protected override async Task OnInitializedAsync()
{
    var query = new GetMoviesQuery("action");
    var result = await _repository.GetMoviesAsync(query);
    Movies = result.Movies;
}
```

---

### ViewService

Le ViewService est le **chef d'orchestre** du slice. Il :
- Expose un `ViewModel` que la page observe pour l'affichage
- Contient toute la logique métier
- Crée les `Command`/`Query`, les passe aux repositories ou services externes
- **Ne dépend d'aucun projet externe** — pas de MediatR, pas de couplage fort

```csharp
// ✅ BON — SearchViewService.cs
public class SearchViewService : ISearchViewService
{
    private readonly IMediaRepository _mediaRepository;

    public SearchViewModel ViewModel { get; private set; } = new();

    public SearchViewService(IMediaRepository mediaRepository)
    {
        _mediaRepository = mediaRepository;
    }

    public async Task RechercherAsync(string terme)
    {
        var query = new GetMoviesQuery(terme);
        var result = await _mediaRepository.GetMoviesAsync(query);

        ViewModel.Resultats = result.Movies;
        ViewModel.EstCharge = true;
    }
}
```

---

### Models

Tous les modèles du slice — pas de distinctions de sous-dossiers, tout est dans `Models/` :

| Type | Rôle |
|---|---|
| `ViewModel` | Données exposées à la page pour l'affichage |
| `Query` | Paramètres d'une lecture (ne modifie pas l'état) |
| `Command` | Paramètres d'une écriture (modifie l'état) |
| `Result` | Résultat retourné par un repository ou une API |

---

### Repository

Isole l'accès aux données (base de données, API externe, fichier, etc.). Toujours une interface + implémentation.

```csharp
// IMediaRepository.cs
public interface IMediaRepository
{
    Task<GetMoviesResult> GetMoviesAsync(GetMoviesQuery query);
}

// MediaRepository.cs
public class MediaRepository : IMediaRepository
{
    public async Task<GetMoviesResult> GetMoviesAsync(GetMoviesQuery query)
    {
        // accès EF Core, HttpClient, etc.
    }
}
```

---

### Configurations (DI)

Chaque slice enregistre ses propres dépendances dans `Configurations/`. Cela évite de polluer le `Program.cs` global.

```csharp
// RechercheConfiguration.cs
public static class RechercheConfiguration
{
    public static IServiceCollection AddRechercheFeature(this IServiceCollection services)
    {
        services.AddScoped<ISearchViewService, SearchViewService>();
        services.AddScoped<IMediaRepository, MediaRepository>();
        return services;
    }
}
```

Et dans `Program.cs` :
```csharp
builder.Services.AddRechercheFeature();
```

---

### Shared

Le dossier `Shared` (au même niveau que les features) contient les éléments **partagés entre plusieurs features** : modèles communs, repositories ou services transversaux. Il suit la même structure interne (`Models/`, `Repositories/`, `Services/`).

> **Règle :** si un élément n'est utilisé que par une seule feature, il reste dans cette feature. Il ne monte dans `Shared` que quand une seconde feature en a besoin.

---

## Violations fréquentes à signaler

Lors de l'analyse d'un slice existant, vérifier :

| Violation | Symptôme |
|---|---|
| Logique dans la page | `OnInitializedAsync` contient des appels à des repositories ou de la transformation de données |
| ViewService absent | La page injecte directement un repository ou un `HttpClient` |
| ViewModel absent | La page gère ses propres variables d'état avec des champs privés à la place d'un ViewModel |
| Pas d'interface | `ViewService` ou `Repository` n'a pas d'interface `I{Nom}` |
| Composant feature mal placé | Un composant utilisé exclusivement par une feature est dans un dossier `Shared/Components` ou à la racine du projet au lieu d'être dans `{Feature}/Components/` |
| Couplage inter-features | Un service d'une feature est injecté dans une autre feature (hors Shared) |
| DI dans Program.cs | Les enregistrements du slice ne sont pas dans `Configurations/` |
| Command/Query dans Services/ | Ces modèles doivent être dans `Models/`, pas dans `Services/` |

---

## Générer un nouveau slice

Pour créer le squelette complet d'un nouveau slice à partir d'un nom de feature (ex: `GestionUtilisateurs`) :

1. Créer le dossier `GestionUtilisateursFeature/` avec les sous-dossiers `Configurations/`, `Models/`, `Repositories/`, `Services/` (et `Components/` si la page a des composants dédiés)
2. Créer les fichiers de page : `GestionUtilisateursPage.razor`, `.razor.cs`, `.razor.css`
3. Créer `GestionUtilisateursViewModel.cs` dans `Models/`
4. Créer `IGestionUtilisateursViewService.cs` + `GestionUtilisateursViewService.cs` dans `Services/`
5. Créer les interfaces et implémentations de repository dans `Repositories/` selon le besoin
6. Créer `GestionUtilisateursConfiguration.cs` dans `Configurations/`
7. Ajouter l'appel `builder.Services.Add{Nom}Feature()` dans `Program.cs`
