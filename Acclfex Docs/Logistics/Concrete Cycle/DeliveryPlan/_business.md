# DeliveryPlanFromSalesOrder — Business Documentation

> **Layer:** API Domain — Aggregate Root
> **Application:** Logistics / Production
> **Cycle:** Concrete Cycle (Ready Mix)
> **Source Path:** `Logistics.Production.Logic/DeliveryPlanAggregate/OnSalesOrder/DeliveryPlanFromSalesOrder.cs`
> **Last Updated:** 2026-06-10

---

## What Is This?

A Delivery Plan is a scheduled dispatching event for ready-mix concrete. It groups one or more mixer truck trips (details) and optionally one or more pump assignments going to the same customer site on the same date and time. Each Delivery Plan is created against a confirmed Sales Order and represents a real-world concrete pour event.

The Delivery Plan is the operational heart of the concrete cycle: it creates Goods Issues (raw material consumption), triggers accounting journals, and drives the customer's delivered quantity on the Sales Order.

---

## Business Rules

### Rule 1: Sales Order Must Exist and Be Valid
- **Plain language:** You cannot create a delivery plan without a valid, existing Sales Order. The SO must also not have `StopIssuing` on its detail.
- **Trigger:** Delivery plan creation.
- **Violation result:** Errors: `SalesOrderNotFound`, `ItemHasBeenStopIssuingForThisSalesOrder`
- **Source:** `DeliveryPlanFromSalesOrder.CreateDeliveryPlan:59-66`

### Rule 2: Customer Delivery Address Is Required
- **Plain language:** Every delivery plan must have a concrete site address — the destination where the mixer trucks are going.
- **Trigger:** Delivery plan creation or update.
- **Violation result:** Error: `CustomerDeliveryAddressNotFound`
- **Source:** `DeliveryPlanFromSalesOrder.ValidateDeliveryPlan:448`

### Rule 3: Delivery Plan Date Cannot Be in the Future
- **Plain language:** You can only create a delivery plan for today or a past date. You cannot pre-schedule a delivery plan dated in the future, unless it is a scheduled delivery (linked to a `SalesOrderDetailDeliveySchedule`).
- **Trigger:** Delivery plan creation or update.
- **Violation result:** Error: `DeliveryPlanCanNotBeInFuture`
- **Source:** `DeliveryPlanFromSalesOrder.ValidateDeliveryPlan:492`

### Rule 4: Delivery Plan Date Cannot Be Before the Sales Order Date
- **Plain language:** Concrete cannot be delivered before the order was placed.
- **Trigger:** Delivery plan creation or update.
- **Violation result:** Error: `CannotCreateDeliveryPlanWithDateBeforeSalesOrderDate`
- **Source:** `DeliveryPlanFromSalesOrder.ValidateDeliveryPlan:497`

### Rule 5: Truck Vehicles Must Be Category "Truck"
- **Plain language:** The vehicles assigned to mixer trip details must be registered as Trucks. Pump vehicles can only be assigned to the pump section.
- **Trigger:** Delivery plan creation or update.
- **Violation result:** Error: `CarMustBeTruck`
- **Source:** `DeliveryPlanFromSalesOrder.ValidateDeliveryPlan:482`

### Rule 6: Driver Cannot Be Shared Between Mixer and Pump
- **Plain language:** The same driver cannot be assigned to both a mixer truck trip and a pump on the same delivery plan.
- **Trigger:** Delivery plan creation or update.
- **Violation result:** Error: `DriverDupplicatedForMixerAndPump`
- **Source:** `DeliveryPlanFromSalesOrder.ValidateDeliveryPlan:413`

### Rule 7: If Pumps Exist, All Mixer Trips Must Be Linked to a Pump
- **Plain language:** If a delivery plan includes pumps, every mixer truck detail must be linked to one of those pumps. You cannot have some trucks with a pump and some without.
- **Trigger:** Delivery plan creation or update.
- **Violation result:** Error: `ThePumpForProductsIsRequired`
- **Source:** `DeliveryPlanFromSalesOrder.ValidateDeliveryPlan:422`

### Rule 8: Delivery Plan Cannot Be Updated If Has Active OnWay Trips With Goods Issues
- **Plain language:** Once a truck is on the road and a Goods Issue document has been created for that trip, you cannot edit the delivery plan header data (date, address, etc.).
- **Trigger:** Delivery plan update.
- **Violation result:** Error: `CanNotUpdateDeliveryPlanAlreadyHasCreatedProductionOrder`
- **Source:** `DeliveryPlanBase.ValidateUpdatesOnHeaderData:484`

### Rule 9: Rush Sales Order Delivery Plans Cannot Be Updated or Deleted
- **Plain language:** If the source Sales Order is of type "Rush Commercial", the delivery plan created for it is locked. It cannot be modified or deleted after creation.
- **Trigger:** Update or delete of a delivery plan.
- **Violation result:** Errors: `CanNotUpdateDeliveryPlanCreatedFromRushSalesOrder`, `CanNotDeleteDeliveryPlanCreatedFromRushSalesOrder`
- **Source:** `DeliveryPlanFromSalesOrder.UpdateDeliveryPlan:189`, `ValidateForDelete:628`

### Rule 10: Cannot Delete If Source Sales Order Has Been Invoiced
- **Plain language:** If an invoice has been created for the sales order referenced by this delivery plan, the delivery plan cannot be deleted.
- **Trigger:** Delete.
- **Violation result:** Error: `CanNotDeleteDeliveryPlanItsSalesOrderInvoiced`
- **Source:** `DeliveryPlanFromSalesOrder.ValidateForDelete:656`

### Rule 11: Cannot Delete If Has Active Redirected Delivery Plans
- **Plain language:** If any truck in this delivery plan was redirected to another customer (creating an auto delivery plan), the original delivery plan cannot be deleted until the redirected plans are resolved.
- **Trigger:** Delete.
- **Violation result:** Error: `CantDeleteDeliveryPlanHaveRedirectedDeliveryPlan`
- **Source:** `DeliveryPlanFromSalesOrder.ValidateForDelete:664`

### Rule 12: Accounting Period Must Be Open
- **Plain language:** The accounting sub-period for the delivery plan date must exist, be unlocked, not accounting-closed, and not balance-transferred. This ensures journals can be posted.
- **Trigger:** Delivery plan delete, journal creation.
- **Violation result:** Errors from `AccountingPeriodErrors` enum
- **Source:** `DeliveryPlanBase.CommonValidateForDelete:521`

### Rule 13: Scheduled Delivery Cannot Exceed Window Quantity
- **Plain language:** When a delivery plan is linked to a delivery schedule window, the total quantity across all truck trips cannot exceed the scheduled window quantity (plus the SO detail's increase tolerance).
- **Trigger:** Delivery plan creation or update when schedule is set.
- **Violation result:** Error: `CanNotExceedScheduleQuantity`
- **Source:** `DeliveryPlanFromSalesOrder.ValidateDeliveryPlan:435`

### Rule 14: Project (If Specified) Must Belong to the Same Customer
- **Plain language:** If you attach a project to a delivery plan, that project must be registered under the same customer as the Sales Order.
- **Trigger:** Delivery plan creation or update.
- **Violation result:** Error: `ProjectNotRelatedToCustomer`
- **Source:** `DeliveryPlanFromSalesOrder.ValidateDeliveryPlan:473`

---

## Auto-Created Delivery Plans

When a truck is **redirected to another customer**, the system automatically creates a new `DeliveryPlanFromSalesOrder` with `IsAutoCreated = true`. These auto-created plans have restricted behavior:

| Auto-Created Plan Rules | |
|---|---|
| Only the detail **status** can be changed | `StatusOnlyCanUpdateForAutoCreatedDeliveryPlan` |
| Cannot add more truck trips | `CanNotAddProductsToAutoCreatedDeliveryPlans` |
| Cannot change the date | Same error as status-only |
| Auto-created details cannot be set back to NotStarted | `AutoCreatedCanNotBeNotStarted` |

---

## Domain Events Fired

| Event | When | Effect |
|-------|------|--------|
| `CreateGoodIssueOnDeliveryPlanEvent` | On create/update (if any detail is OnWay) | Background job creates Goods Issue documents in Inventory |
| `CreateDeliveryPlanJournalDomainEvent` | On create | Background job creates cost accounting journals |
| `UpdateDeliveryPlanJournalDomainEvent` | On update | Background job updates existing journals |
| `DeleteGoodIssuesOnDeliveryPlanDomainEvent` | On update/delete | Background job removes Goods Issue documents for deleted/changed trips |
| `CreateRedirectedSalesOrderForDeliveryPlanDomainEvent` | On create/update | Background job creates new SalesOrders for redirected trucks |
| `CreateRedirectedDeliveryPlanForRedirectedDeliveryPlansDomainEvent` | On update | Background job creates auto delivery plans for redirected trips |
| `DeleteRedirectedDeliveryPlanDomainEvent` | On update/delete | Background job removes auto delivery plans for removed trips |
| `ReCalculateSalesOrderTotalActualIssuedQuantityAndUpdateQuantityEvent` | On create/update | Updates delivered quantities on the Sales Order |

---

## State Machine

A Delivery Plan doesn't have a header status. Its "state" is inferred from its detail statuses. A delivery plan is "complete" when all details have reached a terminal status.

---

## Lifecycle

1. **Created by:** Dispatcher
2. **Starts at:** Plan date = current time (or future if scheduled)
3. **Truck trips:** Each detail starts at NotStarted, moved to OnWay when dispatched, then to a terminal status
4. **Goods Issue:** Auto-created in background when first OnWay detail exists
5. **Journals:** Auto-created in background after status changes
6. **Redirections:** If a truck is turned away, auto delivery plan created for redirect customer
7. **Deleted:** Only if no invoiced SOs, no active redirects, and accounting period is open

---

## Who Uses This?

- **Users/Roles:** Dispatchers (create, update), Truck coordinators (update trip status and timestamps)
- **Screens:** Delivery Plan form, Truck Trip status screen, Redirect customer screen
- **Reports:** Daily delivery reports, batch ticket reports, driver commission reports
