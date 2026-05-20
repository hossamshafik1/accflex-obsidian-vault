---
description: Use when implementing or editing feature Data Services: the data layer between Angular components and API clients. Covers API clients, SubSink, reactive loading from static resources, local caching, and CRUD patterns. Apply in feature modules that use *-data-service or *data.service.
globs:
  - "**/*-data-service*.ts"
  - "**/*data.service.ts"
alwaysApply: false
---
# Overview #

This service act as data layer intermediaries between Angular components and API clients, managing data retrieval, caching, and state management for feature operations.

## Common Patterns & Functionality ##

### 1. Service Architecture ###

#### Dependency Injection ####

service is decorated with @Injectable() and inject the following types of dependencies:

- API Clients: Generated client classes for HTTP operations (i.e., FeatureClient)
- Static Resource Service: LogisticsStaticResourcesService for shared reference data, MUST check before make direct call to get data.
- Utility Services: Error handling and other cross-cutting concerns

Constructor Pattern

```typescript
constructor(
  private apiClient: Client,
  private staticResource: LogisticsStaticResourcesService,
  // other dependencies...
)
```

### 2. Subscription Management ###

#### SubSink Pattern ####

- Uses SubSink library for automatic subscription cleanup
- Prevents memory leaks by managing observable subscriptions lifecycle
- Pattern: sub = new SubSink() at class level
- Subscriptions added via: this.sub.sink = observable.subscribe(...)

### 3. Reactive Data Loading ###

#### Initial Data Loading Strategy ####

Service implement a sophisticated data loading mechanism in their constructors:
Step 1: Watch for Refresh Triggers

```typescript
const updatesSubList: BehaviorSubject<void>[] = [];
updatesSubList.push(this.staticResource.refreshMaterialSubject);
updatesSubList.push(this.staticResource.refreshInventoryLocationSubject);
// ... other refresh subjects

combineLatest(updatesSubList).subscribe(() => { /* reload data */ })
```

Step 2: Combine Multiple Data Sources

```typescript
combineLatest(this.getSubsDataSource()).subscribe({
  next: (res) => {
    // Assign results to service properties
    this.materialList = res[0];
    this.inventoryLocationList = res[1];
    // ...
  },
  error: (error) => {
    // Handle errors
  }
});
```

Step 3: Private Data Source Aggregation

```typescript
private getSubsDataSource(): Observable<any>[] {
  const subList: Observable<any>[] = [];
  subList.push(this.staticResource.MaterialWithOrganizationList);
  subList.push(this.staticResource.InventoryLocationList);
  // ... other observables
  return subList;
}
```

### 4. Local Data Caching ###

#### In-Memory Cache Properties ####

Service maintain local caches of frequently accessed reference data:

```typescript
materialList: MaterialWithOrganizationsDto[] = [];
inventoryLocationList: InventoryLocationListDto[] = [];
//... other cached data
```

### 5. CRUD Operation Patterns ###

#### Create Operations ####

Accept DTO objects as parameters
Instantiate command objects
Initialize command with DTO data
Return Observable from API client

```typescript
createEntity = (dto: EntityDto): Observable<EntityDto> => {
  const command = new CreateEntityCommand();
  command.init(dto);
  return this.client.createEntity(command);
};
```

#### Update Operations ####

Similar pattern to create operations
Use update-specific command objects
May include ID parameters

```typescript
updateEntity = (dto: EntityDto): Observable<EntityDto> => {
  const command = new UpdateEntityCommand();
  command.init(dto);
  return this.client.updateEntity(command);
};
```

#### Delete Operations ####

```typescript
deleteEntity = (deleteCommand: DeleteCommand): Observable<void> => {
  return this.client.deleteEntity(deleteCommand);
};
```

#### Read Operations ####

```typescript
getById = (id: number): Observable<EntityDto> => {
  return this.client.getEntityById(id);
};
```