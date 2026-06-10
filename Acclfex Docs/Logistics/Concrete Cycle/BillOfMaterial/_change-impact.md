# BillOfMaterial — Dependency & Change Impact

> **Application:** Logistics / Production
> **Cycle:** Concrete Cycle (Ready Mix)
> **Last Updated:** 2026-06-10

---

## What BillOfMaterial Depends On

### API Dependencies
| Dependency | Type | Why | Risk if Changed |
|------------|------|-----|----------------|
| `Material` | Hard — `ProductMaterialId` FK | The finished concrete product this BOM produces | High — if material's `ManufacturerList` or `ManufactureUsageId` changes, BOM validation breaks |
| `Units` | Hard — `UnitId` FK | Production unit of measure (m³) | Medium — unit must exist in material's unit list |
| `SalesOrganization` | Soft — `SalesOrganizationId` FK (nullable) | Required only when BOM has scrap lines | Low — only needed for scrap sales org assignment |
| `InventoryManagement` | Hard — via `BillOfMaterialInventoryManagement` | Links BOM to batching plants | Critical — delivery plan validation checks this junction |
| `BillOfMaterialUsage` (enum-class) | Hard — `BillOfMaterialUsageId` | Must be `ReadyMix` for concrete cycle | High — controls date validation logic and plant type validation |
| `BillOfMaterialType` (enum-class) | Hard — `BillOfMaterialTypeId` | Categorizes the BOM | Low for concrete cycle |

---

## What Depends On BillOfMaterial

### Ring 1 — Direct Dependents

| Dependent | Repo | How It Uses This | Impact of Breaking Change |
|-----------|------|-----------------|--------------------------|
| `SalesOrderDetail` | API | `BillOfMaterialId` FK — selects which recipe applies to the concrete order | **Critical** — every concrete order line needs a BOM |
| `DeliveryPlanDetailBase` | API | `BillOfMaterialId` FK — validated at dispatch time via `ValidateBomRelatedToManagementWithinDate` | **Critical** — dispatch blocked if BOM is inactive or mismatched with plant |
| `DeliveryPlanDetailRawMaterial` | API | Lists actual raw material quantities drawn from the BOM | **High** — raw material lines are populated from BOM details |

### Ring 2 — Indirect Dependents

| Dependent | Repo | Coupling Type | Impact |
|-----------|------|--------------|--------|
| `RoutingAggregate` | API | References BOM for routing/operation assignment | **Medium** — if BOM raw materials change, routing needs redistribution (event fired) |
| Batch ticket / dispatch report | DB/Reports | Reads BOM details for print output | **Medium** — schema changes affect report output |
| `BillOfMaterialTechnicalSpecifications` | API child | Spec groups copied to SO and DP details | **Medium** — spec structure changes need mirroring on SalesOrderDetail and DeliveryPlanDetail |

### Ring 3 — Risk Flags

- ⚠️ **BOM validity at dispatch time** — `ValidateBomRelatedToManagementWithinDate` uses only `IsActive` and `BillOfMaterialInventoryManagements`. If a BOM is deactivated after a Sales Order is created but before the delivery plan, the delivery is blocked at dispatch time, not at order time. This is intentional but can surprise operators.
- ⚠️ **`BillOfMaterialUsage.ReadyMix.Id`** is compared numerically in `BillOfMaterial.Create:116` and in multiple validators. If the enum-class ID values are ever renumbered, all these checks silently break.
- ⚠️ **Technical Specs are copied by reference (IDs only)** — `BillOfMaterialTechnicalSpecificationsGrade` stores only `GradeId`. If the Grade lookup table entry is deleted, the spec becomes a dangling reference with no error.
- ⚠️ **`RoutingBomRedistributeNeededEvent`** is fired when raw material quantities change on update. If you remove this event or its handler, routing assignments silently become stale.

---

## Change Scenarios

### Scenario: Deactivate a BOM currently used in open Sales Orders
| What to check | Priority |
|--------------|---------|
| All open SOs with this `BillOfMaterialId` on any detail | 🔴 Critical — their delivery plans will be blocked at dispatch |
| Notify dispatchers before deactivation | 🟠 High |

### Scenario: Add a new raw material to an existing BOM
| What to update | Priority |
|----------------|---------|
| `BillOfMaterialDetail` — add new line | 🔴 Critical |
| DB migration | 🔴 Critical |
| `RoutingBomRedistributeNeededEvent` fires automatically | 🟠 High — routing needs re-assignment |
| Existing open Delivery Plans that already have raw material quantities loaded | 🟠 High — old trips use old recipe; only new trips get new recipe |

### Scenario: Add a new Technical Spec field to BillOfMaterialTechnicalSpecifications
| What to update | Priority |
|----------------|---------|
| `BillOfMaterialTechnicalSpecifications.cs` | 🔴 Critical |
| `BillOfMaterialTechnicalSpecificationsPoco.cs` | 🔴 Critical |
| DB migration | 🔴 Critical |
| `SalesOrderDetail` — spec is copied here | 🟠 High — needs new column + copy logic |
| `DeliveryPlanDetailBase` — spec also copied here | 🟠 High |
| Angular BOM form, SO form, DP form | 🟡 Medium |

### Scenario: Change BillOfMaterialUsage enum ID values
| What to update | Priority |
|----------------|---------|
| `BillOfMaterial.Create:116` — hardcoded `ReadyMix.Id` check | 🔴 Critical |
| `BillOfMaterial.Validate:385,389,414,419,424` — usage type guards | 🔴 Critical |
| DB data migration for existing BOM records | 🔴 Critical |
