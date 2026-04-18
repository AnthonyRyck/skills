---
name: dbup-tsql-scripts
description: Use when writing a DbUp migration script — modifying schema (tables, columns, indexes, constraints) or stored procedures. Utility stored procedures exist for idempotent conditional checks.
---

# DbUp T-SQL Migration Scripts

## Vue d'ensemble

DbUp exécute des scripts SQL ordonnés **une seule fois** dans l'ordre alphabétique. Des procédures utilitaires `sp_Util_*` sont disponibles dans la base de données — **toujours les utiliser** avant un `ALTER`, `CREATE` ou `DROP` conditionnel pour garantir l'idempotence.

---

## Procédures utilitaires disponibles

| Procédure | Paramètres | Vérifie |
|-----------|------------|---------|
| `dbo.sp_Util_ColumnExists` | `@TableName`, `@ColumnName`, `@Exists OUTPUT` | Existence d'une colonne |
| `dbo.sp_Util_TableExists` | `@TableName`, `@Exists OUTPUT` | Existence d'une table |
| `dbo.sp_Util_IndexExists` | `@TableName`, `@IndexName`, `@Exists OUTPUT` | Existence d'un index |
| `dbo.sp_Util_ConstraintExists` | `@ConstraintName`, `@Exists OUTPUT` | Existence d'une contrainte (DEFAULT, FK, CHECK, PK, UQ) |
| `dbo.sp_Util_StoredProcedureExists` | `@ProcedureName`, `@Exists OUTPUT` | Existence d'une procédure stockée |

> Ces procédures sont créées par `Scripts/202511/01_ED-2680_procedure-stocke_PourScripts.sql`.

---

## Patterns d'utilisation

### Ajouter une colonne (si absente)

```sql
DECLARE @Exists BIT;
EXEC dbo.sp_Util_ColumnExists
    @TableName  = 'dbo.ActionFormation',
    @ColumnName = 'EstPublie',
    @Exists     = @Exists OUTPUT;

IF @Exists = 0
BEGIN
    ALTER TABLE dbo.ActionFormation ADD EstPublie BIT NOT NULL;
END
GO
```

### Créer une table (si absente)

```sql
DECLARE @Exists BIT;
EXEC dbo.sp_Util_TableExists
    @TableName = 'dbo.NouvelleTable',
    @Exists    = @Exists OUTPUT;

IF @Exists = 0
BEGIN
    CREATE TABLE dbo.NouvelleTable
    (
        Id   INT          NOT NULL,
        Code NVARCHAR(50) NOT NULL,
        CONSTRAINT PK_NouvelleTable PRIMARY KEY (Id)
    );
END
GO
```

### Créer un index (si absent)

```sql
DECLARE @Exists BIT;
EXEC dbo.sp_Util_IndexExists
    @TableName = 'dbo.ActionFormation',
    @IndexName = 'IX_ActionFormation_EtablissementId',
    @Exists    = @Exists OUTPUT;

IF @Exists = 0
BEGIN
    CREATE INDEX IX_ActionFormation_EtablissementId
        ON dbo.ActionFormation (EtablissementId);
END
GO
```

### Ajouter une contrainte (si absente)

```sql
DECLARE @Exists BIT;
EXEC dbo.sp_Util_ConstraintExists
    @ConstraintName = 'FK_ActionFormation_Etablissement',
    @Exists         = @Exists OUTPUT;

IF @Exists = 0
BEGIN
    ALTER TABLE dbo.ActionFormation
        ADD CONSTRAINT FK_ActionFormation_Etablissement
        FOREIGN KEY (EtablissementId)
        REFERENCES dbo.Etablissement (Id);
END
GO
```

### Supprimer/recréer une procédure stockée

```sql
DECLARE @Exists BIT;
EXEC dbo.sp_Util_StoredProcedureExists
    @ProcedureName = 'dbo.sp_MaProcedure',
    @Exists        = @Exists OUTPUT;

IF @Exists = 1
BEGIN
    DROP PROCEDURE dbo.sp_MaProcedure;
END
GO

CREATE PROCEDURE dbo.sp_MaProcedure
    -- ...
AS
BEGIN
    SET NOCOUNT ON;
    -- ...
END
GO
```

---

## Convention de nommage des scripts DbUp

```
Scripts/
  YYYYMM/
    NN_TICKET_description-courte.sql
```

| Élément | Règle |
|---------|-------|
| `YYYYMM` | Année + mois de création |
| `NN` | Numéro d'ordre sur 2 chiffres (`01`, `02`, ...) |
| `TICKET` | Référence JIRA (ex: `ED-2680`) |
| `description-courte` | Kebab-case, bref |

**Exemple :** `Scripts/202511/01_ED-2680_ajout-colonne-statut.sql`

---

## Règles fondamentales

| Règle | Détail |
|-------|--------|
| Toujours idempotent | Utiliser `sp_Util_*` avant tout `ALTER`/`CREATE`/`DROP` conditionnel |
| `GO` entre les batches | Obligatoire après `DROP`/`CREATE PROCEDURE` et entre blocs DDL |
| Préfixe `dbo.` | Toujours qualifier les objets avec leur schéma |
| Un seul objectif par script | Chaque script = une modification cohérente |
| Standards T-SQL | Mots-clés MAJUSCULES, alias courts, pas de contrainte DEFAULT (voir `tsql` skill) |

---

## Erreurs courantes

| Erreur | Correction |
|--------|------------|
| `ALTER TABLE … ADD colonne` sans vérification | Appeler `sp_Util_ColumnExists` avant |
| `CREATE TABLE` sans vérification | Appeler `sp_Util_TableExists` avant |
| `CREATE INDEX` sans vérification | Appeler `sp_Util_IndexExists` avant |
| `ADD CONSTRAINT` sans vérification | Appeler `sp_Util_ConstraintExists` avant |
| `GO` absent après `DROP`/`CREATE PROCEDURE` | Obligatoire — les procédures sont des batches DDL séparés |
| Script non idempotent | Un script rejoué ne doit jamais échouer |
