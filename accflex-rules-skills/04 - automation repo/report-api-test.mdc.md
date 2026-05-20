---
description: Patterns for testing Accflex ERP report endpoints in the AnalyticsReader API. Covers report test structure (read-only, no CRUD cycle), OData endpoint testing, AngularReportApi controller testing, parameter validation, and DataSource/Parameters verification. Use when creating or modifying tests for any report endpoint or OData query.
globs:
  - "**/*Report*Test*.cs"
  - "**/*report*test*.cs"
  - "**/AnalyticsReader*Test*/**/*.cs"
alwaysApply: false
---

# Report API Test Patterns

## Overview

Report tests differ fundamentally from CRUD tests. Reports are **read-only** — there is no create/update/delete cycle. Instead, the pattern is:

1. **Seed prerequisite data** (transactions, master data)
2. **Call the report endpoint** with parameters
3. **Verify the DataSource** (not empty, correct values, proper aggregation)
4. **Verify the Parameters** dictionary (display values for report header)

**Cross-References**: Report backend architecture — see `mdc:report-backend`. Test framework and base classes — see `mdc:api-test-core` and `mdc:core`.

---

## Report Test Class Structure

```csharp
[TestFixture]
[janono.ado.testcase.associate.Organization("hadafsolutions")]
public class BankBalanceReportTests : TestBase
{
    public const string language = "en-us";

    readonly ReportService _reportService;
    readonly BankService _bankService;          // For seeding prerequisite data
    readonly TransactionService _txnService;    // For seeding transactions

    public BankBalanceReportTests()
    {
        ServiceFactory.Init();
        _reportService = new ReportService();
        _bankService = new BankService();
        _txnService = new TransactionService();
    }

    [OneTimeSetUp]
    public async Task Setup()
    {
        var tokenService = ServiceFactory.GetTokenService();
        await tokenService.Login(Constants.UserName, Constants.Password, Constants.TenantId);
        var tenant = await ServiceFactory.GetTenantService().GetTenant();
        await DatabaseService.InitRespawner(tenant.ConnectionString);
    }

    [SetUp]
    public async Task TestSetup()
    {
        await DatabaseService.RestDatabase();
    }
}
```

### Key Differences from CRUD Tests

| Aspect | CRUD Test | Report Test |
|--------|-----------|-------------|
| Primary action | Create/Update/Delete entity | Call report endpoint with parameters |
| Setup | Minimal (entity under test) | Seed transactions and master data |
| Assertions | Entity saved correctly | DataSource contains expected aggregations |
| Return type | `(Dto, ApiError)` | `AngularDataSourceContainter` (DataSource + Parameters) |
| Idempotency | Each test modifies data | Read-only, multiple calls return same results |

---

## Testing the AngularReportApi Endpoints

The AngularReportApi controller exposes these endpoints:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `angular/ExportTo` | GET/POST | Export report data |
| `angular/AngularReportPdf` | GET/POST | Generate PDF |
| `angular/AngularReportXls` | GET/POST | Generate Excel |
| `angular/GetReportDataType` | GET | Get report column metadata |

### Testing Report Data Retrieval

```csharp
[Test]
[janono.ado.testcase.associate.TestCase(12345)]
[Description("Related Test Case ==> https://hadafsolutions.visualstudio.com/AccFlex_ERP/_workitems/edit/12345")]
public async Task BankBalanceReport_WithValidDateRange_ShouldReturnData()
{
    // Arrange — Seed prerequisite data
    var bank = await SeedBankPredecessor();
    await SeedBankTransactions(bank.BankId);

    // Act — Call report with parameters
    var reportName = "BankBalanceReport";
    var queryOptions = new Dictionary<string, string>
    {
        { "fromDate", new DateTime(CurrentYear, 1, 1).ToString("o") },
        { "toDate", new DateTime(CurrentYear, 12, 31).ToString("o") },
        { "bankIdList", bank.BankId.ToString() }
    };

    var (result, errors) = await _reportService.GetReportData(language, reportName, queryOptions);

    // Assert
    errors.ShouldHasNoError();
    result.Should().NotBeNull();
    result.DataSource.Should().NotBeNull();

    var data = result.DataSource as List<dynamic>;
    data.Should().NotBeEmpty();
    data.Should().HaveCountGreaterThan(0);
}
```

### Testing Report with No Data

```csharp
[Test]
[janono.ado.testcase.associate.TestCase(12346)]
[Description("Related Test Case ==> https://hadafsolutions.visualstudio.com/AccFlex_ERP/_workitems/edit/12346")]
public async Task BankBalanceReport_WithNoMatchingData_ShouldReturnEmptyList()
{
    // Act — Call report with date range that has no transactions
    var queryOptions = new Dictionary<string, string>
    {
        { "fromDate", new DateTime(2000, 1, 1).ToString("o") },
        { "toDate", new DateTime(2000, 12, 31).ToString("o") }
    };

    var (result, errors) = await _reportService.GetReportData(language, "BankBalanceReport", queryOptions);

    // Assert
    errors.ShouldHasNoError();
    result.DataSource.Should().NotBeNull();
}
```

---

## Parameter Testing Patterns

### NULL Parameters (Optional Filters)

```csharp
[Test]
public async Task Report_WithNullOptionalParams_ShouldReturnAllData()
{
    // Arrange
    await SeedMultipleBanks();

    // Act — omit optional bankIdList parameter
    var queryOptions = new Dictionary<string, string>
    {
        { "fromDate", new DateTime(CurrentYear, 1, 1).ToString("o") },
        { "toDate", new DateTime(CurrentYear, 12, 31).ToString("o") }
        // bankIdList intentionally omitted — should return all banks
    };

    var (result, errors) = await _reportService.GetReportData(language, "BankBalanceReport", queryOptions);

    errors.ShouldHasNoError();
    var data = result.DataSource as List<dynamic>;
    data.Should().HaveCountGreaterThanOrEqualTo(2); // All banks returned
}
```

### Multiple IDs (Comma-Separated)

```csharp
[Test]
public async Task Report_WithMultipleIds_ShouldFilterCorrectly()
{
    // Arrange
    var bank1 = await SeedBankPredecessor("Bank A");
    var bank2 = await SeedBankPredecessor("Bank B");
    var bank3 = await SeedBankPredecessor("Bank C");

    // Act — pass comma-separated IDs
    var queryOptions = new Dictionary<string, string>
    {
        { "fromDate", new DateTime(CurrentYear, 1, 1).ToString("o") },
        { "toDate", new DateTime(CurrentYear, 12, 31).ToString("o") },
        { "bankIdList", $"{bank1.BankId},{bank2.BankId}" }  // Only banks 1 and 2
    };

    var (result, errors) = await _reportService.GetReportData(language, "BankBalanceReport", queryOptions);

    errors.ShouldHasNoError();
    var data = result.DataSource as List<dynamic>;
    data.Should().HaveCount(2);  // Only the 2 requested banks
}
```

### Date Range Edge Cases

```csharp
[Test]
public async Task Report_WithSameDayRange_ShouldReturnDayData()
{
    // Act — single day range
    var today = DateTime.Today;
    var queryOptions = new Dictionary<string, string>
    {
        { "fromDate", today.ToString("o") },
        { "toDate", today.ToString("o") }
    };

    var (result, errors) = await _reportService.GetReportData(language, "BankBalanceReport", queryOptions);
    errors.ShouldHasNoError();
}

[Test]
public async Task Report_WithInvalidDateRange_ShouldReturnEmpty()
{
    // Act — toDate before fromDate
    var queryOptions = new Dictionary<string, string>
    {
        { "fromDate", new DateTime(CurrentYear, 12, 31).ToString("o") },
        { "toDate", new DateTime(CurrentYear, 1, 1).ToString("o") }
    };

    var (result, errors) = await _reportService.GetReportData(language, "BankBalanceReport", queryOptions);
    // Report should handle gracefully — either empty or error
    errors.ShouldHasNoError();
}
```

---

## DataSource Assertions

### Verify Column Values

```csharp
// Verify specific row data
var firstRow = data.First();
firstRow.BankName.Should().Be("Test Bank");
firstRow.BankId.Should().Be(expectedBankId);

// Verify calculated fields
firstRow.Balance.Should().Be(firstRow.BeginningBalance + firstRow.TotalReceive - firstRow.TotalPaid);
firstRow.ExpectedBalance.Should().Be(firstRow.Balance + firstRow.TotalRecievedUnderCollect - firstRow.TotalPaidUnderCollect);
```

### Verify Aggregation Correctness

```csharp
// Verify totals match expected values
var bankRow = data.First(x => x.BankId == bank.BankId);
bankRow.TotalReceive.Should().Be(expectedTotalReceive);
bankRow.TotalPaid.Should().Be(expectedTotalPaid);
bankRow.BeginningBalance.Should().Be(expectedOpeningBalance);
```

### Verify Collection Properties

```csharp
// Verify row count matches expected entities
data.Should().HaveCount(expectedBankCount);

// Verify all rows have RowNo (primary key)
data.All(x => x.RowNo > 0).Should().BeTrue();

// Verify no duplicates
data.Select(x => x.RowNo).Distinct().Should().HaveCount(data.Count);
```

---

## Parameters Dictionary Verification

The `Parameters` dictionary contains display values for the report header (e.g., formatted dates).

```csharp
// Verify Parameters exist
result.Parameters.Should().NotBeNull();
result.Parameters.Should().ContainKey("fromDate");
result.Parameters.Should().ContainKey("toDate");

// Verify date formatting
result.Parameters["fromDate"].Should().Be("01/01/2026");  // dd/MM/yyyy format
result.Parameters["toDate"].Should().Be("31/12/2026");
```

---

## Test Data Prerequisites

Reports need existing transactional data to verify. Use predecessor methods to seed data before report calls.

### Predecessor Pattern for Reports

```csharp
private async Task<BankDto> SeedBankPredecessor(string bankName = "Test Bank")
{
    var (bank, errors) = await _bankService.CreateBank(language, new CreateBankCommand
    {
        BankName = bankName,
        AccountId = defaultAccountId,
        CurrencyId = defaultCurrencyId,
        BranchId = defaultBranchId
    });
    errors.ShouldHasNoError();
    bank.BankId.Should().BePositive();
    return bank;
}

private async Task SeedBankTransactions(int bankId)
{
    // Seed receivable transfers
    var (transfer, errors) = await _txnService.CreateReceivableBankTransfer(language, new CreateTransferCommand
    {
        BankId = bankId,
        Amount = 10000m,
        TransferDate = new DateTime(CurrentYear, 6, 15)
    });
    errors.ShouldHasNoError();

    // Seed payable transfers
    var (payable, errors2) = await _txnService.CreatePayableBankTransfer(language, new CreateTransferCommand
    {
        BankId = bankId,
        Amount = 3000m,
        TransferDate = new DateTime(CurrentYear, 6, 20)
    });
    errors2.ShouldHasNoError();
}
```

### Test Data Seeding Order

For report tests, seed data in this order:
1. **Master data** — banks, accounts, cost centers, branches
2. **Configuration** — accounting periods, currencies, exchange rates
3. **Transactions** — invoices, transfers, payments, journals
4. **Verify** — call report and assert results

---

## OData Endpoint Testing

For reports exposed as OData endpoints (via `IODataEntity`), test the OData query syntax:

```csharp
[Test]
public async Task ODataEndpoint_WithFilter_ShouldReturnFilteredData()
{
    // Act — call OData endpoint with $filter
    var odataClient = ServiceFactory.GetODataClient();
    var result = await odataClient.GetAsync<List<BankBalanceReport>>(
        "BankBalanceReport",
        queryOptions: new Dictionary<string, string>
        {
            { "fromDate", fromDate.ToString("o") },
            { "toDate", toDate.ToString("o") },
            { "bankIdList", "1,2,3" }
        });

    result.Should().NotBeNull();
    result.Should().NotBeEmpty();
}
```

---

## Test Ordering for Report Tests

When report tests depend on seeded data from earlier tests, use `[Order]`:

```csharp
[Order(1)]
[Test]
public async Task SeedTestData_ForReportTests()
{
    // Create master data and transactions
    await SeedBankPredecessor();
    await SeedBankTransactions(bankId);
}

[Order(2)]
[Test]
public async Task BankBalanceReport_VerifyTotals()
{
    // This test depends on data from Order(1)
    var (result, errors) = await _reportService.GetReportData(language, "BankBalanceReport", queryOptions);
    errors.ShouldHasNoError();
    // ... assertions
}
```

**Prefer independent tests** — seed data in each test's Arrange phase when possible. Use `[Order]` only when seeding is expensive and multiple tests verify different aspects of the same report.

---

## Best Practices

1. **Always seed data before testing** — don't rely on existing database state
2. **Test with NULL optional parameters** — verify "show all" behavior
3. **Test with multiple comma-separated IDs** — verify `Acc_ConvertIntListToTable` works
4. **Test date boundaries** — same day, year boundaries, start/end of day
5. **Verify calculated fields** — Balance = BeginningBalance + TotalReceive - TotalPaid
6. **Verify RowNo uniqueness** — no duplicate primary keys in results
7. **Check Parameters dictionary** — ensure display values are formatted correctly
8. **Test empty results gracefully** — report should return empty list, not error
9. **Follow AAA pattern** — Arrange (seed data), Act (call report), Assert (verify DataSource)
10. **Use FluentAssertions** for all assertions, `errors.ShouldHasNoError()` after every API call
