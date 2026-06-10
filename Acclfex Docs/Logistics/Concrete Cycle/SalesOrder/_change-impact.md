# SalesOrder — Dependency & Change Impact

> **Application:** Logistics / Production
> **Cycle:** Concrete Cycle (Ready Mix)
> **Last Updated:** 2026-06-10

---

## What SalesOrder Depends On

### API Dependencies
| Dependency | Type | Why | Risk if Changed |
|------------|------|-----|----------------|
| `Customer` | Hard — FK | Every SO belongs to a customer | High — removing customer FK breaks SO creation |
| `SalesOrganization` | Hard — FK | Defines the selling entity | High — drives price procedures and redirect validation |
| `BillOfMaterial` | Hard — FK on `SalesOrderDetail.BillOfMaterialId` | Links the concrete recipe to the order line | High — delivery plans check BOM validity against SO |
| `SalesOrderType` (enum-class) | Hard — `SalesOrderTypeId` | Determines dispatch rules (Rush vs Standard) | High — Rush type has special update/delete restrictions throughout the delivery cycle |

---

## What Depends On SalesOrder

### Ring 1 — Direct Dependents

| Dependent | Repo | How It Uses This | Impact of Breaking Change |
|-----------|------|-----------------|--------------------------|
| `DeliveryPlanFromSalesOrder` | API | `SalesOrderId` FK — reads `IsConfirm`, `StopIssuing`, `SalesOrderTypeId` | **Critical** — delivery plan creation and all dispatch logic depends on SO state |
| `DeliveryPlanDetailFromSalesOrder` | API | `SalesOrderDetailId` FK — validates quantity limits against SO detail | **Critical** — quantity guard uses SO detail's `GrossQuantity` + `AllowedIncreasePercent` |
| `Invoice` | API | `SalesOrderId` FK — invoice is generated per SO | **Critical** — invoice creation fails without valid SO reference |
| `SalesOrderDetailDeliveySchedule` | API | Child of SalesOrderDetail — constrains delivery volumes per window | **High** — delivery plan validates quantity against schedule window |

### Ring 2 — Indirect Dependents

| Dependent | Repo | Coupling Type | Impact |
|-----------|------|--------------|--------|
| `ReCalculateSalesOrderTotalActualIssuedQuantityAndUpdateQuantityEvent` | API domain event | Fires on every delivery plan create/update to update SO delivered quantities | **High** — if SO detail quantity fields change shape, event handler breaks |
| `CreateRedirectedSalesOrderForDeliveryPlanDomainEvent` | API domain event | Creates a new SO from a redirected truck trip | **Medium** — clones SO structure; changes to SO fields need mirroring in clone logic |
| Delivery plan delete guard | API | Checks if `InvoicedSalesOrderId` overlaps with SO | **Medium** — delete protection relies on SO id being stable |

### Ring 3 — Risk Flags

- ⚠️ **`SalesOrderTypeId == RushCommercial`** is checked by string-based comparison in multiple places (`DeliveryPlanFromSalesOrder.cs:83`, `:189`, `:628`). Adding a new SalesOrderType that should behave like Rush requires updates in all three locations.
- ⚠️ **`AllowedIncreasePercent`** on `SalesOrderDetail` is used in the quantity ceiling formula. If this field is renamed or its semantics change (e.g., absolute vs. fractional), delivery plan quantity validation silently breaks.
- ⚠️ **`GuidNumber`** on both header and detail is used as the cross-aggregate reference key for domain events. It must never be regenerated after creation.

---

## Change Scenarios

### Scenario: Add a new SalesOrderType
| What to update                                                       | Priority    |
| -------------------------------------------------------------------- | ----------- |
| `SalesOrderType` enum-class                                          | 🔴 Critical |
| `DeliveryPlanFromSalesOrder` — 3 places checking `RushCommercial.Id` | 🟠 High     |
| Application layer validators that filter by type                     | 🟡 Medium   |

### Scenario: Add a new field to SalesOrderDetail
| What to update | Priority |
|----------------|---------|
| `SalesOrderDetail.cs` | 🔴 Critical |
| DB table migration | 🔴 Critical |
| `CreateRedirectedSalesOrderForDeliveryPlanDomainEvent` handler — clones SO detail | 🟠 High |
| Angular Sales Order form | 🟡 Medium |

### Scenario: Change quantity tolerance logic (AllowedIncreasePercent)
| What to update | Priority |
|----------------|---------|
| `DeliveryPlanDetailFromSalesOrder.ValidateDetailQuantityRelatedToSalesOrderQuantity` | 🔴 Critical |
| `ValidateOldSalesOrderToLink` — same formula for redirect path | 🟠 High |
| Angular SO form (displays tolerance to user) | 🟡 Medium |
