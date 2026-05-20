# Owner Memo — Price-Differences Memo (CR 116591) — Implementation Plan

> **ADO Change Request:** [#116591 — اضافة الخصومات و الاضافات في شاشة إشعار المالك](https://hadafsolutions.visualstudio.com/AccFlex_ERP/_workitems/edit/116591)
> **Customer:** دار المعمار · **WorkType:** API+Desktop+Web · **SP:** 10 · **Due:** 2026-07-15
> **Parent:** 92847 · **Related:** 119445
> **Module:** Construction → Owner Memo

## 1. Context

Customer (Dar Al Mimar) issues a periodic "Currency-Differences Statement" through Owner Memo. The current screen lacks Additions/Subtractions for standalone memos, and the journal always nets to a single fixed memo account — the per-account legs (retention, work-guarantee, …) the business needs to move are not touched.

CR 116591 asks for:
- **A**: Make Additions/Subtractions available on every Owner Memo + include them in the default print.
- **B**: A new flag `IsPriceDifferenceMemo` — posts each detail/addition/subtraction to its own account, omits the fixed memo-account leg, Debit-only, mutually exclusive with `IsAffectStatement`.

Desktop ships first; API contract bumps first.

## 2. Locked decisions

| # | Decision |
|---|---|
| **D1** | No journal-builder refactor. OwnerMemo owns its own price-diff journal; OwnerStatement.Journals.cs is read-only reference. |
| **D2** | Tax % comes from the linked Owner Statement's `TaxPercentage`. |
| **D3** | Source tax allowed on price-diff memos. |
| **D4** | Exactly one source-tax subtraction line per memo. |
| **D5** | Rounding plug absorbed into the owner journal line (tolerance `10^(-approximateNo)`). |
| **D6** | `IsPriceDifferenceMemo` set-on-create only. Immutable on Update. |
| **D7** | Create/Update/Delete adjusts linked Owner Statement's remaining amount via MediatR domain events in the same `IApplicationTransaction`. |
| **D8** | Blocked when a later non-deleted Owner Statement exists in the project's positional sequence (`ExtractionNumber > current`). `IsLast` flag is NOT the check. |
| **D9** | E-invoice guard: each doc's own sync protects itself; a synced child Memo also blocks parent Statement Update/Delete (D9-iii). A synced parent Statement does NOT block Memo Create/Update/Delete (D9-iv, D9-v). KSA on-boarded tenant trumps. |

## 3. Implementation summary

### Phase 4 — API / Logic / DB

- **4.1 OwnerMemo aggregate** — add `IsPriceDifferenceMemo` (private setter), extend Create/Update/ValidateOwnerMemo signatures with `isPriceDifferenceMemo` + `hasLaterNonDeletedStatement`; add new error keys (`MutuallyExclusiveMemoModes`, `PriceDifferenceMemoMustBeDebit`, `OnlyOneSourceTaxSubtractionAllowed`, `CannotChangePriceDifferenceFlag`, `CannotModifyPriceDiffMemoWhenLaterStatementExists`, `JobItemAccountRequired`); on Update verify flag immutability (D6); change `if (isAffectStatement)` build/tax-% blocks to `if (isAffectStatement || isPriceDifferenceMemo)`.
- **4.2 New journal branch** — `CreatePriceDifferenceJournalActionList` in `OwnerMemo.Journal.cs`, mirrors `OwnerStatement.Journals.cs:172-282`:
  - Subtractions → `sub.AccountId` (Debit `sub.MemoValue`)
  - Additions → `add.AccountId` (Credit `add.MemoValue`)
  - Job items grouped by item account → Credit `Σ(detail.Total)`
  - VAT → tax account (Credit `totalMemoTax`)
  - **Owner line** → Debit plug, with rounding reconciliation (D5)
  - No memo-account leg.
- **4.3 VAT** — drop any `if (isAffectStatement)` gates in `CreateValueAddedTaxs`.
- **4.4 Domain events** — `PriceDifferenceOwnerMemoCreated/Updated/Deleted` records emitted from aggregate gated on `IsPriceDifferenceMemo` (existing `TaxedOwnersMemo*` events still fire in parallel).
- **4.5 OwnerStatement aggregate** — add `decimal PriceDifferenceMemoTotal`; new methods `ApplyPriceDifferenceMemo` / `ApplyPriceDifferenceMemoDelta` / `RevertPriceDifferenceMemo` returning `Result`; fold into the existing remaining-amount calc in `OwnerStatement.Balance.cs`; extend `IsValidToUpdate` / `IsValidToDelete` with `IReadOnlyList<EInvoice> linkedMemoEInvoices` for D9-iii; new errors `LinkedMemoEInvoiceIsReadyToSync` / `LinkedMemoEInvoiceIsSynced`.
- **4.6 Application layer** — commands + DTOs add the flag; `GetOwnerMemoByIdQuery` projects it; new `IConstructionContext` helpers:
  - `AnyLaterNonDeletedStatementExistsAsync(projectId, referenceStatementId, ct)` — D8
  - `GetOwnerMemoEInvoicesForStatementAsync(ownerStatementId, ct)` — D9-iii
  Update/Delete handlers pass results in; `UpdateOwnerMemoCommandHandler` reads existing aggregate's flag and passes it in (don't trust client payload). Three `INotificationHandler<>` for the new events — load statement, call apply/revert/delta, then `LinkMemoToOwnerStatement` / `UnLinkMemoToOwnerStatement` (existing methods on `OwnerMemo.cs:1099` / `:1109`).
- **4.7 Data / EF / Migration:**
  ```sql
  ALTER TABLE OwnerMemo
    ADD IsPriceDifferenceMemo BIT NOT NULL
        CONSTRAINT DF_OwnerMemo_IsPriceDifferenceMemo DEFAULT 0;

  ALTER TABLE OwnerStatement
    ADD PriceDifferenceMemoTotal DECIMAL(18,4) NOT NULL
        CONSTRAINT DF_OwnerStatement_PriceDifferenceMemoTotal DEFAULT 0;
  ```
  EF default mappings on both configurations; regen NSwag → publish `Construction.API.Client`.

### Phase 1 — Desktop UI (`AccFlex_ERP\Constraction\Constraction.UI\OwnerMemo`)

| # | File | Change |
|---|---|---|
| 1 | `Vms\OwnerMemoVm.cs` | Add `bool IsPriceDifferenceMemo`; keep `OwnerMemoAdditionList` / `OwnerMemoSubtractionList` always populated. |
| 2 | `frmOwnerMemoForm.Designer.cs` | Add `chkboxIsPriceDifferenceMemo`. |
| 3 | `frmOwnerMemoForm.cs` — `ManageAdditionAndSubtractionTabsVisiblility` | Tabs **always visible**; drop the `IsAffectStatement` gate. |
| 4 | `rdMemoType_EditValueChanged` | Enable new checkbox only on Debit; uncheck on Credit. |
| 5 | New `chkboxIsPriceDifferenceMemo_CheckedChanged` | Mutual exclusion with `chkboxisAffectStatement`; pre-load `ownerStatementDto`; lock `txtTotalTax` to statement's tax % (D2). |
| 6 | `ManageAffectsStatementQuantitiesConsequences` | Stop wiping addition/subtraction lists when `IsAffectStatement` off; source from masters when neither flag is on. |
| 7 | `frmOwnerMemoForm.Api.cs` | Map flag + always send additions/subtractions. |
| 8 | Validation | Block both flags ticked; block flag + Credit; surface server-side errors. |
| 9 | `FillControls` / `FillData` | Bind new checkbox; hydrate lists in all modes. |
| 10 | `Open(entityId)` | Disable both flag checkboxes after load (D6). |
| 11 | `brBtnNew_ItemClick` / `lkUpOwnerStatement_EditValueChanged` | Pre-flight `CanCreatePriceDifferenceMemoOnStatement(statementId)`; disable flag if false (D8). |
| 12 | Grid setup | Expose Account column on Additions/Subtractions grids when flag is on; guard against second `IsSourceTax` row (D4). |
| 13 | `.resx` / `.ar-EG.resx` / `.en-US.resx` | Labels + new error messages. |
| 14 | `frmOwnerMemoSearchForm.cs` | New column / filter. |

### Phase 3 — Desktop print

Files under `…\Constraction.Reports\…\OwnerReport\Memo\`:
- `dsOwnerMemo.xsd` — add `OwnerMemoAdditionList` and `OwnerMemoSubtractionList` data tables.
- `rptOwnerMemoReport.cs` / `.Designer.cs` — add Additions / Subtractions detail bands + totals footer.
- Show "إشعار فروق أسعار" badge when `IsPriceDifferenceMemo == true`.

### Phase 5 — Angular (after desktop ships)

Files under `…\projects\construction\src\app\modules\owner-memo\`:
- `shared\owner-memo-form.service.ts` — add control + validators.
- `containers\owner-memo-shell\` — header checkbox; tabs always visible.
- `components\owner-memo-detail-grid\` — item-account column when flag is on.
- `src\assets\i18n\{ar-EG,en-US}\construction.json` — new keys.
- Disable checkbox when `entityId > 0` (D6).
- Pre-flight `CanCreatePriceDifferenceMemoOnStatement` (D8).

## 4. Critical files map

**Read-only references:**
- `…\OwnerStatementAggregate\OwnerStatement.Journals.cs` — canonical pattern (D1).
- `OwnerMemo.cs:1099` — `LinkMemoToOwnerStatement` (reuse).
- `OwnerMemo.cs:1109` — `UnLinkMemoToOwnerStatement` (reuse).

**Files to be modified:**
- `Construction.Logic\OwnerMemoAggregate\OwnerMemo.cs` · `OwnerMemo.Journal.cs` · `OwnerMemoErrors.cs` · `Events\` (new)
- `Construction.Logic\OwnerStatementAggregate\OwnerStatement.cs` · `OwnerStatement.Balance.cs`
- `Construction.Logic\Resources\Resource.resx` + `Resource.en.resx`
- `Construction.Application\OwnerMemos\Commands\{Create,Update,Delete}\…Command.cs`
- `Construction.Application\OwnerMemos\Queries\GetOwnerMemoById\GetOwnerMemoByIdQuery.cs`
- `Construction.Application\OwnerMemos\EventHandlers\` (3 new)
- `Construction.Application\Common\Interfaces\IConstructionContext.cs`
- `Construction.Data\ConstructionContext.cs` · `OwnerMemoConfiguration.cs` · `OwnerStatementConfiguration.cs`
- `AccFlex_Cloud_Database_State\…\OwnerMemo` + `OwnerStatement` table scripts
- `AccFlex_ERP\Constraction\Constraction.UI\OwnerMemo\` (form, designer, API, search, VM, resx)
- `AccFlex_ERP\Constraction\Constraction.Reports\…\Memo\dsOwnerMemo.xsd` · `rptOwnerMemoReport.cs/.Designer.cs`
- `AccFlex_Cloud_Front\accflex-erp\projects\construction\src\app\modules\owner-memo\` + i18n

## 5. Test catalog

**Journal:**
1. Items only → per-item-account credit + owner debit, no memo-account leg, balanced.
2. Items + additions + non-source subs + VAT → 4-segment balanced, VAT correct, rounding plug on owner line (D5).
3. One source-tax sub → `CustomerSourceTax` created with sub's account, percent = amount/totalJobItemAmount (D3).
4. Two source-tax subs → fail `OnlyOneSourceTaxSubtractionAllowed` (D4).

**Validation:**
5. Flag + `IsAffectStatement` → fail.
6. Flag + Credit → fail.
7. Detail without item account → fail.
8. Tax % override (D2): user-typed % ignored, statement's value used.

**Regression:**
9. Existing `IsAffectStatement` + standalone journals byte-equal to baseline.

**Domain events (D7):**
10. Create → `PriceDifferenceOwnerMemoCreated` fires; statement's `PriceDifferenceMemoTotal` += memo total; `memo.AffectedStatementId == OwnerStatementId`.
11. Update → `PriceDifferenceOwnerMemoUpdated`; statement reflects delta only.
12. Delete → `PriceDifferenceOwnerMemoDeleted`; statement reverts; `AffectedStatementId` cleared.
13. Transactional rollback on handler failure → no persistence.
14. Re-applying after Delete: no double-apply.

**Immutability (D6):**
15. Update flipping flag → fail `CannotChangePriceDifferenceFlag`; no events.

**Sequence guard (D8):**
16. Statements (1,2,3,4); Create memo on 3 → fail.
17. Soft-delete 4 → Create on 3 succeeds.
18. Memo on 3, then add 4 → Update fails.
19. Same → Delete fails.
20. `IsLast = false` mid-flow with no statement 4 → Create on 3 succeeds.

**E-invoice (D9):**
21. Memo Pending, statement Synced → Update succeeds (D9-iv).
22. Same → Delete succeeds (D9-iv).
23. Memo Synced → Update fails (D9-i).
24. Memo Synced → Delete fails (D9-i).
25. Statement Synced → memo Create succeeds (D9-v).
26. Statement Ready_To_Sync → memo Create succeeds (D9-v).
27. Create-on-Synced then sync memo → memo Update fails (D9-i).
28. Synced child memo → Statement Update fails `LinkedMemoEInvoiceIsSynced` (D9-iii).
29. Ready_To_Sync child memo → Statement Delete fails (D9-iii).
30. Only Pending child memos → Statement Update succeeds.
31. KSA on-boarded tenant — all directions block.

## 6. Sequencing

| Day | Work |
|---|---|
| D1 | Phase 4.1 + 4.5 (domain fields, validation, errors, EF, SQL migrations). |
| D2 | Phase 4.2 / 4.3 / 4.4 (journal branch, VAT, events) + 4.6 (commands, handlers, context helpers). |
| D3 | NSwag regen. Tests 1–9. Tests 10–31. |
| D4–5 | Phase 1 desktop UI. |
| D5 | Phase 3 print. |
| D6 | PO demo. |
| D7+ | Phase 5 Angular. |

## 7. Verification

**Build/run:**
```
dotnet build
dotnet run --project AccFlex_Cloud_ERP/Construction/Construction.API/Construction.API.csproj
```

**Tests:**
```
dotnet test Construction.Logic.Tests --filter "FullyQualifiedName~OwnerMemo"
dotnet test ConstructionAPITests --filter "Category=OwnerMemo"
```

**Desktop smoke:**
1. New → project → statement with no later non-deleted statements → Debit → tick Price-Differences Memo → confirm tabs + account column visible → add item + addition + non-source sub → Save → reopen → flag disabled.
2. Print preview → bands + badge.
3. API GET → flag and totals match.
4. DB: `SELECT IsPriceDifferenceMemo, TotalAmount FROM OwnerMemo`; `SELECT PriceDifferenceMemoTotal FROM OwnerStatement` matches.
5. Add later statement → Update memo → expect D8 error.
6. Sync memo's e-invoice → Update → D9-i. Update parent Statement → D9-iii.
7. Delete memo → `PriceDifferenceMemoTotal` reverts.

**Pre-merge:** all tests green · NSwag client published · DB migration reviewed · AR+EN resources reviewed · PO sign-off.
