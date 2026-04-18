---
name: csharp-standards
description: C# coding standards, naming conventions, and readability rules. Use this skill whenever writing, reviewing, or editing C# code — including classes, records, services, repositories, endpoints, DTOs, commands, interfaces, or any .cs file. Trigger when the user asks to implement a feature, fix a bug, create a new type, or review C# code.
---

# Standards C# — Conventions de code

Appliquer ces règles systématiquement lors de l'écriture ou de la relecture de tout code C#.

---

## Classes et Records

- Toutes les classes et records sont **`sealed` par défaut**
- Retirer `sealed` uniquement si l'héritage est explicitement nécessaire
- `record` pour les modèles immuables (DTOs, Commands, Messages)
- `class` pour les services et repositories

```csharp
// ✅
public sealed record CreateActionFormationCommand(string Nom, int EtablissementId);
public sealed class ActionFormationService : IActionFormationService { }

// ❌
public record CreateActionFormationCommand(string Nom, int EtablissementId);
public class ActionFormationService : IActionFormationService { }
```

---

## Constructeurs — Primary Constructors INTERDITS

Toujours utiliser le constructeur classique avec champs `private readonly`. Les Primary Constructors (C# 12) sont interdits.

```csharp
// ✅
public sealed class ActionFormationService : IActionFormationService
{
    private readonly IActionFormationRepository _repository;

    public ActionFormationService(IActionFormationRepository repository)
    {
        _repository = repository;
    }
}

// ❌ — Primary Constructor interdit
public sealed class ActionFormationService(IActionFormationRepository repository)
    : IActionFormationService
{
}
```

---

## Comparaisons et nullité

- **Code C# standard** : utiliser `is` / `is not`
- **LINQ** (`Where`, `Select`, `FirstOrDefault`, etc.) : utiliser `==` / `!=` — EF Core ne traduit pas `is`/`is not` en SQL

```csharp
// ✅ — code C# standard
if (result is null) { }
if (service is not null) { }

// ✅ — dans LINQ / EF Core
var actives = context.ActionFormations.Where(af => af.DateSuppression == null);

// ❌ — is/is not dans LINQ (non traduit en SQL)
var actives = context.ActionFormations.Where(af => af.DateSuppression is null);
```

---

## Injection de dépendances

- Toujours via constructeur
- Enregistrement dans `{Feature}Configuration.cs`
- `AddScoped<T>()` pour les services avec état de requête
- `AddTransient<T>()` pour les services sans état

```csharp
// ✅
public static IServiceCollection AddActionFormationFeature(this IServiceCollection services)
{
    services.AddScoped<IActionFormationService, ActionFormationService>();
    services.AddScoped<IActionFormationRepository, ActionFormationRepository>();
    return services;
}
```

---

## Documentation XML — obligatoire

`/// <summary>` obligatoire pour toutes les propriétés publiques et **toutes les méthodes** (private, protected, public). Exception : méthodes de test (le nom suffit).

```csharp
// ✅
/// <summary>
/// Crée une nouvelle action de formation à partir de la commande fournie.
/// </summary>
public async Task<ResultOf<ActionFormationDto>> CreateAsync(CreateActionFormationCommand command)
{ }

/// <summary>
/// Identifiant unique de l'action de formation.
/// </summary>
public int Id { get; set; }

// ❌ — documentation manquante
public async Task<ResultOf<ActionFormationDto>> CreateAsync(CreateActionFormationCommand command)
{ }
```

---

## Magic Numbers / Magic Strings

Extraire en constante si :
- la valeur est utilisée plusieurs fois
- elle représente une règle métier
- son sens n'est pas évident

Ne pas extraire si :
- valeur par défaut d'une propriété
- message d'erreur unique
- valeur évidente (0, 1, -1, "")

```csharp
// ✅ — règle métier non évidente
private const int NombreMaxParticipants = 30;
private const string CodePrefixeFormationContinue = "FC";

// ✅ — valeur évidente, pas besoin de constante
return ResultOf<T>.Failure("Action de formation introuvable.");
```

---

## Async / Await

- Toujours `async`/`await` pour les opérations IO
- Suffixe `Async` sur toutes les méthodes asynchrones
- Jamais `.Result` ou `.Wait()`
- Dans une méthode async, toujours la variante async : `ToListAsync()`, `FirstOrDefaultAsync()`, `AnyAsync()`, `CountAsync()`, `SingleOrDefaultAsync()`, `MaxAsync()`, etc.

```csharp
// ✅
public async Task<ResultOf<List<ActionFormationDto>>> GetAllAsync(CancellationToken ct)
{
    var entities = await _context.ActionFormations
        .Where(af => af.DateSuppression == null)
        .ToListAsync(ct);
}

// ❌
public ResultOf<List<ActionFormationDto>> GetAll()
{
    var entities = _context.ActionFormations
        .Where(af => af.DateSuppression == null)
        .ToList(); // pas async
}

// ❌
var result = GetAllAsync().Result; // .Result interdit
```

---

## Collections

- Matérialiser avec `.ToList()` ou `.ToArray()` si la collection est parcourue plusieurs fois
- Garder `IEnumerable` uniquement pour les flux ou la composition LINQ to SQL

```csharp
// ✅ — parcourue plusieurs fois : matérialiser
var items = await query.ToListAsync(ct);
var count = items.Count;
foreach (var item in items) { }

// ✅ — composition LINQ to SQL : garder IEnumerable/IQueryable
IQueryable<ActionFormation> query = _context.ActionFormations
    .Where(af => af.EstActif == true);
```

---

## Conventions de nommage

### Suffixes des types

| Suffixe | Rôle | Couche |
|---------|------|--------|
| `Db` | Données brutes de la DB | Sortie Repository |
| `Dto` | Transfert entre couches/API | Frontière système |
| `Command` | Action modifiant l'état (CQRS) | Application |
| `Query` | Lecture sans effet de bord (CQRS) | Application |
| `Message` | Message RabbitMQ | Infrastructure/Messaging |
| `Service` | Logique métier | Application/Domaine |
| `Repository` | Accès données | Infrastructure |

### Commands — forme : `VerbeMétier + Objet + Command`

Verbe métier du langage ubiquitaire, pas de verbes vagues (`Save`, `Update`, `Process`).

```csharp
// ✅
CreateActionFormationCommand
ChangeEtablissementAdresseCommand
PublierCatalogueCommand
ArchiverMarqueCommand

// ❌
SaveActionFormationCommand
UpdateAdresseCommand
ProcessMarqueCommand
```

### Interfaces

Préfixe `I` systématique, une interface par service.

```csharp
// ✅
public interface IActionFormationService { }
public interface IEtablissementRepository { }
```

---

## Lisibilité du code

### Conditions multi-lignes — `||` / `&&` en début de ligne

```csharp
// ✅
if (action.EstActif
    && action.DateDebut <= DateTime.UtcNow
    && action.DateFin >= DateTime.UtcNow)
{ }

// ❌
if (action.EstActif &&
    action.DateDebut <= DateTime.UtcNow &&
    action.DateFin >= DateTime.UtcNow)
{ }
```

### Chaînes LINQ — une méthode par ligne

```csharp
// ✅
var result = await _context.ActionFormations
    .Where(af => af.EtablissementId == etablissementId)
    .Where(af => af.DateSuppression == null)
    .OrderBy(af => af.Nom)
    .Select(af => af.ToDto())
    .ToListAsync(ct);

// ❌
var result = await _context.ActionFormations.Where(af => af.EtablissementId == etablissementId).Where(af => af.DateSuppression == null).OrderBy(af => af.Nom).ToListAsync(ct);
```

### Pas de traitement dans les paramètres

```csharp
// ✅
var nomNormalise = command.Nom.Trim().ToLowerInvariant();
var existe = await _repository.ExisteAsync(nomNormalise, ct);

// ❌
var existe = await _repository.ExisteAsync(command.Nom.Trim().ToLowerInvariant(), ct);
```

### Initialisations — object initializer sur plusieurs lignes si > 1 propriété

```csharp
// ✅
var action = new ActionFormation
{
    Nom = command.Nom,
    EtablissementId = command.EtablissementId,
    DateCreation = DateTime.UtcNow
};

// ❌
var action = new ActionFormation { Nom = command.Nom, EtablissementId = command.EtablissementId, DateCreation = DateTime.UtcNow };
```

### KISS — boucle simple plutôt que LINQ complexe

```csharp
// ✅ — lisible
var ids = new List<int>();
foreach (var item in source)
{
    if (item.EstValide)
        ids.Add(item.Id);
}

// ❌ — LINQ trop complexe pour rien
var ids = source.Where(x => x.EstValide).Select(x => x.Id).ToList();
// (acceptable si simple, problématique si chaîné avec 5+ transformations)
```

### Noms positifs — éviter les doubles négations

```csharp
// ✅
bool estActif = true;
if (estActif) { }

// ❌
bool estInactif = false;
if (!estInactif) { } // double négation
```
