---
name: tsql
description: T-SQL (SQL Server) writing standards and conventions. Use this skill whenever writing, reviewing, or editing any SQL file or T-SQL script — including CREATE TABLE, ALTER TABLE, stored procedures, views, functions, migration scripts, and ad-hoc queries. Trigger on any .sql file, any inline SQL block, or whenever the user asks to write or fix SQL code.
---

# Standards d'écriture T-SQL (SQL Server)

Appliquer ces règles systématiquement lors de l'écriture de tout code T-SQL.

---

## Mots-clés — MAJUSCULES

Tous les mots réservés SQL en **MAJUSCULES**, sans exception.

```sql
-- ✅
SELECT af.Id, af.Nom
FROM dbo.ActionFormation af
WHERE af.EstActif = 1

-- ❌
select af.Id, af.Nom
from dbo.ActionFormation af
where af.EstActif = 1
```

---

## Crochets — mots réservés uniquement

Utiliser `[ ]` **uniquement** quand l'identifiant est un mot réservé SQL Server. Les identifiants normaux n'ont pas de crochets.

```sql
-- ✅
af.[Description],   -- mot réservé
u.[User],           -- mot réservé
af.Nom,             -- identifiant normal, pas de crochets

-- ❌
[af].[Nom],         -- crochets sur un identifiant normal
[dbo].[ActionFormation]
```

Mots réservés courants nécessitant des crochets : `Description`, `User`, `Role`, `Name`, `Date`, `Key`, `Value`, `Status`, `Level`, `Order`, `Table`, `File`.

---

## Préfixe de schéma — obligatoire

Toute référence à une table doit inclure le préfixe de schéma.

```sql
-- ✅
FROM dbo.ActionFormation af
JOIN dbo.Etablissement etab ON etab.Id = af.EtablissementId

-- ❌
FROM ActionFormation af
JOIN Etablissement etab ON etab.Id = af.EtablissementId
```

---

## Alias de tables — courts et significatifs

Les alias doivent être courts mais lisibles. Jamais `t1`, `t2`, `a`, `b` ou une lettre seule sans sens.

```sql
-- ✅
FROM dbo.ActionFormation af
JOIN dbo.Etablissement etab ON etab.Id = af.EtablissementId
JOIN dbo.Marque mq ON mq.Id = etab.MarqueId

-- ❌
FROM dbo.ActionFormation a
JOIN dbo.Etablissement t1 ON t1.Id = a.EtablissementId
```

---

## Conditions multi-lignes — AND / OR en début de ligne

Quand une clause `WHERE`, `ON` ou `HAVING` s'étend sur plusieurs lignes, placer `AND` / `OR` en **début** de la ligne de continuation, aligné avec la première condition.

```sql
-- ✅
WHERE af.EstActif = 1
AND af.DateCreation >= '2024-01-01'
AND (af.TypeId = 1
     OR af.TypeId = 2)

-- ❌
WHERE af.EstActif = 1 AND
      af.DateCreation >= '2024-01-01' AND
      af.TypeId = 1
```

---

## Nommage des contraintes — toujours explicite

Ne jamais laisser une contrainte sans nom. Utiliser le préfixe approprié :

| Préfixe | Type |
|---------|------|
| `PK_` | Clé primaire |
| `FK_` | Clé étrangère |
| `UK_` | Clé unique / index unique |
| `CK_` | Contrainte de vérification |
| `IX_` | Index non unique |

```sql
-- ✅
CONSTRAINT PK_ActionFormation PRIMARY KEY (Id),
CONSTRAINT FK_ActionFormation_Etablissement FOREIGN KEY (EtablissementId)
    REFERENCES dbo.Etablissement (Id),
CONSTRAINT UK_ActionFormation_Code UNIQUE (Code),
CONSTRAINT CK_ActionFormation_NiveauValide CHECK (Niveau IN (1, 2, 3))

-- ❌
PRIMARY KEY (Id),
FOREIGN KEY (EtablissementId) REFERENCES dbo.Etablissement (Id)
```

---

## Pas de contrainte DEFAULT

Ne **pas** définir de valeurs `DEFAULT` sur les colonnes. Les valeurs par défaut sont la responsabilité du code applicatif (C#), pas du schéma de base de données.

```sql
-- ✅
EstActif BIT NOT NULL,
DateCreation DATETIME2 NOT NULL

-- ❌
EstActif BIT NOT NULL DEFAULT 1,
DateCreation DATETIME2 NOT NULL DEFAULT GETUTCDATE()
```
