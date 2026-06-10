
## Context

The AccflexOfflineReadyMix desktop app lets field operators record ready‑mix delivery
work while offline. When connectivity returns, those local transactions must be pushed to
the Logistics server. The user described **two types** of unsynced transactions and a
conflict rule. Exploration shows most of the machinery already exists — this plan fills
the **two real gaps** and refines conflict handling, rather than building from scratch.

### What already exists (verified in code)

**Type 1 — reference‑bound** (a truck/edit made against a real Sales‑Order delivery
schedule) is **fully implemented**:
- Outbox kinds `AddTruck`, `UpdateDeliveryStatus`, `DeleteDelivery`, `PutDeliveryTracking`
  ([`OutboxKinds`](AccflexOfflineReadyMix/Data/Repositories/OutboxRepo.cs)) replay against
  the **same** online endpoints used while online — no separate endpoint needed.
- A Phase‑1 conflict pre‑check re‑reads the server `TimeStamp` (rowversion) for the target
  `DeliveryPlanDetail` and compares to the snapshot captured at enqueue time
  ([`OutboxSyncService.CheckConflictAsync`](AccflexOfflineReadyMix/Sync/OutboxSyncService.cs:244)).
  Mismatch ⇒ row held as `Conflict` ([`OutboxRepo.MarkConflict`](AccflexOfflineReadyMix/Data/Repositories/OutboxRepo.cs:335)).
- The [`UnsyncedItemsTab`](AccflexOfflineReadyMix/Forms/MonitoringConcreteProducts/UnsyncedItemsTab.cs)
  shows conflicts in orange with a Retry / Cancel / View‑payload context menu.

**Type 2 — unreferenced** (a row inserted offline via
[`InsertScheduleRowPopup`](AccflexOfflineReadyMix/Forms/MonitoringConcreteProducts/InsertScheduleRowPopup.cs)
with no Sales Order) is **partially implemented — link‑to‑existing only**:
- Stored in `draft_schedules` + a `CreateScheduleRow` outbox row, which
  [`OutboxSyncService.PushAsync`](AccflexOfflineReadyMix/Sync/OutboxSyncService.cs:121)
  deliberately **skips** ("reserved for Phase 2").
- [`LinkDraftSchedulePopup`](AccflexOfflineReadyMix/Forms/MonitoringConcreteProducts/LinkDraftSchedulePopup.cs)
  + [`DraftLinkService.LinkAndPushAsync`](AccflexOfflineReadyMix/Sync/DraftLinkService.cs:31)
  let the user pick an **existing** SO delivery schedule, map the draft to a
  `CreateTruckCommand`, push via `AddTruck`, mark the draft synced, and discard the
  `CreateScheduleRow` row.

### The two gaps to close

1. **Type 2 "insert a new line"** — today the user can only link a draft to a *pre‑existing*
   SO schedule. If the chosen Sales Order has **no matching detail line**, there is no way
   to insert one. The user chose *"Both, user chooses per draft"* — so we add an
   **insert‑new‑line** path + a **new backend endpoint**.
2. **Type 1 conflict resolution UX** — a `Conflict` row is currently a dead end (Retry just
   re‑conflicts; Cancel blind‑discards). The user chose *"Hold for review, remote‑win
   default"* — so we make the conflict actionable with **remote‑wins as the default** and a
   deliberate **override** escape hatch.

---

## Part A — Type 1 conflict resolution (remote wins, with review)

Goal: when the server changed the detail while offline, the **default** resolution discards
the local edit (remote wins); the user may explicitly override to force their version.

1. **Backend force path** — add an optional `force` flag through the push so an override can
   bypass the Phase‑1 pre‑check:
   - [`OutboxSyncService.PushOneAsync`](AccflexOfflineReadyMix/Sync/OutboxSyncService.cs:138)
     and `PushSingleAsync` gain a `bool bypassConflictCheck` param; when set, skip
     `CheckConflictAsync` and dispatch directly. The server's own `TimeStamp` check remains
     the final backstop (a true stale write still 409s and lands as `Failed`).
2. **UnsyncedItemsTab conflict actions** — in the row context menu
   ([UnsyncedItemsTab.cs:193](AccflexOfflineReadyMix/Forms/MonitoringConcreteProducts/UnsyncedItemsTab.cs:193)),
   when the focused row is `Conflict`, surface two explicit items:
   - **"Keep server version (discard mine)"** — *primary/default*. Reuses the existing
     `CancelFocused()` rollback logic (revert optimistic truck via
     `TrucksRepo.MarkFailedByCorrelation`, then `OutboxRepo.Discard`). Reword the confirm
     text to make "the server copy wins" explicit.
   - **"Override — push my version"** — secondary, guarded by a clear warning dialog. Calls
     `PushOneAsync(id, bypassConflictCheck: true)`.
3. **Optional auto‑resolve note** — leave the auto‑drain
   ([`OnConnectivityChanged`](AccflexOfflineReadyMix/Sync/OutboxSyncService.cs:60)) as‑is:
   it still routes conflicts to the held `Conflict` state for review (matching the chosen
   policy), rather than silently discarding.

No new outbox kind and no schema change are needed for Part A.

---

## Part B — Type 2 "insert new line under Sales Order"

### B.1 New backend endpoint (Logistics API)

A new endpoint inserts a ready‑mix delivery line into an **existing** Sales Order and returns
enough context for the desktop to then add the truck via the existing `AddTruck` flow — keeping
symmetry with `DraftLinkService.LinkAndPushAsync`.

- **Route:** `POST api/MonitoringConcreteProducts/InsertOfflineDeliveryIntoSalesOrder`
  in [`MonitoringConcreteProductsController`](../AccFlex_Cloud_ERP/AccFlex_Cloud_ERP/Logistics/Logistics.API/ApiControllers/Production/MonitoringConcreteProductsController/MonitoringConcreteProductsController.cs).
- **Command:** `InsertOfflineDeliveryIntoSalesOrderCommand` (new), mirroring the CQRS shape
  of [`CreateTruckCommand`](../AccFlex_Cloud_ERP/AccFlex_Cloud_ERP/Logistics/Logistics.Application/Production/MonitoringConcreteProducts/Commands/DeliverySchedules/CreateTruck/CreateTruckCommand.cs).
  Request fields (sourced from the draft + the chosen SO):
  - `SalesOrderId` (required), optional `SalesOrderDetailId` (link into an existing line vs.
    create a new line), `MaterialId`, `Quantity`, `DeliveryScheduleDate`,
    `CustomerDeliveryAddressId`/`DeliveryAddressName`, cement spec fields
    (`CementType`/`CementContent`/`CementStrength`), `SlumpLevelIdList`,
    `AddedMaterialsIdList`, `WaterAndIceIdList`, and the SO‑level `TimeStamp`.
- **Handler behavior** — reuse existing Sales Order aggregate logic. Mirror the patterns in
  the SalesOrder detail/delivery‑schedule writers
  (`UpdateSalesOrderDetailDeliveryScheduleCommand`, `CreateSalesOrderCommand` under
  `Logistics.Application/.../Sales/...`):
  1. Load the `SalesOrder` (+ details + `SalesOrderDetailDeliveyScheduleList`) with concurrency
     check against `TimeStamp`.
  2. If `SalesOrderDetailId` provided ⇒ append a new `SalesOrderDetailDeliverySchedule` to that
     detail; else create a new `SalesOrderDetail` (material, qty, default pricing/conditions
     consistent with the SO) and its first delivery schedule.
  3. Persist and **return the new `SalesOrderDetailDeliveryScheduleId`** plus `DeliveryPlanId`
     (if applicable) and the refreshed `TimeStamp`.
- **Response:** `ResponseValidationWrapper<InsertOfflineDeliveryResultDto>` with
  `SalesOrderDetailDeliveryScheduleId`, `DeliveryPlanId`, `TimeStamp` — exactly the fields
  `DraftLinkService.BuildCommand` needs to construct the follow‑up `CreateTruckCommand`.

> Decision captured: the new endpoint only *inserts the line/schedule*; the existing
> `AddTruck` endpoint still creates the truck/`DeliveryPlanDetail`. This minimizes new
> backend surface and reuses the proven path. (Alternative — one atomic endpoint that also
> creates the truck — is possible but heavier; not recommended for the first cut.)

### B.2 Desktop: insert‑new‑line path

- **`MonitoringApi`** — add `InsertOfflineDeliveryIntoSalesOrderAsync(commandJson, ct)`
  following the existing `PostAndExtractIdAsync` pattern in
  [MonitoringApi.cs:230](AccflexOfflineReadyMix/Api/MonitoringApi.cs:230) but returning the
  result DTO (schedule id + DeliveryPlanId + TimeStamp), not just an id.
- **`DraftLinkService`** — add
  `InsertNewLineAndPushAsync(DraftScheduleEntry draft, SalesOrderPick targetSalesOrder, ct)`
  next to the existing [`LinkAndPushAsync`](AccflexOfflineReadyMix/Sync/DraftLinkService.cs:31):
  1. Call the new endpoint with the draft's data + chosen `SalesOrderId`.
  2. Synthesize a `DeliveryScheduleForListDto` from the result (schedule id, DeliveryPlanId,
     TimeStamp, material/qty) and reuse the existing `BuildCommand` → `AddTruckAsync` →
     `MarkSynced` → `DiscardCreateScheduleRowOutboxEntry` tail. (Refactor that tail into a
     shared private helper so both paths share it.)
- **`LinkDraftSchedulePopup`** — extend the existing popup so the user "chooses per draft":
  keep the current **"Link to existing schedule"** action, and add a second footer button
  **"Insert as new line under SO…"** that requires only a Sales Order selection (SO search,
  not a specific schedule) and calls `InsertNewLineAndPushAsync`. Localize EN/AR via
  `AppUiCulture.Select` to match the file's existing strings.

No SQLite schema change is required — drafts already carry everything needed; only the
target resolution differs.

---

## Critical files

| Area | File |
|------|------|
| Conflict push + force flag | [Sync/OutboxSyncService.cs](AccflexOfflineReadyMix/Sync/OutboxSyncService.cs) |
| Conflict UX | [Forms/MonitoringConcreteProducts/UnsyncedItemsTab.cs](AccflexOfflineReadyMix/Forms/MonitoringConcreteProducts/UnsyncedItemsTab.cs) |
| Type 2 link engine | [Sync/DraftLinkService.cs](AccflexOfflineReadyMix/Sync/DraftLinkService.cs) |
| Type 2 link UI | [Forms/MonitoringConcreteProducts/LinkDraftSchedulePopup.cs](AccflexOfflineReadyMix/Forms/MonitoringConcreteProducts/LinkDraftSchedulePopup.cs) |
| Desktop API wrapper | [Api/MonitoringApi.cs](AccflexOfflineReadyMix/Api/MonitoringApi.cs) |
| New backend endpoint | `Logistics.API/ApiControllers/Production/MonitoringConcreteProductsController/` + new command/handler under `Logistics.Application/Production/MonitoringConcreteProducts/Commands/` |
| Backend pattern source | `CreateTruckCommand`, `UpdateSalesOrderDetailDeliveryScheduleCommand` |

## Verification (end‑to‑end)

1. **Type 1 conflict, remote wins:** Online, open a delivery schedule; go offline; edit a
   truck/status (queues outbox row). Separately bump the same `DeliveryPlanDetail` server‑side.
   Reconnect ⇒ row lands as `Conflict` (orange) in Unsynced Items. Choose **"Keep server
   version"** ⇒ row discarded, local optimistic effect reverted, grid reflects server copy.
2. **Type 1 override:** Repeat, choose **"Override — push my version"** ⇒ pushes; if server
   truly stale it returns and shows `Failed` with the server message.
3. **Type 2 insert‑new‑line:** Offline, `InsertScheduleRowPopup` to create a draft for a
   material not present on any SO line. Reconnect ⇒ Unsynced Items → "Link offline inserts…"
   → pick the draft → **"Insert as new line under SO…"** → choose a Sales Order → confirm.
   Verify on the web monitoring module (Module 621) that a new detail line + delivery
   schedule + truck appears under that Sales Order; the draft and its `CreateScheduleRow`
   outbox row are gone locally.
4. **Type 2 link‑existing regression:** Confirm the original "Link to existing schedule" path
   still works unchanged.
5. **Build:** `AccflexOfflineReadyMix` (x64, .NET 4.7.2) compiles; new files registered in
   `.csproj`. Backend solution builds; new endpoint reachable via Swagger.