# DeliveryPlanDetailFromSalesOrder — Dependency & Change Impact

> **Application:** Logistics / Production
> **Cycle:** Concrete Cycle (Ready Mix)
> **Last Updated:** 2026-06-10

---

## What DeliveryPlanDetailFromSalesOrder Depends On

### API Dependencies
| Dependency | Type | Why | Risk if Changed |
|------------|------|-----|----------------|
| `SalesOrderDetail` | Hard — `SalesOrderDetailId` FK | Validates quantity limits; copied at redirect | Critical — quantity ceiling check uses SO detail fields |
| `BillOfMaterial` | Hard — `BillOfMaterialId` FK | Recipe for the batch; validated at create/update | Critical — BOM must be active and plant-linked |
| `InventoryManagement` (×2) | Hard — `RawMaterialInventoryManagementId` + `ReadyMixInventoryManagementId` | Source (raw) and target (finished product) plants | Critical — BOM validity check, scrap account lookup |
| `Car` | Hard — `CarId` FK | Assigned mixer truck | High — max load check uses `car.MaxLoad` and `car.UnitId` |
| `Driver` | Soft — `DriverId` FK (nullable) | Truck driver | Medium — duplicate driver check across mixer/pump |
| `Customer` (redirect) | Soft — `RedirectedCustomerId` nullable | Target customer for a redirected trip | High — must exist and be active; cannot be same as original customer |
| `SalesOrder` (redirect) | Soft — `RedirectedSalesOrderId` nullable | Linked existing SO for redirect | High — must be confirmed, same sales org, no prior deliveries |
| `GoodsIssue` (cross-BC) | Soft — `GoodIssueNumber` (int?) | Set by Inventory BC after raw material consumption | Critical — journal cost calculation divides by recipe quantity based on GI lines |
| `Journal` (cross-BC) | Soft — `JournalGuidNumber` (string) | GL journal linked to this trip | Critical — journal update/void uses this link |
| `SystemConfiguration` | Hard — for journal creation | Provides `MaterialCostAccountId` and `WorkInProgressAccountId` | High — both accounts must be configured for journal to post |
| `Station` | Soft — `StationId` nullable | Batching station at the plant | Low — informational; batch ticket reference |

---

## What Depends On DeliveryPlanDetailFromSalesOrder

### Ring 1 — Direct Dependents

| Dependent | Repo | How It Uses This | Impact of Breaking Change |
|-----------|------|-----------------|--------------------------|
| `DeliveryPlanDetailRawMaterial` | API | Children owned by this detail; lists actual raw material quantities | **Critical** — raw material lines are core to Goods Issue creation and costing |
| Concrete spec child lists | API | 9 spec lists (Grade, Slump, MaxAgg, Temp, AggMix, Admixture, Water, AddedMat, Mixture) | **Medium** — informational copies from SO and BOM |
| `DeliveryPlanDetailCondition` | API | Pricing conditions for redirect sales orders (SPrice etc.) | **High** — required when creating a new SO via redirect |

### Ring 2 — Indirect Dependents

| Dependent | Repo | Coupling Type | Impact |
|-----------|------|--------------|--------|
| `GoodsIssue` (Inventory BC) | Cross-BC | Created based on detail's raw material list; GoodIssueNumber stamped back | **Critical** — if raw material structure changes, GI creation breaks |
| `Journal` (GL BC) | Cross-BC | Journal entries calculated from `DeliveredQuantity / RecipeQty × GI cost` | **Critical** — cost formula is sensitive to both quantity fields |
| `DriverCommission` | API | Commission calculation uses trip data | **Medium** — `UpdateCommissionData` writes back commission amount |
| `SalesOrder.TotalActualIssuedQuantityInMinimumUnit` | API | Updated by domain event after delivery | **High** — delivered quantity field drives SO completion status |
| Redirected `DeliveryPlanFromSalesOrder` (auto-created) | API | `GetRedirectInstance` clones this detail's data | **High** — any field added to detail needs mirroring in `GetRedirectInstance` + `SetRedirectInstance` |

### Ring 3 — Risk Flags

- ⚠️ **`ClaimCustomerWithDelivedQuantity` flag** — when `true`, uses `DeliveredQuantity` (actual) for journal costing instead of `ReceipeQuantity` (recipe). This flag is set to `true` on create (`Create:161`) and conditionally preserved on update. Misunderstanding this flag leads to wrong COGS entries.
- ⚠️ **Journal cost formula** (`DeliveredQty / RecipeQty × GICost`) — if `RecipeQty` is zero (e.g., a data issue), this causes a divide-by-zero at runtime. There is no guard in the domain code.
- ⚠️ **`GetEqualityComponents`** determines if a detail "changed" for optimistic lock purposes. If a new field needs to participate in change detection, it must be added to this method; otherwise updates that only change that field will bypass the concurrency check silently.
- ⚠️ **Redirect clone logic** in `SetRedirectInstance` manually copies every field. Any new field added to `DeliveryPlanDetailBase` or `DeliveryPlanDetailFromSalesOrder` must be explicitly added to `SetRedirectInstance` and `GetRedirectInstance`, or the cloned redirect trip will have missing data.
- ⚠️ **`StationBatchTicket`** — stored as a free-text string with no format validation. If a batch ticket numbering format needs to change or be validated, this is a gap.
- ⚠️ **9 concrete spec lists** are all parallel structures (Grade, Slump, MaxAgg, Temp, AggMix, Admixture, Water, Ice, Mixture) that follow identical patterns. They exist on `SalesOrderDetail`, `DeliveryPlanDetailBase`, and `BillOfMaterialTechnicalSpecifications`. Any new spec type requires changes in all three places.

---

## Change Scenarios

### Scenario: Add a new concrete spec type (e.g., "FiberContent")
| What to update | Priority |
|----------------|---------|
| New lookup table + migration | 🔴 Critical |
| `DeliveryPlanDetailBase.cs` — new list property | 🔴 Critical |
| `DeliveryPlanDetailFromSalesOrder.Create` — populate from poco | 🔴 Critical |
| `DeliveryPlanDetailFromSalesOrder.Update` — clear and re-add | 🔴 Critical |
| `SetRedirectInstance` — copy to redirect clone | 🔴 Critical |
| `SalesOrderDetail.cs` — same spec list | 🟠 High |
| `BillOfMaterialTechnicalSpecifications.cs` — same spec list | 🟠 High |
| Angular forms: SO, DP detail, BOM tech spec | 🟡 Medium |

### Scenario: Change the journal costing formula (e.g., use recipe qty instead of delivered qty)
| What to update | Priority |
|----------------|---------|
| `DeliveryPlanDetailBase.CreateJournal` — `deliveredQuantityCost` calculation | 🔴 Critical |
| `DeliveryPlanDetailBase.UpdateJournal` — same formula | 🔴 Critical |
| `DeliveryPlanBase.AddJournalsToDeliveryPlan` — passes `topLevelReceipeQuantity` | 🟠 High |
| `DeliveryPlanBase.UpdateJournalsToDeliveryPlan` — redirect chain traversal | 🟠 High |

### Scenario: Add a new terminal DetailStatus (e.g., "Cancelled")
| What to update | Priority |
|----------------|---------|
| `DeliveryPlanDetailStatus` enum-class | 🔴 Critical |
| `IsValidTransaction` state machine | 🔴 Critical |
| `DeliveryPlanDetailBase.ValidateUpdatesForOnWayStatus` — terminal status list | 🔴 Critical |
| `CreateJournalActionList` — add journal handling for new status | 🟠 High |
| `ValidateUpdates` — includes this status in "can only change from NotStarted" | 🟠 High |
| `SalesOrder.TotalActualIssuedQuantity` event — exclude new status from count | 🟠 High |
| Angular status badge + transition buttons | 🟡 Medium |

### Scenario: Change the redirect clone behavior (what gets copied to a redirect trip)
| What to update | Priority |
|----------------|---------|
| `DeliveryPlanDetailBase.SetRedirectInstance` | 🔴 Critical |
| `DeliveryPlanDetailFromSalesOrder.GetRedirectInstance` | 🔴 Critical |
| `DeliveryPlanFromSalesOrder.RedirectDeliveryPlanDetails` | 🟠 High |
