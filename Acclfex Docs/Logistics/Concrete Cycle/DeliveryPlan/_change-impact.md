# DeliveryPlanFromSalesOrder — Dependency & Change Impact

> **Application:** Logistics / Production
> **Cycle:** Concrete Cycle (Ready Mix)
> **Last Updated:** 2026-06-10

---

## What DeliveryPlanFromSalesOrder Depends On

### API Dependencies
| Dependency | Type | Why | Risk if Changed |
|------------|------|-----|----------------|
| `SalesOrder` | Hard — `SalesOrderId` FK | Source of the delivery; drives validation and event firing | Critical — most business rules read SO state |
| `Customer` + `CustomerDeliveryAddress` | Hard — FKs | Delivery destination | High — address is required and drives redirect customer validation |
| `Car` (Truck / Pump) | Hard — via details and pumps | Physical vehicles assigned to each trip | High — car category drives validation (Truck vs Pump) |
| `Driver` | Hard — via detail `DriverId` | Assigned driver per trip | Medium — driver duplicate check (mixer vs pump) |
| `Project` | Soft — `ProjectId` nullable | Optional project assignment | Low — only validated that it belongs to the same customer |
| `BillOfMaterial` | Hard — via each detail | Recipe for the concrete batch | Critical — each trip detail requires a valid, active BOM |
| `InventoryManagement` | Hard — two per detail (raw materials + ready mix) | Batching plant and finished goods plant | Critical — BOM validity and scrap account are checked against these |
| `AccountingPeriod` | Hard — runtime check | Must be open for the plan date | High — a closed accounting period blocks all creates, updates, and deletes |
| `SystemConfiguration` | Hard — for journals | Provides `MaterialCostAccountId` and `WorkInProgressAccountId` | High — journal creation fails if these accounts are not configured |

---

## What Depends On DeliveryPlanFromSalesOrder

### Ring 1 — Direct Dependents

| Dependent | Repo | How It Uses This | Impact of Breaking Change |
|-----------|------|-----------------|--------------------------|
| `DeliveryPlanDetailFromSalesOrder` | API | Children owned by this aggregate | **Critical** — details cannot exist without their parent plan |
| `DeliveryPlanPumpFromSalesOrder` | API | Pump assignments owned by this aggregate | **High** — pump validation checks the detail list of the parent plan |
| `GoodsIssue` (Inventory BC) | Cross-BC | Created by background job triggered by `CreateGoodIssueOnDeliveryPlanEvent` | **Critical** — GoodIssueNumber is stamped back onto each detail |
| `Journal` (GeneralLedger BC) | Cross-BC | Created by `CreateDeliveryPlanJournalDomainEvent`; linked via `JournalGuidNumber` on each detail | **Critical** — voiding journals on delete depends on this link |

### Ring 2 — Indirect Dependents

| Dependent | Repo | Coupling Type | Impact |
|-----------|------|--------------|--------|
| `SalesOrder.TotalActualIssuedQuantityInMinimumUnit` | API | Updated by `ReCalculateSalesOrderTotalActualIssuedQuantityAndUpdateQuantityEvent` | **High** — SO delivered quantity reflects all non-scrapped/non-redirected DP details |
| `Invoice` (Logistics BC) | API | Delete guard checks if SO is invoiced | **High** — an invoiced SO blocks delivery plan deletion |
| Redirected `DeliveryPlanFromSalesOrder` (auto-created) | API | Auto-generated when a truck is redirected; `RedirectDeliveryPlanDetailId` links back | **High** — delete cascade logic must traverse this redirect chain |
| Redirected `SalesOrder` (auto-created) | API | Created by `CreateRedirectedSalesOrderForDeliveryPlanDomainEvent` | **High** — these SOs inherit structure from the original SO |
| `DriverCommission` | API | Commission is stamped on each detail after delivery | **Low** — separate aggregate, updated via `UpdateCommissionData` |

### Ring 3 — Risk Flags

- ⚠️ **Redirect chain traversal** — `UpdateJournalsToDeliveryPlan` walks the `RedirectDeliveryPlanDetailId` chain with cycle detection. This is a manually managed linked list. If a redirect is deleted without resetting the pointer, the chain breaks silently.
- ⚠️ **`IsAutoCreated` flag** is the only indicator that a delivery plan was system-generated. If this flag gets reset (e.g., by a manual DB update), auto-created plans lose their restrictions and can be freely edited.
- ⚠️ **`GuidNumber`** on the delivery plan header is the idempotency key for all background jobs. It is passed to every domain event. Regenerating it after creation would cause duplicate journal entries.
- ⚠️ **Optimistic concurrency** — `TimeStamp` (rowversion) is checked on update and delete. If the UI sends a stale timestamp, the user gets a conflict error. Any batch update tool that bypasses this check will silently overwrite concurrent changes.
- ⚠️ **Background jobs are fire-and-forget** — all domain events enqueue Hangfire jobs. If a job fails after the plan is saved, the plan exists but has no Goods Issue or journal. The plan appears "stuck" with no visible error in the UI.

---

## Change Scenarios

### Scenario: Add a new field to the Delivery Plan header
| What to update | Priority |
|----------------|---------|
| `DeliveryPlanBase.cs` or `DeliveryPlanFromSalesOrder.cs` | 🔴 Critical |
| DB migration | 🔴 Critical |
| `GetEqualityComponents()` — if field participates in optimistic lock comparison | 🟠 High |
| Auto-create clone logic in `RedirectDeliveryPlanDetails` | 🟠 High |
| Angular form + Angular service | 🟡 Medium |

### Scenario: Change accounting period validation rules
| What to update | Priority |
|----------------|---------|
| `DeliveryPlanBase.ValidateAccountingPeriodToChange` | 🔴 Critical |
| `DeliveryPlanBase.CommonValidateForDelete` | 🔴 Critical |
| `DeliveryPlanBase.ValidateJournal` | 🔴 Critical |

### Scenario: Add a new domain event on delivery plan creation
| What to update | Priority |
|----------------|---------|
| `DeliveryPlanFromSalesOrder.CreateEvents` or `CommonCreateEvents` | 🔴 Critical |
| `DeliveryPlanFromSalesOrder.UpdateEvents` — add matching update event | 🟠 High |
| `CommonDeleteEvents` — add matching delete/void event | 🟠 High |
| Background job handler registration | 🔴 Critical |

### Scenario: Change the "no future date" rule for all delivery plans
| What to update | Priority |
|----------------|---------|
| `DeliveryPlanFromSalesOrder.ValidateDeliveryPlan:492` | 🔴 Critical |
| `DeliveryPlanOnCostCenter` and `DeliveryPlanFromProject` — same validation exists | 🟠 High |
| Angular date picker constraints | 🟡 Medium |
