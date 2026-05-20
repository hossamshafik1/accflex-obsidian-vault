---
description: "End-to-end scaffolding guide for Angular report modules. Covers module creation, routing, lazy loading, OData endpoint connection to AnalyticsReader API, report-grid component selection, i18n translation keys, query options, filter forms, and ApplicationId mapping. Use when creating a new report module from scratch or wiring a report to the backend. Complements report-module.mdc which covers the 3 component patterns in detail."
globs:
  - "**/basic-reports/**/*"
  - "**/*-report/**/*"
  - "**/*report*.module.ts"
  - "**/*report*.component.ts"
  - "**/*report*shell*.ts"
  - "**/*report*routing*.ts"
alwaysApply: false
---

# Report Module Scaffolding Guide

## Overview

This skill guides creating a **new report module from scratch** in the Angular frontend. It covers the full scaffolding chain: module → routing → component → OData connection → i18n.

For the 3 component patterns (Simple OData, Master-Detail, Popup Details), see `mdc:report-module`.

For the backend 4-artifact pattern, see `mdc:report-backend`.

For SQL report functions, see `mdc:report-database-functions`.

---

## Step-by-Step Scaffolding Checklist

| # | Step | Location |
|---|------|----------|
| 1 | Create module folder | `projects/{app}/src/app/modules/{report-name}/` |
| 2 | Create module file | `{report-name}.module.ts` |
| 3 | Add lazy-loaded route in parent routing | Parent routing module |
| 4 | Create component folder | `components/{report-name}/` or `{report-name}/` |
| 5 | Create component `.ts` + `.html` | Component files |
| 6 | Add i18n keys | `src/assets/i18n/en-US/{app}.json` + `ar-EG/{app}.json` |
| 7 | Wire report-grid to OData endpoint | Component template |
| 8 | Configure query options | Component class |

---

## Step 1: Module File

```typescript
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterModule } from '@angular/router';
import { AccflexSharedModule } from 'src/app/shared/accflex-shared.module';
import { CashControlSharedModule } from '../../shared/cash-control-shared.module'; // or project-specific shared
import { ReportComponent } from './components/report-name/report-name.component';

@NgModule({
  declarations: [ReportComponent],
  imports: [
    CommonModule,
    AccflexSharedModule,
    CashControlSharedModule,  // Replace with project-specific shared module
    RouterModule.forChild([
      { path: '', component: ReportComponent }
    ])
  ],
  providers: [
    ODataService,
    // Add API clients needed for lookup dropdowns
    // e.g., BankClient, CustomerClient, BranchClient
  ]
})
export class ReportNameModule {}
```

### Key Rules

- Always import `AccflexSharedModule` (provides report-grid, shared components)
- Always import the project-specific shared module (provides project lookups)
- Use `RouterModule.forChild` with empty path — the parent does lazy loading
- Declare only the report component(s) in this module
- Provide `ODataService` and any API clients for filter dropdowns

---

## Step 2: Lazy-Loaded Route in Parent

Add the route in the parent routing module (e.g., `cash-control-routing.module.ts`):

```typescript
{
  path: 'bank-balance-report',
  loadChildren: () =>
    import('./modules/bank-balance-report/bank-balance-report.module')
      .then(m => m.BankBalanceReportModule),
  data: { permissionId: 1234 }  // Permission ID from backend
}
```

### Route Naming Convention

- Use **kebab-case** for route paths: `bank-balance-report`, `treasury-transactions-report`
- Match the module folder name to the route path
- Always include `data: { permissionId }` for access control

---

## Step 3: OData Endpoint Connection

### How the Frontend Connects to AnalyticsReader

```
Angular Component
  → report-grid / report-grid-multi-request component
    → DevExtreme ODataStore
      → POST {analyticsreaderapi}/odata/{EndPoint}/$query
        → AnalyticsReader API (backend)
          → SQL Function (database)
```

### API Base URL

Configured in `src/assets/configs/config.json`:

```json
{
  "analyticsreaderapi": "https://localhost:5055/"
}
```

Accessed at runtime via `configStorageGetter()?.apiUrls?.analyticsreaderapi`.

### Endpoint Name

The `endpoint` or `headerendpoint` input on the report-grid component must match the **EntitySet name** defined in the backend OData Entity's `Configure()` method:

```
Frontend: headerendpoint="BankBalanceReport"
  ↕ must match ↕
Backend:  builder.EntitySet<BankBalanceReport>("BankBalanceReport")
```

### Authentication

All requests automatically include:
- `Authorization: Bearer {access_token}` (from OIDC service)
- `Accept-Language: {language}` (from RTL service)
- `Content-Type: text/plain`
- Method: `POST`

---

## Step 4: Choosing the Right Report-Grid Component

| Component | Use When | Endpoint Inputs |
|-----------|----------|-----------------|
| `report-grid` | Simple single-level report with OData filters | `endpoint` |
| `report-grid-multi-request` | Master-detail with expandable rows and tabs | `headerendpoint` + `detailendpoints[]` |
| `report-grid-multi-request-with-popups` | Multi-level drill-down with popup details | `headerendpoint` (details via events) |

### Pattern 1: Simple `report-grid`

```html
<report-grid
  [ApplicationId]="7"
  [keyGrid]="keyGrid"
  [ReportName]="reportName"
  [QueryFilter]="QueryFilter"
  endpoint="BankBalanceReport"
  [ClearFilterClicked]="ClearFilters"
  [DisabledPreview]="!reportForm.valid"
  [moneyColumns]="[{ DataField: 'Amount' }]"
  [orderby]="['DateField']"
>
</report-grid>
```

### Pattern 2: `report-grid-multi-request`

```html
<report-grid-multi-request
  [ApplicationId]="7"
  headerendpoint="BankGuaranteeReport"
  [keyGrid]="keyGrid"
  [details]="Details"
  [detailendpoints]="['TransactionDetail']"
  [ReportName]="reportName"
  [DisabledPreview]="!reportForm.valid"
  [headerqueryoptions]="HeaderQueryOptions"
  [detailqueryoptions]="[DetailQueryOptions]"
  [ClearFilterClicked]="ClearFilters"
  [columnIndex]="columnsIndex"
  [detailColumnIndex]="DetailColumnIndex"
  [headerfilter]="headerFilters"
  [moneyColumns]="MoneyColumns"
  (onDetailExpand)="onDetailExpand($event)"
>
</report-grid-multi-request>
```

### Pattern 3: `report-grid-multi-request-with-popups`

```html
<accflex-report-grid-multi-request-with-popups
  [ApplicationId]="7"
  headerendpoint="HeaderEndpoint"
  keyGrid="Id"
  ReportName="ReportName"
  [DisabledPreview]="!reportForm.valid"
  [headerqueryoptions]="HeaderQueryOptions"
  [ClearFilterClicked]="ClearFilters"
  [headerColumns]="headerColumns"
  [headerfilter]="headerFilters"
  (onDetailClicked)="onDetailClicked($event)"
  (OnPreviewClick)="OnPreviewClick($event)"
>
</accflex-report-grid-multi-request-with-popups>
```

---

## Step 5: ApplicationId Mapping

Each sub-application has a unique ApplicationId used by the report grid for layout persistence:

| ApplicationId | Sub-Application |
|---------------|-----------------|
| 7 | CashControl |
| 8 | Construction |
| 10 | GeneralAccounting |
| 12 | Logistics |
| 13 | Payroll |
| 14 | PropertyManager |

Always pass the correct `[ApplicationId]` for your sub-application.

---

## Step 6: ReportName Usage

The `ReportName` string is used for:

1. **Grid layout persistence** — save/load user column customizations
   - Header grid key: `{ReportName}WebReportGrid`
   - Detail grid keys: `{ReportName}WebDetailGrid0`, `{ReportName}WebDetailGrid1`, etc.
2. **User design storage** — `{ReportName}UserDesign -- {userId}`
3. **Print/export** — identifies the report in the print layout system

```typescript
reportName = 'BankBalanceReport';  // Must be unique across the application
```

**Convention**: Use the same string as the backend `IAngularReport<T>.ReportName` property.

---

## Step 7: Query Options vs OData Filters

There are **two ways** to pass parameters to the backend:

### QueryOptions (Custom Parameters → IAngularReport.GetDataSource)

Passed via `[headerqueryoptions]` and `[detailqueryoptions]`. These become the `queryOptions` parameter in the backend `GetDataSource()` method.

```typescript
// Component
HeaderQueryOptions: any = {};

initializeQueryOptions() {
  this.HeaderQueryOptions['fromDate'] = startOfDay(new Date(this.reportForm.value.FromDate));
  this.HeaderQueryOptions['toDate'] = endOfDay(new Date(this.reportForm.value.ToDate));
  this.HeaderQueryOptions['bankIdList'] = this.reportForm.value.BankIds?.join(',');
}
```

**Use for**: dates, ID lists, flags, year — parameters consumed by the SQL function.

### OData Filters (→ $filter query string)

Passed via `[QueryFilter]` (Pattern 1) or `[headerfilter]` (Patterns 2/3). These become standard OData `$filter` expressions.

```typescript
// Pattern 1: buildQuery
import buildQuery from 'odata-query';
this.QueryFilter = buildQuery({ filter: { DateField: { ge: fromDate, le: toDate } } });

// Patterns 2/3: ODataService
this.headerFilters = this.odataService.jsonConcat(
  { BankId: { in: this.reportForm.controls.BankIds?.value } },
  this.headerFilters
);
```

**Use for**: filtering OData entity properties directly (when not using SQL function parameters).

### Which to Use?

- **SQL function-backed reports** (most reports): Use **QueryOptions** → parameters go to `GetDataSource` → `FromSqlInterpolated`
- **View/table-backed reports** (rare): Use **OData Filters** → standard `$filter` applied by OData middleware

---

## Step 8: Filter Form Pattern

```typescript
import { FormBuilder, FormControl, FormGroup, Validators } from '@angular/forms';
import { endOfDay, startOfDay } from 'date-fns';

// Initialize form
private initForm() {
  this.reportForm = this.formBuilder.group({
    FromDate: [startOfDay(new Date()), Validators.required],
    ToDate: [endOfDay(new Date()), Validators.required],
    BankIds: [[]],              // Multi-select lookup
    ShowInactive: [false],      // Checkbox filter
  });

  // Auto-regenerate on changes
  this.reportForm.valueChanges
    .pipe(catchError(err => EMPTY))
    .subscribe(() => this.generateRequest());
}

// Clear filters
ClearFilters = () => {
  this.reportForm.patchValue({
    FromDate: startOfDay(new Date()),
    ToDate: endOfDay(new Date()),
    BankIds: [],
    ShowInactive: false,
  });
};
```

### Date Range Validation (Required in Every Report)

```typescript
if (this.reportForm.controls.ToDate.value < this.reportForm.controls.FromDate.value) {
  this.reportForm.controls.FromDate.setErrors(
    this.translate.instant('COMMON._dateOutOfRangeMessage_')
  );
  this.toastrService.showErrorToast(
    this.translate.instant('COMMON._dateOutOfRangeMessage_')
  );
  return;
}
```

---

## Step 9: i18n Translation Keys

### File Locations

| Language | Path |
|----------|------|
| English | `src/assets/i18n/en-US/{app-name}.json` |
| Arabic | `src/assets/i18n/ar-EG/{app-name}.json` |

### Key Structure

```json
{
  "BANKREPORT": {
    "_title_": "Bank Balance Report",
    "_fromDate_": "From Date",
    "_toDate_": "To Date",
    "_bankName_": "Bank Name",
    "_balance_": "Balance",
    "_expectedBalance_": "Expected Balance"
  }
}
```

### Common Shared Keys (Already Available)

Use keys from `COMMON` and `ODATAREPORTS` namespaces — don't duplicate:

```typescript
// Already available — don't create new keys for these
'COMMON._FromDate_'              // "From Date"
'COMMON._ToDate_'                // "To Date"
'COMMON._dateOutOfRangeMessage_' // Date validation error
'ODATAREPORTS.SaveGridDesign'    // Grid design buttons
'ODATAREPORTS.ResetGridDesign'   // Grid design buttons
```

### Usage in Templates

```html
<h5>{{ 'BANKREPORT._title_' | translate }}</h5>

<dx-date-box
  [label]="('COMMON._FromDate_' | translate) + ' * '"
  formControlName="FromDate"
></dx-date-box>
```

---

## Step 10: Column Configuration

### ColumnIndex (Patterns 1 & 2)

```typescript
import { ColumnIndex } from 'src/app/shared/components/report-grid/ColumnIndex';

columnsIndex: ColumnIndex[] = [
  { DataField: 'BankName', Index: 0 },
  { DataField: 'BeginningBalance', Index: 1 },
  { DataField: 'TotalReceive', Index: 2 },
  { DataField: 'TotalPaid', Index: 3 },
  { DataField: 'Balance', Index: 4 },
  // RTL-aware column
  { DataField: 'NameAr', Index: 5, Visible: this.isRtlEnabled },
  { DataField: 'NameEn', Index: 5, Visible: !this.isRtlEnabled },
  // Hidden column
  { DataField: 'BankId', Index: 6, Visible: false },
];
```

### MoneyColumn (Numeric Formatting)

```typescript
import { MoneyColumn } from 'src/app/shared/components/report-grid/MoneyColumn';

moneyColumns: MoneyColumn[] = [
  { DataField: 'BeginningBalance' },
  { DataField: 'TotalReceive' },
  { DataField: 'TotalPaid' },
  { DataField: 'Balance' },
  { DataField: 'ExpectedBalance' },
];
```

### Column Service (Pattern 3 Only)

For complex reports with drill-down popups, create a dedicated column service:

```typescript
// report-columns.service.ts
import { Injectable } from '@angular/core';
import { Column } from 'devextreme/ui/data_grid';

@Injectable()
export class ReportColumnsService {
  constructor(
    private rtlService: RTLService,
    private translate: TranslateService
  ) {}

  get headerColumns(): Column[] {
    return [
      {
        dataField: 'BankName',
        caption: this.translate.instant('BANKREPORT._bankName_'),
        alignment: 'center',
      },
      {
        dataField: 'Balance',
        caption: this.translate.instant('BANKREPORT._balance_'),
        dataType: 'number',
        format: { type: 'fixedPoint', precision: 2 },
      },
      {
        name: 'details',
        caption: this.translate.instant('BANKREPORT._details_'),
        cellTemplate: 'detailTemplate',  // Drill-down button
        alignment: 'center',
      },
    ];
  }
}
```

---

## Step 11: RTL Support (Required)

Every report component must support RTL:

```typescript
public get isRtlEnabled(): boolean {
  return this.rtlService.IsRtlEnabled;
}

public get dateTimeFormat(): string {
  return this.rtlService.DateTimeFormatOnRTL;
}
```

Use in templates:

```html
<dx-date-box
  [displayFormat]="dateTimeFormat"
  [rtlEnabled]="isRtlEnabled"
></dx-date-box>
```

For bilingual columns:

```typescript
{ DataField: this.isRtlEnabled ? 'BankNameAr' : 'BankNameEn', Index: 0 }
```

---

## Step 12: Detail Expansion (Patterns 2 & 3)

### Pattern 2: Build Detail Filters on Row Expand

```typescript
onDetailExpand = (e) => {
  const row = e?.parentRowData;
  this.detailFilters.length = 0;

  // Build filter for detail tab
  let detailFilter = new Object();
  if (row?.BankId) {
    detailFilter = this.odataService.jsonConcat(
      { BankId: { in: [row?.BankId] } },
      detailFilter
    );
  }
  this.detailFilters.push(detailFilter);
};
```

### Pattern 3: Handle Drill-Down Clicks

```typescript
onDetailClicked = (e: DetailClickedEventArgs) => {
  if (e?.event?.column.name === 'details') {
    e.DetailEndPoint = 'DetailEndpointName';
    e.DetailQueryOptions = {
      parentId: e.row?.data?.Id,
      fromDate: this.reportForm?.controls.FromDate?.value,
      toDate: this.reportForm?.controls.ToDate?.value,
    };
    e.detailColumns = this.columnsService.detailColumns;
    e.detailTitle = this.translate.instant('REPORT._DetailTitle_');
    e.detialKeyGrid = 'DetailId';
  }
};
```

---

## Complete Module Folder Structure

```
projects/{app}/src/app/modules/{report-name}/
├── {report-name}.module.ts
└── components/
    └── {report-name}/
        ├── {report-name}.component.ts
        ├── {report-name}.component.html
        └── {report-name}-columns.service.ts     (Pattern 3 only)
```

---

## Common Imports Reference

```typescript
// Always needed
import { Component, OnDestroy } from '@angular/core';
import { FormBuilder, FormControl, FormGroup, Validators } from '@angular/forms';
import { TranslateService } from '@ngx-translate/core';
import { endOfDay, startOfDay } from 'date-fns';
import { EMPTY, catchError } from 'rxjs';
import { RTLService } from 'src/app/shared/services/RTL/rtl.service';
import { SubSink } from 'subsink';

// Pattern 1
import buildQuery from 'odata-query';
import { ColumnIndex } from 'src/app/shared/components/report-grid/ColumnIndex';

// Patterns 2 & 3
import { ODataService } from 'src/app/shared/services/ODataServices/odata-service';
import { MoneyColumn } from 'src/app/shared/components/report-grid/MoneyColumn';

// Pattern 3
import { Column } from 'devextreme/ui/data_grid';
import { DetailClickedEventArgs } from 'src/app/shared/components/report-grid-multi-request-with-popups/detail-clicked-event';
import { PreviewArgs } from 'src/app/shared/components/report-grid/PreviewArgs';
```
