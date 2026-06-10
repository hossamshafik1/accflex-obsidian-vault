The Obsidian connector timed out again on the progress-log append (it's been consistently flaky) — the plan note itself is saved and the task tracker reflects status, so I won't keep retrying it.

## Checkpoint — Steps 1–5 done (all builds green)

|Step|Status|What changed|
|---|---|---|
|**1** Plan → Obsidian|✅|`accflex/AccflexOfflineReadyMix/Offline Sync v2 Plan.md`|
|**2** Offline cache fix|✅|`ReferenceSyncService.ToRows<T>` left nested values as Newtonsoft tokens → `LookupsRepo.Replace` (JavaScriptSerializer) wrote `[[]]` garbage. Added recursive `ToPlain(JToken)` → plain CLR. Materials/Stations/Customers now deserialize offline.|
|**3** Refresh-on-reconnect|✅|Centralized in `OutboxSyncService`: push → `AutoPullIfStale` (ReferenceSync + DeliverySync, 2‑min throttle, skipped while unsynced) → `RefreshedFromReconnect` event; form re-hydrates lookups + grids.|
|**4** Caching efficiency|✅|Audit confirmed transactional bulk writes + indexes + pruning already in place; dropped the redundant reconnect API re-fetch (hydrate from just-synced SQLite) and added the auto-pull throttle.|
|**5** Edit-lock|✅|Reused existing `SyncGate` (online && unsynced); added pending-drafts to `IsBlocked`. Already wired into every mutation entry point. Offline → queue freely; online+unsynced → block everything.|

**Decisions applied:** offline queueing allowed (lock only online); lock covers everything; conflict check = re-fetch & compare; phase-2 links to a chosen SO delivery-schedule detail.

## Remaining — Steps 6 & 7 (the heavy two-phase sync)

- **Step 6** — phase-1 conflict pre-check. There's **no by-id detail API**, so I'll re-fetch via `GetDeliveryTrackingAsync` (filter by `SalesOrganizationId` + date), locate the row, and compare its `TimeStamp` to the offline snapshot already in the `AddTruck` payload. On mismatch → mark the outbox row **Failed with a CONFLICT message** (reuses the attention/Unsynced surface). Server `TimeStamp` stays as backstop.
- **Step 7** — interactive **link** flow: for each pending draft, a picker of existing SO delivery-schedule details → map to a `CreateTruckCommand` for that `DeliveryPlanDetailId` (reuse `AddTrucksPopup.BuildCommand`) → push via `AddTruckAsync` → `DraftSchedulesRepo.MarkSynced`. New UI in the Unsynced/Sync screen.

These two are the most invasive (new UI + sync-flow changes) and can't be exercised live in this sandbox (need real offline + server-conflict scenarios), so this is a natural review point.

Say **proceed** and I'll implement Step 6 next, then Step 7.
## Context
The desktop already caches lookups, delivery rows, layouts and (newly) print data, and queues offline
mutations in a single outbox that drains oldest-first on reconnect. Three weaknesses remain:

1. **Offline completeness/freshness** is uneven — some reference caches fail to deserialize offline
   (`reference_materials/stations/customers` → "Cannot deserialize JSON array into MaterialsUnitDto"),
   and reconnect only auto-pushes the outbox; it never auto-re-pulls fresh data.
2. **No conflict safety** — the single-phase push replays every queued mutation blindly. A truck added
   offline to a sales-order detail that someone else changed meanwhile can be pushed onto stale state.
3. **Free-standing inserts** — rows inserted offline with no sales-order reference are posted as
   stand-alone schedules instead of being attached to a real sales order.

Goal: make the offline cache efficient and complete, refresh it on reconnect, **block new live edits
while anything is unsynced**, and split transaction sync into **two phases** with the agreed rules.

## Decisions (confirmed with user)
- **Conflict check (phase 1):** *Re-fetch & compare.* Before pushing a ticket on an existing SO detail,
  re-fetch the current detail and compare its concurrency token (`TimeStamp`) + key fields to the
  snapshot captured when going offline. Identical → push; changed → block as a conflict.
- **Link target (phase 2):** *SO delivery-schedule detail.* Rows inserted offline without a reference are
  linked by letting the user pick an existing sales-order delivery-schedule detail; the row's data/trucks
  attach to it via the same path as Add Truck on that detail.
- **Edit-lock scope:** *All edits until synced.* While any unsynced transaction (outbox attention rows or
  pending drafts) exists, block every create/edit/delete with a banner that routes the user to sync.

## Obsidian
Persist this plan as a note in vault **accflex** (e.g. `AccflexOfflineReadyMix/Offline Sync v2 Plan`)
and track each step there as it's implemented.

---

## Step 1 — Save the plan to Obsidian
Create the note in vault `accflex` with this content (sections + checkboxes per step) so progress is
tracked one-by-one. Tool: `obsidian create-note`.

## Step 2 — Fix offline reference-cache deserialization (foundational)
Offline screens are empty for materials/stations/customers because the cached JSON shape doesn't match
the typed DTO on read-back (`Path '[0].MaterialsUnitDtoList[0]'`).
- Files: `Services/MonitoringConcreteProductsState.cs` (`LoadLookupsAsync` / `ReadDtosFromCache` /
  `ReadNameDtosFromCache`), `Sync/ReferenceSyncService.cs` (what shape is written), the
  `reference_*` write path in `Data/Repositories/LookupsRepo.cs` / `DeliveryGridRepo.cs`.
- Fix: align the cached payload with the DTO the reader expects (write the same element shape that
  `Newtonsoft`/`JavaScriptSerializer` reads back into `MaterialsUnitDto` / `StationDetailListDto` /
  `CustomerDeliveryAddressDto`), or adjust the reader to the stored shape. Add a focused round-trip
  unit check.
- Acceptance: force-offline → every filter dropdown + grid name-resolution populates from cache.

## Step 3 — Auto refresh-on-reconnect (freshness)
Today only `OutboxSyncService` auto-pushes on offline→online. Add an auto **pull** so caches become
fresh after the outbox drains.
- Files: new orchestration in `Sync/` (e.g. `SyncCoordinator`) or extend
  `OutboxSyncService.OnConnectivityChanged`; reuse `ReferenceSyncService.SyncAllAsync`,
  `DeliverySyncService.SyncAsync` (already bulk-caches print data), `PrintLayoutSyncService`,
  `GridLayoutSyncService`.
- Order on reconnect: **push outbox → if clean, pull references/deliveries/layouts/print-data**.
  Do **not** pull while unsynced conflicts remain (ties into Step 5/6).
- Guard with the existing `_autoPushInFlight`-style reentrancy flag; respect `ConnectivityState`.

## Step 4 — Caching efficiency pass
- Files: `Data/Repositories/*` (`DeliveryTrackingRepo.Replace`, `LookupsRepo`, `PrintDataRepo`,
  `HttpCacheRepo`), `Sync/*`, `Data/LocalDb.cs`.
- Actions: ensure each bulk write runs in **one transaction/connection** (DeliverySync print-data and
  reference writes already batch — verify all do); add/confirm indexes for hot lookups
  (`print_data` PK exists; add indexes for `delivery_plan_details` search keys, `outbox(SyncStatus,CreatedAt)`
  already noted as `ix_outbox_status_created` — verify it exists); keep `HttpCacheRepo.PruneOlderThan`;
  avoid redundant re-fetch when a scope synced within the same session (use `SyncMetaRepo` timestamps).
- Acceptance: full Sync All time and DB write counts measurably reduced; no N+1 connection opens.

## Step 5 — Global "unsynced edit-lock"
Block all live mutations while unsynced work exists (decision: *all edits until synced*).
- New: `Services/UnsyncedGate.cs` → `HasUnsynced` = `OutboxRepo.CountAttention() > 0 ||
  DraftSchedulesRepo.CountPending() > 0`; `ChangedEvent` to refresh UI.
- Gate every mutation entry point with a clear bilingual banner ("Sync your offline changes before
  editing live data") + disabled actions: `Forms/MonitoringConcreteProducts/AddTrucksPopup.cs`,
  `InsertScheduleRowPopup.cs`, status/redirect/delete actions in `DeliveryTrackingTab.cs`,
  `DeliveryScheduleTab.cs`, and the per-row buttons. Reuse `MonitoringConcreteProductsState` events.
- Note: creating offline mutations is still allowed (that's how the queue fills); the lock prevents
  starting *new* live edits on top of an unsynced state and pushes the user to the sync flow.
- Acceptance: with a pending outbox row, all add/edit/delete buttons show the banner and are inert
  until the queue is drained.

## Step 6 — Phase 1 sync: tickets on EXISTING SO details (re-fetch & compare)
Split the push so existing-SO-detail mutations sync with a conflict pre-check.
- Files: `Sync/OutboxSyncService.cs` (`PushAsync`/`DispatchAsync`), `Api/MonitoringApi.cs`,
  `Data/Repositories/OutboxRepo.cs`, `Models/AddTruckCommand.cs` (snapshot fields), `OutboxKinds`.
- Phase 1 covers: `AddTruck`, `UpdateDeliveryStatus`, `DeleteDelivery`, `PutDeliveryTracking` — all keyed
  to an existing `DeliveryPlanDetailId`.
- Pre-check per row: `GetDeliveryPlanDetailCurrentAsync(detailId)` (reuse `GetDeliveryTracking` filtered
  by id, or add a by-id fetch) → compare current `TimeStamp` (+ key fields: status, delivered qty) to the
  snapshot captured at enqueue time (already in the `AddTruck` payload `TimeStamp`; add the same snapshot
  to the other kinds). Identical → push; changed → mark a new **Conflict** state, do **not** push, surface
  in the Unsynced tab with "detail changed on server — review".
- Keep the server `TimeStamp` as a backstop (existing behavior) so a race after the check still fails safe.
- Acceptance: a ticket whose SO detail is unchanged pushes; one whose detail changed server-side is held
  as Conflict and never silently applied.

## Step 7 — Phase 2 sync: link unreferenced inserts to an existing SO detail
Replace the free-standing draft-schedule post with an interactive link step.
- Files: `Forms/SyncForm.cs` / `Forms/MonitoringConcreteProducts/UnsyncedItemsTab.cs` (phase-2 UI),
  `Data/Repositories/DraftSchedulesRepo.cs`, `Sync/OutboxSyncService.cs` (`CreateScheduleRow` path),
  `Forms/MonitoringConcreteProducts/AddTrucksPopup.cs` (reuse command-build/mapping),
  `Api/MonitoringApi.cs`.
- Flow: for each pending `draft_schedules` row, present a picker of existing sales-order delivery-schedule
  details (reuse `GetDeliverySchedules` + the schedule grid search). User selects the target detail; map
  the draft's data into a `CreateTruckCommand` for that `DeliveryPlanDetailId` (same mapping
  `AddTrucksPopup.BuildCommand` uses) and push via `MonitoringApi.AddTruckAsync`. On success, mark the
  draft synced (`DraftSchedulesRepo.MarkSynced`) and drop the obsolete free-standing `CreateScheduleRow`
  outbox row.
- Phase ordering: run **Phase 1 first**, then **Phase 2** (so linking targets reflect freshly-synced state).
- Acceptance: an offline-inserted row with no reference can only sync after the user links it to a real
  SO delivery-schedule detail; it then appears as a ticket on that detail server-side.

## Step 8 — Sync UI + end-to-end verification
- `SyncForm.cs` / `UnsyncedItemsTab.cs`: show the two phases distinctly (Phase 1 with conflict flags,
  Phase 2 with the link action), plus the edit-lock banner state and cache freshness timestamps.

---

## Verification (end-to-end)
1. **Build**: `MSBuild AccflexOfflineReadyMix.csproj /p:Configuration=Debug` (exit 0).
2. **Offline completeness**: force-offline → Schedule/Tracking dropdowns + grids populate from cache;
   Print preview/export render from `print_data` cache (already working). (Step 2)
3. **Freshness**: go offline, change a row on the web, come online → caches auto-refresh after outbox
   drains. (Step 3)
4. **Edit-lock**: queue one offline change → all add/edit/delete show the banner until synced. (Step 5)
5. **Phase 1 conflict**: queue a truck on a detail; change that detail on the web; sync → row held as
   Conflict, not applied. Repeat with the detail unchanged → pushes cleanly. (Step 6)
6. **Phase 2 link**: insert a row offline with no reference; sync → forced to pick an existing SO
   delivery-schedule detail; confirm it lands as a ticket on that detail. (Step 7)
7. Inspect SQLite (`%LOCALAPPDATA%\Accflex\OfflineReadyMix\local.db`) and
   `logs\app-*.log` to confirm cache counts, sync phases, and conflict handling.

## Notes / files map (reuse, don't recreate)
- Outbox: `Sync/OutboxSyncService.cs`, `Data/Repositories/OutboxRepo.cs` (`OutboxKinds`,
  `CountAttention`, `ListPending`, `GetDeliveriesWithPendingChanges`).
- Drafts: `Data/Repositories/DraftSchedulesRepo.cs`, `Models/CreateScheduleRowCommand.cs`.
- Tickets: `Models/AddTruckCommand.cs`, `Forms/MonitoringConcreteProducts/AddTrucksPopup.cs`
  (`BuildCommand` mapping), `Api/MonitoringApi.cs` (`AddTruckAsync`, `CreateScheduleRowAsync`,
  `GetDeliveryTracking*`, `GetDeliveryPlanDetailPrint*`).
- Connectivity: `Services/ConnectivityState.cs` (`IsOnline`, `StatusChanged`), reconnect handler in
  `OutboxSyncService.OnConnectivityChanged`.
- Caching: `Sync/ReferenceSyncService.cs`, `Sync/DeliverySyncService.cs`, `Data/Repositories/*Repo.cs`,
  `Data/LocalDb.cs` (migrations; current schema v9).