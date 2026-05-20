## Context

Field workers at concrete/ready-mix plants operate in deserts with poor connectivity. They need to create delivery tracking records (trucks, pumps, materials) while offline and sync them manually when back online. The existing `Monitoring Concrete Products` module only works online via `DeliveryPlanFromSalesOrder`. This plan adds a parallel offline lifecycle that syncs to a new `DeliveryPlanFromOffline` (SourceType=4) and becomes linkable to sales orders only after sync succeeds.

**Phase 1 scope**: Create-only offline. No offline editing of server records. No offline sales-order usage. No conflict resolution. **Sales order linkage deferred to Phase 2.**

**Offline form required fields**: Truck (Car), Driver, Concrete Grade (Material), Quantity, Production Date. Production Time is optional.

---

## Technical Decisions

### D1: Local Database -- Dexie.js v4

**Choice**: Dexie.js over raw IndexedDB or `idb`.

- First-class TypeScript generics on table definitions
- Built-in versioned schema migrations (handles app updates while user has unsynced data)
- Promise-based API wraps cleanly with RxJS `from()`
- ~28KB gzipped, actively maintained
- `idb` is too low-level (no migration support); raw IndexedDB is boilerplate-heavy

### D2: Server Storage -- Same Tables, New SourceType (4)

**Choice**: Insert synced records into `Lgx_DeliveryPlan` + `Lgx_DeliveryPlanDetail` with `DeliveryPlanSourceTypeId = 4`.

- Follows the existing abstract-base pattern: `OnSalesOrder(1)`, `OnProject(2)`, `OnCostCenter(3)` -> add `OnOfflineEntry(4)`
- Avoids duplicating the 90-column DeliveryPlanDetail schema into a staging table
- Existing queries filter by SourceTypeId, so offline records are automatically excluded from online flows
- Linkage deferred to Phase 2 -- offline records stay as SourceType=4 until then

### D3: Client ID Strategy -- Reuse Existing `GuidNumber` Column

**Choice**: Use the existing `GuidNumber` (NVARCHAR(50), DEFAULT newid()) as the idempotency key.

- Column already exists on `DeliveryPlanBase` and `DeliveryPlanDetailBase`
- The codebase already generates client-side GUIDs (`angular2-uuid` installed in frontend, `Guid.NewGuid()` in backend)
- No schema migration needed -- just populate `GuidNumber` with the client-generated UUID before sync

### D4: Data Cleanup Strategy

|Trigger|Action|
|---|---|
|Record synced successfully|Retain in IndexedDB for 7 days, then auto-purge on next app open|
|User logs out|Show warning with pending count, do NOT delete. Records keyed by `userId + tenantId`|
|Manual clear|User can "Clear Synced Records" from offline tab|

### D5: Service Worker Scope -- Root Shell Registration

- `@angular/service-worker` supports only one SW per origin. Register at root `accflex-erp/src/`.
- `ngsw-config.json` caches: shell, logistics lazy chunk, DevExtreme CSS, i18n JSON
- The SW does NOT intercept API calls. All offline data flows through Dexie/IndexedDB at the application layer.
- SW purpose: make app installable (manifest), prevent browser error page when offline

### D6: API Design -- Single Batch Endpoint

**Endpoint**: `POST api/MonitoringConcreteProducts/SyncOfflineTrucks`

- Accepts `List<OfflineTruckSyncItem>` (batch)
- Returns `List<SyncItemResult>` with per-row `clientGuidNumber`, `success`, `serverDeliveryPlanDetailId`, `errorMessage`
- Each item processed independently -- partial success allowed
- Idempotent: if `GuidNumber` already exists server-side, return success with existing ID

### D7: Domain Entity -- New `DeliveryPlanFromOffline` Subclass

- Create `DeliveryPlanFromOffline` + `DeliveryPlanDetailFromOffline` following the OnSalesOrder/OnProject/OnCostCenter pattern
- Minimal required fields: CarId, DriverId, MaterialId, Quantity, UnitId, DeliveryPlanDate, ProductionTime (optional)
- No SalesOrderDetailId, no ProjectId, no CostCenterId
- Looser validation (inventory/sales order checks deferred to Phase 2 linkage)
- Factory method `CreateDeliveryPlan(OfflineTruckPoco poco)` on the aggregate

### D8: Offline Indicator Placement

- **Monitoring screen only** -- small banner within the Monitoring Concrete Products component
- Not app-shell level for Phase 1 (minimizes cross-module impact)

### D9: NSwag Client Regeneration

- **Manual workflow**: Developer runs `nswag run` locally after backend endpoint is created, commits generated client, publishes npm package
- Step 6 depends on this being done after Step 5

---

## Implementation Steps

### Step 1: PWA Foundation & Connectivity Awareness

**Goal**: Make the app installable as PWA and expose `isOnline$` observable app-wide.

**Files to create**:

- `AccFlex_Cloud_Front/accflex-erp/src/manifest.webmanifest` -- PWA manifest
- `AccFlex_Cloud_Front/accflex-erp/src/ngsw-config.json` -- Angular SW config
- `AccFlex_Cloud_Front/accflex-erp/src/app/core/services/pwa/network-status.service.ts` -- `isOnline$: BehaviorSubject<boolean>`
- `AccFlex_Cloud_Front/accflex-erp/src/app/core/services/pwa/pwa-update.service.ts` -- SW version update handler

**Files to modify**:

- `AccFlex_Cloud_Front/accflex-erp/angular.json` -- add `"serviceWorker": true`, manifest to assets
- `AccFlex_Cloud_Front/accflex-erp/src/index.html` -- `<link rel="manifest">`
- `AccFlex_Cloud_Front/accflex-erp/package.json` -- add `@angular/service-worker`

**Technical approach**:

- `ng add @angular/service-worker` at workspace root
- `NetworkStatusService`: wraps `window.addEventListener('online'/'offline')` + `navigator.onLine` initial value in a `BehaviorSubject<boolean>`. Injectable globally via `providedIn: 'root'`.
- `ngsw-config.json`: cache `index.html`, main/polyfills/runtime bundles, logistics lazy chunk, `assets/i18n/**/*.json`, DevExtreme CSS. Use `freshness` strategy for API calls (don't cache them).

**Test before moving on**:

- `ng build logistics --configuration production` produces `ngsw-worker.js` in dist
- Open in Chrome > Application tab: service worker registered, manifest detected
- Toggle DevTools offline: app shell still loads from cache
- Inject `NetworkStatusService`, verify `isOnline$` emits `false` when offline, `true` when online

---

### Step 2: IndexedDB Schema & Dexie Setup

**Goal**: Create the local database layer with typed CRUD operations.

**Install**: `npm install dexie@^4`

**Files to create** (all under `projects/logistics/src/app/modules/monitoring-concrete-products/offline/`):

- `models/offline-sync-status.enum.ts`
- `models/offline-truck-record.ts`
- `offline-db.ts` -- Dexie database class
- `offline-db.service.ts` -- Angular injectable wrapping Dexie CRUD

**Key types**:

typescript

```typescript
// offline-sync-status.enum.ts
export enum OfflineSyncStatus {
  PendingSync = 'PendingSync',
  Syncing = 'Syncing',
  Synced = 'Synced',
  SyncFailed = 'SyncFailed'
  // Linked = 'Linked' -- Phase 2
}

// offline-truck-record.ts
export interface OfflineTruckRecord {
  clientGuidNumber: string;       // PK, UUID v4
  syncStatus: OfflineSyncStatus;
  syncError?: string;
  serverDeliveryPlanDetailId?: number;

  // Required business fields
  deliveryPlanDate: string;       // ISO date (default: today)
  productionTime?: string;        // HH:mm (optional)
  carId: number;
  carName?: string;               // denormalized for offline display
  driverId: number;
  driverName?: string;
  materialId: number;             // concrete grade e.g. C30
  materialName?: string;
  quantity: number;               // m³ or unit
  unitId: number;

  // Metadata
  userId: number;
  tenantId: string;
  createdAt: string;
  updatedAt: string;
  schemaVersion: number;          // for future migrations
}
```

**Dexie DB**:

typescript

```typescript
export class OfflineMonitoringDb extends Dexie {
  offlineTrucks!: Table<OfflineTruckRecord, string>;
  referenceDataCache!: Table<{ key: string; data: any; lastUpdated: string }, string>;

  constructor() {
    super('AccflexOfflineMonitoring');
    this.version(1).stores({
      offlineTrucks: 'clientGuidNumber, syncStatus, userId, tenantId, createdAt',
      referenceDataCache: 'key'
    });
  }
}
```

**OfflineDbService methods**:

- `addRecord(record): Observable<string>`
- `getRecordsByStatus(status, userId, tenantId): Observable<OfflineTruckRecord[]>`
- `getAllUserRecords(userId, tenantId): Observable<OfflineTruckRecord[]>`
- `updateRecord(clientGuidNumber, partial): Observable<void>`
- `deleteRecord(clientGuidNumber): Observable<void>`
- `getPendingCount(userId, tenantId): Observable<number>`
- `purgeOldRecords(olderThanDays, statuses): Observable<number>`

**Test before moving on**:

- Open DevTools > Application > IndexedDB: `AccflexOfflineMonitoring` DB appears with 2 tables
- Call `addRecord()` from browser console, verify record in IndexedDB
- Call `getRecordsByStatus(PendingSync)`, verify filtering works
- Call `purgeOldRecords(0, ['Synced'])`, verify cleanup

---

### Step 3: Reference Data Caching for Offline Forms

**Goal**: Cache lookup data (trucks, drivers, materials, stations) into IndexedDB so the offline form has populated dropdowns.

**Files to create**:

- `.../offline/reference-data-cache.service.ts`

**Files to modify**:

- `.../shared/monitoring-concrete-products-data-service.ts` -- trigger caching after successful online data load

**Reference data to cache** (only what the minimal offline form needs):

|Key|Source|Size estimate|
|---|---|---|
|`trucks`|`staticResource.cars$` (filtered CarCategory=Truck)|~100-500 items|
|`drivers`|`staticResource.Drivers`|~50-200 items|
|`materials`|`staticResource.MaterialWithOrganization` (concrete grades)|~100-2000 items|
|`units`|`staticResource.Units`|~20-50 items|

**Technical approach**:

- `ReferenceDataCacheService` is injected into `MonitoringConcreteProductsDataService`
- After each online data fetch, call `cacheService.cacheAll({ trucks, drivers, ... })`
- Each list is serialized to `referenceDataCache` table with `lastUpdated` timestamp
- When offline, `MonitoringConcreteProductsDataService` reads from cache instead of API
- Use a `DataSourceProxy` approach: `getTrucks(): Observable<TruckDto[]>` checks `NetworkStatusService.isOnline$` -> online? API call + cache update. Offline? read from IndexedDB cache.

**Large data concern**: If `materials` or `customers` exceeds 5,000 items, filter by user's SalesOrganizationId before caching. Only cache materials relevant to the user's org.

**Test before moving on**:

- Load monitoring page online -> check IndexedDB `referenceDataCache` table has entries for all keys
- Switch to offline mode -> reload page -> verify dropdowns still populate
- Verify `lastUpdated` refreshes on next online load

---

### Step 4: Offline Truck Entry UI

**Goal**: Add UI for creating, viewing, editing, and deleting local offline truck records.

**Files to create**:

- `.../offline/components/offline-truck-list/offline-truck-list.component.ts|html|scss`
- `.../offline/components/sync-status-badge/sync-status-badge.component.ts|html|scss`
- `.../offline/offline-entry.service.ts`

**Files to modify**:

- `.../containers/monitoring-concrete-products-shell/monitoring-concrete-products-shell.component.html` -- add "Offline Entries" tab to `dx-tab-panel`
- `.../containers/monitoring-concrete-products-shell/monitoring-concrete-products-shell.component.ts` -- inject offline services, expose pending count
- `.../monitoring-concrete-products.module.ts` -- declare new components, provide services
- i18n files (`en-US/logistics.json`, `ar-EG/logistics.json`) -- add translation keys

**Technical approach**:

- New tab "Offline Entries" / "ادخال بدون اتصال" in the existing `dx-tab-panel`. Show pending count badge on tab title.
- `offline-truck-list` uses `dx-data-grid` with columns: Status (badge), Truck, Driver, Material, Qty, Date, Created, Error, Actions
- "Add Offline Truck" button opens the existing `add-trucks-pupop` form (reuse). Intercept the save action:
    - If offline OR user explicitly chose "Save Locally": save to IndexedDB via `OfflineEntryService`
    - `OfflineEntryService.createLocalRecord(formData)` generates UUID via `UUID.UUID()`, maps form data to `OfflineTruckRecord`, saves via `OfflineDbService`
- Row actions on the grid:
    - **Edit** (PendingSync/SyncFailed only): reopens form with local data
    - **Delete** (PendingSync/SyncFailed only): confirmation popup -> removes from IndexedDB
    - **View** (any status): read-only detail popup
- Online/offline indicator: small banner at top of monitoring screen bound to `NetworkStatusService.isOnline$`

**Test before moving on**:

- Third tab appears with correct Arabic/English labels
- Click "Add Offline Truck" while offline: form opens with cached dropdown data
- Fill and save: record appears in grid with yellow "Pending Sync" badge
- Edit the record: form opens pre-filled, save updates IndexedDB
- Delete the record: confirmation dialog, record removed
- Tab badge shows correct count

---

### Step 5: Backend Sync API & Domain Entities

**Goal**: Create the batch sync endpoint and the `DeliveryPlanFromOffline` domain entity.

**Files to create** (Backend):

Domain layer:

- `Logistics.Production.Logic/DeliveryPlanAggregate/OnOfflineEntry/DeliveryPlanFromOffline.cs`
- `Logistics.Production.Logic/DeliveryPlanAggregate/OnOfflineEntry/DeliveryPlanDetailFromOffline.cs`
- `Logistics.Production.Logic/DeliveryPlanAggregate/OnOfflineEntry/Pocos/DeliveryPlanDetailFromOfflinePoco.cs`

Application layer:

- `Logistics.Application/Production/MonitoringConcreteProducts/Commands/SyncOfflineTrucks/SyncOfflineTrucksCommand.cs`
- `Logistics.Application/Production/MonitoringConcreteProducts/Commands/SyncOfflineTrucks/SyncOfflineTrucksCommandHandler.cs`
- `Logistics.Application/Production/MonitoringConcreteProducts/Commands/SyncOfflineTrucks/OfflineTruckSyncItem.cs`
- `Logistics.Application/Production/MonitoringConcreteProducts/Commands/SyncOfflineTrucks/SyncOfflineTrucksResponse.cs`
- `Logistics.Application/Production/MonitoringConcreteProducts/Commands/SyncOfflineTrucks/SyncItemResult.cs`

Data layer:

- `Logistics.Production.Data/EFConfigurations/DeliveryPlanFromOfflineConfiguration.cs`
- `Logistics.Production.Data/EFConfigurations/DeliveryPlanDetailFromOfflineConfiguration.cs`

**Files to modify**:

- `Logistics.Common.Logic/Enums/DeliveryPlanSourceType.cs` -- add `OnOfflineEntry = new(4, "On Offline Entry", "ادخال بدون اتصال")`
- `Logistics.API/ApiControllers/Production/MonitoringConcreteProductsController/MonitoringConcreteProductsController.cs` -- add sync endpoint
- `Logistics.Application/Common/Interfaces/IProductionContext.cs` -- add `DbSet<DeliveryPlanFromOffline>`
- `Logistics.Production.Data/ProductionContext.cs` -- add `DbSet<DeliveryPlanFromOffline>`

**Command handler logic per item**:

```
1. Check idempotency: query WHERE GuidNumber == item.ClientGuidNumber
   -> If exists: return Success with existing ServerId (skip creation)
2. Validate: CarId exists + is Truck category, DriverId exists, MaterialId exists, etc.
   -> If validation fails: return Failure with error message (continue to next item)
3. Create DeliveryPlanFromOffline aggregate with:
   - DeliveryPlanSourceTypeId = 4
   - GuidNumber = item.ClientGuidNumber
   - DeliveryPlanDate = item.DeliveryPlanDate
   - BranchId from user session
4. Add DeliveryPlanDetailFromOffline child with truck/driver/material data
5. Save -> return Success with new DeliveryPlanDetailId
```

All items processed in a single `IApplicationTransaction`. Individual failures do not roll back successful items -- use try/catch per item within the transaction.

**Domain entity `DeliveryPlanFromOffline`**:

- Inherits `DeliveryPlanBase`
- Factory: `static Result<DeliveryPlanFromOffline> CreateDeliveryPlan(DeliveryPlanDetailFromOfflinePoco poco, IDateTime dateTimeProvider, int maxSerial)`
- Sets `DeliveryPlanSourceTypeId = 4`
- Skips: SalesOrder validation, inventory availability check, production order creation events
- Does NOT fire `CreateGoodIssueOnDeliveryPlanEvent` or `CreateRedirectedSalesOrderForDeliveryPlanDomainEvent` (those are for linked records only)
- Override `RedirectDeliveryPlanDetails` to return failure (offline records can't be redirected until linked)

**EF Configuration** (TPH discriminator):

- Map `DeliveryPlanFromOffline` in the existing TPH hierarchy using `HasDiscriminator(x => x.DeliveryPlanSourceTypeId).HasValue(4)`
- All columns already exist on `Lgx_DeliveryPlan` / `Lgx_DeliveryPlanDetail` -- nullable FKs like `SalesOrderDetailId` are simply left null

**Test before moving on**:

- POST to `/api/MonitoringConcreteProducts/SyncOfflineTrucks` with 3 valid items via Postman -> all return success with server IDs
- POST same 3 `ClientGuidNumber` values again -> returns same server IDs (idempotency verified)
- POST with 1 valid + 1 invalid (bad CarId) -> valid succeeds, invalid returns error
- Check `Lgx_DeliveryPlan` table -> new records with `DeliveryPlanSourceTypeId = 4`
- Check `Lgx_DeliveryPlanDetail` -> records have `SalesOrderDetailId = NULL`

---

### Step 6: Frontend Manual Sync Service

**Goal**: Wire the sync button to the batch API and update local record statuses.

**Files to create**:

- `.../offline/sync.service.ts`

**Files to modify**:

- `.../offline/components/offline-truck-list/offline-truck-list.component.ts` -- add sync button handler
- Auto-generated API client (NSwag regeneration) OR manually add method to logistics API client

**SyncService flow**:

```
1. Query IndexedDB: all records WHERE syncStatus IN (PendingSync, SyncFailed)
2. If none: show "Nothing to sync" toast -> return
3. Check NetworkStatusService.isOnline$
   -> If offline: show "No internet connection" toast -> return
4. Update all selected records to syncStatus = Syncing
5. Map records to OfflineTruckSyncItem[] (strip display-only fields)
6. POST to SyncOfflineTrucks endpoint
7. For each result:
   - Success: update record -> syncStatus = Synced, serverDeliveryPlanDetailId = result.serverId
   - Failure: update record -> syncStatus = SyncFailed, syncError = result.errorMessage
8. On HTTP error (network failure, 401, 500):
   - Revert all Syncing records back to PendingSync
   - If 401: trigger OIDC silent refresh, show "Please login again" if refresh fails
9. Show summary toast: "3 synced, 1 failed"
10. Refresh the dx-data-grid
```

**Sync button UX**:

- Disabled when offline (bound to `isOnline$ | async`)
- Shows spinner while syncing
- "Retry Failed" button: filters only SyncFailed records for re-sync

**Test before moving on**:

- Create 5 offline records -> click Sync -> all show "Synced" with green badges
- Create 3 records -> disconnect -> Sync button disabled
- Create records with 1 invalid -> sync -> 2 green, 1 red with error tooltip
- Click "Retry" on failed record after fixing -> becomes Synced
- Kill network mid-sync -> records revert to PendingSync (not stuck in Syncing)

---

### Step 7: Post-Sync Status UI & Error Display Polish

**Goal**: Rich status display, error details, summary counts, and row-level retry/delete.

**Files to modify**:

- `.../offline/components/offline-truck-list/offline-truck-list.component.html` -- enhanced grid template
- `.../offline/components/sync-status-badge/sync-status-badge.component.ts` -- color-coded badges

**Status badge colors**:

|Status|Color|Icon|
|---|---|---|
|PendingSync|Yellow/Amber|Clock|
|Syncing|Blue|Spinner|
|Synced|Green|Checkmark|
|SyncFailed|Red|X-mark|

**Summary bar** above grid: "5 Pending | 2 Failed | 12 Synced"

**Error column**: Truncated text + "Details" button -> opens `dx-popup` with full error message and field-level validation errors.

**Row actions column** (dx-data-grid cell template):

- PendingSync: [Edit] [Delete]
- SyncFailed: [Edit] [Retry] [Delete]
- Synced: [View]

**Test before moving on**:

- Each status shows correct color badge
- Error popup displays full validation message
- Summary bar counts are accurate and update after sync
- Retry button on failed row triggers sync for that single row

---

### Step 8: Sales Order Linkage -- DEFERRED TO PHASE 2

Sales order linkage is out of scope for Phase 1. Synced records remain as `SourceType=4` in the database. In Phase 2, a linkage mechanism will associate these records with sales orders. Key decisions for Phase 2:

- Whether to add `LinkedSalesOrderDetailId` FK or delete-and-recreate as SourceType=1
- How to handle EF Core discriminator constraints

For now, synced records are visible in the main Delivery Tracking grid (filtered by SourceType=4) but cannot be linked.

---

### Step 8: Edge Cases -- Logout Warning, Auth, Multi-Device

**Goal**: Handle authentication, logout, and multi-device scenarios safely.

**Files to modify**:

- Logout/signout component -- add pending-records check before logout
- `.../offline/offline-db.service.ts` -- add `hasPendingRecords(userId, tenantId)` method

**Files to create**:

- `.../offline/guards/pending-data.guard.ts` -- CanDeactivate guard on monitoring route

**Logout handling**:

- Before logout: check `offlineDbService.getPendingCount(userId, tenantId)`
- If > 0: show confirmation dialog: "You have {n} unsynced records on this device. They will NOT be deleted but you won't be able to sync until you log in again. Continue?"
- Records persist in IndexedDB (keyed by userId+tenantId). When user logs back in, records are still there.

**Auth token expiry**:

- The `SyncService` catches HTTP 401 errors
- On 401: call OIDC silent refresh (`AccflexOidcService.signinSilent()`)
- If silent refresh succeeds: retry the sync call
- If silent refresh fails: show toast "Session expired, please log in again"
- Records remain PendingSync -- no data loss

**Multi-device**:

- Each device has its own IndexedDB (no cross-device sync of local data)
- Records are keyed by `userId + tenantId` so shared tablets isolate per-user
- If same user syncs the same record from two devices: `GuidNumber` idempotency prevents duplicate server records
- Second sync attempt returns success with existing `serverId`

**CanDeactivate guard**:

- When navigating away from monitoring module with pending records: show warning
- "You have unsaved offline records. Leave anyway?"

**Test before moving on**:

- Create pending records -> attempt logout -> warning dialog appears
- Confirm logout -> log back in -> records still visible in offline tab
- Navigate away from monitoring with pending records -> guard warning appears
- Simulate token expiry (short-lived token in dev) -> sync fails -> re-auth -> sync succeeds

---

### Step 9: Cleanup, Storage Quota & Schema Versioning

**Goal**: Auto-purge old data, monitor storage, handle schema evolution.

**Files to modify**:

- `.../offline/offline-db.service.ts` -- add `checkStorageQuota()`, `purgeOldSyncedRecords()`
- `.../offline/offline-db.ts` -- add Dexie version upgrade handlers for future schema changes

**Files to create**:

- `.../offline/components/storage-warning-banner/storage-warning-banner.component.ts`

**Auto-purge logic** (runs on monitoring module init):

typescript

```typescript
async purgeExpiredRecords(): Promise<void> {
  const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString();
  const oneDayAgo = new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString();

  // Purge Synced records older than 7 days
  await this.db.offlineTrucks
    .where('syncStatus').equals('Synced')
    .and(r => r.updatedAt < sevenDaysAgo)
    .delete();

  // Phase 2: will also purge Linked records when linkage is implemented
}
```

**Storage quota check**:

typescript

```typescript
async checkStorageQuota(): Promise<{ usedMB: number; quotaMB: number; percentUsed: number }> {
  if ('storage' in navigator && 'estimate' in navigator.storage) {
    const { usage, quota } = await navigator.storage.estimate();
    return {
      usedMB: (usage || 0) / (1024 * 1024),
      quotaMB: (quota || 0) / (1024 * 1024),
      percentUsed: ((usage || 0) / (quota || 1)) * 100
    };
  }
  return { usedMB: 0, quotaMB: 0, percentUsed: 0 };
}
```

- If `percentUsed > 80%`: show warning banner with "Sync and clear old records"

**Schema versioning**:

- Each `OfflineTruckRecord` has `schemaVersion: 1`
- When app updates with new fields, add Dexie `version(2).stores(...)` with upgrade function
- Backend sync endpoint checks `schemaVersion` and applies defaults for missing fields
- This prevents data loss when app updates while user has unsynced records

**"Clear Synced" button**:

- Manual action in offline tab toolbar
- Deletes all records where `syncStatus = Synced` for current user
- Shows confirmation with count

**Test before moving on**:

- Create records, sync, wait (or mock dates) -> purge runs on next app open
- Check storage quota API returns values
- Simulate schema v1 record with v2 app -> Dexie migration runs without data loss
- "Clear Synced" button removes correct records

---

## Gaps Identified & Recommendations

### Gap 1: EF Core TPH discriminator mapping

Verify the existing `DeliveryPlanBase` hierarchy uses TPH (not TPT). If TPT, each subclass has its own table, and the approach changes. Check `DeliveryPlanBaseConfiguration.cs` for `HasDiscriminator()` calls.

### Gap 2: What about concurrent quantity exhaustion?

If two users offline-create trucks for the same material/station, and the total exceeds available quantity, the sync should validate. But the current sync design validates per-item. **Recommend**: validate cumulative quantity within the batch, and also against current server inventory at sync time.

### Gap 3: Offline indicator placement -- RESOLVED

Monitoring screen only (per user decision).

### Gap 4: NSwag client regeneration -- RESOLVED

Manual workflow: developer runs `nswag run` locally after backend changes, commits and publishes.

---

## Verification Plan

|#|Scenario|Expected Result|
|---|---|---|
|1|Create 5 records offline, close browser, reopen|All 5 records persist in offline tab|
|2|Edit unsynced record, then sync|Edited values sync correctly|
|3|Delete unsynced record|Removed from IndexedDB permanently|
|4|Sync while offline|Button disabled, toast "No connection"|
|5|Sync 3 valid + 1 invalid|3 Synced, 1 SyncFailed with error|
|6|Retry failed record after fixing|Record syncs successfully|
|7|Sync same record twice (network retry)|No duplicate on server (idempotency)|
|8|Synced record visible in delivery tracking grid|Appears with SourceType=4 indicator, not linked|
|9|Logout with pending records|Warning dialog, records persist|
|10|Existing online AddTruck flow|Completely unchanged, works as before|
|11|Large batch sync (50 records)|All processed, reasonable response time (<10s)|
|12|Token expiry during offline period|Re-auth on sync, records not lost|
|13|App update with schema v2, existing v1 records|Dexie migration preserves data|
|14|Two devices sync same GuidNumber|Only one server record created|

---

## Critical File References

|Purpose|Path|
|---|---|
|DeliveryPlan SourceType enum|`AccFlex_Cloud_ERP/.../Logistics.Common.Logic/Enums/DeliveryPlanSourceType.cs`|
|DeliveryPlanBase aggregate|`AccFlex_Cloud_ERP/.../Logistics.Production.Logic/DeliveryPlanAggregate/Base/DeliveryPlanBase.cs`|
|OnSalesOrder subclass (template)|`AccFlex_Cloud_ERP/.../DeliveryPlanAggregate/OnSalesOrder/DeliveryPlanFromSalesOrder.cs`|
|CreateTruck command (template)|`AccFlex_Cloud_ERP/.../MonitoringConcreteProducts/Commands/DeliverySchedules/CreateTruck/`|
|Monitoring controller|`AccFlex_Cloud_ERP/.../ApiControllers/Production/MonitoringConcreteProductsController/`|
|Production context|`AccFlex_Cloud_ERP/.../Logistics.Production.Data/ProductionContext.cs`|
|Frontend monitoring module|`AccFlex_Cloud_Front/.../modules/monitoring-concrete-products/`|
|Frontend data service|`AccFlex_Cloud_Front/.../monitoring-concrete-products-data-service.ts`|
|Frontend add-trucks popup|`AccFlex_Cloud_Front/.../componants/add-trucks-pupop/`|
|Frontend shell component|`AccFlex_Cloud_Front/.../containers/monitoring-concrete-products-shell/`|
|Frontend package.json|`AccFlex_Cloud_Front/accflex-erp/package.json`|
|Angular config|`AccFlex_Cloud_Front/accflex-erp/angular.json`|
|DB DeliveryPlan table|`AccFlex_Cloud_Database/.../dbo/Tables/Lgx_DeliveryPlan.sql`|
|DB DeliveryPlanDetail table|`AccFlex_Cloud_Database/.../dbo/Tables/Lgx_DeliveryPlanDetail.sql`|
