---
description: "SQL table-valued function patterns for Accflex ERP reports. Covers naming by module (Saf_Rpt_, Acc_, Cnt_, Lgx_, PR_), standard parameters, Acc_ConvertIntListToTable helper, inline vs multi-statement TVFs, and ROW_NUMBER() keying. Use when creating or modifying report functions in the SQL Database Project."
globs:
  - "**/dbo/Functions/*Report*.sql"
  - "**/dbo/Functions/*Rpt*.sql"
  - "**/dbo/Functions/Saf_Rpt_*.sql"
  - "**/dbo/Functions/Acc_*Report*.sql"
  - "**/dbo/Functions/Cnt_*Report*.sql"
  - "**/dbo/Functions/Lgx_*Report*.sql"
  - "**/dbo/Functions/PR_*Report*.sql"
alwaysApply: false
---

# Report Database Functions

## Overview

Report data is served by **table-valued functions (TVFs)** in the SQL Server database project. These functions are consumed by the AnalyticsReader API via Entity Framework Core's `builder.ToFunction("dbo.FunctionName")` mapping. The function name in SQL must **exactly match** the string in the C# configuration class.

**File Location**: `AccFlex_Cloud_Database_State/AccFlex_Cloud_Database_State/dbo/Functions/{FunctionName}.sql`

**Cross-Reference**: After creating the SQL function, 4 backend artifacts must be created in AnalyticsReader — see `mdc:report-backend`.

---

## Naming Conventions by Module

| Module | Prefix | Example |
|--------|--------|---------|
| CashControl | `Saf_Rpt_` | `Saf_Rpt_BankBalance`, `Saf_Rpt_TreasuryTransactionsReport` |
| GeneralLedger | `Acc_` | `Acc_AccountBalanceReport`, `Acc_CostCenterBalanceReport` |
| Construction | `Cnt_` | `Cnt_ProjectDetailedReport`, `Cnt_OwnersAgingReportHeader` |
| Logistics | `Lgx_` | `Lgx_InventoryOnHandReport`, `Lgx_InventoryManagementOnHandReport` |
| Payroll | `PR_` | `PR_EmployeeAccrualsReport` |

---

## ROW_NUMBER() Keying (Required)

Every report function **must** include `ROW_NUMBER()` as the first column in the final SELECT. This serves as the primary key for EF Core's entity mapping.

```sql
SELECT ROW_NUMBER() OVER (ORDER BY {column}) AS RowNo
     , ...other columns...
FROM ...
```

- Data type must be `BIGINT` (maps to C# `long RowNo`)
- Must be the **first column** in the result set
- The ORDER BY clause should use a meaningful column (usually the primary entity ID)

---

## Standard Parameter Patterns

```sql
-- Date range (most common)
@FromDate DATETIME,
@ToDate DATETIME

-- Alternative date naming
@from DATETIME,
@to DATETIME

-- Comma-separated ID lists (parsed by helper function)
@BankIds NVARCHAR(MAX) = NULL,
@BranchIds NVARCHAR(MAX) = NULL,
@AccountIds NVARCHAR(MAX) = NULL,
@CustomerIds NVARCHAR(MAX) = NULL

-- Filter parameters
@JournalRevise NVARCHAR(50) = NULL,
@Status INT = NULL,
@IsActive BIT = NULL

-- Language/culture
@IsArabic BIT = 0

-- Security
@SecurityLevel INT = NULL

-- Year parameter
@Year INT
```

**Key Rule**: All ID list parameters should default to `NULL` and be treated as "show all" when NULL.

---

## Comma-Separated ID List Handling

Use the `Acc_ConvertIntListToTable()` helper function to parse comma-separated ID strings into a table:

```sql
-- Standard pattern: filter only when parameter is provided
WHERE (@BankIds IS NULL OR Saf_Bank.BankId IN (
    SELECT number FROM dbo.Acc_ConvertIntListToTable(@BankIds)
))
```

This pattern ensures:
- When `@BankIds` is `NULL` → no filtering (show all)
- When `@BankIds` is `'1,2,3'` → filter to those specific IDs

---

## TVF Style 1: Inline TVF (Preferred)

Use for most reports. The query optimizer can inline these for better performance.

```sql
CREATE FUNCTION dbo.{Module}_{Description}Report
(
      @from DATETIME
    , @to DATETIME
    , @IdList NVARCHAR(MAX) = NULL
)
RETURNS TABLE
AS
    RETURN (
    WITH CTE1 AS
    (
        SELECT ...
        FROM ...
        WHERE (SomeDate BETWEEN @from AND @to)
        GROUP BY ...
    ),
    CTE2 AS
    (
        SELECT ...
        FROM ...
        WHERE (SomeDate BETWEEN @from AND @to)
        GROUP BY ...
    ),
    FinalCTE AS
    (
        SELECT MainTable.Name
             , MainTable.Id
             , ISNULL(CTE1.Amount, 0) AS Amount1
             , ISNULL(CTE2.Amount, 0) AS Amount2
        FROM MainTable
            LEFT OUTER JOIN CTE1 ON MainTable.Id = CTE1.Id
            LEFT OUTER JOIN CTE2 ON MainTable.Id = CTE2.Id
        WHERE (@IdList IS NULL OR MainTable.Id IN (
                SELECT number FROM dbo.Acc_ConvertIntListToTable(@IdList)
            ))
    )
    SELECT ROW_NUMBER() OVER (ORDER BY Id) AS RowNo
         , *
         , (b.CalculatedField) AS DerivedColumn
    FROM FinalCTE
        CROSS APPLY (
            SELECT (Amount1 - Amount2) AS CalculatedField
        ) AS b
    )
GO
```

### Real Example: `Saf_Rpt_BankBalance`

```sql
CREATE FUNCTION dbo.Saf_Rpt_BankBalance
(
      @from DATETIME
    , @to DATETIME
    , @BankIds NVARCHAR(MAX) = NULL
)
RETURNS TABLE
AS
    RETURN (
    WITH payableUnderCollection AS
    (
        SELECT Saf_Bank.BankId
             , SUM(Saf_PayableNote.TotalAmount) AS TotalPaidUnderCollect
        FROM Saf_Bank
            LEFT OUTER JOIN Saf_PayableNote ON Saf_Bank.BankId = Saf_PayableNote.DrwenBankId
        WHERE (Saf_PayableNote.PayableNoteStatusId = 1)
            AND (Saf_PayableNote.MaturityDate BETWEEN @from AND @to)
        GROUP BY Saf_Bank.BankId
    ),
    ReceivableUnderCollection AS
    (
        -- ... similar CTE pattern
    ),
    -- ... more CTEs for each data source
    cte AS
    (
        SELECT Saf_Bank_2.BankName
             , Saf_Bank_2.BankId
             , ISNULL(AccountBalance_1.BeginingBalance, 0) AS BeginningBalance
             , ISNULL(ReceivableBankTransfer_1.TotalRecieved, 0)
               + ISNULL(CollectedReceivableNote_1.TotalRecievedNote, 0) AS TotalReceive
             , ISNULL(PayableBankTransfer_1.TotalPaid, 0)
               + ISNULL(CollectedPayableNote_1.TotalPaidNote, 0) AS TotalPaid
        FROM Saf_Bank AS Saf_Bank_2
            LEFT OUTER JOIN ... ON ...
        WHERE (@BankIds IS NULL OR Saf_Bank_2.BankId IN (
                SELECT number FROM dbo.Acc_ConvertIntListToTable(@BankIds)
            ))
    )
    SELECT ROW_NUMBER() OVER (ORDER BY BankId) AS RowNo
         , *
         , (b.Balance + TotalRecievedUnderCollect - TotalPaidUnderCollect) AS ExpectedBalance
    FROM cte
        CROSS APPLY (
            SELECT (BeginningBalance + TotalReceive - TotalPaid) AS Balance
        ) AS b
    )
GO
```

---

## TVF Style 2: Multi-Statement TVF

Use when procedural logic is needed (running totals, cursors, cumulative balances).

```sql
CREATE FUNCTION dbo.{Module}_{Description}Report
(
      @from DATETIME
    , @to DATETIME
    , @IdList NVARCHAR(MAX) = NULL
)
RETURNS @Result TABLE
(
    RowNo BIGINT,
    EntityId INT,
    EntityName NVARCHAR(200),
    Amount MONEY,
    RunningTotal MONEY
)
AS
BEGIN
    -- Insert base data
    INSERT INTO @Result (RowNo, EntityId, EntityName, Amount, RunningTotal)
    SELECT ROW_NUMBER() OVER (ORDER BY EntityId) AS RowNo
         , EntityId
         , EntityName
         , Amount
         , 0 -- Will be calculated below
    FROM SourceTable
    WHERE (@IdList IS NULL OR EntityId IN (
        SELECT number FROM dbo.Acc_ConvertIntListToTable(@IdList)
    ))
    AND (TransactionDate BETWEEN @from AND @to)

    -- Calculate running totals
    UPDATE r
    SET RunningTotal = (
        SELECT SUM(r2.Amount)
        FROM @Result r2
        WHERE r2.RowNo <= r.RowNo
    )
    FROM @Result r

    RETURN
END
GO
```

---

## CTE Composition Pattern

Complex reports typically chain 5-10+ CTEs together, each calculating one aspect of the data:

```sql
WITH
    -- Step 1: Aggregate from source A
    SourceA_CTE AS (SELECT Id, SUM(Amount) AS Total FROM TableA GROUP BY Id),

    -- Step 2: Aggregate from source B
    SourceB_CTE AS (SELECT Id, SUM(Amount) AS Total FROM TableB GROUP BY Id),

    -- Step 3: Get opening balances
    OpeningBalance_CTE AS (SELECT Id, dbo.GetBalance(Id) AS Balance FROM MainTable),

    -- Step 4: Combine all into final result
    Final_CTE AS (
        SELECT m.Id, m.Name
             , ISNULL(a.Total, 0) AS SourceATotal
             , ISNULL(b.Total, 0) AS SourceBTotal
             , ISNULL(ob.Balance, 0) AS OpeningBalance
        FROM MainTable m
            LEFT OUTER JOIN SourceA_CTE a ON m.Id = a.Id
            LEFT OUTER JOIN SourceB_CTE b ON m.Id = b.Id
            LEFT OUTER JOIN OpeningBalance_CTE ob ON m.Id = ob.Id
    )
SELECT ROW_NUMBER() OVER (ORDER BY Id) AS RowNo, *
FROM Final_CTE
```

---

## Data Type Conventions

| Purpose | SQL Type | C# Type |
|---------|----------|---------|
| RowNo (key) | `BIGINT` (via ROW_NUMBER) | `long` |
| Money/amounts | `MONEY` or `DECIMAL(19,4)` | `decimal?` |
| Dates | `DATETIME` | `DateTime?` |
| Names/descriptions | `NVARCHAR(length)` | `string?` |
| IDs | `INT` | `int?` |
| Flags | `BIT` | `bool?` |
| Counts | `INT` | `int?` |

**Important**: Column names in the SQL result must **exactly match** the C# property names in the OData View class.

---

## NULL Handling Best Practices

```sql
-- Use ISNULL for aggregations to avoid NULL propagation
ISNULL(SUM(Amount), 0) AS TotalAmount

-- Use ISNULL when combining CTEs
ISNULL(CTE1.Total, 0) + ISNULL(CTE2.Total, 0) AS CombinedTotal

-- Optional parameter filtering
WHERE (@Param IS NULL OR Column = @Param)

-- Optional ID list filtering
WHERE (@IdList IS NULL OR Id IN (
    SELECT number FROM dbo.Acc_ConvertIntListToTable(@IdList)
))
```

---

## CROSS APPLY for Calculated Fields

Use `CROSS APPLY` for derived calculations that depend on aggregated values:

```sql
SELECT ROW_NUMBER() OVER (ORDER BY Id) AS RowNo
     , *
     , b.Balance
     , (b.Balance + TotalReceived - TotalPaid) AS ExpectedBalance
FROM FinalCTE
    CROSS APPLY (
        SELECT (OpeningBalance + TotalCredit - TotalDebit) AS Balance
    ) AS b
```

This avoids repeating complex expressions and keeps the final SELECT clean.

---

## Performance Guidelines

1. **Filter early in CTEs** — apply WHERE clauses in the first CTE, not the final SELECT
2. **Avoid cursors** in inline TVFs (not supported) — use multi-statement TVF if needed
3. **Use `ISNULL(SUM(...), 0)`** instead of checking for NULL after aggregation
4. **LEFT OUTER JOIN** from the main entity table to CTEs (not INNER JOIN) to include entities with zero values
5. **Index hints** — avoid unless proven necessary by query plan analysis
6. **Return only necessary columns** — don't SELECT * from source tables within CTEs
