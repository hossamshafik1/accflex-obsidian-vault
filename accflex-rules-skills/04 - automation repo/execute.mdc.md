---
description: Rules for executing test implementation
globs:
alwaysApply: false
---
# Execute Test Implementation Plan

**Goal:** Execute commits from test development plan systematically, ensuring all verification steps pass before proceeding.

## Setup

Check plan file exists in `.cursor/plans/` directory.
If missing, stop and notify user.

## Execution Cycle

For each commit in plan file:

1. **Implement:** Execute commit instructions exactly as written
2. **Verify:** Run ALL verification steps specified for this commit
3. **Commit:** Stage changes and GIT commit with exact message from plan file
4. **Continue:** Immediately proceed to next commit (no feedback requests)

## Implementation Rules

### Service Initialization

- Declare services as `readonly` fields
- Initialize in constructor: `_service = new ServiceName();`
- Follow existing patterns in the codebase

### Test Method Structure

```csharp
[Order(n)]
[Test]
[Description("Related Test Case ==> {TFS_URL}")]
[janono.ado.testcase.associate.TestCase(testCaseId)]
public async Task TestMethodName()
{
    // Arrange: Setup test data (use predecessor if needed)
    var entity = await CreateEntityPredecessor12345();
    
    // Act: Call service method
    var (result, errors) = await _service.PostEntity(language, command);
    
    // Assert: Verify results
    errors.ShouldHasNoError();
    result.Id.Should().BePositive();
    result.Name.Should().Be("Expected");
}
```

### Error Checking

**Always check for errors after API calls:**
```csharp
var (result, errors) = await _service.PostEntity(language, command);
errors.ShouldHasNoError(); // REQUIRED - never skip this
```

### Assertions

- Use FluentAssertions for all assertions
- Chain related assertions
- Verify errors before verifying data
- Use descriptive assertion messages when needed

### Predecessor Methods

**Private helper method pattern:**
```csharp
private async Task<EntityDto> CreateEntityPredecessor12345()
{
    // Create dependencies
    var (dependency, errors) = await _dependencyService.Create(...);
    errors.ShouldHasNoError();
    
    // Create main entity
    var (entity, entityErrors) = await _service.Create(...);
    entityErrors.ShouldHasNoError();
    
    return entity;
}
```

**Predecessor service pattern:**
```csharp
// In test class constructor
private readonly EntityPredecessorService _predecessorService;

// In constructor
_predecessorService = new EntityPredecessorService();

// In test method
await _predecessorService.Predecessor12345(predecessorData);
```

## Verification Steps

### Before Each Commit

1. **Compilation:** Ensure code compiles without errors
2. **Linting:** Fix any linting errors
3. **Test Execution:** Run specific test(s) for this commit

### Test Execution Commands

```bash
# Run specific test class
dotnet test --filter TestClassName

# Run specific test method
dotnet test --filter TestClassName.TestMethodName

# Run all tests in project
dotnet test

# Run with verbose output
dotnet test --logger "console;verbosity=detailed"
```

### Expected Verification Outcomes

For each commit, verify:
- **Compilation:** No build errors
- **Test Execution:** All tests pass
- **Assertions:** All FluentAssertions pass
- **Error Checking:** `errors.ShouldHasNoError()` passes
- **Data Verification:** All data assertions pass

### Error Scenarios

If verification fails:
1. **Stop execution**
2. **Fix the issue**
3. **Re-run verification**
4. **Continue only after all verifications pass**

## Commit Patterns

### Commit Message Format

Use exact message from plan file:
```
test: add CreateInvoice test with predecessor setup
```

### Staging Changes

```bash
# Stage all changes
git add .

# Or stage specific files
git add Tests/Invoice/InvoiceTest.cs
git add Services/InvoicePredecessorService.cs
```

### Commit Command

```bash
git commit -m "test: add CreateInvoice test with predecessor setup"
```

## Task Completion Criteria

Task is complete when:
- All commits from plan file created successfully
- All automated tests pass
- All verification steps completed successfully
- All test methods have proper attributes
- All TFS test case associations are in place

## Output Rules

- **NO chatter** between commits
- **Output only:**
  - Command results (if any)
  - File change summaries
  - Test execution results
- **After completion:** Summary of implemented tests

**Execute → Verify → Commit → Proceed. No conversation until task complete.**

## Common Patterns

### Creating Test Class

1. Create test class file: `Tests/{Feature}/{Feature}Tests.cs`
2. Add required attributes: `[TestFixture]`, `[Organization("hadafsolutions")]`
3. Inherit from appropriate TestBase
4. Initialize services in constructor
5. Add test methods with proper attributes

### Creating Predecessor Service

1. Create service file: `Services/{Entity}PredecessorService.cs`
2. Initialize required services in constructor
3. Implement predecessor method(s)
4. Use in test class via field initialization

### Adding Test Method

1. Add `[Test]` attribute
2. Add `[TestCase(testCaseId)]` attribute
3. Add `[Description]` with TFS URL
4. Add `[Order(n)]` if test has dependencies
5. Implement test logic with proper error checking

## Anti-Patterns to Avoid

- **Don't skip error checking** - Always call `errors.ShouldHasNoError()`
- **Don't hardcode values** - Use Constants class
- **Don't create test dependencies** - Use `[Order]` only when necessary
- **Don't skip verification** - Always run tests before committing
- **Don't commit broken tests** - Fix issues before proceeding
