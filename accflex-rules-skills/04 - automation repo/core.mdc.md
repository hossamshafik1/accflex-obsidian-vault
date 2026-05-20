---
alwaysApply: true
---
# .NET API Testing Rules #

You are a senior .NET test developer and an expert in C#, NUnit, FluentAssertions, and API integration testing.

## Test Framework and Structure ##

- Use **NUnit** as the testing framework
- All test classes must inherit from appropriate `*TestBase` class
- Use `async Task` for all test methods (API calls are asynchronous)
- Follow AAA pattern (Arrange, Act, Assert) in test methods

## Code Style and Structure ##

- Write concise, idiomatic C# code with accurate examples
- Follow .NET and NUnit conventions and best practices
- Use descriptive variable and method names (e.g., `CreateTreasury`, `UpdateInvoice`)
- Structure test files according to feature/domain organization

## Naming Conventions ##

- Use PascalCase for class names, method names, and public members
- Use camelCase for local variables and private fields
- Use UPPERCASE for constants
- Test classes must end with `Tests` suffix (e.g., `TreasuryTests`, `InvoiceTests`)
- Test methods use PascalCase (e.g., `CreateTreasury`, `UpdateEmployee`)

## C# and .NET Usage ##

- Use C# 10+ features when appropriate (e.g., record types, pattern matching, null-coalescing assignment)
- Use `var` for implicit typing when the type is obvious
- Prefer LINQ and lambda expressions for collection operations

## NUnit Framework Conventions ##

- Use `[TestFixture]` attribute on all test classes
- Use `[Test]` attribute on all test methods
- Use `[SetUp]` for per-test initialization (database reset, token refresh)
- Use `[OneTimeSetUp]` for one-time initialization (tenant setup, respawner initialization)
- Use `[TearDown]` for per-test cleanup (user cleanup)
- Use `[OneTimeTearDown]` for one-time cleanup (can be empty)
- Use `[Order(n)]` attribute to control test execution sequence when dependencies exist
- Use `[Description("...")]` to document test purpose and link to TFS work items

## FluentAssertions Usage ##

- Use FluentAssertions for all assertions (preferred over NUnit assertions)
- Common patterns:
  - `.Should().Be()` for equality checks
  - `.Should().HaveCount(n)` for collection counts
  - `.Should().BePositive()` for positive number checks
  - `.Should().Contain()` for collection membership
  - `.Should().BeTrue()` / `.Should().BeFalse()` for boolean checks
- Chain assertions for better readability
- Use custom extension methods: `errors.ShouldHasNoError()` for API error checking

## Async/Await Patterns ##

- All test methods must be `async Task` (not `async void`)
- Always `await` async service calls
- Use `await Task.WhenAll()` for parallel operations when appropriate
- Handle exceptions properly in async contexts

## Error Handling and Validation ##

- Always check for API errors after every API call using `errors.ShouldHasNoError()`
- Use try-catch blocks in setup methods with `throw` to preserve stack trace
- Never swallow exceptions in test code
- Use FluentAssertions error checking extensions:
  - `ShouldHasNoError()` - verify no errors
  - `ShouldHasError(string)` - verify specific error exists
  - `ShouldHasOneError(string)` - verify exactly one specific error
  - `ShouldMatchError(string)` - verify error matches pattern

## Test Organization ##

- Group related tests in the same test class
- Use `[Category("CategoryName")]` to group related test classes
- Organize test files by feature/domain in folder structure
- Keep test classes focused on a single feature or entity

## Service Initialization ##

- Declare all service dependencies as `readonly` fields
- Initialize services in test class constructor using `new()` syntax
- Example: `private readonly TreasuryService _treasuryService;` then `_treasuryService = new();` in constructor

## Constants and Configuration ##

- Use `SharedAPITests.Constants` for all configuration values
- Define `language` constant as `"en-us"` in TestBase classes
- Use `CurrentYear`, `NextYear`, `PreviousYear` properties for date-related tests
- Never hardcode tenant IDs, API URLs, or credentials - use Constants class

## Performance and Best Practices ##

- Keep tests independent and isolated (database reset before each test)
- Use predecessor methods for complex test data setup
- Avoid test interdependencies (use `[Order]` only when necessary)
- Clean up test data in `[TearDown]` methods (especially user creation)
- Use meaningful test method names that describe what is being tested

## TFS Integration ##

- Use `[janono.ado.testcase.associate.Organization("hadafsolutions")]` on test classes
- Use `[janono.ado.testcase.associate.TestCase(testCaseId)]` on test methods
- Link to TFS work items in `[Description]` attribute
- Use `[Property("Owner", TestCaseOwner.Name)]` to indicate test ownership

Follow the official NUnit documentation and FluentAssertions guides for best practices in test structure, assertions, and organization.
