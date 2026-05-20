# Owner Memo — Module Overview (Baseline, pre-CR-116591)

> **Application scope:** Cloud API (Construction.*), WinForms Desktop (Constraction.UI), Angular ERP, SQL DB, Print reports.
> **Bounded context:** Construction → Owner Memo
> **Aggregate root:** `OwnerMemo`
> **Status:** As-is baseline captured **before** CR 116591 (Price-Differences Memo) is implemented. To be updated after the CR ships.

---

## 1. What it is, in business language

An **Owner Memo (إشعار مالك)** is a transactional document a contractor raises against an Owner (client) to **adjust amounts owed** outside the regular Owner Statement (مستخلص مالك) flow. It is always linked to a single Owner Statement.

**Who uses it.** Project finance / billing staff in construction companies.

**Why it exists.** Owner Statements are formal milestone billings. After a statement is issued, real-life events (small extras, deductions, partial reversals) need a lighter-weight document — that's the memo. Two operating modes today:

| Flavor | Selector | Purpose |
|---|---|---|
| **Standalone Memo** | `IsAffectStatement = false` | Add an item-level adjustment to the owner's running balance, without modifying the statement's billed quantities. |
| **Affects-Statement Memo** | `IsAffectStatement = true` (Credit only) | Reverse part of a previously-billed statement: reduce billed quantities, bring back statement-level additions/subtractions, lock tax % to the statement's tax. |

A memo is **never independent of a statement** — `OwnerStatementId` is required. Print, journal posting, VAT and source-tax all key off the linked statement.

---

## 2. Application map

| Layer | Path | Purpose |
|---|---|---|
| Desktop UI (primary today) | `AccFlex_ERP\Constraction\Constraction.UI\OwnerMemo\` | WinForms `frmOwnerMemoForm`, search form, view-models. |
| Desktop print | `AccFlex_ERP\Constraction\Constraction.Reports\…\OwnerReport\Memo\` | DevExpress XtraReport `rptOwnerMemoReport` + dataset `dsOwnerMemo`. |
| Angular ERP frontend | `…\projects\construction\src\app\modules\owner-memo\` | Shell, detail-grid, form service. |
| API | `…\Construction\Construction.API\ApiControllers\` | Thin controllers calling MediatR. |
| Application (CQRS) | `…\Construction\Construction.Application\OwnerMemos\` | Commands, queries, handlers. |
| Domain | `…\Construction\Construction.Logic\OwnerMemoAggregate\` | `OwnerMemo` aggregate root + child entities + domain events. |
| Data | `…\Construction\Construction.Data\` | `ConstructionContext` + EF configurations. |
| Database | `AccFlex_Cloud_Database_State\` | SQL project (tables, FKs, indexes). |
| Tests | `…\ConstructionAPITests\Services\OwnerMemoService.cs` | Integration harness. |

NSwag generates both the .NET and TypeScript clients; desktop and Angular consume `Construction.API.Client`.

---

## 3. Domain model — `OwnerMemo` aggregate

File: `Construction.Logic\OwnerMemoAggregate\OwnerMemo.cs`. Implements `Entity, IConcurrencyBase, IAggregateRoot, IAuditedEntityBase`.

### 3.1 Key columns

| Field | Type | Notes |
|---|---|---|
| `Id` | `int` | PK, mapped to `OwnerMemoId`. |
| `MemoNumber` | `int` | Per-period serial. |
| `JournalId` | `int?` | Posted journal reference. |
| `DocumentNumber` | `int?` | User-entered document number. |
| `MemoDate` | `DateTime` | Must be ≥ statement `ExtractionDate`. |
| `ProjectId` | `int` | FK → `Project`. |
| `CurrencyId` | `int` | FK → `Currency`. |
| `AccountId` | `int` | The single "memo account" used by the standard journal. |
| `BranchId` | `int` | Memo branch. |
| `OwnerStatementId` | `int` | Linked statement (always required). |
| `CostCenterId` | `int?` | Required when account is cost-center-linked. |
| `CurrencyFactor` | `decimal` | Snapshot of factor active on `MemoDate`. |
| `TotalAmount` | `decimal` | Memo total after tax, rounded to GL `ApproximateNo`. |
| `TaxPercent` | `decimal` | Effective VAT %. Locked to statement's when `IsAffectStatement`. |
| `SourceTaxPercent` | `decimal` | Effective source-tax %. |
| `IsMemoDebit` | `bool` | Debit from owner-balance perspective. |
| `Description` | `string` | Free text. |
| `GuidNumber`, `JournalGuidNumber` | `string` | Stable cross-aggregate refs. |
| `CustomerSourceTaxId` | `int?` | FK to `CustomerSourceTax` when emitted. |
| `IsAffectStatement` | `bool` | Mode selector. |
| `AffectedStatementId` | `int?` | Set when memo reverses statement quantities. |
| `TimeStamp` | `byte[]` | Optimistic-concurrency token. |
| audit fields | — | `CreatedById/On`, `LastModifiedById/On`. |

### 3.2 Child collections

| Entity | Role |
|---|---|
| `OwnerMemoDetail` | Line per `ProjectJobItem` — quantity, unit price, memo quantity, tax / source-tax values. |
| `OwnerMemoAddition` | Per-statement addition surfaced into the memo (today only on `IsAffectStatement`). |
| `OwnerMemoSubtraction` | Per-statement subtraction (today only on `IsAffectStatement`). May be flagged `IsSourceTax`. |
| `ValueAddedTax` | VAT bookkeeping rows generated per addition/subtraction/detail. |
| `MemoJournal` (`Journal`) | Generated by `CreateOwnerMemoJournal`. |
| `CustomerSourceTax` | Generated via `CreateSourceTax`. |

### 3.3 Cross-aggregate touchpoints

- `Project`, `Owner` (tax-exemption + tax account), `OwnerStatement` (drives tax %, sequence, addition/subtraction terms).
- `Currency` + `CurrencyFactor` (snapshot at `MemoDate`).
- GL `Account`, `Branch`, `CostCenter`, `AccountingPeriod`.
- `EInvoice` (sync status governs editability).

All cross-aggregate references are int FKs (or stable GUIDs) — no navigation across bounded contexts per ERP conventions.

---

## 4. Business rules catalog

Sources: `OwnerMemo.ValidateOwnerMemo`, `IsValidToUpdate`, `IsValidToDelete`, child-entity validators.

### Always applied

| # | Rule | Error key |
|---|---|---|
| R1 | Project must exist. | `ProjectErrors.ProjectNotFound` |
| R2 | Linked Owner Statement must exist. | `OwnerStatementNotFound` |
| R3 | Memo date ≥ statement's `ExtractionDate`. | `MemoDateMustBeAfterStatementDate` |
| R4 | Statement not deleted. | `StatementsNotFound` |
| R5 | Memo account exists. | `AccountErrors.AccountNotFound` |
| R6 | Memo branch exists. | `BranchErrors.BranchNotFound` |
| R7 | Cost center required when account is cost-center-linked. | `CostCenterErrors.CostCenterNotFount` |
| R8 | Currency + active currency factor exist. | `EnterCurrency`, `CurrencyErrors.CurrencyFactorNotFound` |
| R9 | `MemoDate` is a valid date. | `InvalidDate` |
| R10 | At least one detail line. | `EnterDetails` |
| R11 | No duplicate `ProjectJobItemId`. | `DuplicatedDetails` |
| R12 | Both Construction + GL system configs exist. | `SystemConfigurationErrors.SystemConfigurationNotFound` |
| R13 | Any detail with `SourceTaxValue > 0` requires `SystemConfig.OwnerSourceTaxAccountId`. | `MustSelectOwnerSourceTaxAccount` |
| R14 | Any tax > 0 requires a resolvable tax account. | `OwnerTaxAccountNotFound` |

### Mode-specific (`IsMemoDebit` × `IsAffectStatement`)

| # | Rule | Error key |
|---|---|---|
| R15 | Debit memo cannot have `IsAffectStatement = true`. | `CannotEditIsAffectStatments` |
| R16 | When `IsAffectStatement` + Credit: statement must not be `IsLast`. | `CannotCreateAMemoOnALastStatement` |
| R17 | Memo quantity per line ≤ statement quantity; not negative. | `CannotExceedStatementQuantity` |
| R18 | Percent-band job items cannot carry memo quantity. | `PercentBandCannotBeAdded` |
| R19 | Subtraction memo value ≤ matching statement subtraction value. | `CannotExceedStatementSubtractionQuantity` |
| R20 | Addition memo value ≤ matching statement addition value. | `CannotExceedStatementAdditionQuantity` |
| R21 | Sequence relationship guard (counter-intuitive naming — see `OwnerMemo.cs:202`). | `CannotEditMemoWithLaterStatements` |
| R22 | No final statement may exist beyond this one. | `FinalStatementExists` |

### Delete / Update

| # | Rule | Error key |
|---|---|---|
| R23 | `IsAffectStatement` + `AffectedStatementId` set ⇒ affected statement must be deleted first (Delete). | `CannotDeleteMemoThatAffectStatement` |
| R24 | KSA on-boarded tenant freezes e-invoice transactions. | `EInvoiceErrors.TransactionCannotUpdateOrDelete` |
| R25 | Memo e-invoice `Ready_To_Sync` / `Processing` ⇒ block. | `EInvoiceErrors.TransactionIsReadyToSync` |
| R26 | Memo e-invoice `Synced` ⇒ block. | `EInvoiceErrors.TransactionIsSynced` |
| R27 | Optimistic concurrency: `TimeStamp` must match. | `OwnerMemoUpdated` |

---

## 5. Lifecycle / state

```
   New ──▶ AddingNew ──Save──▶ Persisted
                                  │
                  ┌── Open ──▶ Editing ──Save──▶ Persisted (re-validated, journal rebuilt)
                  │
                  ├── Delete ──▶ R23–R27 guards ──▶ soft delete + journal reversal
                  │
                  └── E-Invoice pipeline ── Ready_To_Sync → Processing → Synced
                                             (after Synced, R25/R26 freeze the memo)
```

No explicit `Status` enum. State is implied by:
- `JournalId.HasValue` — posted to GL.
- `AffectedStatementId.HasValue` — quantity-affect linkage active.
- `CustomerSourceTaxId.HasValue` — source-tax record emitted.
- Memo e-invoice (separate aggregate) — sync state.

---

## 6. Data flow — Create

```
Desktop / Angular form
   │ payload: ProjectId, OwnerStatementId, IsMemoDebit, IsAffectStatement,
   │          OwnerMemoDetailList, AdditionList, SubtractionList, …
   ▼
POST /api/OwnerMemo
   ▼
CreateOwnerMemoCommand (MediatR)
   ├─ Begin IApplicationTransaction
   ├─ Load Project + JobItems, Owner, Additions, Subtractions, Currency + factors,
   │  Accounts, Branches, CostCenters, AccountingPeriodWithNextSerials,
   │  OwnerStatement (with details/additions/subtractions),
   │  hasLaterStatement / hasFinalStatement
   ▼
OwnerMemo.CreateOwnerMemo(...)   (domain factory)
   ├─ ValidateOwnerMemo (R1–R22)
   ├─ Build OwnerMemoDetail list (per-line CreateOwnerMemoDetail)
   ├─ If IsAffectStatement:
   │     • taxPercent ← statement.TaxPercentage
   │     • build Addition / Subtraction lists
   │     • compute SubtractionTaxableAmount, subTaxValue per sub
   ├─ Compute memoTotalsBeforeTax + totalMemoTax → TotalAmount (rounded)
   ├─ CreateOwnerMemoJournal → MemoJournal
   ├─ CreateValueAddedTaxs   → MemoValueAddedTaxList
   ├─ CreateSourceTax (if applicable) → CustomerSourceTax
   ├─ Emit TaxedOwnersMemoCreated if TaxPercent > 0
   ▼
ConstructionContext.SaveChangesAsync (pre/post domain-event dispatch)
   ▼
Commit IApplicationTransaction → HTTP 200 → Id
```

Update mirrors this: load existing aggregate, call `IsValidToUpdate` (R24–R27), then `UpdateOwnerMemo` which re-runs `ValidateOwnerMemo`, `UpdateDetails` / `UpdateAdditions` / `UpdateSubtractions`, and rebuilds the journal via `UpdateOwnerMemoJournal` (or `CreateOwnerMemoJournal` if no journal yet).

Delete: `IsValidToDelete` (R23–R27) → soft-delete; `OwnerMemo.Delete()` emits `TaxedOwnersMemoDeleted` when `TaxPercent > 0`.

---

## 7. Journal posting (today)

File: `OwnerMemo.Journal.cs`. The single builder `CreateJournalActionList` produces a flat `List<NormalJournalActionBuilder>`:

- Owner running account leg (sign per `IsMemoDebit`).
- **Memo account leg** (the single `AccountId` on the header) — opposite side, takes the net of all detail/addition/subtraction values.
- Per-detail VAT account leg when `TaxValue > 0`.
- Per-detail source-tax account leg when `SourceTaxValue > 0`.
- Cost-center association on memo + project sides.

Type: `JournalType.OwnerMemo`. Description: `JournalForOwnerMemo` formatted with `MemoNumber`.

**Key behaviour:** there is only **one** memo account on the header, and the journal nets all lines into / out of that single account. The per-account legs (retention, work-guarantee, etc. — i.e., the addition/subtraction master accounts) are **not posted on the memo journal**; they are only touched via the linked statement's journal. This is the core limitation CR 116591 addresses.

Journal is rebuilt from scratch on every update.

---

## 8. VAT (`ValueAddedTax`)

File: `OwnerMemo.cs` — `CreateValueAddedTaxs`. Writes per-line tax records (in `GeneralLedger.ValueAddedTax`) attributed via `OwnerMemo.MemoValueAddedTaxList`. Inputs: linked statement, owner, currency, tax account, the three DTO lists. Output: zero-or-more `ValueAddedTax` rows for VAT compliance reports.

Math reference: `OwnerMemo.Helper.cs` — `CalculateTotalTaxValue` / `CalculateTaxForAffectStatement` / `CalculateTaxForNonAffectStatement` / `CalculateSubtractionValues`.

---

## 9. Source Tax (`CustomerSourceTax`)

File: `OwnerMemo.cs:1040-1090` — `CreateSourceTax`. Two paths today:

| Mode | Source-tax amount | Account |
|---|---|---|
| Standalone | `Σ detail.SourceTaxValue` | `SystemConfig.OwnerSourceTaxAccountId` |
| Affects-statement | `Σ sub.MemoValue WHERE IsSourceTax` | Subtraction's own account |

When `IsAffectStatement`, `SourceTaxPercent = sourceTaxAmount / totalJobItemAmount` is computed inline. The resulting `CustomerSourceTax` is a separate aggregate with its own journal and serial (`AccountingPeriodWithNextSerials.serialNumberForCustomerSourceTax`). Memo's `CustomerSourceTaxId` is updated post-save via `UpdateCustomerSourceTaxId`.

---

## 10. Linking to the Owner Statement

- `LinkMemoToOwnerStatement(int statementId)` (`OwnerMemo.cs:1099`) — sets `AffectedStatementId`. Idempotent.
- `UnLinkMemoToOwnerStatement(int statementId)` (`OwnerMemo.cs:1109`) — clears it on matching id.

Called by `UpdateOwnerStatementRemaining` service (referenced in `CreateOwnerMemoCommandHandler.cs:60`) when `IsAffectStatement` memos post, so the statement's remaining-quantity calculation can detect them.

---

## 11. Domain events (today)

| Event | Fires when | Used for |
|---|---|---|
| `TaxedOwnersMemoCreated(guidNumber)` | After Create, `TaxPercent > 0`. | Promoted to integration event for VAT/e-invoice pipeline. |
| `TaxedOwnersMemoUpdated(ownerMemoId, totalBeforeTax, taxValue, transactionDate, ownerId, transactionSerial, transactionCode, internalId)` | After Update when tax changed or > 0. | Same. |
| `TaxedOwnersMemoDeleted(ownerMemoId)` | On `Delete()` when `TaxPercent > 0`. | Cleanup downstream. |

Live under `…\OwnerMemoAggregate\Events\`. Dispatched by `IDomainEventDispatcher` registered in `Construction.Data`.

---

## 12. API surface

Controller: `Construction.API\ApiControllers\OwnerMemoController\OwnerMemoController.cs`. `ModuleId` from `Modules` enum (legacy mirror in `AccModuleList.Owner_Memo`).

| Verb | Route | Command/Query | Policy |
|---|---|---|---|
| GET | `OwnerMemo/{id}` | `GetOwnerMemoByIdQuery` | `ViewPolicy` |
| GET | `OwnerMemo/search/...` | search query | `ViewPolicy` |
| GET | `OwnerMemo/jobitems/...` | helper for `IsAffectStatement` UI | `ViewPolicy` |
| POST | `OwnerMemo` | `CreateOwnerMemoCommand` | `AddPolicy` |
| PUT | `OwnerMemo/{id}` | `UpdateOwnerMemoCommand` | `EditPolicy` |
| DELETE | `OwnerMemo/{id}` | `DeleteOwnerMemoCommand` | `DeletePolicy` |
| GET | `OwnerMemo/{id}/print` | print data | `PrintPolicy` |

All return `ProblemDetails` on failure with stable error codes.

---

## 13. Application layer (CQRS)

Path: `Construction.Application\OwnerMemos\`

```
OwnerMemos/
  Commands/CreateOwnerMemo/CreateOwnerMemoCommand.cs    + Handler
  Commands/UpdateOwnerMemo/UpdateOwnerMemoCommand.cs    + Handler
  Commands/DeleteOwnerMemo/DeleteOwnerMemoCommand.cs    + Handler
  Queries/GetOwnerMemoById/...
  Queries/SearchOwnerMemos/...
  Queries/GetOwnerMemoJobItemsAdditionsSubtractions/... (UI hydration)
  Services/OwnerMemoService.cs
  MappingProfiles/OwnerMemoMappingProfile.cs
```

Each handler runs inside `IApplicationTransaction.BeginTransaction()` / `Commit()` / `RollBack()`. Flow: load referenced aggregates → call domain factory/mutation → save via `ConstructionContext.SaveChangesAsync` (pre/post-dispatches events) → commit or rollback. Returns `Result<int>` (Create) or `Result` (Update/Delete).

`UpdateOwnerStatementRemaining` listens to memo events and adjusts statement remaining quantities for `IsAffectStatement` memos.

---

## 14. Desktop screen — `frmOwnerMemoForm`

File: `…\Constraction.UI\OwnerMemo\frmOwnerMemoForm.cs` (DevExpress `RibbonForm`).

**Header controls:** `txtDocumentNumber`, `dtMemoDate`, `txtMemoNumber`, `txtJournalNumber`, `searchLkUpProject`, `txtOwner`, `searchLkUpCurrency`, `lkUpOwnerStatement`, `lkUpAccount`, `lkUpBranch`, `lkUpCostCenter`, `rdMemoType` (Debit/Credit), `chkboxisAffectStatement` (enabled on Credit only), `memoDescription`. Totals: `txtTotalAmount`, `txtTotalTax`, `txtTotalSourceTax`, `txtTaxValue`, `txtSourceTaxValue`. Audit: `dtCreationDate/ModificationDate`, `txtCreatedBy/ModifiedBy`. E-Invoice indicator: `chkIsSync` + `lySync` tooltip.

**Grids:** `grdVwMemo` (details), `grdVwAddtions` and `grdVwSubtractions` (visible only on `IsAffectStatement` today).

**Ribbon:** New, Open, Reset, Save, Delete, Print, View Journals, Attachments — state-driven via `ModuleRibbonButton.SetButtonsState` per `FormStates`.

**Validation pipeline:** `ValidateOwnerMemoForm` (required fields) → `ValidateOwnerMemo` (empty / duplicate guard) → per-grid `ValidateRow` (exceed-quantity / exceed-percentage). Server is source of truth.

**Authorization:** `Program.ValidateModuleOperation(this, ModuleOperation.AddNew/Update/Delete/Print)` against `AccModuleList.Owner_Memo`.

---

## 15. Desktop search — `frmOwnerMemoSearchForm`

Standard search dialog. Filters: project, owner, currency, date range, memo number, document number. Returns selected memo's `OwnerMemoId` to `Open()`.

---

## 16. Print — `rptOwnerMemoReport`

Files: `AccFlex_ERP\Constraction\Constraction.Reports\…\OwnerReport\Memo\`
- `dsOwnerMemo.xsd` — typed dataset (memo header + details only today).
- `rptOwnerMemoReport.cs/.Designer.cs`.

Constructor: `rptOwnerMemoReport(int ownerMemoId, bool isEInvoiceFeatureEnabled, bool isEInvocieSynced)`.

Layout is **details-only today** — no Additions/Subtractions bands. (The CR will add them.)

---

## 17. Angular surface

Path: `…\projects\construction\src\app\modules\owner-memo\`

```
owner-memo/
  containers/owner-memo-shell/owner-memo-shell.component.{ts,html}
  components/owner-memo-detail-grid/owner-memo-detail-grid.component.{ts,html}
  shared/owner-memo-form.service.ts
```

Mirrors the desktop two-mode behaviour. Additions/Subtractions visibility is gated on `IsAffectStatement` today. i18n keys live in `src\assets\i18n\{ar-EG,en-US}\construction.json`.

---

## 18. Reporting (AnalyticsReader)

Memo data participates in cross-module Owner reports via the OData + EF + `IAngularReport<T>` 4-artifact pattern under `AnalyticsReader.Api\…\Construction\`. Memo data feeds into `Cnt_Rpt_OwnerSubLedger` and `Cnt_ProjectDetailedReport` (SQL functions in `AccFlex_Cloud_Database_State\dbo\Functions\`). No dedicated memo report exists today.

---

## 19. Tests

| Project | Path | Scope |
|---|---|---|
| Unit (if present) | `Construction.Logic.Tests` | Aggregate factory + journal + helpers. |
| Integration | `…\ConstructionAPITests\Services\OwnerMemoService.cs` | E2E command/query against test DB. |

---

## 20. Known constraints / gaps (baseline — motivate CR 116591)

- **Standalone memos cannot post per-account legs.** The single `AccountId` is the only non-owner journal leg.
- **Standalone memos cannot carry additions/subtractions** even though the collections exist on the aggregate.
- **`IsAffectStatement` is Credit-only** (R15).
- **`IsLast` statement rule** blocks adjustments to the final statement (R16).
- **`hasLaterStatement` semantics inverted** in `ValidateOwnerMemo:202` — beware when reading.
- **No explicit `Status` enum.** State is derived.
- **No per-statement-sequence guard for standalone memos** — once additions/subtractions are added to the standalone path (CR 116591), the new positional guard becomes important.
- **No cross-document e-invoice rule today** — only the document's own e-invoice protects itself.

---

## 21. Pointers for new readers

| Concern | File |
|---|---|
| Aggregate root + factory + validation | `Construction.Logic\OwnerMemoAggregate\OwnerMemo.cs` |
| Journal-action construction | `…\OwnerMemoAggregate\OwnerMemo.Journal.cs` |
| Totals / tax / source-tax math | `…\OwnerMemoAggregate\OwnerMemo.Helper.cs` |
| Error catalog | `…\OwnerMemoAggregate\OwnerMemoErrors.cs` |
| Child entities | `…\OwnerMemoDetail.cs`, `…\OwnerMemoAddition.cs`, `…\OwnerMemoSubtraction.cs` |
| Commands | `Construction.Application\OwnerMemos\Commands\…\…Command.cs` |
| Desktop form | `…\Constraction.UI\OwnerMemo\frmOwnerMemoForm.cs` |
| Print | `…\Constraction.Reports\…\OwnerReport\Memo\rptOwnerMemoReport.cs` |
| Angular | `…\projects\construction\src\app\modules\owner-memo\` |
| Cross-aggregate canonical journal reference | `…\OwnerStatementAggregate\OwnerStatement.Journals.cs` |

---

## 22. Change log

| Date       | Author                           | Change                                                                                                                                                                                                                                       |
| ---------- | -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-05-20 | hossam.shafik@hadafsolutions.net | Baseline as-is documentation captured pre-CR-116591.                                                                                                                                                                                         |
| _pending_  | _pending_                        | Update after CR 116591 (Price-Differences Memo) implementation lands: new flag, journal branch, domain events, statement remaining-amount linkage, additions/subtractions in standalone mode, sequence guard, cross-document e-invoice rule. |
