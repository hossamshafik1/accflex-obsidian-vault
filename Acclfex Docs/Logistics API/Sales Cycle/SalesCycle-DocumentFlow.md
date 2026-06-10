---
title: Sales Cycle — Document Flow, Status Engine & Test Scenarios
application: Logistics API
module: Sales
audience: Developer · QA
last_updated: 2026-06-04
---

# Sales Cycle — Document Flow, Status Engine & Test Scenarios

> **Type:** Developer & QA deep-dive
> **Parent:** [[_overview|Logistics Sales Cycle — Overview]]
> **Primary code:** `Logistics.Sales.Logic` (aggregates), `Logistics.Sales.Data` (EF),
> `Logistics.Application` (CQRS handlers), `Logistics.API/ApiControllers/Sales`

## 1. Document Flow — Per-Line Traceability

Traceability is implemented at the **detail-line** level. Each downstream detail row stores the
identifier of the line it originated from. The `DocumentFlowViews/*.cs` projections expose this
chain to the UI. Forward chain of `*DetailId` keys:

```
SalesInquiryDetailId
   → SalesQuotationDetailId
       → SalesOrderDetailId      (also reachable from SalesContractDetailId)
           → OutboundDeliveryDetailId
               → InvoiceDetailId
           ↘ SalesReturnDetailId   (alt: standalone customer return)
```

Each flow view (`SalesOrderDocumnetFlow`, `OutboundDocumentFlow`, `InvoiceDocumentFlow`,
`SalesReturnDocumentFlow`) carries the **nullable upstream IDs** so a line can be traced back to
any ancestor, plus `Quantity`, `QuantityInMin`, `Unit`, `Currency`, `Serial`, `SerialDocument`.
`InvoiceDocumentFlow` additionally exposes `JournalNumber`, `Total`, `IsAdvancePaymentInvoice`.

**Source enums / type discriminators:**
- `SalesQuotation`: `FromSalesInquiry` vs `QuotationManually` (subfolders under `SalesQuotationAggregate`).
- `SalesOrder`: `IsFromSalesQuotation`, `ContractId`, `SalesQuotationId`, `IsCreatedFromDeliveryPlanRedirected`.
- `SalesReturn`: `SalesReturnSource` → `CustomerReturn` (1) | `ReturnFromSalesOrder` (2).

## 2. Aggregate Roots & Key Files

| Aggregate | Root file | Notable interfaces |
|-----------|-----------|--------------------|
| Sales Inquiry | `SalesInquiryAggregate/SalesInquiry.cs` | `IAggregateRoot, IAuditedEntityBase, IAccountingPeriodTransactionAutoSerial` |
| Sales Quotation | `SalesQuotationAggregate/Base/SalesQuotationBase.cs` | base + `FromSalesInquiry` / `QuotationManually` |
| Sales Order | `SalesOrderAggregate/SalesOrder.cs` (+ `.RushOrder`, partial) | `IAggregateRoot, IApprovedEntity, IConcurrencyBase, IHeaderPackageEntity` |
| Outbound Delivery | `OutboundDeliveryAggregate/OutboundDelivery.cs` | builder: `OutBoundDeliveryBuilder.cs` |
| Goods Issue | `GoodIssueAggregate/GoodIssue.cs` | many sources (outbound, production, movement, cancellation) |
| Invoice | `InvoiceAggregate/Invoice.cs` (+ `.EInvoice`, `.SalesReturn`) | GL: `JournalDescriptionFactoryForInvoice.cs` |
| Sales Return | `SalesReturnAggregate/Base/...` | `FromSalesOrder` / `FromCustomerReturn` |

Domain errors live in `*Errors.cs` per aggregate (e.g. `SalesOrderErrors.SalesOrderStopIssuing`).
All creation/mutation methods return `Result` / `Result<T>` (CSharpFunctionalExtensions) — **business
validation never throws**, per `accflex-erp-core` rules.

## 3. Status Recalculation Engine (Sales Order)

The Sales Order recomputes its three trackers from a list of
`SalesOrderDetailTransactionsQuantities` (delivered / issued / invoiced quantities aggregated per
line). Logic in `SalesOrder.cs` ~lines 2300–2360:

**Outbound Delivery status**
- `Fully` — every non-service line is fully delivered, *or* flagged `StopIssuing` (Supplies directing).
- `Not Yet` — nothing delivered.
- `Partially` — otherwise.

**Goods Issue status** — same shape, using `IssuedQuantity` (StopIssuing Supplies lines count as
their full SO quantity).

**Invoice status** — `Fully` / `Not Yet` / `Partially` from invoiced quantity vs SO quantity.

> Status fields are `private set` and reset to `Not Yet` when downstream docs are cancelled
> (`TransactionCancellationAggregate`). Outbound delivery also carries its own `GoodsIssueStatusId`.

**Confirmation gate** (`IsConfirm`): set true by `CreateStandardSalesOrder` / confirm methods
*only* when permission/approval checks pass. Confirmation can be revoked (`IsConfirm = false`)
but **not** if `StopIssuing` is set on header or any line — returns `SalesOrderErrors.SalesOrderStopIssuing`.

## 4. CQRS / API Surface

Controllers in `Logistics.API/ApiControllers/Sales/` (and `.../Inventory/GoodIssueController`,
`.../Production/DeliveryPlanController`). Each thin controller calls MediatR commands/queries in
`Logistics.Application`; policy authorization per CRUD action (`ViewPolicy`/`AddPolicy`/
`EditPolicy`/`DeletePolicy`/`PrintPolicy`).

| Screen | Controller |
|--------|-----------|
| Sales Inquiry | `SalesInquiryController` |
| Sales Quotation | `SalesQuotationController`, `SalesQuotationFromInquiryController` |
| Sales Contract | `SalesContractController` |
| Sales Order | `SalesOrderController` (+ `SalesOrderImportController`, `SalesOrderDetailImportController`) |
| Outbound Delivery | `OutboundDeliveryController` |
| Goods Issue | `GoodIssueController` (+ `GoodIssueDetailImportController`) |
| Invoice | `InvoiceController`, `LogisticsEInvoiceController` |
| Sales Return | `SalesReturnController` |
| Settlement | `CustomerInvoiceSettlementController` |

Cross-context posting (GL journals, currency) is wrapped in `IApplicationTransaction` in the
Application layer — one transaction per command.

## 5. Business Rules Reference

| #   | Rule                                                              | Source                                               | Violation result                      |
| --- | ----------------------------------------------------------------- | ---------------------------------------------------- | ------------------------------------- |
| 1   | SO must be confirmed before delivery/issue/invoice                | `SalesOrder` confirm flow                            | Downstream creation blocked           |
| 2   | Cannot ship/issue when `StopIssuing` set (header or line)         | `SalesOrder.cs:2182-2186`                            | `SalesOrderStopIssuing`               |
| 3   | Cannot deliver/issue/invoice more than open qty                   | transaction-qty validation                           | `Result.Failure` per line             |
| 4   | Confirmation enforces credit limit / max debit / max debit period | `CreateStandardSalesOrder` permission params         | Blocked unless override permission    |
| 5   | Confirmation enforces stock availability                          | `AbilityToCreateSalesOrderWithUnavailableQuantities` | Blocked unless permission             |
| 6   | Quotation/Inquiry have validity (final) dates                     | `SalesInquiry.FinialSalesInquiryDate` etc.           | Validation failure                    |
| 7   | Return source must be Customer Return or Return-from-SO           | `SalesReturnSource`                                  | Invalid source rejected               |
| 8   | Approval workflow applies when license enables it                 | `isLicenseContainsApprovalSystem`                    | SO stays unconfirmed pending approval |

## 6. QA — End-to-End Test Scenarios

### Happy path (full cycle)
| # | Scenario | Expected |
|---|----------|----------|
| 1 | Create Inquiry → convert to Quotation → convert to Sales Order | Each doc links to parent; `SalesQuotationDetailId`/`SalesInquiryDetailId` populated on SO lines |
| 2 | Confirm SO (within credit limit, stock available) | `IsConfirm = true`; all 3 statuses = `Not Yet` |
| 3 | Create Outbound Delivery for full qty → Goods Issue | `OutboundDeliveryStatus = Fully`, `GoodsIssueStatus = Fully`; inventory reduced; GL movement posted |
| 4 | Create Invoice for full qty | `InvoiceStatus = Fully`; receivable/revenue journal posted; `JournalNumber` on flow view |
| 5 | Open Document Flow on the Invoice | Full chain Inquiry→Quotation→Order→Outbound→Invoice visible & traceable backward |

### Partial fulfillment
| # | Scenario | Expected |
|---|----------|----------|
| 6 | Deliver 40% of an order line | `OutboundDeliveryStatus = Partially` |
| 7 | Invoice in two batches (60% then 40%) | Status `Partially` then `Fully`; sum of invoiced qty = SO qty |
| 8 | Try to deliver more than open qty | `Result.Failure`, no document created |

### Negative / guards
| # | Scenario | Expected |
|---|----------|----------|
| 9 | Create Outbound from an **unconfirmed** SO | Rejected |
| 10 | Set `StopIssuing`, then attempt Goods Issue | Rejected with `SalesOrderStopIssuing` |
| 11 | Confirm SO exceeding credit limit without permission | Rejected; with permission → allowed |
| 12 | Revoke confirmation while `StopIssuing` set | Rejected with `SalesOrderStopIssuing` |

### Returns & cancellation
| # | Scenario | Expected |
|---|----------|----------|
| 13 | Sales Return *from Sales Order* line | `SalesReturnSource = ReturnFromSalesOrder`; inventory increased; credit memo posted |
| 14 | Standalone Customer Return | `SalesReturnSource = CustomerReturn`; no SO link required |
| 15 | Cancel the Goods Issue / Invoice | SO status reverts toward `Partially`/`Not Yet`; quantities released |

### Integration
| # | Scenario | Expected |
|---|----------|----------|
| 16 | E-Invoice submission on Invoice | E-invoice payload generated (`Invoice.EInvoice`) and submitted via `LogisticsEInvoiceController` |
| 17 | Make-to-order: SO → Delivery Plan → Production Order → Goods Issue | Production document-flow links resolve; goods issued from production |

## 7. Risk & Impact Notes

- `SalesOrder.cs` is large and central — changes to the status engine affect delivery, issue,
  invoice, and reporting simultaneously. Re-run all section 6 scenarios on any change there.
- Status recalculation depends on accurate `SalesOrderDetailTransactionsQuantities` aggregation;
  a missed cancellation path leaves stale `Fully` statuses.
- GL posting (Goods Issue, Invoice, Return credit memo) crosses bounded contexts — verify the
  `IApplicationTransaction` wrapping so a failed journal rolls back the document.

## Open Questions

- [ ] Exact decision logic for order-based vs delivery-based invoicing.
- [ ] Advance-payment invoice interaction with `InvoicedPaidAmount` / `AdvancePaymentsPaidAmount`.
- [ ] Full list of permissions gating confirmation overrides (enumerate from `CreateStandardSalesOrder`).
