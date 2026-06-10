# Invoice — Dependency & Change Impact

> **Application:** Logistics / Production
> **Cycle:** Concrete Cycle (Ready Mix)
> **Last Updated:** 2026-06-10

---

## What Invoice Depends On

### API Dependencies
| Dependency | Type | Why | Risk if Changed |
|------------|------|-----|----------------|
| `SalesOrder` | Hard — `SalesOrderId` FK | Invoice is issued against a Sales Order | Critical — if SO is deleted, invoices become orphans |
| `Customer` | Hard — `CustomerId` FK | The billed party | High — customer identity affects tax and account assignment |
| `SalesOrganization` | Hard — `SalesOrganizationId` FK | Owns the invoice; drives tax rules and GL mapping | High |
| `Currency` | Hard — `CurrencyId` + `CurrencyFactor` | Invoice currency and conversion rate | Medium — changes affect amount displays, not domain logic |
| `GoodsIssue` (Inventory BC) | Soft — `RelatedGoodIssueId` nullable | Links the delivery event to the bill | Medium — informational; not a hard guard in Production Logic |
| `Journal` (GL BC) | Soft — `JournalGUIDNumber` string | GL journal for the invoice | Critical — voiding the invoice uses this to reverse entries |
| `SalesOffice` | Soft — `SalesOfficeId` nullable | Optional sales office assignment | Low |
| `SmartSalesSettings` | Soft — `SmartSalesSettingsId` nullable | Automation rules for smart billing | Low — only applies when Smart Sales is enabled |

### Database Dependencies
| Table / Object | Type | Why |
|----------------|------|-----|
| `Invoice` table | Primary | Header record |
| `InvoiceDetail` table | Child FK | Line items per SO detail |
| GL journal table (cross-BC) | Cross-BC | `JournalGUIDNumber` links here |

---

## What Depends On Invoice

### Ring 1 — Direct Dependents

| Dependent | Repo | How It Uses This | Impact of Breaking Change |
|-----------|------|-----------------|--------------------------|
| `DeliveryPlanFromSalesOrder.ValidateForDelete` | API (Production Logic) | Checks if any invoice exists for the plan's SO — blocks delete if so | **High** — removing or changing Invoice lookup breaks the delete guard |
| `DeliveryPlanDetailFromSalesOrder.ValidateInvoiceDeliveryPlan` | API (Production Logic) | Checks if the trip's SO detail is invoiced — blocks update if so | **High** — removing or changing Invoice lookup breaks the update guard |

### Ring 2 — Indirect Dependents

| Dependent | Repo | Coupling Type | Impact |
|-----------|------|--------------|--------|
| `SalesOrder.OutboundDeliveryStatusId` | API | Invoice existence affects SO's delivery/billing status | **Medium** — SO status may reflect invoiced flag |
| Payment tracking screens | Angular | Display `PaidAmount`, `RemainingAmount`, `IsPaid` | **Medium** — UI reflects payment state; field renames break display |
| Customer account statement report | DB (SP/View) | JOINs Invoice + InvoiceDetail for AR balance | **Medium** — schema changes break the report |
| E-Invoice gateway integration | External | Reads `EInvoiceTaxableCode` from `InvoiceDetail` | **High** — government tax submission depends on these codes |

### Ring 3 — Risk Flags

- ⚠️ **Invoice is read-only in Production Logic** — the domain methods for creating and voiding invoices live in the Sales BC. If you search for `Invoice.Create(...)` in this project, you will not find it. Any business logic change to how invoices are created or voided must be made in the Sales bounded context, not here.
- ⚠️ **Delete guard is a query-time check** — `ValidateForDelete` queries the database for existing invoices on the SO. If the invoice is in a different schema or database (multi-tenancy edge case), this check could silently pass even when invoices exist.
- ⚠️ **`Void` is a simple boolean field** — there is no state machine enforcing transitions to/from Void in this project. If void logic changes (e.g., partial void, void reversal), the Sales BC and any code reading `Void = true` in queries must be updated in sync.
- ⚠️ **`IsAutoCreatedFromRushOrder` and `IsAdvancePaymentInvoice`** — these are boolean flags with no enum-class pattern. If a third invoice type is added, it requires a new boolean field and all guards that check invoice type must be updated.
- ⚠️ **`JournalGUIDNumber` is a free-text string** — there is no FK constraint or format validation. If the GL BC changes how it generates GUIDs, the link between Invoice and its journal could become stale without any database-level error.
- ⚠️ **`InvoiceDetail.SalesOrderDetailId`** — this nullable FK is the key link for the delivery plan detail update guard. If it becomes null (e.g., for advance payment invoices), the guard check in `ValidateInvoiceDeliveryPlan` may incorrectly pass or fail depending on how the query is written.

---

## Change Scenarios

### Scenario: Add a new Invoice type (e.g., "CreditNote")
| What to update | Priority |
|----------------|---------|
| Add boolean field or migrate to enum-class for `InvoiceTypeId` | 🔴 Critical |
| Sales BC — Invoice creation logic | 🔴 Critical |
| `ValidateForDelete` and `ValidateInvoiceDeliveryPlan` — check if new type should block | 🟠 High |
| Angular invoice type badge + filter dropdowns | 🟡 Medium |

### Scenario: Change the delete guard (e.g., allow deletion of invoiced plans under certain conditions)
| What to update | Priority |
|----------------|---------|
| `DeliveryPlanFromSalesOrder.ValidateForDelete:656` | 🔴 Critical |
| Test: create an invoiced SO → delivery plan → attempt delete | 🟠 High |
| Angular delete button visibility / confirmation dialog | 🟡 Medium |

### Scenario: Add E-Invoice fields to InvoiceDetail
| What to update | Priority |
|----------------|---------|
| `InvoiceDetail.cs` — new property | 🔴 Critical |
| DB migration | 🔴 Critical |
| Sales BC — Invoice creation handler populates the new field | 🔴 Critical |
| E-Invoice gateway integration code | 🟠 High |
| Angular invoice detail form | 🟡 Medium |

### Scenario: Rename or remove the `Void` field
| What to update | Priority |
|----------------|---------|
| `Invoice.cs` — rename property | 🔴 Critical |
| DB migration | 🔴 Critical |
| Sales BC — void command handler | 🔴 Critical |
| All queries checking `Void == false` or `Void == true` in both Production Logic and Sales BC | 🟠 High |
| GL BC — journal reversal logic keyed on this flag | 🟠 High |
| Angular — void button visibility guard | 🟡 Medium |
