# SalesOrder — Business Documentation

> **Layer:** API Domain — Aggregate Root
> **Application:** Logistics / Production
> **Cycle:** Concrete Cycle (Ready Mix)
> **Source Path:** `Logistics.Production.Logic/SalesOrderAggregate/SalesOrder.cs`
> **Last Updated:** 2026-06-10

---

## What Is This?

A Sales Order is the entry point of the Concrete Cycle. It is created when a customer places an order for ready-mix concrete. The Sales Order captures what concrete products are needed, in what quantities, with what technical specifications, and by when. Every subsequent step — delivery plans, truck trips, goods issues, invoicing — traces back to a Sales Order.

---

## Business Rules

### Rule 1: Sales Order Must Be Confirmed Before Dispatching
- **Plain language:** A delivery plan cannot be created for a Sales Order that has not been confirmed.
- **Trigger:** When a dispatcher tries to create a delivery plan from the Sales Order.
- **Violation result:** Error: `SalesOrderMustBeConfirmed`
- **Source:** `SalesOrderErrors.cs`, enforced in application layer validators

### Rule 2: Stop-Issuing Blocks All New Deliveries
- **Plain language:** If `StopIssuing` is set to `true` on a Sales Order detail, no new delivery plan can be dispatched for that product line.
- **Trigger:** When creating a new `DeliveryPlanFromSalesOrder`.
- **Violation result:** Error: `ItemHasBeenStopIssuingForThisSalesOrder`
- **Source:** `DeliveryPlanFromSalesOrder.cs:59` — first guard in `CreateDeliveryPlan`

### Rule 3: Delivery Plan Date Cannot Precede Sales Order Date
- **Plain language:** You cannot dispatch concrete on a date earlier than the Sales Order was placed.
- **Trigger:** When creating or updating a Delivery Plan.
- **Violation result:** Error: `CannotCreateDeliveryPlanWithDateBeforeSalesOrderDate`
- **Source:** `DeliveryPlanFromSalesOrder.cs:497`

### Rule 4: Quantity Delivery Cannot Exceed Order Quantity + Tolerance
- **Plain language:** The cumulative quantity of all truck trips for a product line cannot exceed the ordered quantity plus the `AllowedIncreasePercent` tolerance.
- **Trigger:** When adding a truck trip (DeliveryPlanDetail) for a Sales Order line.
- **Violation result:** Error: `MaterialNumber_ExceededSalesOrderOrderQuantity`
- **Source:** `DeliveryPlanDetailFromSalesOrder.cs:594` — `ValidateDetailQuantityRelatedToSalesOrderQuantity`
- **Formula:** `MaxAllowed = GrossQuantity + GrossQuantity × AllowedIncreasePercent`

### Rule 5: Rush Commercial Orders Have Special Dispatch Rules
- **Plain language:** Delivery plans created from a Rush Commercial Sales Order cannot be updated or deleted after creation. They also skip the recalculation event when dispatched.
- **Trigger:** On update or delete of a DeliveryPlan.
- **Violation result:** Errors: `CanNotUpdateDeliveryPlanCreatedFromRushSalesOrder` / `CanNotDeleteDeliveryPlanCreatedFromRushSalesOrder`
- **Source:** `DeliveryPlanFromSalesOrder.cs:189, 628`

---

## Key Fields

| Field | Business Meaning | Constraints |
|-------|-----------------|-------------|
| `IsConfirm` | Order has been reviewed and approved for dispatch | Must be `true` before delivery plans can be created |
| `StopIssuing` | Blocks any further delivery dispatches | Set manually by a coordinator; halts the entire order |
| `OutboundDeliveryStatusId` | Tracks how much of the order has been dispatched | Updated after each delivery |
| `SalesOrderTypeId` | Type of order (Standard, Rush Commercial, etc.) | Rush orders have special update/delete restrictions |
| `FirstDeliveryDate` | Date of the first actual delivery | Set automatically when first delivery plan is dispatched |
| `SupplyOrderReferenceNumber` | External reference from the customer | Optional; used for cross-referencing |

---

## State Machine

Sales Orders don't have an explicit status enum in the domain. Status is inferred from:

```
Created → [Confirmed: IsConfirm=true] → Active for Dispatch
                                              │
                          ┌───────────────────┘
                          ▼
                   Deliveries in Progress
                   (OutboundDeliveryStatusId updates)
                          │
                          ▼
                   Fully Delivered → Ready for Invoicing
```

`StopIssuing = true` can be set at any point and blocks further dispatches without closing the order.

---

## Lifecycle

1. **Created by:** Sales team / office staff
2. **Confirmed by:** Authorized user setting `IsConfirm = true`
3. **Used for dispatch:** Dispatcher creates `DeliveryPlanFromSalesOrder` referencing this SO
4. **Delivery quantity updated:** After each delivery, `TotalActualIssuedQuantityInMinimumUnit` on `SalesOrderDetail` recalculates via domain event
5. **Invoiced:** Finance creates an `Invoice` referencing the SO after deliveries complete
6. **Deleted:** Only possible before any deliveries exist; blocked once invoiced or has active delivery plans

---

## Key Operations

| Operation | Triggered By | Business Effect |
|-----------|-------------|----------------|
| Confirm | Finance/Sales | Enables delivery plan creation |
| Set StopIssuing | Coordinator | Blocks all future delivery plan creation |
| ReCalculateTotalActualIssuedQuantity | Domain event after delivery | Updates delivered quantities on all detail lines |

---

## Who Uses This?

- **Users/Roles:** Sales team (create), Finance (confirm, invoice), Dispatcher (create delivery plans)
- **Screens:** Sales Order form, Delivery Plan form (references SO), Invoice form
- **Reports:** Delivery tracking reports, Outstanding orders reports
