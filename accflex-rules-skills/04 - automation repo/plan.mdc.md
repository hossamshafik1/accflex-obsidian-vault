---
description: Rules for creating test development plans
globs:
alwaysApply: false
---
# Create Test Development Plan

**Core Task:** Generate multi-commit task plan for test implementation with comprehensive test data setup, predecessor identification, and verification steps.

## Setup

1. **Get timestamp:** Use current date format `YYYY-MM-DD` for plan filename
2. **Check files:** Verify project structure and existing test patterns
3. **Review TFS work item:** Understand test case requirements and acceptance criteria

## Information Gathering (MANDATORY)

**MUST read all relevant files before planning:**
- TFS work item/test case description
- Related API endpoint documentation
- Existing similar test implementations
- Predecessor test cases (if test has dependencies)
- Service layer documentation
- Enum and constant definitions

## Output Rules

- **ONLY output:** Complete Markdown content for the task plan file
- **NO conversation:** No preamble, commentary, or summaries
- **Missing info:** Use `<!-- TODO: [specify what's missing] -->` in file

## Required for Each Commit

### Test Data Setup (Priority Order)
1. **Identify predecessors** - What entities must exist before this test?
2. **Create predecessor methods** - Reusable helper methods for test data
3. **Service initialization** - Required services for the test
4. **Test data creation** - Entities needed for the test scenario

### Predecessor Identification

For each test, identify:
- **Direct dependencies**: Entities that must be created first
  - Example: Invoice test needs Customer, SalesOrder, Material
- **Indirect dependencies**: Entities needed by direct dependencies
  - Example: Customer needs Branch, Account
- **System configuration**: Settings that must be configured
  - Example: System configuration, permissions, licenses

### Predecessor Method Patterns

**Naming:**
- `Create{Entity}Predecessor{TestCaseId}()` - Returns created entity
- `Predecessor{TestCaseId}()` - Sets up data without return value

**Structure:**
```csharp
private async Task<EntityDto> CreateEntityPredecessor12345()
{
    // Step 1: Create dependency A
    var (dependencyA, errorsA) = await _serviceA.Create(...);
    errorsA.ShouldHasNoError();
    
    // Step 2: Create dependency B
    var (dependencyB, errorsB) = await _serviceB.Create(...);
    errorsB.ShouldHasNoError();
    
    // Step 3: Create main entity
    var (entity, errors) = await _service.Create(...);
    errors.ShouldHasNoError();
    
    return entity;
}
```

### Testing (Priority Order)
1. **Test method implementation** - Main test logic
2. **Assertions** - Verify expected behavior
3. **Error handling** - Verify error scenarios (if applicable)
4. **Edge cases** - Boundary conditions (if applicable)

### Verification Format

For each commit specify:
- **Test Command:** Exact command (e.g., `dotnet test --filter InvoiceTest`)
- **Expected Outcome:** Key assertions/behavior
- **Predecessor Verification:** How to verify test data setup
- **Error Scenarios:** How to verify error handling (if applicable)

## File Structure

```markdown
# Test: [Brief Title] - Test Case [ID]

## Overview
- **Test Case ID:** [TFS Test Case ID]
- **Work Item:** [TFS Work Item URL]
- **Feature:** [Feature name]
- **Module:** [Module name]

## Dependencies
- **Predecessor Test Cases:** [List of test case IDs that must run first]
- **Required Entities:** [List of entities that must exist]
- **System Configuration:** [Required system settings]

## Commit 1: [type: description]
**Description:**
[Specific details: file paths, method names, service initialization, predecessor method creation]

**Predecessor Setup:**
- Create Entity A (using ServiceA)
- Create Entity B (using ServiceB)
- Configure System Settings

**Verification:**
1. **Predecessor Method:**
   * **Method:** `CreateEntityPredecessor12345()`
   * **Expected Outcome:** Returns EntityDto with Id > 0, all dependencies created
2. **Automated Test:**
   * **Command:** `dotnet test --filter TestClass.CreateEntityPredecessor12345`
   * **Expected Outcome:** Test passes, entity created successfully

---

## Commit 2: [type: description]
**Description:**
[Test method implementation, assertions, error checking]

**Test Implementation:**
- Test method: `CreateEntity_ShouldSave()`
- Use predecessor method to setup data
- Call service method
- Verify response

**Verification:**
1. **Automated Test:**
   * **Command:** `dotnet test --filter TestClass.CreateEntity_ShouldSave`
   * **Expected Outcome:** 
     - `errors.ShouldHasNoError()` passes
     - `result.Id.Should().BePositive()` passes
     - `result.Name.Should().Be("Expected")` passes
2. **Error Checking:**
   * **Action:** Verify `errors.ShouldHasNoError()` after API call
   * **Expected:** No errors in response

---

## Commit 3: [type: description]
[Additional test scenarios, edge cases, or cleanup]

**Description:**
[Additional test methods or scenarios]

**Verification:**
[Test commands and expected outcomes]
```

## Semantic Commit Types
Use: `test:`, `feat:`, `fix:`, `docs:`, `chore:`

Examples:
- `test: add CreateInvoice test with predecessor setup`
- `test: add UpdateInvoice test with validation`
- `test: add DeleteInvoice test with error scenarios`

## Predecessor Service Creation

When predecessor logic is complex or reusable across multiple tests:

**Commit Structure:**
1. Create predecessor service class
2. Initialize required services in constructor
3. Implement predecessor method(s)
4. Add unit tests for predecessor service (if needed)

**Example:**
```markdown
## Commit 1: test: create InvoicePredecessorService
**Description:**
- Create `Logistics.Shared.Api.Test/Services/InvoicePredecessorService.cs`
- Initialize all required services in constructor
- Implement `Predecessor64765(Predecessor_81340Data data)` method
- Method creates Customer, SalesOrder, Material, and related entities

**Verification:**
- Service class compiles
- All required services initialized
- Predecessor method signature matches pattern
```

## Test Method Planning

For each test method, plan:
1. **Test data setup** - What entities are needed?
2. **Service calls** - What API methods will be called?
3. **Assertions** - What should be verified?
4. **Error scenarios** - What error cases should be tested?

## Multi-Test Planning

When planning multiple related tests:
- Identify common predecessor logic
- Create reusable predecessor methods/services
- Plan test execution order using `[Order]` attribute
- Group related tests in same test class

## After Creation

1. Output ONLY the task plan file content
2. Provide brief executive summary of planned commits
3. Wait for user feedback
