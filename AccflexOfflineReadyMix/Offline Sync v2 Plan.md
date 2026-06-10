---
project: AccflexOfflineReadyMix
module: monitoring-concrete-products (Module 621)
topic: offline sync, caching, two-phase transaction sync
status: in-progress
created: 2026-06-03
---

# AccflexOfflineReadyMix — Offline Sync v2

Efficient cache · full offline coverage + freshness · unsynced edit-lock · two-phase transaction sync.

## Context
The desktop already caches lookups, delivery rows, layouts and print data, and queues offline mutations
in a single outbox that drains oldest-first on reconnect. Three weaknesses remain:

1. **Offline completeness/freshness** — some reference caches fail to deserialize offline
   (`reference_materials/stations/customers` → *"Cannot deserialize JSON array into MaterialsUnitDto"*),
   and reconnect only auto-pushes the outbox; it never auto-re-pulls fresh data.
2. **No conflict safety** — the single-phase push replays every queued mutation blindly. A truck added
   offline to a sales-order detail that was changed meanwhile can be pushed onto stale state.
3. **Free-standing inserts** — rows inserted offline with no sales-order reference are posted as
   stand-alone schedules instead of being attached to a real sales order.

**Goal:** make the offline cache efficient and complete, refresh it on reconnect, **block new live edits
while anything is unsynced**, and split transaction sync into **two phases**.

## Decisions (confirmed)
- **Conflict check (phase 1):** *Re-fetch & compare* — before pushing a ticket on an existing SO detail,
  re-fetch the current detail and compare its concurrency token (`TimeStamp`) + key fields to the snapshot
  captured when going offline. Identical → push; changed → block as a conflict.
- **Link target (phase 2):** *SO delivery-schedule detail* — rows inserted offline without a reference are
  linked by letting the user pick an existing sales-order delivery-schedule detail; the row's data/trucks
  attach to it via the same path as Add Truck on that detail.
- **Edit-lock scope:** *All edits until synced* — while any unsynced transaction exists, block every
  create/edit/delete with a banner that routes the user to sync.

---

## Steps (implement one by one)

- [x] **Step 1 — Save this plan to Obsidian** (vault `accflex`).
- [ ] **Step 2 — Fix offline reference-cache deserialization (foundational).**
  Align cached `reference_*` JSON with the typed DTO read-back
  (`MaterialsUnitDto` / `StationDetailListDto` / `CustomerDeliveryAddressDto`).
  Files: `Services/MonitoringConcreteProductsState.cs`, `Sync/ReferenceSyncService.cs`,
  `Data/Repositories/LookupsRepo.cs`, `Data/Repositories/DeliveryGridRepo.cs`.
  Acceptance: force-offline → all filter dropdowns + grid name-resolution populate from cache.
- [ ] **Step 3 — Auto refresh-on-reconnect.**
  After outbox drains on offline→online, auto re-pull references/deliveries/layouts/print-data.
  Files: `Sync/OutboxSyncService.cs` (`OnConnectivityChanged`) or new `Sync/SyncCoordinator`;
  reuse `ReferenceSyncService.SyncAllAsync`, `DeliverySyncService.SyncAsync`,
  `PrintLayoutSyncService`, `GridLayoutSyncService`. Order: push → if clean, pull.
- [ ] **Step 4 — Caching efficiency pass.**
  One transaction/connection per bulk write; confirm/add indexes (`outbox(SyncStatus,CreatedAt)`,
  `delivery_plan_details` search keys); keep `HttpCacheRepo.PruneOlderThan`; skip redundant re-fetch via
  `SyncMetaRepo` timestamps. Files: `Data/Repositories/*`, `Sync/*`, `Data/LocalDb.cs`.
- [ ] **Step 5 — Global unsynced edit-lock.**
  New `Services/UnsyncedGate.cs` (`HasUnsynced = OutboxRepo.CountAttention()>0 || DraftSchedulesRepo.CountPending()>0`).
  Gate every mutation entry point with a bilingual banner: `AddTrucksPopup`, `InsertScheduleRowPopup`,
  status/redirect/delete in `DeliveryTrackingTab`/`DeliveryScheduleTab`.
- [ ] **Step 6 — Phase 1 sync: tickets on EXISTING SO details (re-fetch & compare).**
  Split push; pre-check each row: re-fetch detail, compare `TimeStamp`+key fields to enqueue snapshot;
  identical → push, changed → new **Conflict** state (don't push). Server `TimeStamp` as backstop.
  Covers `AddTruck`, `UpdateDeliveryStatus`, `DeleteDelivery`, `PutDeliveryTracking`.
  Files: `Sync/OutboxSyncService.cs`, `Api/MonitoringApi.cs`, `Data/Repositories/OutboxRepo.cs`,
  `Models/AddTruckCommand.cs`.
- [ ] **Step 7 — Phase 2 sync: link unreferenced inserts to an existing SO detail.**
  For each pending `draft_schedules` row, show a picker of existing SO delivery-schedule details
  (reuse `GetDeliverySchedules`); map draft → `CreateTruckCommand` for the chosen `DeliveryPlanDetailId`
  (reuse `AddTrucksPopup.BuildCommand`); push via `MonitoringApi.AddTruckAsync`; `DraftSchedulesRepo.MarkSynced`.
  Run **Phase 1 first, then Phase 2.**
  Files: `Forms/SyncForm.cs`, `Forms/MonitoringConcreteProducts/UnsyncedItemsTab.cs`,
  `Data/Repositories/DraftSchedulesRepo.cs`, `Sync/OutboxSyncService.cs`, `AddTrucksPopup.cs`, `Api/MonitoringApi.cs`.
- [ ] **Step 8 — Sync UI + end-to-end verification.**
  Two phases shown distinctly (Phase 1 conflict flags, Phase 2 link action), edit-lock banner, freshness
  timestamps. Files: `SyncForm.cs`, `UnsyncedItemsTab.cs`.

## Verification
1. Build: `MSBuild AccflexOfflineReadyMix.csproj /p:Configuration=Debug` (exit 0).
2. Offline completeness: force-offline → dropdowns/grids + print render from cache.
3. Freshness: offline → change row on web → online → caches auto-refresh after outbox drains.
4. Edit-lock: queue 1 offline change → all add/edit/delete blocked with banner.
5. Phase 1 conflict: queue truck; change that detail on web; sync → held as Conflict. Unchanged → pushes.
6. Phase 2 link: insert row offline (no ref); sync → must pick existing SO detail; lands as ticket on it.
7. Inspect `%LOCALAPPDATA%\Accflex\OfflineReadyMix\local.db` + `logs\app-*.log`.

## File map (reuse, don't recreate)
- Outbox: `Sync/OutboxSyncService.cs`, `Data/Repositories/OutboxRepo.cs` (`OutboxKinds`, `CountAttention`, `ListPending`).
- Drafts: `Data/Repositories/DraftSchedulesRepo.cs`, `Models/CreateScheduleRowCommand.cs`.
- Tickets: `Models/AddTruckCommand.cs`, `Forms/MonitoringConcreteProducts/AddTrucksPopup.cs` (`BuildCommand`),
  `Api/MonitoringApi.cs` (`AddTruckAsync`, `CreateScheduleRowAsync`, `GetDeliveryTracking*`, `GetDeliveryPlanDetailPrint*`).
- Connectivity: `Services/ConnectivityState.cs` (`IsOnline`, `StatusChanged`).
- Caching: `Sync/ReferenceSyncService.cs`, `Sync/DeliverySyncService.cs`, `Data/Repositories/*Repo.cs`,
  `Data/LocalDb.cs` (schema v9).
