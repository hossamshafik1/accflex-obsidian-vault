---
alwaysApply: true
description: Overview and index of all AccFlex ERP API Testing Cursor Rules
---
# AccFlex ERP API Testing Cursor Rules Overview

## Project Folder structure ##

AccFlex ERP API Testing Consist of multiple test projects organized by module:

- **Logistics Tests**: Multiple sub-projects for different Logistics bounded contexts
  - `Logistics.Inventory.Api.Test` - Inventory management tests
  - `Logistics.Sales.Api.Test` - Sales module tests
  - `Logistics.Purchase.Api.Test` - Purchase module tests
  - `Logistics.Production.Api.Test` - Production module tests
  - `Logistics.Maintenance.Api.Test` - Maintenance module tests
  - `Logistics.Shared.Api.Test` - Shared services, enums, and utilities
  - **[Project rules](mdc:api-test-core.mdc)** - project specific rules
  - **[General Testing rules](mdc:core.mdc)** - General .NET/NUnit rules
- **CashControl Tests**: `CashControlAPITests` - Treasury and cash management tests
- **Construction Tests**: `ConstructionAPITests` - Construction project tests
- **GeneralLedger Tests**: `GenaralLedgerAPITests` - Accounting and financial tests
- **Payroll Tests**: `PayrollAPITests` - Payroll and HR tests
- **Misc Tests**: `MiscAPITests` - Miscellaneous feature tests
- **PropertyManager Tests**: `PropertyManagerAPITests` - Property management tests
- **Shared Tests**: `SharedAPITests` - Common utilities, extensions, and constants

## Related Projects ##

- **Backend APIs**: in folder `../../../../AccFlex_Cloud_ERP`
  - **[Backend Project rules](mdc:../../../../AccFlex_Cloud_ERP/.cursor/rules/accflex-erp-core.mdc)** - Backend API architecture rules
  - **[Backend General Coding rules](mdc:../../../../AccFlex_Cloud_ERP/.cursor/rules/core.mdc)** - General C# rules

## Rules Files ##

- **[General Testing Rules](mdc:core.mdc)** - General .NET, NUnit, and FluentAssertions conventions
- **[API Test Project Rules](mdc:api-test-core.mdc)** - Project-specific API testing patterns and conventions
- **[Planning Rules](mdc:plan.mdc)** - Rules for creating test development plans
- **[Execution Rules](mdc:execute.mdc)** - Rules for executing test implementation
