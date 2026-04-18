# Marketplace de mes Skills Claude

Marketplace personnel de skills pour Claude Code.

## Ajouter le marketplace

Note : Claude code n'a pas accès directement au GitLab, il faut d'abord cloner le Repository.   

Ajouter le marketplace :  
```shell
/plugin marketplace add AnthonyRyck/skills
```

## Installer les skills

Une fois le marketplace installé, il ne reste plus qu'à installer le plugin.  

```shell
/plugin install dev-skills@anthonyryck
```

Ou directement dans le menu de `/plugin`  

## Mettre à jour le marketplace

```shell
/plugin marketplace update anthonyryck
```

---

## Catalogue des skills

### `blazor-component`

Conventions pour la création et l'édition de composants Blazor (`.razor`).

- Distinction **Page** (`@page`) vs **Composant** (sans `@page`)
- Structure en 3 fichiers : `.razor` (markup) / `.razor.cs` (code-behind) / `.razor.css` (styles scopés)
- Règle d'autonomie : pas de `[Inject]` dans un composant, données via `[Parameter]` uniquement
- Communication montante via `EventCallback<T>`
- Indentation des attributs : un attribut par ligne, alignés verticalement
- Nommage : suffixe `Component` ou `Page` obligatoire

### `csharp-standards`

Standards de code C# appliqués systématiquement à tout fichier `.cs`.

- Classes et records **`sealed` par défaut**
- Primary Constructors (C# 12) **interdits** — constructeur classique avec `private readonly`
- Comparaisons : `is` / `is not` en C# standard, `==` / `!=` dans LINQ/EF Core
- Injection de dépendances toujours via constructeur
- Documentation XML `/// <summary>` obligatoire sur toutes les méthodes et propriétés publiques
- `async`/`await` systématique pour les IO, jamais `.Result` ou `.Wait()`
- Conventions de nommage : suffixes `Db`, `Dto`, `Command`, `Query`, `Service`, `Repository`
- Commands : forme `VerbeMétier + Objet + Command` (pas de verbes vagues `Save`, `Update`, `Process`)
- Lisibilité : `&&`/`||` en début de ligne, LINQ sur plusieurs lignes, pas de traitement dans les paramètres

### `tsql`

Standards d'écriture T-SQL pour SQL Server, appliqués à tout fichier `.sql` ou bloc SQL inline.

- Mots-clés SQL en **MAJUSCULES** sans exception
- Crochets `[ ]` uniquement sur les mots réservés (`Description`, `User`, `Role`...), jamais sur les identifiants normaux
- Préfixe de schéma **obligatoire** (`dbo.ActionFormation`, pas `ActionFormation`)
- Alias de tables courts et significatifs (pas `t1`, `a`, `b`)
- `AND` / `OR` en début de ligne pour les conditions multi-lignes
- Nommage explicite des contraintes : `PK_`, `FK_`, `UK_`, `CK_`, `IX_`
- Pas de `DEFAULT` sur les colonnes — les valeurs par défaut sont la responsabilité du code C#

### `dbup-tsql-scripts`

Standards DbUp pour les scripts de migration T-SQL.

- Scripts de migration numérotés avec préfixes (migrations, seeds, etc.)
- Conventions de nommage et structure des migrations
- Gestion des rollbacks et migrations réversibles

---

## Ajouter un nouveau skill

1. Crée un dossier dans `skills/` :
   ```
   skills/<nom-du-skill>/
   └── SKILL.md
   ```

2. Ajoute le skill dans `.claude-plugin/marketplace.json` :
   ```json
   {
     "skills": [
       "./skills/blazor-component",
       "./skills/csharp-standards",
       "./skills/tsql",
       "./skills/dbup-tsql-scripts",
       "./skills/<nom-du-skill>"
     ]
   }
   ```

3. Documente le skill dans ce README.

4. Commit et push.

### Format du `SKILL.md`

```markdown
---
name: nom-du-skill
description: Ce que fait le skill (affiché dans /help, et en anglais)   
---

# Nom du Skill

Instructions détaillées pour Claude...

## Exemples

## Guidelines
```
