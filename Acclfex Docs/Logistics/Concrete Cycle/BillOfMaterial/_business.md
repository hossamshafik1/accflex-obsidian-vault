# BillOfMaterial — Business Documentation

> **Layer:** API Domain — Aggregate Root
> **Application:** Logistics / Production
> **Cycle:** Concrete Cycle (Ready Mix)
> **Source Path:** `Logistics.Production.Logic/BillOfMaterialAggregate/BillOfMaterial.cs`
> **Last Updated:** 2026-06-10

---

## What Is This?

A Bill of Material (BOM) is the concrete mix recipe. It defines exactly which raw materials — cement, sand, gravel, water, ice, admixtures — are required and in what quantities to produce one unit of a ready-mix concrete product. Every mixer truck trip carries a BOM reference so the batching plant knows exactly what to load.

For Ready Mix, the BOM also carries a set of **Technical Specifications** that describe the expected quality profile: compressive strength grade, slump level, maximum aggregate size, temperature range, and admixture dosage. These specs travel with the order from Sales Order through to the Delivery Plan detail.

---

## Business Rules

### Rule 1: BOM Must Be Active for Use in Delivery Plans
- **Plain language:** Only BOMs with `IsActive = true` can be referenced on delivery plan details. An inactive BOM cannot be dispatched.
- **Trigger:** When creating or updating a `DeliveryPlanDetail`.
- **Violation result:** Error: `BomError` / `BillOfMaterialNotValid`
- **Source:** `DeliveryPlanDetailBase.ValidateBomRelatedToManagementWithinDate` — checks `bom.IsActive`

### Rule 2: BOM Must Be Linked to the Correct Batching Plant
- **Plain language:** A BOM is valid only for the inventory management (batching plant) it was assigned to. You cannot use a BOM from Plant A for a delivery dispatched from Plant B.
- **Trigger:** When creating or updating a `DeliveryPlanDetail`.
- **Violation result:** Error: `BomError`
- **Source:** `DeliveryPlanDetailBase.ValidateBomRelatedToManagementWithinDate:538` — checks `BillOfMaterialInventoryManagements` contains the requested `InventoryManagementId`

### Rule 3: All Inventory Managements Must Be ReadyMix Type
- **Plain language:** When creating a ReadyMix BOM, every batching plant you assign it to must be configured as a ReadyMix plant. You cannot mix ReadyMix and Manufacturing plants on the same BOM.
- **Trigger:** BOM creation or update.
- **Violation result:** Error: `AllInventoryManagementsMustBeReadyMixManagements`
- **Source:** `BillOfMaterial.Validate:419`

### Rule 4: ReadyMix BOMs Skip Date Validity
- **Plain language:** Unlike manufacturing BOMs, a ReadyMix BOM is always "open" — its `FromDate` and `ToDate` are set to `DateTime.Now` at creation and are not validated. A ReadyMix BOM is valid as long as it is active.
- **Trigger:** BOM creation when `BillOfMaterialUsageId == ReadyMix`.
- **Source:** `BillOfMaterial.Create:116-117`

### Rule 5: The Finished Concrete Product Cannot Be Its Own Ingredient
- **Plain language:** You cannot add the same product material (e.g., C30 concrete) as a raw material ingredient in its own BOM. This prevents circular recipes.
- **Trigger:** BOM creation or update.
- **Violation result:** Error: `CannotUseProductInRawMaterials`
- **Source:** `BillOfMaterial.Validate:444`

### Rule 6: No Duplicate Raw Materials in the BOM
- **Plain language:** Each raw material (cement, sand, etc.) can appear only once in a BOM's ingredient list.
- **Trigger:** BOM creation or update.
- **Violation result:** Error: `BomDetailMaterialDuplicated`
- **Source:** `BillOfMaterial.Validate:438`

### Rule 7: BOM Cannot Be Deleted If In Use
- **Plain language:** A BOM that is referenced by an existing Sales Order detail cannot be deleted.
- **Trigger:** BOM delete.
- **Violation result:** Error: `BomUsedInSalesOrder`
- **Source:** `BillOfMaterial.ValidateForDelete:329`

### Rule 8: Product Material Must Be Linked to All Selected Plants
- **Plain language:** The finished product material must be registered as a manufactured product in every batching plant selected on the BOM.
- **Trigger:** BOM creation or update.
- **Violation result:** Error: `ProductMaterialNotLinkedToSelectedInventoriesManagement`
- **Source:** `BillOfMaterial.Validate:358`

### Rule 9: At Least One Ingredient Required
- **Plain language:** A BOM must have at least one raw material line. An empty recipe is rejected.
- **Trigger:** BOM creation or update.
- **Violation result:** Error: `BomDetailListIsEmpty`
- **Source:** `BillOfMaterial.Validate:433`

---

## State Machine

BOM doesn't have a status enum. Its availability is controlled by `IsActive`:

```
Active (IsActive=true)  ←──── Create / Update
        │
        │ Deactivate
        ▼
Inactive (IsActive=false)  ← Cannot be used in new delivery plans
```

---

## Technical Specifications (BillOfMaterialTechnicalSpecifications)

Each BOM can have one or more Technical Spec profiles. A profile groups:

| Spec | Business Meaning |
|------|-----------------|
| `Code` | Unique spec identifier (e.g., "C30-S3") |
| `CementStrength` | Target compressive strength |
| `CementContent` | Cement content per m³ |
| `CementTypeId` | Type of cement (OPC, SRPC, etc.) |
| `Grades` | Concrete strength class (C20, C25, C30...) |
| `SlumpLevels` | Workability (S1=dry, S5=flowing) |
| `MaxAggregateSizes` | Largest aggregate particle allowed (mm) |
| `Temperatures` | Target pour temperature range |
| `AggregateMixes` | Aggregate blend ratios |
| `AdmixtureDosageQuantities` | Chemical admixture dosages |
| `WaterAndIces` | Water/ice ratio |
| `AddedMaterials` | Supplementary materials (silica fume, fly ash) |

These specs are copied onto `SalesOrderDetail` and `DeliveryPlanDetail` at order creation and travel with each truck trip.

---

## Lifecycle

1. **Created by:** Mix design engineers / QC lab
2. **Linked to:** One or more batching plants (via `BillOfMaterialInventoryManagement`)
3. **Referenced by:** Sales Order details when a product is ordered
4. **Used at dispatch:** Each truck trip (DeliveryPlanDetail) carries this BOM to define what to load
5. **Updated:** When recipe changes; triggers `RoutingBomRedistributeNeededEvent` if raw material quantities change
6. **Deactivated:** When the recipe is superseded; existing SO references are retained

---

## Key Operations

| Operation | Triggered By | Business Effect |
|-----------|-------------|----------------|
| `Create` | Mix design engineer | Registers a new concrete recipe |
| `Update` | Mix design engineer | Updates ingredients; fires routing redistribution event if quantities change |
| `ValidateForDelete` | Delete command | Checks no SO or routing uses this BOM before allowing deletion |

---

## Who Uses This?

- **Users/Roles:** Mix design engineers (create/update), Dispatchers (select on delivery plans), QC (verify specs)
- **Screens:** BOM master form, Sales Order form (select BOM per detail), Delivery Plan form (BOM auto-loaded)
- **Reports:** Batch ticket reports, mix design reports
