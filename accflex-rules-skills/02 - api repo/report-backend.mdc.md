---
description: "Backend report implementation patterns for the AnalyticsReader API. Covers the 4 coordinated artifacts per report: (1) OData Entity (IODataEntity), (2) EF Configuration (IEntityTypeConfiguration), (3) DataContext DbSet, (4) Angular Report (IAngularReport<T>). Use when creating or modifying reports in AnalyticsReader."
globs:
  - "**/AnalyticsReader/**/*.cs"
  - "**/ODataViews/**/*.cs"
  - "**/ODataViewsConfigurations/**/*.cs"
  - "**/AngularReports/**/*.cs"
alwaysApply: false
---

# Report Backend Implementation Patterns

## Architecture Overview

The reporting system follows a three-layer architecture:

```
Frontend Request → Angular Report (IAngularReport<T>) → SQL Function → OData View → DataContext → Database
```

- **AnalyticsReader.Api** is a standalone API project serving OData endpoints and report data
- Reports are **auto-discovered** via Scrutor DI — no manual registration needed
- EF configs are applied by `ApplyConfigurationsFromAssembly`
- OData model is built from all `IODataEntity.Configure` implementations

**Cross-References**: SQL functions — see `mdc:report-database-functions`. Frontend Angular reports — see `mdc:report-module`.

---

## The 4-Artifact Checklist

Every new report requires **exactly 4 coordinated artifacts**. Missing any one causes a silent failure.

| # | Artifact | Interface | Location |
|---|----------|-----------|----------|
| 1 | OData Entity | `IODataEntity` | `ODataViews/{Module}/{SubFolder}/{EntityName}.cs` |
| 2 | EF Configuration | `IEntityTypeConfiguration<T>` | `ODataViewsConfigurations/{Module}/{SubFolder}/{EntityName}Config.cs` |
| 3 | DataContext DbSet | — | `Models/DataContext.cs` |
| 4 | Angular Report | `IAngularReport<T>` | `AngularReports/{Module}/{SubFolder}/{EntityName}Report.cs` |

---

## Artifact 1: OData Entity (IODataEntity)

**File**: `AnalyticsReader.Api/ODataViews/{Module}/{SubFolder}/{EntityName}.cs`

The OData entity defines the data model returned by the report. Properties must **exactly match** the SQL function's result set columns.

```csharp
using AnalyticsReader.Api.Helpers.ODataHelpers;
using Microsoft.OData.ModelBuilder;

namespace AnalyticsReader.Api.ODataViews.{Module}.{SubFolder}
{
    public class {EntityName} : IODataEntity
    {
        public long RowNo { get; set; }          // Always required — maps to ROW_NUMBER() in SQL
        public string? Name { get; set; }         // Use nullable types for all columns
        public int? EntityId { get; set; }
        public decimal? Amount { get; set; }
        public decimal? Balance { get; set; }
        public DateTime? TransactionDate { get; set; }

        public void Configure(ODataConventionModelBuilder builder)
        {
            var entityCfg = builder.EntitySet<{EntityName}>("{EntityName}").EntityType;
            entityCfg.HasKey(x => x.RowNo);       // Always key on RowNo
        }
    }
}
```

### Key Rules

- **`long RowNo`** is mandatory — maps to `ROW_NUMBER()` in the SQL function
- **EntitySet name** must match the class name exactly (e.g., `"BankBalanceReport"` for class `BankBalanceReport`)
- **All properties nullable** except RowNo — use `decimal?`, `string?`, `int?`, `DateTime?`
- **HasKey on RowNo** — always use `entityCfg.HasKey(x => x.RowNo)`

### Master-Detail Reports (Multiple Entities in One File)

When a report has detail grids, define additional IODataEntity classes in the same file:

```csharp
// In the same file as the main entity
public class BankBalanceReportDetailUnderCollect : IODataEntity
{
    public long RowNo { get; set; }
    public int? BankId { get; set; }
    public decimal? TotalRecieved { get; set; }
    public decimal? TotalPaid { get; set; }
    public decimal? EndBalance { get; set; }

    public void Configure(ODataConventionModelBuilder builder)
    {
        var entityCfg = builder.EntitySet<BankBalanceReportDetailUnderCollect>("BankBalanceReportDetailUnderCollect").EntityType;
        entityCfg.HasKey(x => x.RowNo);
    }
}
```

---

## Artifact 2: EF Configuration (IEntityTypeConfiguration)

**File**: `AnalyticsReader.Api/ODataViewsConfigurations/{Module}/{SubFolder}/{EntityName}Config.cs`

Maps the OData entity to the SQL function via `ToFunction()`.

```csharp
using AnalyticsReader.Api.ODataViews.{Module}.{SubFolder};
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace AnalyticsReader.Api.ODataViewsConfigurations.{Module}.{SubFolder}
{
    public class {EntityName}Config : IEntityTypeConfiguration<{EntityName}>
    {
        public void Configure(EntityTypeBuilder<{EntityName}> builder)
        {
            builder.HasKey(x => x.RowNo);
            builder.ToFunction("{ExactSqlFunctionName}");  // Must match SQL function name exactly
        }
    }
}
```

### Key Rules

- **`builder.HasKey(x => x.RowNo)`** — matches the OData entity key
- **`builder.ToFunction("FunctionName")`** — the string must **exactly match** the SQL function name (no `dbo.` prefix needed)
- **One config class per entity** — for master-detail reports, put multiple configs in the same file:

```csharp
// Multiple configs in one file for master-detail reports
public class BankBalanceReportConfig : IEntityTypeConfiguration<BankBalanceReport>
{
    public void Configure(EntityTypeBuilder<BankBalanceReport> builder)
    {
        builder.HasKey(x => x.RowNo);
        builder.ToFunction("Saf_Rpt_BankBalance");
    }
}

public class BankBalanceReportDetailUnderCollectConfig : IEntityTypeConfiguration<BankBalanceReportDetailUnderCollect>
{
    public void Configure(EntityTypeBuilder<BankBalanceReportDetailUnderCollect> builder)
    {
        builder.HasKey(x => x.RowNo);
        builder.ToFunction("Saf_Rpt_BankBalanceDetailUnderCollect");
    }
}
```

---

## Artifact 3: DataContext DbSet

**File**: `AnalyticsReader.Api/Models/DataContext.cs`

Add a `DbSet<T>` property for each OData entity. Group by module.

```csharp
// In DataContext.cs — add alphabetically within the module section
public DbSet<BankBalanceReport> BankBalanceReport { get; set; }
public DbSet<BankBalanceReportDetailUnderCollect> BankBalanceReportDetailUnderCollect { get; set; }
public DbSet<BankBalanceReportDetailCollected> BankBalanceReportDetailCollected { get; set; }
```

### Key Rules

- Property name matches the class name
- Group by module sections: CashControl, Construction, GeneralLedger, Logistics, Payroll
- No other change needed — `ApplyConfigurationsFromAssembly` auto-discovers configs

---

## Artifact 4: Angular Report (IAngularReport<T>)

**File**: `AnalyticsReader.Api/AngularReports/{Module}/{SubFolder}/{EntityName}Report.cs`

Handles parameter extraction, SQL function invocation, and data return.

```csharp
using AnalyticsReader.Api.Helpers;
using AnalyticsReader.Api.Models;
using AnalyticsReader.Api.ODataViews.{Module}.{SubFolder};
using AnalyticsReader.Api.Reports.AngularReportHelper;
using Microsoft.EntityFrameworkCore;
using SharedKernel.Logic;
using SharedKernel.Logic.Helpers;

namespace AnalyticsReader.Api.AngularReports.{Module}.{SubFolder}
{
    public class {EntityName}Report : IAngularReport<{EntityName}>
    {
        private readonly DataContext _context;

        public {EntityName}Report(DataContext Context)
        {
            _context = Context;
        }

        public string ReportName => "{EntityName}Report";   // Must match frontend report name
        public Type EntityType => typeof({EntityName});

        public async Task<AngularDataSourceContainter> GetDataSource(
            string filters,
            IEnumerable<string> queryOptions)
        {
            // 1. Extract parameters
            var fromDate = queryOptions.GetQueryOptionValue("fromDate");
            var toDate = queryOptions.GetQueryOptionValue("toDate");
            var idList = queryOptions.GetQueryOptionValue("idList");

            // 2. Clean parameters
            fromDate = fromDate?.Trim().Replace("'", "").Replace("\"", "");
            toDate = toDate?.Trim().Replace("'", "").Replace("\"", "");
            idList = idList?.Trim().Replace("'", "").Replace("\"", "");

            // 3. Parse dates
            var (_fromDate, _toDate) = ReportExtensions.ParseUtcDateRange(fromDate, toDate);

            if (!(DateTime.TryParse(fromDate, out _fromDate)
                && DateTime.TryParse(toDate, out _toDate)))
                return new AngularDataSourceContainter { DataSource = new List<{EntityName}>() };

            // 4. Call SQL function
            var result = await _context.{EntityName}
                .FromSqlInterpolated(@$"SELECT * FROM {SqlFunctionName}
                ({_fromDate.GetDayStart()},{_toDate.GetDayEnd()},{idList})")
                .ToListAsync();

            // 5. Build display parameters (optional)
            var parameters = new Dictionary<string, string>
            {
                { "fromDate", string.Format($"{{0:{SharedConstants.DateFormat}}}", _fromDate) },
                { "toDate", string.Format($"{{0:{SharedConstants.DateFormat}}}", _toDate) }
            };

            return new AngularDataSourceContainter { DataSource = result, Parameters = parameters };
        }
    }
}
```

### Key Rules

- **`ReportName`** must be a unique string matching what the frontend sends
- **`EntityType`** returns `typeof(T)` where T is the OData entity
- **`GetDataSource`** extracts params → cleans → parses → executes SQL → returns container
- **`FromSqlInterpolated`** — always use interpolated string for SQL injection protection
- **`GetDayStart()` / `GetDayEnd()`** — extension methods to normalize date boundaries
- **Return empty list** if date parsing fails (don't throw)

---

## Parameter Extraction Patterns

### Standard Pattern (GetQueryOptionValue)

```csharp
var fromDate = queryOptions.GetQueryOptionValue("fromDate");
var toDate = queryOptions.GetQueryOptionValue("toDate");
var bankIdList = queryOptions.GetQueryOptionValue("bankIdList");

// Clean all parameters
fromDate = fromDate?.Trim().Replace("'", "").Replace("\"", "");
toDate = toDate?.Trim().Replace("'", "").Replace("\"", "");
bankIdList = bankIdList?.Trim().Replace("'", "").Replace("\"", "");
```

### RemoveSingleQuotes Shorthand

```csharp
var branchIds = queryOptions.GetQueryOptionValue("branchIds")?.RemoveSingleQuotes();
```

### Date Parsing with ReportExtensions

```csharp
var (_fromDate, _toDate) = ReportExtensions.ParseUtcDateRange(fromDate, toDate);
```

### Year-Based Date Range

```csharp
var year = Convert.ToInt32(
    queryOptions.GetQueryOptionValue("year")?.RemoveSingleQuotes()
    ?? DateTime.Now.Year.ToString()
);
var fromDate = new DateTime(year, 1, 1);
var toDate = new DateTime(year, 12, 31, 23, 59, 59);
```

### OData Filter-Based Extraction (Advanced)

```csharp
var filterDic = filters.FiltersDataForParameters();
var entityId = filterDic.GetValueFromHashTable<int>("entityId");
```

---

## Auto-Discovery (No Manual Registration)

Reports are automatically registered via Scrutor in `DependencyInjectionHelperExtension.cs`:
- All classes implementing `IAngularReport<T>` are scanned from the entry assembly
- Registered with **scoped lifetime**
- No need to add anything to `Startup.cs` or DI container manually

Similarly:
- EF configurations are auto-discovered via `ApplyConfigurationsFromAssembly`
- OData entities are auto-discovered via `IODataEntity.Configure` scanning

---

## Naming Conventions

| Component | Pattern | Example |
|-----------|---------|---------|
| OData Entity class | `{EntityName}` | `BankBalanceReport` |
| OData Entity file | `{EntityName}.cs` | `BankBalanceReport.cs` |
| Configuration class | `{EntityName}Config` | `BankBalanceReportConfig` |
| Configuration file | `{EntityName}Config.cs` | `BankBalanceReportConfig.cs` |
| Angular Report class | `{ShortName}Rpt` or `{EntityName}Report` | `BankBalanceRpt` |
| Angular Report file | `{ShortName}Rpt.cs` or `{EntityName}Report.cs` | `BankBalanceRpt.cs` |
| ReportName property | `"{EntityName}Report"` | `"BankBalanceReport"` |
| DbSet property | `{EntityName}` | `BankBalanceReport` |
| EntitySet name | `"{EntityName}"` | `"BankBalanceReport"` |
| SQL function | `{ModulePrefix}_{Description}` | `Saf_Rpt_BankBalance` |

### Module Folder Mapping

| Module | ODataViews Folder | AngularReports Folder | Config Folder |
|--------|-------------------|----------------------|---------------|
| CashControl | `ODataViews/CashControl/` | `AngularReports/CashControl/` | `ODataViewsConfigurations/CashControl/` |
| Construction | `ODataViews/Construction/` | `AngularReports/Construction/` | `ODataViewsConfigurations/Construction/` |
| GeneralLedger | `ODataViews/GL/` | `AngularReports/GeneralLedger/` | `ODataViewsConfigurations/GL/` |
| Logistics | `ODataViews/Logistics/` | `AngularReports/Logistics/` | `ODataViewsConfigurations/Logistics/` |
| Payroll | `ODataViews/Payroll/` | `AngularReports/Payroll/` | `ODataViewsConfigurations/Payroll/` |

---

## IAngularPrint<T> (Print-Specific Reports)

For print-specific reports (e.g., invoice prints), implement `IAngularPrint<T>` instead:

```csharp
public class InvoicePrintReport : IAngularPrint<InvoicePrintEntity>
{
    public string ReportName => "InvoicePrint";
    public Type EntityType => typeof(InvoicePrintEntity);

    public async Task<AngularDataSourceContainter> GetDataSource(
        int headerId,
        string printData)
    {
        // headerId is the document ID
        // printData contains additional print configuration
        var result = await _context.InvoicePrintEntity
            .FromSqlInterpolated($"SELECT * FROM dbo.PrintFunction({headerId})")
            .ToListAsync();

        return new AngularDataSourceContainter { DataSource = result };
    }
}
```

---

## Common Mistakes and Gotchas

### 1. Missing DbSet Registration
**Symptom**: Entity not found at runtime, no OData endpoint available
**Fix**: Add `public DbSet<YourEntity> YourEntity { get; set; }` to `DataContext.cs`

### 2. Function Name Mismatch
**Symptom**: `Microsoft.Data.SqlClient.SqlException: Invalid object name`
**Fix**: Ensure `builder.ToFunction("FunctionName")` exactly matches the SQL function name

### 3. Property/Column Name Mismatch
**Symptom**: Data conversion errors or null values where data exists
**Fix**: OData Entity property names must exactly match SQL function column names

### 4. Missing RowNo in SQL Function
**Symptom**: EF Core mapping errors
**Fix**: Add `ROW_NUMBER() OVER (ORDER BY {column}) AS RowNo` as the first column

### 5. EntitySet Name Mismatch
**Symptom**: OData endpoint returns 404
**Fix**: EntitySet name in `Configure()` must match the class name: `builder.EntitySet<MyReport>("MyReport")`

### 6. Null QueryOptions
**Symptom**: `NullReferenceException` when extracting parameters
**Fix**: Use null-conditional: `queryOptions?.GetQueryOptionValue(...)` and return empty list on parse failure

### 7. SQL Injection Risk
**Symptom**: Security vulnerability
**Fix**: Always use `FromSqlInterpolated($"...")` — never concatenate user input into SQL strings

---

## Full Working Example: BankBalanceReport

### OData Entity (`ODataViews/CashControl/BankBalanceRpt/BankBalanceReport.cs`)

```csharp
using AnalyticsReader.Api.Helpers.ODataHelpers;
using Microsoft.OData.ModelBuilder;

namespace AnalyticsReader.Api.ODataViews.CashControl.BankBalanceRpt
{
    public class BankBalanceReport : IODataEntity
    {
        public long RowNo { get; set; }
        public string? BankName { get; set; }
        public int? BankId { get; set; }
        public decimal? BeginningBalance { get; set; }
        public decimal? TotalReceive { get; set; }
        public decimal? TotalPaid { get; set; }
        public decimal? TotalPaidUnderCollect { get; set; }
        public decimal? TotalRecievedUnderCollect { get; set; }
        public decimal? Balance { get; set; }
        public decimal? ExpectedBalance { get; set; }

        public void Configure(ODataConventionModelBuilder builder)
        {
            var entityCfg = builder.EntitySet<BankBalanceReport>("BankBalanceReport").EntityType;
            entityCfg.HasKey(x => x.RowNo);
        }
    }
}
```

### EF Config (`ODataViewsConfigurations/CashControl/BankBalanceReportConfigs/BankBalanceReportConfig.cs`)

```csharp
using AnalyticsReader.Api.ODataViews.CashControl.BankBalanceRpt;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace AnalyticsReader.Api.ODataViewsConfigurations.CashControl.BankBalanceReportConfigs
{
    public class BankBalanceReportConfig : IEntityTypeConfiguration<BankBalanceReport>
    {
        public void Configure(EntityTypeBuilder<BankBalanceReport> builder)
        {
            builder.HasKey(x => x.RowNo);
            builder.ToFunction("Saf_Rpt_BankBalance");
        }
    }
}
```

### DataContext (`Models/DataContext.cs`)

```csharp
public DbSet<BankBalanceReport> BankBalanceReport { get; set; }
```

### Angular Report (`AngularReports/CashControl/BankBalanceReportAng/BankBalanceRpt.cs`)

```csharp
using AnalyticsReader.Api.Helpers;
using AnalyticsReader.Api.Models;
using AnalyticsReader.Api.ODataViews.CashControl.BankBalanceRpt;
using AnalyticsReader.Api.Reports.AngularReportHelper;
using Microsoft.EntityFrameworkCore;
using SharedKernel.Logic.Helpers;

namespace AnalyticsReader.Api.AngularReports.CashControl.BankBalanceReportAng
{
    public class BankBalanceRpt : IAngularReport<BankBalanceReport>
    {
        private readonly DataContext _context;

        public BankBalanceRpt(DataContext Context)
        {
            _context = Context;
        }

        public string ReportName => "BankBalanceReport";
        public Type EntityType => typeof(BankBalanceReport);

        public async Task<AngularDataSourceContainter> GetDataSource(
            string filters,
            IEnumerable<string> queryOptions)
        {
            var fromDate = queryOptions.GetQueryOptionValue("fromDate");
            var toDate = queryOptions.GetQueryOptionValue("toDate");
            var bankIdList = queryOptions.GetQueryOptionValue("bankIdList");

            fromDate = fromDate?.Trim().Replace("'", "").Replace("\"", "");
            toDate = toDate?.Trim().Replace("'", "").Replace("\"", "");
            bankIdList = bankIdList?.Trim().Replace("'", "").Replace("\"", "");

            var (_fromDate, _toDate) = ReportExtensions.ParseUtcDateRange(fromDate, toDate);

            if (!(DateTime.TryParse(fromDate, out _fromDate)
                && DateTime.TryParse(toDate, out _toDate)))
                return new AngularDataSourceContainter { DataSource = new List<BankBalanceReport>() };

            var result = await _context.BankBalanceReport
                .FromSqlInterpolated(@$"SELECT * FROM Saf_Rpt_BankBalance
                ({_fromDate.GetDayStart()},{_toDate.GetDayEnd()},{bankIdList})")
                .ToListAsync();

            var parameters = new Dictionary<string, string>
            {
                { "fromDate", string.Format($"{{0:{SharedConstants.DateFormat}}}", _fromDate) },
                { "toDate", string.Format($"{{0:{SharedConstants.DateFormat}}}", _toDate) }
            };

            return new AngularDataSourceContainter { DataSource = result, Parameters = parameters };
        }
    }
}
```
