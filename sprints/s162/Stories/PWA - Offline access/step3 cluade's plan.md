# Step 3: Reference Data Caching for Offline Forms

## Context

Steps 1 (PWA/service worker) and 2 (Dexie IndexedDB + `OfflineDbService`) are complete and verified working. Step 3 caches the 4 lookup lists that offline entry forms need — cars, drivers, materials, units — into IndexedDB so dropdowns populate when the device has no connectivity.

**Project root**: `D:\Hadaf Work TFS\AccFlex_Cloud_Front\accflex-erp`
**Target module**: `projects/logistics/src/app/modules/monitoring-concrete-products/`

---

## Codebase Context

### Existing infrastructure (Steps 1 & 2)

- **`NetworkStatusService`** — `providedIn: 'root'`, `isOnline$: BehaviorSubject<boolean>`, sync read via `.value`
- **`OfflineDbService`** — `@Injectable()` module-scoped, in `offline/offline-db.service.ts`:
  - `getCachedData(key: string): Observable<ReferenceDataCacheEntry | undefined>`
  - `setCachedData(entry: ReferenceDataCacheEntry): Observable<string>` (Dexie `put` = upsert)
- **`ReferenceDataCacheEntry`** model — `{ key: string; data: unknown; lastUpdated: string }`
- **`OfflineDbService`** already registered in `monitoring-concrete-products.module.ts` providers

### Data service loading pattern

`MonitoringConcreteProductsDataService.fillLookupData()` (called in constructor) subscribes to `LogisticsStaticResourcesService` refresh subjects. All 4 `BehaviorSubject<void>` subjects fire immediately on subscribe (initial load), then again when data refreshes:

| Refresh subject | Observable | Data service property |
|---|---|---|
| `refreshDriverSubject` | `Drivers: Observable<DriverForListDto[]>` | `drivers` (filtered `IsActive`) |
| `refreshCarsSubject` | `cars$: Observable<CarForListDto[]>` | `TruckList` (Truck), `PumpList`, `pumpListForDeliveryPlan` |
| `refreshUnitsSubject` | `UnitList: Observable<UnitDtoForList[]>` | `unitList` |
| `refreshMaterialSubject` | `MaterialWithOrganizationList` | `materialList`, `materialUnitList` |

### Key DTOs (from `logistics.api.client`)
- `CarForListDto`: `CarId, CarNumber, CarCategoryId, CarDriverId, UnitId, ...`
- `DriverForListDto`: `DriverId, DriverName, IsActive, Mobile, ...`
- `MaterialWithOrganizationsDto`: `MaterialId, MaterialName, Code, MinimumUnitId, MaterialsUnitDtoList, ...`
- `UnitDtoForList`: `UnitId, UnitName, DimensionId`
- `CarCategory` enum: `Truck = 1, Pump = 2` (from `../../delivery-plan/shared/delivery-plan.enums`)

---

## Implementation Plan

### 1. Create `offline/reference-data-cache.service.ts`

**Path**: `projects/logistics/src/app/modules/monitoring-concrete-products/offline/reference-data-cache.service.ts`

Typed wrapper over `OfflineDbService`. Exposes named read/write pairs per entity. Cache key `'cars'` stores ALL cars (trucks + pumps combined) so both `TruckList` and `PumpList` can be derived when offline.

```typescript
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import {
  CarForListDto,
  DriverForListDto,
  MaterialWithOrganizationsDto,
  UnitDtoForList
} from 'logistics.api.client';
import { OfflineDbService } from './offline-db.service';
import { ReferenceDataCacheEntry } from './models';

export const CACHE_KEYS = {
  CARS:      'cars',
  DRIVERS:   'drivers',
  MATERIALS: 'materials',
  UNITS:     'units',
} as const;

@Injectable()
export class ReferenceDataCacheService {

  constructor(private readonly offlineDb: OfflineDbService) {}

  // ── Cars ──────────────────────────────────────────────────────────────────

  getCars(): Observable<CarForListDto[] | undefined> {
    return this.offlineDb.getCachedData(CACHE_KEYS.CARS).pipe(
      map(entry => entry?.data as CarForListDto[] | undefined)
    );
  }

  setCars(cars: CarForListDto[]): Observable<string> {
    return this.offlineDb.setCachedData({
      key: CACHE_KEYS.CARS,
      data: cars,
      lastUpdated: new Date().toISOString()
    });
  }

  // ── Drivers ───────────────────────────────────────────────────────────────

  getDrivers(): Observable<DriverForListDto[] | undefined> {
    return this.offlineDb.getCachedData(CACHE_KEYS.DRIVERS).pipe(
      map(entry => entry?.data as DriverForListDto[] | undefined)
    );
  }

  setDrivers(drivers: DriverForListDto[]): Observable<string> {
    return this.offlineDb.setCachedData({
      key: CACHE_KEYS.DRIVERS,
      data: drivers,
      lastUpdated: new Date().toISOString()
    });
  }

  // ── Materials ─────────────────────────────────────────────────────────────

  getMaterials(): Observable<MaterialWithOrganizationsDto[] | undefined> {
    return this.offlineDb.getCachedData(CACHE_KEYS.MATERIALS).pipe(
      map(entry => entry?.data as MaterialWithOrganizationsDto[] | undefined)
    );
  }

  setMaterials(materials: MaterialWithOrganizationsDto[]): Observable<string> {
    return this.offlineDb.setCachedData({
      key: CACHE_KEYS.MATERIALS,
      data: materials,
      lastUpdated: new Date().toISOString()
    });
  }

  // ── Units ─────────────────────────────────────────────────────────────────

  getUnits(): Observable<UnitDtoForList[] | undefined> {
    return this.offlineDb.getCachedData(CACHE_KEYS.UNITS).pipe(
      map(entry => entry?.data as UnitDtoForList[] | undefined)
    );
  }

  setUnits(units: UnitDtoForList[]): Observable<string> {
    return this.offlineDb.setCachedData({
      key: CACHE_KEYS.UNITS,
      data: units,
      lastUpdated: new Date().toISOString()
    });
  }
}
```

**Notes**:
- `as const` on `CACHE_KEYS` prevents typos at call sites
- All `get*` return `T | undefined` — callers handle the missing-cache case with an `if (cached)` guard
- `UnitDtoForList[]` stored (not `UnitDto[]`) — matches what `UnitList` emits; structural typing handles the property mismatch silently

---

### 2. Modify `shared/monitoring-concrete-products-data-service.ts`

**Path**: `projects/logistics/src/app/modules/monitoring-concrete-products/shared/monitoring-concrete-products-data-service.ts`

**A. Add imports** (append to existing import block):
```typescript
import { ReferenceDataCacheService } from '../offline/reference-data-cache.service';
import { NetworkStatusService } from 'src/app/core/services/pwa/network-status.service';
```

**B. Add constructor parameters** (append to existing constructor):
```typescript
constructor(
  // ... existing 12 parameters unchanged ...
  private referenceDataCache: ReferenceDataCacheService,  // NEW
  private networkStatus: NetworkStatusService              // NEW
) {
  this.fillLookupData();
}
```

**C. Replace the 4 lookup subscription blocks in `fillLookupData()`**

Replace from `this.sub.sink = this.staticResource.refreshDriverSubject.subscribe(...)` through the end of the `refreshMaterialSubject` block (lines ~130–211) with:

```typescript
if (this.networkStatus.isOnline$.value) {
  // ── Online: existing API subscriptions + fire-and-forget cache writes ──

  this.sub.sink = this.staticResource.refreshDriverSubject.subscribe({
    next: () => {
      this.sub.sink = this.staticResource.Drivers.subscribe({
        next: (res) => {
          this.drivers = res.filter((a) => a.IsActive);
          this.referenceDataCache.setDrivers(this.drivers).subscribe({
            error: (err) => console.error('[ReferenceDataCache] setDrivers failed', err)
          });
        }
      });
    }
  });

  this.sub.sink = this.staticResource.refreshCarsSubject.subscribe({
    next: () => {
      this.sub.sink = this.staticResource.cars$.subscribe({
        next: (res) => {
          this.TruckList = res.filter((x) => x.CarCategoryId === CarCategory.Truck);
          this.PumpList = res
            .filter((x) => x.CarCategoryId === CarCategory.Pump)
            .map((pump) => ({ PumpId: pump.CarId, PumpName: pump?.CarNumber }));
          this.pumpListForDeliveryPlan = this.PumpList.map((pump) => ({
            ...pump,
            GuidNumber: ''
          }));
          // cache ALL cars so offline can reconstruct both TruckList and PumpList
          this.referenceDataCache.setCars(res).subscribe({
            error: (err) => console.error('[ReferenceDataCache] setCars failed', err)
          });
        }
      });
    }
  });

  this.sub.sink = this.staticResource.refreshUnitsSubject.subscribe({
    next: () => {
      this.staticResource.UnitList.subscribe({
        next: (src) => {
          this.unitList = src;
          this.referenceDataCache.setUnits(src).subscribe({
            error: (err) => console.error('[ReferenceDataCache] setUnits failed', err)
          });
        }
      });
    }
  });

  this.sub.sink = this.staticResource.refreshMaterialSubject.subscribe({
    next: () => {
      this.sub.sink = this.staticResource.MaterialWithOrganizationList.subscribe({
        next: (res) => {
          this.materialList = res;
          this.materialUnitList = res.flatMap((x) => x.MaterialsUnitDtoList);
          this.referenceDataCache.setMaterials(res).subscribe({
            error: (err) => console.error('[ReferenceDataCache] setMaterials failed', err)
          });
        }
      });
    }
  });

} else {
  // ── Offline: load all 4 lists from IndexedDB cache ─────────────────────

  this.sub.sink = this.referenceDataCache.getDrivers().subscribe({
    next: (cached) => {
      if (cached) { this.drivers = cached; }
    },
    error: (err) => console.error('[ReferenceDataCache] getDrivers failed', err)
  });

  this.sub.sink = this.referenceDataCache.getCars().subscribe({
    next: (cached) => {
      if (cached) {
        this.TruckList = cached.filter((x) => x.CarCategoryId === CarCategory.Truck);
        this.PumpList = cached
          .filter((x) => x.CarCategoryId === CarCategory.Pump)
          .map((pump) => ({ PumpId: pump.CarId, PumpName: pump?.CarNumber }));
        this.pumpListForDeliveryPlan = this.PumpList.map((pump) => ({
          ...pump,
          GuidNumber: ''
        }));
      }
    },
    error: (err) => console.error('[ReferenceDataCache] getCars failed', err)
  });

  this.sub.sink = this.referenceDataCache.getUnits().subscribe({
    next: (cached) => {
      if (cached) { this.unitList = cached; }
    },
    error: (err) => console.error('[ReferenceDataCache] getUnits failed', err)
  });

  this.sub.sink = this.referenceDataCache.getMaterials().subscribe({
    next: (cached) => {
      if (cached) {
        this.materialList = cached;
        this.materialUnitList = cached.flatMap((x) => x.MaterialsUnitDtoList);
      }
    },
    error: (err) => console.error('[ReferenceDataCache] getMaterials failed', err)
  });
}
```

Everything after the 4 lookup blocks (inventory location, stations, customers, cost centres, projects, sales org, price procedure, condition types — lines ~176–251) is **left unchanged**.

---

### 3. Modify `monitoring-concrete-products.module.ts`

**Path**: `projects/logistics/src/app/modules/monitoring-concrete-products/monitoring-concrete-products.module.ts`

Add import and provider entry:

```typescript
import { ReferenceDataCacheService } from './offline/reference-data-cache.service';

// providers array — add after OfflineDbService:
OfflineDbService,
ReferenceDataCacheService
```

`NetworkStatusService` is `providedIn: 'root'` — no module registration needed.

---

## Files Summary

| Action | File |
|--------|-------|
| **Create** | `offline/reference-data-cache.service.ts` |
| **Modify** | `shared/monitoring-concrete-products-data-service.ts` |
| **Modify** | `monitoring-concrete-products.module.ts` |

All paths relative to `projects/logistics/src/app/modules/monitoring-concrete-products/`.

---

## Key Design Decisions

| Decision | Rationale |
|---|---|
| `isOnline$.value` synchronous snapshot in constructor | `fillLookupData()` runs in the constructor; BehaviorSubject's `.value` is the correct synchronous read pattern here |
| Cache ALL cars under `'cars'` key | Both `TruckList` and `PumpList` are derived by filtering `CarCategoryId`; the split logic is re-applied identically in the offline branch |
| Fire-and-forget cache writes via `.subscribe({ error: ... })` | Cache writes must not block the UI; inline subscribe with error logger is clean and avoids `tap`/`catchError` overhead |
| No staleness check for Step 3 | Online = always refresh; offline = use whatever is cached. TTL/stale-cache warnings are a Step 9 concern |
| Remaining lookups (customers, stations, etc.) NOT cached | Only the 4 lists needed by the offline entry form are in scope for Step 3 |

---

## Verification

**Build check**:
```bash
node --max_old_space_size=12288 ./node_modules/@angular/cli/bin/ng build --configuration production
node serve.js
```

**Test 1 — Online caching** (navigate to monitoring page while online):

1. Open DevTools → Application → IndexedDB → `AccflexOfflineMonitoring` → `referenceDataCache`
2. Should see 4 rows with keys: `cars`, `drivers`, `materials`, `units`
3. Each row has a `lastUpdated` ISO timestamp
4. `cars` entry's `data` array contains both trucks (`CarCategoryId: 1`) and pumps (`CarCategoryId: 2`)

**Test 2 — Offline read** (simulate offline and reload):

1. DevTools → Application → Service Workers → check **Offline**
2. Hard reload (`Ctrl+Shift+R`) to force a fresh module instantiation
3. Navigate to the monitoring module
4. In Console: no `[ReferenceDataCache] get* failed` errors
5. The truck/driver/material/unit dropdowns in the existing online UI still populate (or check `dataService.TruckList.length > 0` in console via `ng.probe` or a temporary `console.log`)

**Test 3 — `lastUpdated` refreshes**:

1. Go online, navigate to monitoring page → note `lastUpdated` timestamps
2. Hard reload → navigate again
3. `lastUpdated` values should be newer (Dexie `put` upserts the entry)