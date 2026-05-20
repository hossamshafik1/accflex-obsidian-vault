---
alwaysApply: true
---
# Database Migration & Schema Rules

This document defines the rules and conventions for managing database schema and data migrations using **MS SQL Projects** and **4tecture.CustomSSDTMigrationScripts**.  
All contributors **must** follow these rules to ensure consistency, safety, and reliability of our database deployments.

---

## General Principles

- All schema changes are managed via **SQL Database Project** (`.sqlproj`) and published as a `.dacpac`.
- **Data migrations** (pre/post deployment) must be handled through **custom scripts** placed in the correct `Pre`/`Post` folders.
- **Idempotency** is mandatory: all scripts must be re-runnable without side effects.
- Never modify the main `Pre` or `Post` script files directly — always create **new numbered scripts**.

---

## Schema Change Rules

1. **New columns in existing tables** must always be added **at the end** of the table definition.
2. Use `decimal(19,4)` for money values — never use `float` or `double`.
3. Always name constraints explicitly:
   - `DF_TableName_ColumnName` → Default constraint  
   - `CK_TableName_ColumnName` → Check constraint  
   - `UQ_TableName_ColumnName` → Unique constraint  
   - `PK_TableName` → Primary key constraint  
   - `FK_FromTable_ToTable[_Col]` → Foreign key constraint
4. For **new tables**, do **not** add default values for non-nullable columns unless required by business rules.
5. New columns must be added **nullable first**, then backfilled in `Post` scripts, and finally altered to `NOT NULL` (if required).
6. All new tables must include a `RowVer` (`rowversion`) column for concurrency if they will be updated frequently.

---

## Naming Conventions

- **Tables, Functions, Views, Stored Procedures:**
  - `Saf_*` → Cash Control  
  - `Cnt_*` → Construction  
  - `Acc_*` → General Ledger  
  - `Lgx_*` → Logistics  
  - `ACFX_*` → Misc / AnalyticsReader
- **Indexes:**  
  - `IX_Table_Col1[_Col2]`  
  - `UIX_Table_Col1[_Col2]` for unique non-PK indexes
- **Sequences:**  
  - `SEQ_<Entity>`

---

## Data Types

- Dates: use `datetime2(3)` instead of `datetime` or `smalldatetime`.
- Strings: use `nvarchar(length)` with a **defined length**; use `nvarchar(max)` only if truly needed.
- Booleans: use `bit NOT NULL` with default value.
- GUIDs: use `uniqueidentifier` only when global uniqueness is required, default with `newsequentialid()`.

---

## Pre/Post Deployment Scripts

- Always include **guards** before running:

```sql
  SET XACT_ABORT ON;
  SET NOCOUNT ON;
```

- Scripts must be idempotent:

```sql
IF NOT EXISTS (SELECT 1 FROM sys.columns WHERE name = 'NewCol' AND object_id = OBJECT_ID('Acc_Table'))
    ALTER TABLE Acc_Table ADD NewCol nvarchar(50) NULL;
```

- Number scripts in sequence (001_AddColumn_X.sql, 002_Backfill_X.sql) to ensure deterministic order.

- Reference data generation:

```sql
SET NOCOUNT ON

SET IDENTITY_INSERT [TABLE] ON

MERGE INTO [TABLE] AS [Target]
USING (VALUES
  (1,N'Value 1')
 ,(2,N'Value 2')
 ,(3,N'Value 3')
) AS [Source] ([ID],[Name])
ON ([Target].[ID] = [Source].[ID])
WHEN MATCHED AND (
	NULLIF([Source].[Name], [Target].[Name]) IS NOT NULL OR NULLIF([Target].[Name], [Source].[Name]) IS NOT NULL) THEN
 UPDATE SET
  [Target].[Name] = [Source].[Name]
WHEN NOT MATCHED BY TARGET THEN
 INSERT([ID],[Name])
 VALUES([Source].[ID],[Source].[Name])
WHEN NOT MATCHED BY SOURCE THEN 
 DELETE;

DECLARE @mergeError int
 , @mergeCount int
SELECT @mergeError = @@ERROR, @mergeCount = @@ROWCOUNT
IF @mergeError != 0
 BEGIN
 PRINT 'ERROR OCCURRED IN MERGE FOR [TABLE]. Rows affected: ' + CAST(@mergeCount AS VARCHAR(100)); -- SQL should always return zero rows affected
 END
ELSE
 BEGIN
 PRINT '[TABLE] rows affected by MERGE: ' + CAST(@mergeCount AS VARCHAR(100));
 END

```

## Data Migration Rules

- Backfill in batches to avoid locking large tables:

```sql
DECLARE @BatchSize int = 10000;
WHILE 1=1
BEGIN
  ;WITH c AS (
    SELECT TOP (@BatchSize) pk
    FROM Acc_Table WITH (READPAST)
    WHERE NewCol IS NULL
    ORDER BY pk
  )
  UPDATE t SET NewCol = 'Default'
  FROM Acc_Table t
  JOIN c ON t.pk = c.pk;

  IF @@ROWCOUNT = 0 BREAK;
END
```

-When adding NOT NULL columns:

1. Add as NULL.

2. Backfill data.

3. Alter to NOT NULL with default constraint.

## Performance

- Only add indexes when justified by query needs.

- Keep INCLUDE lists minimal in covering indexes.

- Partition large tables (>50M rows) upfront, with documented boundary strategy.
