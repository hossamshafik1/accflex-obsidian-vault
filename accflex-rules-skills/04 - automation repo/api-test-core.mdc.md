---
alwaysApply: true
---
# API Test Project Rules (Project-Specific)

This document defines **Accflex ERP API Testing solution's** architecture and the rules to follow when adding or changing test code. Any generic testing guidance lives in `core.mdc`.

## Test Base Classes ##

### Base Class Structure

All test projects must have a `*TestBase` class that inherits from `TestBase` (from `Logistics.Shared.Api.Test.Utilities` or project-specific base):

```csharp
public class InventoryTestBase : TestBase
{
    public const string language = "en-us";
    protected int CurrentYear => DateTime.Now.Year;
    protected int NextYear => DateTime.Now.Year + 1;
    protected int PreviousYear => DateTime.Now.Year - 1;
    
    public InventoryTestBase()
    {
        ServiceFactory.Init();
    }
    
    [OneTimeSetUp]
    public async Task Setup() { /* ... */ }
    
    [SetUp]
    public async Task TestSetup() { /* ... */ }
    
    [TearDown]
    public async Task TearDown() { /* ... */ }
}
```

### Required Patterns

- **Constructor**: Must call `ServiceFactory.Init()`
- **OneTimeSetUp**: 
  - Login with tenant-specific credentials
  - Get tenant information
  - Initialize database respawner: `await DatabaseService.InitRespawner(tenant.ConnectionString)`
  - Initialize OData client and user access control client (for Logistics projects)
- **SetUp**: 
  - Reset database: `await DatabaseService.RestDatabase()` or `await RestDatabase()`
  - Refresh token: `await _tokenService.RefreshToken()` (for Logistics projects)
  - Track users before test (for Logistics projects)
- **TearDown**: 
  - Clean up users created during test (for Logistics projects)
- **Language Constant**: Always `"en-us"`

### Database Reset Patterns

**Simple Reset** (CashControl, Construction, GeneralLedger, Payroll):
```csharp
await DatabaseService.RestDatabase();
// Optional: with accounting period date
await DatabaseService.RestDatabase(accountingPeriodMinimumDate: new DateTime(PreviousYear, 1, 1));
```

**Reset with Retry Logic** (Logistics projects):
```csharp
private async Task RestDatabase()
{
    string restDatabaseFailureError = string.Empty;
    int resetDBTimes = 5;

    do
    {
        restDatabaseFailureError = await DatabaseService.RestDatabase(seedInitialData: true);
        if (!string.IsNullOrEmpty(restDatabaseFailureError))
            System.Threading.Thread.Sleep(5000); // wait 5 Seconds
        resetDBTimes--;
    } while (!string.IsNullOrEmpty(restDatabaseFailureError) && resetDBTimes > 0);

    if (!string.IsNullOrEmpty(restDatabaseFailureError))
        throw new Exception(restDatabaseFailureError);
    
    await DatabaseService.MakeConstructionIntegratedWithLogistics();
}
```

### User Management (Logistics Projects)

- Track users before test in `[SetUp]`
- Clean up users created during test in `[TearDown]`
- Use `_userAccessControlClient.GetAllActivePerTenantAsync()` to get user list
- Delete users that weren't in the initial list

## Test Class Structure ##

### Naming Conventions

- Test class names **must** end with `Tests` suffix
- Examples: `TreasuryTests`, `InvoiceTests`, `GoodReceiptFromPurchaseOrderTests`

### Inheritance

- All test classes must inherit from appropriate `*TestBase`:
  - Logistics tests: `InventoryTestBase`, `SalesTestBase`, `PurchaseTestBase`, `ProductionTestBase`, `MaintenanceTestBase`
  - Other projects: `TestBase` (project-specific)

### Required Attributes

```csharp
[TestFixture]
[janono.ado.testcase.associate.Organization("hadafsolutions")]
public class InvoiceTest : SalesTestBase
{
    // ...
}
```

### Optional Attributes

- `[Category("CategoryName")]` - Group related tests
- `[Property("Owner", TestCaseOwner.Name)]` - Indicate test ownership

### Service Initialization

- Declare services as `readonly` fields
- Initialize in constructor using `new()` syntax:

```csharp
readonly InvoiceService _invoiceService;
readonly SalesOrderService _salesOrderService;

public InvoiceTest()
{
    _invoiceService = new InvoiceService();
    _salesOrderService = new SalesOrderService();
}
```

## Test Method Patterns ##

### Required Attributes

```csharp
[Test]
[janono.ado.testcase.associate.TestCase(testCaseId)]
[Description("Related Test Case ==> https://hadafsolutions.visualstudio.com/AccFlex_ERP/_workitems/edit/61007")]
public async Task CreateCreditInvoiceMemo_ShouldSave()
{
    // Test implementation
}
```

### Test Method Naming

- Use PascalCase
- Descriptive names that indicate what is being tested
- Examples: `CreateTreasury`, `UpdateInvoice`, `CreateGoodReceiptCompleteThePO`

### Order Attribute

Use `[Order(n)]` when tests have dependencies:

```csharp
[Order(1)]
[Test]
[janono.ado.testcase.associate.TestCase(112935)]
public async Task CreateTransportSupplierData() { /* ... */ }

[Order(2)]
[Test]
[janono.ado.testcase.associate.TestCase(112936)]
public async Task CreateTransportationPurchaseOrder() { /* ... */ }
```

### Async Pattern

All test methods must be `async Task`:

```csharp
public async Task CreateTreasury()
{
    var result = await _treasuryService.PostTreasury(language, createTreasuryDto);
    result.errors.ShouldHasNoError();
}
```

## Service Layer Patterns ##

### Service Organization

- Services organized in `Services/` folders
- Shared services in `Logistics.Shared.Api.Test/Services/` or `SharedAPITests/Services/`
- Project-specific services in project's `Services/` folder

### Service Return Type Pattern

All service methods return tuple: `Task<(Dto dto, ApiError errors)>`

```csharp
public async Task<(TreasuryDto dto, ApiError errors)> PostTreasury(string language, CreateTreasuryCommand command)
{
    // Implementation
    return (dto, errors);
}
```

### Error Handling in Services

- Services catch exceptions and convert to `ApiError` format
- Use `ex.GetError()` extension method to convert exceptions
- Always return tuple with errors dictionary

### Service Initialization

- Services initialize their own API clients in constructor
- Use `ServiceFactory.GetClient<T>()` to get API clients
- Services are stateless and can be reused

## Predecessor/Helper Methods ##

### Naming Conventions

**Private Helper Methods** (within test class):
- `Create{Entity}Predecessor{TestCaseId}()` - Returns created entity
- `Predecessor{TestCaseId}()` - Sets up test data without return value
- Example: `CreateTreasuryPredecessor20238()`, `Predecessor_61007()`

**Predecessor Service Classes**:
- Named as `{Entity}PredecessorService`
- Example: `InvoicePredecessorService`, `GoodIssueFromOutboundDeliveryPredessorService`

### Predecessor Service Pattern

```csharp
public class InvoicePredecessorService
{
    readonly InvoiceService _invoiceService;
    readonly SalesOrderService _salesOrderService;
    // ... other services

    public InvoicePredecessorService()
    {
        _invoiceService = new InvoiceService();
        _salesOrderService = new SalesOrderService();
        // ... initialize all services
    }

    public async Task Predecessor64765(Predecessor_81340Data predecessorData)
    {
        // Setup logic
        var (result, errors) = await _invoiceService.GetInvoiceBySalesOrder(...);
        errors.ShouldHasNoError();
        // ... more setup
    }
}
```

### Helper Method Organization

- Keep helper methods private within test class
- Use predecessor services for complex, reusable setups
- Document helper methods with comments explaining their purpose

## Error Handling & Assertions ##

### Error Checking Pattern

**Always check for errors after API calls**:

```csharp
var (result, errors) = await _service.PostEntity(language, command);
errors.ShouldHasNoError();
```

### FluentAssertions Extensions

Use custom extension methods from `SharedAPITests.FluentAssertionsExtensions`:

- `errors.ShouldHasNoError()` - Verify no errors (most common)
- `errors.ShouldHasError(string)` - Verify specific error exists
- `errors.ShouldHasOneError(string)` - Verify exactly one specific error
- `errors.ShouldMatchError(string)` - Verify error matches pattern (supports * and ?)

### FluentAssertions Patterns

```csharp
// Equality
result.Id.Should().Be(expectedId);
result.Name.Should().Be("Expected Name");

// Collections
result.Items.Should().HaveCount(5);
result.Items.Should().Contain(item => item.Id == expectedId);

// Numbers
result.Amount.Should().BePositive();
result.Quantity.Should().BeGreaterThan(0);

// Booleans
result.IsActive.Should().BeTrue();
result.IsDeleted.Should().BeFalse();

// Null checks
result.Should().NotBeNull();
result.Details.Should().BeNull();
```

### Assertion Best Practices

- Chain assertions for better readability
- Use descriptive assertion messages when needed
- Always verify API response errors first, then verify data

## Enums and Constants ##

### Enum Organization

- Enums organized in `Enums/` folders
- Project-specific enums in project's `Enums/` folder
- Shared enums in `Logistics.Shared.Api.Test/Enums/` or `SharedAPITests/Enums/`

### Enum Naming

- Use PascalCase with underscores for multi-word values
- Examples: `Bill_Journal`, `Employee_Advance`, `Good_Receipt`

### Constants Usage

- Use `SharedAPITests.Constants` for all configuration
- Common constants:
  - `Constants.language` or `TestBase.language` = `"en-us"`
  - `Constants.UserName` = `"admin"`
  - `Constants.Password` = `"admin"`
  - `Constants.{Module}TenantId` - Tenant-specific IDs
  - `Constants.{Module}ApiUrl` - API URLs

### Date Constants

Use properties from TestBase:
- `CurrentYear` - `DateTime.Now.Year`
- `NextYear` - `DateTime.Now.Year + 1`
- `PreviousYear` - `DateTime.Now.Year - 1`

## Global Usings and Namespaces ##

### GlobalUsings.cs Pattern

Create `GlobalUsings.cs` in test projects to reduce repetitive imports:

```csharp
global using ApiError = System.Collections.Generic.Dictionary<string, System.Collections.Generic.List<string>>;
global using Logistics.API.Client.Contracts;
global using SharedAPITests;
global using Logistics.Shared.Api.Test.Enums;
global using System.Threading.Tasks;
```

### Namespace Organization

- Test classes: `{Project}APITests.Tests.{Feature}`
- Services: `{Project}APITests.Services` or `Logistics.Shared.Api.Test.Services`
- Enums: `{Project}APITests.Enums` or `Logistics.Shared.Api.Test.Enums`

### Type Aliases

Use type aliases to disambiguate when same class name exists in multiple namespaces:

```csharp
using CostCenterService_CostCenterService = Logistics.Shared.Api.Test.Services.GLServices.CostCenterService.CostCenterService;
using PriceProcedureService_PriceProcedureService = Logistics.Shared.Api.Test.Services.PriceProcedureService.PriceProcedureService;
```

## Database and Test Data Management ##

### Database Reset

- **Simple projects**: `await DatabaseService.RestDatabase()`
- **Logistics projects**: Use retry logic with `seedInitialData: true`
- **With date filter**: `await DatabaseService.RestDatabase(accountingPeriodMinimumDate: new DateTime(PreviousYear, 1, 1))`

### Test Data Seeding

- Use `seedInitialData: true` parameter for Logistics projects
- Initial data includes default accounts, branches, currencies, etc.
- Database reset happens before each test in `[SetUp]`

### User Cleanup

- Track users before test: `IdentityUserBeforeCurrentTestCaseList`
- In `[TearDown]`, identify new users and delete them
- Use `_userAccessControlClient.DeleteUserAsync(userId, language)`

### Database Respawner

- Initialize once in `[OneTimeSetUp]`: `await DatabaseService.InitRespawner(tenant.ConnectionString)`
- Respawner handles database cleanup and restoration

## Integration Patterns ##

### Cross-Module Integration

- Use `await DatabaseService.MakeConstructionIntegratedWithLogistics()` for Logistics projects
- This enables Construction-Logistics integration features

### Cross-Project Service Usage

- Services from `SharedAPITests` can be used across all projects
- Examples: `BranchService`, `AccountsTreeService`, `CurrencyService`, `TreasuryService`
- Project-specific services stay in their respective projects

### Tenant-Specific Patterns

- Each test project uses its own tenant ID from `Constants`
- Tenant IDs: `Constants.LogisticsSalesTenantId`, `Constants.CashManagementTenantId`, etc.
- Never hardcode tenant IDs - always use Constants

### Multi-Tenant Test Isolation

- Each test project has isolated tenant database
- Database reset ensures test isolation
- No cross-tenant data leakage

## TFS Integration ##

### Required Package

- Install `janono.ado.testcase.associate` NuGet package

### Test Class Attributes

```csharp
[janono.ado.testcase.associate.Organization("hadafsolutions")]
[TestFixture]
public class InvoiceTest : SalesTestBase
{
    // ...
}
```

### Test Method Attributes

```csharp
[janono.ado.testcase.associate.TestCase(61007)]
[Test]
[Description("Related Test Case ==> https://hadafsolutions.visualstudio.com/AccFlex_ERP/_workitems/edit/61007")]
public async Task CreateCreditInvoiceMemo_ShouldSave()
{
    // ...
}
```

### TFS Association Process

1. Add `[Organization("hadafsolutions")]` to test class
2. Add `[TestCase(testCaseId)]` to test method
3. Build project
4. Run CLI tool to associate: `janono.ado.testcase.associate.cli.exe --authMethod PAT --authValue {pat} --path {dllPath} --action Associate --url https://hadafsolutions.visualstudio.com/`

### Work Item Links

- Include TFS work item URL in `[Description]` attribute
- Format: `"Related Test Case ==> https://hadafsolutions.visualstudio.com/AccFlex_ERP/_workitems/edit/{id}"`

## Service Factory Pattern ##

### Initialization

- Call `ServiceFactory.Init()` in TestBase constructor
- ServiceFactory sets up dependency injection container

### Getting Services

```csharp
// Token service
var tokenService = ServiceFactory.GetTokenService();

// API clients
var client = ServiceFactory.GetClient<ITenantClient>();
var odataClient = ServiceFactory.GetLgxODataClient();

// Tenant service
var tenantService = ServiceFactory.GetTenantService();
```

### Service Lifetime

- Services are scoped per request/test
- Use `using var scope = _scopeFactory.CreateScope()` pattern internally

## Test Data Creation Patterns ##

### Simple Entity Creation

```csharp
var (branch, errors) = await _branchService.CreateBranch("Branch Name", "Code", "123", "", true);
errors.ShouldHasNoError();
branch.Id.Should().BePositive();
```

### Complex Setup with Predecessor

```csharp
// Use predecessor method
var treasury = await CreateTreasuryPredecessor20238();

// Or use predecessor service
var predecessorService = new InvoicePredecessorService();
await predecessorService.Predecessor64765(predecessorData);
```

### Reusable Helper Methods

Create private methods for reusable test data:

```csharp
private async Task<TreasuryDto> CreateTreasuryPredecessor20238()
{
    var branch = await _branchService.CreateBranch(...);
    branch.errors.ShouldHasNoError();
    
    var treasury = await _treasuryService.PostTreasury(language, createDto);
    treasury.errors.ShouldHasNoError();
    
    return treasury.Item1;
}
```

## Error Response Pattern ##

### ApiError Type

```csharp
using ApiError = System.Collections.Generic.Dictionary<string, System.Collections.Generic.List<string>>;
```

### Error Structure

- Key: Field name or general error category
- Value: List of error messages for that field/category

### Error Checking

```csharp
// Check no errors
errors.ShouldHasNoError();

// Check specific error
errors.ShouldHasError("fieldName");

// Check error count
errors.Should().HaveCount(1);
errors.ShouldHasOneError("fieldName");

// Pattern matching
errors.ShouldMatchError("*required*");
```

## Best Practices ##

### Test Independence

- Each test should be independent
- Database reset in `[SetUp]` ensures isolation
- Use `[Order]` only when absolutely necessary

### Test Data Cleanup

- Clean up created users in `[TearDown]`
- Database reset handles most data cleanup
- Don't rely on test execution order

### Service Reusability

- Create predecessor services for complex, reusable setups
- Keep helper methods private when only used in one test class
- Share common services through `SharedAPITests`

### Assertion Clarity

- Use descriptive assertion messages
- Chain related assertions
- Verify errors before verifying data

### Code Organization

- Group related tests in same class
- Use folders to organize test files by feature
- Keep test classes focused on single feature/entity

---

When contributing, align changes with these **project rules**. If a change requires bending a rule, document the exception and rationale in the PR description.
