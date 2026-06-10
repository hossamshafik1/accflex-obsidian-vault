# Invoice — Business Documentation

> **Layer:** API Domain — Aggregate Root (Reference Entity in Production Logic)
> **Application:** Logistics / Production
> **Cycle:** Concrete Cycle (Ready Mix)
> **Source Path:** `Logistics.Production.Logic/InvoiceAggregate/Invoice.cs`
> **Last Updated:** 2026-06-10

---

## What Is This?

An Invoice is the billing document issued to a customer after concrete has been delivered. It summarises what was delivered across one or more delivery trips on a Sales Order, carries the monetary amounts (before tax, tax, total), and tracks the document's lifecycle from creation through check → send → payment.

Within the Production Logic bounded context, `Invoice` is a **read-only reference entity** — the domain rules that create, validate, and post invoices live in the Sales bounded context. Production Logic reads Invoice records only to enforce two guards:
1. A Delivery Plan cannot be deleted once its Sales Order has been invoiced.
2. A Delivery Plan Detail cannot be updated once its Sales Order detail has been invoiced.

The Invoice is therefore the financial end-gate of the Concrete Cycle: `SalesOrder → DeliveryPlan → DeliveryPlanDetail (delivered) → Invoice`.

---

## Key Business Rules (As Observed in Production Logic)

### Rule 1: Invoice Blocks Delivery Plan Deletion
- **Plain language:** If at least one Invoice exists for the Sales Order linked to a Delivery Plan, that Delivery Plan cannot be deleted.
- **Trigger:** Delivery plan delete.
- **Violation result:** Error: `CanNotDeleteDeliveryPlanItsSalesOrderInvoiced`
- **Source:** `DeliveryPlanFromSalesOrder.ValidateForDelete:656`

### Rule 2: Invoice Blocks Delivery Plan Detail Update
- **Plain language:** If the Sales Order detail linked to a trip has been invoiced, and the trip is in Delivered status, that trip cannot be updated.
- **Trigger:** Delivery plan detail update.
- **Violation result:** Error: `CanNotUpdateDeliveryPlanRelatedToSalesOrderInvoiced`
- **Source:** `DeliveryPlanDetailFromSalesOrder.ValidateInvoiceDeliveryPlan:643`

---

## Lifecycle

```
[Created] → [Checked] → [Sent] → [Paid]
                │
                └──[Void] (at any point before payment)
```

| Stage | Fields Set | Business Meaning |
|-------|-----------|-----------------|
| Created | `InvoiceDate`, `DueDate`, `TotalBeforeTax`, `Total` | Invoice document generated for the SO |
| Checked | `IsChecked = true`, `CheckDate` | Finance team reviewed the invoice |
| Sent | `IsSent = true`, `SentDate` | Invoice sent to the customer |
| Paid (partial) | `PaidAmount`, `RemainingAmount` | Customer payment registered |
| Paid (full) | `IsPaid = true`, `PaymentDate`, `RemainingAmount = 0` | Invoice fully settled |
| Void | `Void = true` | Invoice cancelled; journal reversed via `JournalGUIDNumber` |

---

## Special Invoice Types

| Type | Indicator Field | Created By |
|------|----------------|-----------|
| Standard Invoice | `IsAutoCreatedFromRushOrder = false` | Dispatcher / Finance |
| Rush Order Auto Invoice | `IsAutoCreatedFromRushOrder = true` | Background job on Rush SO dispatch |
| Advance Payment Invoice | `IsAdvancePaymentInvoice = true` | Finance, before delivery |

---

## Key Fields

| Field | Business Meaning |
|-------|-----------------|
| `SalesOrderId` | The Sales Order this invoice covers |
| `CustomerId` | The billed customer |
| `SalesOrganizationId` | The sales org that owns the invoice |
| `InvoiceGuidNumber` | Cross-BC idempotency key; used by background jobs |
| `JournalGUIDNumber` | Links to the GL journal; used when voiding to reverse entries |
| `RelatedGoodIssueId` | Links to the Goods Issue document for the deliveries |
| `InvoiceSerialNumber` | Sequential human-readable invoice number |
| `InvoiceDate` | Document date |
| `DueDate` | Payment due date |
| `BaselineDate` | Reference date for payment terms calculation |
| `TotalBeforeTax` | Net amount before VAT |
| `TotalTax` | VAT amount |
| `Total` | Total amount payable |
| `TotalGrossPrice` | Gross price before any discounts |
| `TotalDiscountValue` | Total discount applied |
| `PaidAmount` | Amount already received from customer |
| `RemainingAmount` | `Total − PaidAmount` |
| `IsChecked` | Whether finance has reviewed this invoice |
| `IsSent` | Whether the invoice has been sent to the customer |
| `IsPaid` | Whether the invoice is fully settled |
| `Void` | Whether the invoice has been cancelled |
| `HasManualCondition` | Whether a pricing condition was manually overridden |
| `SmartSalesSettingsId` | Links to Smart Sales automation settings (if applicable) |

---

## Invoice Detail (Child)

Each invoice line (`InvoiceDetail`) represents one delivered item, linked back to a `SalesOrderDetailId`:

| Field | Business Meaning |
|-------|-----------------|
| `SalesOrderDetailId` | Which SO line this detail covers |
| `MaterialId` | The concrete product |
| `Quantity` | Delivered quantity in the billing unit |
| `TotalBeforeTax` / `TotalTax` / `Total` | Line-level amounts |
| `DeliveryDate` | Date of the delivery trip |
| `IsTaxableMaterial` | Whether VAT applies to this material |
| `EInvoiceCodeTypeId` + `EInvoiceTaxableCode` | E-Invoice tax classification codes (for government e-invoicing) |
| `CostCenterId` | Cost center allocation for the line |

---

## Who Uses This?

- **Users/Roles:** Finance (check and send), Accountant (post journal), Customer (receives sent invoice)
- **Screens:** Invoice list screen, Invoice detail form, Payment registration form
- **Reports:** Invoice print (PDF), Customer account statement, Accounts receivable aging report
- **Read by:** Delivery Plan deletion guard, Delivery Plan Detail update guard

---

## Integration Points

| Integration | Direction | How |
|------------|-----------|-----|
| GeneralLedger BC | Outbound | `JournalGUIDNumber` links to the posted journal entry |
| GoodsIssue (Inventory BC) | Inbound | `RelatedGoodIssueId` ties the invoice to the physical delivery |
| E-Invoice Gateway | Outbound | `EInvoiceTaxableCode` fields on detail lines feed the tax authority submission |
