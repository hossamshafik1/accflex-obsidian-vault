---
alwaysApply: true
---
# Contributing and Architecture Guide (Project-Specific)

This document defines **Accflex ERP solution’s** architecture and the rules to follow when adding or changing code. Any generic engineering guidance lives in `core.mdc`.

## Solution Architecture ## 

- Project is implement SOA Architecture applying DDD practices bounded context, domain events, shared kernels, cqrs, aggregates, Accflex ERP consist of modules each module is service some large module contain sub-modules each sub-module is bounded context:
  - `AccessControl` manage users ,authentication & login using Identity server 4
  - `AnalyticsReader` is for reporting via odata endpoints
  - `CashControl` manage treasuries and Banks business
  - `Construction` manage Construction business project, progress.
  - `GeneralLedger` accounting entries and financial stuff.
  - `Logistics` has some sub-modules Inventory, sales, procurement and production.
  - `Misc` contain some Miscellaneous stuff.
  -  
- each Module/sub-module contains features represent ui-screen like invoice, customer etc..
- Features in ERP has one of these types
  - Master Data – relatively static business entities like (Customer, Vendor, Material).
    - has CRUD operations.
    - has search functionality
    - has Import functionality
    - No print functionality
  - Transactional Data – day-to-day documents like (Quotation, Sales Order, Invoice).
    - has CRUD operations
    - creation has special flow with search to choose the referencing document
    - has search functionality
    - has print functionality  with multiple layout
  - Configuration Data – how the system works (rules, settings).
    - Opens directly with default data and ability to save it.
    - No Search,No delete or print functionality
  - Organizational Data – structure of the enterprise (Company Code, Plant, Sales Org).
    - No delete or print functionality
    - Search form is a component displayed always on side of edit form
  - Condition/Reference Data – flexible business rules (pricing, discounts).
    - has CRUD operations
    - No print functionality
    - No Delete functionality replaced with active period.
  - Analytical Data – Reporting of historical/derived data.
    - has query form inputs
    - has DevExtreme grid for result
    - has print functionality with multiple layout
	
## Stack and Modules ##

- **Runtime**: .NET 8, EF Core 8
- **Solution layout**: one `API` project, one `Application` project, and per bounded-context `*.Logic` (domain) and `*.Data` (EF) projects
- **Bounded contexts**: is for sub-modules like`Inventory`, `Sales`, `Purchase`, `Production`, `Quality` and each of it should have separate logic project and separate Data project with it's entity framework dbcontext

### Dependency and Layering Rules

- **Allowed**:
  - `API -> Application`
  - `Data -> Application`
  - `Application -> Logic`
- **Forbidden**:
  - Logic layer do not depend on any other layer
  - Cross-`*.Logic` references between bounded contexts (use `Shared` types or orchestrate in `Application`)
  - `API` accessing `*.Data` directly

### API Layer

- thin layer for registering dependencies and controllers call MediatR commands/queries in application layer only
- controllers reside in folder named 'ApiControllers'
- to perform authorization in policy filter each controller has identifying static readonly int field `ModuleId` from Enum SharedKernel.Logic.Modules
- Return **ProblemDetails** on exceptions (stable error shape for clients)
- rout attribute used per action.
- each controller action should have Microsoft.AspNetCore.Mvc.ProducesResponseTypeAttribute attributes identify all possible returns:
  1. [ProducesDefaultResponseType] attribute.
  2. [ProducesResponseType(typeof(Dto), StatusCodes.Status200OK)]
  3. [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
- RESTful resources, stable IDs, ProblemDetails for errors.
- each controller action should has  [Authorize(Policy = "")] related to the crud function of the action from constants at  SharedKernel.Logic.Helpers.SharedConstants.
  - ViewPolicy
  - PrintPolicy
  - EditPolicy
  - DeletePolicy
  - AddPolicy  
- Use deprecation headers and timelines; **do not break contracts** within a version
  
### Application Layer (CQRS with MediatR)

- Perform input validation here
- Gather state, then call `*.Logic` for business rules
- Wrap multi-context work in `IApplicationTransaction` (one transaction per command)
- has interfaces for dbcontext and use it to application layer

### Domain Logic (`*.Logic`)

- Aggregates per feature with a single aggregate root
- Encapsulate invariants; emit domain events to express side-effects
- No EF/HTTP/infra dependencies
- Never throw exception for business validation rather use Result object to caller

### Data Access (`*.Data`)

- Multi-tenant Support
	- Implements IUnitOfWork and it's interface in `Application` interfaces
	- Uses tenant-specific connection strings via ITenantProvider
	- Supports transaction management with DbTransaction
- Domain-Driven Design
	- Implements domain event handling with pre/post event dispatching
	- Uses IDomainEventDispatcher for event publishing
	- Supports workflow events and approval events
	- Implements audit trail functionality
- Environment-Specific Configuration
	- Development: Single query behavior, sensitive data logging
	- Production: Split query behavior for performance
	- Uses SQL Server with EF Core 8
- Model Configuration
	- Auto-applies configurations from assembly
	- Defines sequences (e.g., Lgx_ClassificationSequenceNumber)
	- Maps database views to entities
	- Configures keyless entities for DTOs and views
- Query guidance:
  - Use `AsNoTracking` for reads and project to DTOs
  - Always paginate list queries (see defaults below)

### Transactions and Consistency

- Prefer single-aggregate transactions
- For cross-BC operations, use `IApplicationTransaction` and domain/integration events
- Handlers must be idempotent; persist idempotency keys for background jobs

### Domain and Integration Events

- Raise domain events in-process from aggregates
- Promote selected events to integration events via **Hangfire + MediatR** (fire-and-forget)
- Enqueue jobs with **tenant ID, correlation ID, and idempotency key**; enable retries with backoff

### Authentication and Authorization

- **JWT via IdentityServer4**; extract user and tenant from the token
- Policy-based authorization per ERP feature in `API`
- ERP feature controllers must inherit the base API controller and implement `IHaveModulePermission`

### Multi-tenancy

- Resolve tenant per request; set the connection string before `DbContext` use
- Scope `DbContext`s per request; never reuse across tenants
- Include tenant context in logs and background jobs

### Database and Mapping Conventions

- **Single database per tenant**; schema managed by a SQL project
- Within same aggregate (Header/Detail):
  - Header INT PK; Detail FK to Header; cascade delete
  - Entities: `Header` contains `ICollection<Detail>`; `Detail` has `HeaderId`
- Static references (Orgs, Materials, Units, …):
  - INT FK by ID; no navigation properties in entities
- Cross-aggregate relations:
  - Use **GUID** columns without FK constraints
  - Required columns: Header `XXXXXGUIDNumber`, Detail `XXXXXDetailGUIDNumber`
  - If a cross-aggregate navigation is added, populate it explicitly in commands
- Prefer optimistic concurrency (e.g., `rowversion`) where appropriate
- Entity's Id property map to column 'EntityId' like 'Invoice' Entity has property 'Id' map  to db Column 'InvoiceId'
- Many to many junctions are inherit ValueObject abstract class in CSharpFunctionalExtensions library with two properties references the related entities doesn't require audit infos properties or reference name property or Id PK and contain this implementation for GetEqualityComponents method

```C#
protected override IEnumerable<object> GetEqualityComponents()
{
    yield return firstEntityPK;
    yield return secondEntityPK;
}
```

### Client Generation

- **NSwag** (`nswag.json`) is the single source for .NET and TS clients
- Regenerate clients in CI on API changes and publish to the private package host
- Do not hand-edit generated clients; use semantic versioning

### Validation and Localization

- Application: input validation (types, ranges)
- Domain logic: business validations requiring state (e.g., stock checks)
- All messages are localized via `Common.Logic/Resources` (AR/EN); cultures configured in `API`

### Logging and Observability

- **Serilog + ELK**; structured logs everywhere
- Include **correlation ID** and **tenant** in logs and job payloads
- Log once per concern per layer; avoid noisy logs in loops

### Performance Rules

- Always paginate list endpoints
- Avoid N+1 via explicit loading or projections; index frequently filtered columns
- Review EF-generated SQL for heavy endpoints

### Error Handling

- Map exceptions to **ProblemDetails** at the `API` boundary with a stable error code and `traceId`
- never swallow exceptions

### Testing Strategy (This Solution)

- Unit: `*.Logic` (pure business rules)
- Integration: `Application` with mocked/test data or test DB as needed
- E2E: Running `API` against real DB; deterministic seeds and isolated data per run

### SignalR

- Used in **Access Control API**
- Authenticate connections; authorize per hub/group
- Version hub contracts; use DTOs (no domain entities)

### CI/CD (Azure DevOps)

- Pipeline: Restore → Build (Release) → Unit/Integration tests → Generate NSwag clients → Publish client packages → Publish API artifacts → Deploy
- Validate SQL project schema changes; block on drift
- Enforce API breaking-change checks once versioning is enabled

### Defaults and Operational Constants (Project-specific)

- Paging: default page size **10**; max **10000**
- Background job retries: **3** with exponential backoff
- Correlation header: `X-Correlation-ID`
- Idempotency: persist keys for integration handlers and jobs
- Enums are implemented using static fields in class like 
```C#
public class Role
{
    public static Role Author {get;} = new Role(1, "Author","كاتب");
    public static Role Editor {get;} = new Role(2, "Editor","محرر");
    public static Role Administrator {get;} = new Role(3, "Administrator","مدير");
    
 
    public string NameSecondLanguage { get; private set; }
    public string Name { get; private set; }
    public int Value { get; private set; }
 
    private Role(int val, string name, string nameSecondLanguage) 
    {
        Value = val;
        Name = name;
        NameSecondLanguage = nameSecondLanguage;
    }
 
    public static IEnumerable<Role> List()
    {
        // alternately, use a dictionary keyed by value
        return new[]{Author,Editor,Administrator};
    }
 
}
```
---

When contributing, align changes with these **project rules**. If a change requires bending a rule, document the exception and rationale in the PR description.
