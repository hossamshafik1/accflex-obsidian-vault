# DeliveryPlanDetailFromSalesOrder — Business Documentation

> **Layer:** API Domain — Child Entity (of DeliveryPlanFromSalesOrder)
> **Application:** Logistics / Production
> **Cycle:** Concrete Cycle (Ready Mix)
> **Source Path:** `Logistics.Production.Logic/DeliveryPlanAggregate/OnSalesOrder/DeliveryPlanDetailFromSalesOrder.cs`
> **Last Updated:** 2026-06-10

---

## What Is This?

A Delivery Plan Detail represents one mixer truck's trip from the batching plant to the customer's construction site. It carries a specific quantity of concrete, is assigned to a specific truck (Car) and driver, references the BOM (recipe), and tracks the truck's physical journey in real time.

Each detail has two parallel tracking systems:
1. **`DetailStatusId`** — the operational lifecycle (NotStarted → OnWay → terminal outcome)
2. **`TripStateId`** — the physical GPS-like location of the mixer truck during the OnWay phase

When a trip reaches a terminal status, an accounting journal is created that moves the cost of the raw materials out of Work-In-Progress into either Cost of Sales (delivered) or Scrap Expense (rejected).

---

## Business Rules

### Rule 1: Quantity Must Be Greater Than Zero
- **Plain language:** Every truck trip must carry some concrete. Zero-quantity trips are not allowed.
- **Trigger:** Create or update.
- **Violation result:** Error: `QuantityShouldBeGreaterThanZeroForItemNumber_`
- **Source:** `DeliveryPlanDetailBase.CommonValidate:143`

### Rule 2: BOM Must Be Active and Linked to the Batching Plant
- **Plain language:** The Bill of Material selected for this trip must be active (`IsActive = true`) and must be registered for the inventory management (batching plant) being used for the delivery.
- **Trigger:** Create or update.
- **Violation result:** Error: `BomError`
- **Source:** `DeliveryPlanDetailBase.ValidateBomRelatedToManagementWithinDate:528`

### Rule 3: Material Must Have Usage Type "ReadyMix"
- **Plain language:** The concrete product being delivered must have its `ManufactureUsageId` set to ReadyMix. You cannot dispatch manufacturing or sales materials through the Ready Mix cycle.
- **Trigger:** Create or update.
- **Violation result:** Error: `ProductsUsageMustBeReadyMix`
- **Source:** `DeliveryPlanDetailBase.CommonValidate:175`

### Rule 4: Inventory Management Must Have a Scrap Account Configured
- **Plain language:** The batching plant's finished goods inventory management must have a scrap GL account assigned. This account is needed to book scrap cost if the truck is rejected.
- **Trigger:** Create or update.
- **Violation result:** Error: `YouShouldSelectScarpAccountForInventoryManagement`
- **Source:** `DeliveryPlanDetailBase.CommonValidate:184`

### Rule 5: Reason Is Required When Scrapping
- **Plain language:** If a truck trip is marked as "Rejected and Scrapped," the dispatcher must record a reason why the concrete was poured out.
- **Trigger:** Status change to `RejectedAndScrapped`.
- **Violation result:** Error: `ReasonIsRequired`
- **Source:** `DeliveryPlanDetailBase.CommonValidate:151`

### Rule 6: Data Changes Only Allowed When Status Is NotStarted
- **Plain language:** Once a truck has been dispatched (status moves to OnWay), the detail's data (material, quantity, BOM, truck) cannot be changed. Only the status can be updated.
- **Trigger:** Update where detail is not NotStarted.
- **Violation result:** Error: `CanChangeStatusOnlyOnNotStarted`
- **Source:** `DeliveryPlanDetailBase.ValidateUpdates:476`

### Rule 7: Status Transitions Follow a Strict State Machine
- **Plain language:** Not all status changes are allowed. The only forward path from NotStarted is OnWay. From OnWay, a trip can go to Delivered, Scrapped, or Redirected. Reverse transitions back to OnWay are allowed only as corrections.
- **Trigger:** Any status update.
- **Violation result:** Error: `ChangeStatusIsNotValid`
- **Source:** `DeliveryPlanDetailStatus.IsValidTransaction`

### Rule 8: Cannot Mark Delivered/Rejected Without a Goods Issue
- **Plain language:** A truck trip cannot be finalized (Delivered, Scrapped, or Redirected) if it is in OnWay status, has raw material lines, and no Goods Issue document has been created yet. The raw material consumption must be posted first.
- **Trigger:** Status change to a terminal status from OnWay.
- **Violation result:** Error: `CanNotMakeStatusDeliveredOrRejectedOrRedirectWithoutProductionOrderOrGoodIssueCreated`
- **Source:** `DeliveryPlanDetailBase.ValidateUpdatesForOnWayStatus:402`

### Rule 9: Truck Quantity Cannot Exceed Vehicle Max Load
- **Plain language:** If the assigned mixer truck has a `MaxLoad` configured (in the same unit as the order), the trip quantity cannot exceed that load limit.
- **Trigger:** Create or update.
- **Violation result:** Error: `CannotExceedMixerTruckMaxLoad`
- **Source:** `DeliveryPlanDetailFromSalesOrder.ValidateDeliveryPlanDetail:401`

### Rule 10: Cannot Update a Delivered Trip That Has Been Invoiced
- **Plain language:** If a trip's Sales Order detail has already been invoiced, and the trip is currently in Delivered status, you cannot change it anymore.
- **Trigger:** Update.
- **Violation result:** Error: `CanNotUpdateDeliveryPlanRelatedToSalesOrderInvoiced`
- **Source:** `DeliveryPlanDetailFromSalesOrder.ValidateInvoiceDeliveryPlan:643`

### Rule 11: Redirect — Must Choose New or Existing Sales Order (Not Both)
- **Plain language:** When a truck is redirected to another customer, the dispatcher must choose one path: either create a brand-new Sales Order for the redirect customer, or link to an existing confirmed Sales Order. Choosing neither is not allowed.
- **Trigger:** Status = `RejectedAndRedirectedToAnotherCustomer`.
- **Violation result:** Error: `MustSelectOneOfChoices`
- **Source:** `DeliveryPlanDetailBase.CommonValidate:200`

### Rule 12: Redirect — Linked Sales Order Must Be Confirmed With No Outbound Deliveries Yet
- **Plain language:** When linking to an existing Sales Order for a redirect, that order must be confirmed and must not already have any delivery dispatched against it.
- **Trigger:** Redirect to existing SO.
- **Violation result:** Errors: `SalesOrderWantToLinkMustBeConfirmed`, `SomeOutboundDeliveryWasCreatedOnSalesOrder`
- **Source:** `DeliveryPlanDetailBase.CommonValidateOldSalesOrderToLink:356-373`

### Rule 13: Redirect — Cannot Redirect to the Same Customer
- **Plain language:** The customer you redirect to must be different from the original Sales Order's customer.
- **Trigger:** Redirect validation.
- **Violation result:** Error: `ChooseAnotherCustomer`
- **Source:** `DeliveryPlanDetailFromSalesOrder.ValidateNewSalesOrder:438`

### Rule 14: Redirect — Linked Sales Order Must Have Same Sales Organization
- **Plain language:** When linking to an existing SO for redirect, that SO must belong to the same Sales Organization as the original Sales Order.
- **Trigger:** Redirect to existing SO.
- **Violation result:** Error: `SalesOrderWantToLinkMustHaveSameSalesOrganization`
- **Source:** `DeliveryPlanDetailFromSalesOrder.ValidateOldSalesOrderToLink:544`

### Rule 15: Redirect Quantity Cannot Exceed Available Quantity on Target SO
- **Plain language:** The redirected quantity cannot exceed what is still available (ordered but not yet delivered) on the target Sales Order detail.
- **Trigger:** Redirect to existing SO.
- **Violation result:** Errors: `NoAvailableQuantityOnSalesOrder`, `MaxQuantityInSalesOrder`
- **Source:** `DeliveryPlanDetailFromSalesOrder.ValidateOldSalesOrderToLink:530`

---

## State Machine — DetailStatus

```
NotStarted(1) ──[dispatch]──► OnWay(2) ──[confirmed delivery]──► Delivered(5)
                                  │
                                  ├──[concrete rejected, dumped]──► RejectedAndScrapped(3)
                                  │
                                  └──[redirected to other customer]──► RejectedAndRedirectedToAnotherCustomer(4)

Correction paths (all terminal → OnWay):
Delivered(5) ────────────────────────────────────────────────────► OnWay(2)
RejectedAndScrapped(3) ──────────────────────────────────────────► OnWay(2)
RejectedAndRedirectedToAnotherCustomer(4) ───────────────────────► OnWay(2)
```

---

## State Machine — MixerTripState (Physical Location)

Runs in parallel with DetailStatus during the OnWay phase:

| Step | TripStateId | Timestamp Field | Business Meaning |
|------|-------------|----------------|-----------------|
| NotStarted | 0 | — | Truck is still at the plant |
| ExitPlant | 1 | `TimeEXPlant` | Truck leaves the batching plant |
| OnSite | 2 | `TimeOnSite` | Truck arrives at the construction site |
| StartDischarge | 3 | `StartDischargeDate` | Concrete begins pouring |
| EndDischarge | 4 | `EndDischargeDate` | Concrete pour complete |
| ReturnedPlant | 5 | `TimeReturndPlant` | Truck returns to the plant |

Setting `DetailStatusId = NotStarted` automatically resets `TripStateId = NotStarted`.

---

## Accounting Journal Logic

When a trip is finalized (Delivered or Scrapped), a journal is created using:

**Cost formula:**
```
DeliveredQuantityCost = (DeliveredQty ÷ RecipeQty) × TotalRawMaterialCost
ScrappedCost          = (ScrappedQty  ÷ RecipeQty) × TotalRawMaterialCost
```

Where `TotalRawMaterialCost` = sum of all line costs on the linked GoodsIssue.

**Journal entries:**

| Status | Debit | Credit |
|--------|-------|--------|
| Delivered | Material Cost Account (from SystemConfig) | Work In Progress Account |
| RejectedAndScrapped | Inventory Scrap Account (from InventoryManagement) | Work In Progress Account |
| Partial scrap with delivery | Inventory Scrap Account | (already in WIP credit) |

**Key fields used:**
- `ClaimCustomerWithDelivedQuantity` — when `true`, uses `DeliveredQuantity` (actual poured); when `false`, uses `ReceipeQuantity` (recipe quantity)
- `ScrappedQuantity` — partial scrap amount (some concrete was wasted but truck still delivered)

---

## Key Fields

| Field | Business Meaning |
|-------|-----------------|
| `DetailStatusId` | Current operational status of the trip |
| `TripStateId` | Physical location of the mixer truck |
| `ReceipeQuantity` | Planned/recipe quantity loaded at the plant |
| `DeliveredQuantity` | Actual quantity delivered to customer (may differ from recipe) |
| `ScrappedQuantity` | Quantity wasted/dumped (partial scrap) |
| `RedirectedToAnotherCustomerQuantity` | Quantity redirected to a different customer |
| `GoodIssueNumber` | Set by Inventory BC after raw materials are consumed |
| `JournalGuidNumber` | Links to the cost journal in GeneralLedger BC |
| `StationId` | Which batching plant produced this batch |
| `StationBatchTicket` | Batch ticket number printed at the plant |
| `TechnicalSpecificationsCode` | Code of the concrete spec profile used for this trip |
| `Commission` | Driver commission calculated for this trip |

---

## Lifecycle

1. **Created at:** Delivery plan creation, initial status = NotStarted or OnWay
2. **Dispatched:** Status → OnWay; `GoodIssueNumber` stamped by background job
3. **Trip tracked:** TripState updated at each checkpoint (ExitPlant → OnSite → StartDischarge → EndDischarge → ReturnedPlant)
4. **Finalized:** Status → Delivered, Scrapped, or Redirected
5. **Journal created:** Background job posts cost journal after finalization
6. **Commission calculated:** After delivery, commission is stamped via `UpdateCommissionData`

---

## Who Uses This?

- **Users/Roles:** Dispatchers (create/update status), Truck coordinators (update trip timestamps), QC (verify batch ticket), Finance (read journal data)
- **Screens:** Delivery Plan detail form, Trip tracking screen, Driver commission screen
- **Reports:** Batch ticket print, Driver performance report, Cost of delivery report
