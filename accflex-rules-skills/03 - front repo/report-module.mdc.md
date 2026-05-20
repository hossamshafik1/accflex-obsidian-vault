---
description: "Patterns and structure for Angular report modules that display OData data with filtering. Covers three patterns: (1) Simple OData with report-grid and buildQuery, (2) Master-Detail with report-grid-multi-request and expandable rows, (3) Advanced with report-grid-multi-request-with-popups and column service. Use when implementing or modifying analytical reports, basic-reports, or any screen that uses report-grid, date-range filters, or OData endpoints."
globs:
  - "**/basic-reports/**/*"
  - "**/*-report/**/*"
  - "**/*report*.component.ts"
  - "**/*report*.module.ts"
  - "**/*report*shell*.ts"
alwaysApply: false
---

# Report Module Patterns

## Overview

Report modules are used to display data from OData endpoints with filtering capabilities. There are three distinct patterns based on complexity and functionality requirements.

---

## Pattern 1: Simple OData Report Pattern

**Use Case**: Basic reports with simple filtering and optional detail expansion.

**Identifying Characteristics**:

- Uses `report-grid` component
- Uses `buildQuery` from `odata-query` package
- Simple filter concatenation
- Single OData endpoint
- Optional nested details through DetailPropertyNames

### Module Structure

```typescript
import { NgModule } from "@angular/core";
import { RouterModule } from "@angular/router";
import { AccflexSharedModule } from "src/app/shared/accflex-shared.module";
import { Client1, Client2, Client3 } from "logistics.api.client";
import { LogisticsSharedModule } from "../../shared/logistics-shared.module";
import { SharedService } from "../../shared/services/shared-service/shared.service";
import { ReportComponent } from "./report/report.component";

@NgModule({
  declarations: [ReportComponent],
  imports: [
    AccflexSharedModule,
    LogisticsSharedModule,
    RouterModule.forChild([{ path: "", component: ReportComponent }]),
  ],
  providers: [
    SharedService, // Optional shared service
    Client1, // API clients for lookups
    Client2,
    Client3,
  ],
})
export class ReportModule {}
```

### Component Structure

```typescript
import { Component, OnDestroy } from "@angular/core";
import {
  FormBuilder,
  FormControl,
  FormGroup,
  Validators,
} from "@angular/forms";
import { TranslateService } from "@ngx-translate/core";
import { endOfDay, startOfDay } from "date-fns";
import buildQuery from "odata-query";
import { EMPTY, Observable } from "rxjs";
import { catchError } from "rxjs/operators";
import { ColumnIndex } from "src/app/shared/components/report-grid/ColumnIndex";
import { ErrorHandlerService } from "src/app/shared/services/error-handler/error-handler.service";
import { SubSink } from "subsink";
import { ODataService } from "src/app/shared/services/ODataServices/odata-service";
import { RTLService } from "src/app/shared/services/RTL/rtl.service";

@Component({
  selector: "report-component",
  templateUrl: "./report.component.html",
})
export class ReportComponent implements OnDestroy {
  // Grid configuration
  keyGrid = "IdField"; // Primary key field
  odataRequesturl: string;
  reportForm: FormGroup;
  reportName = "ReportName"; // Used for saving user preferences
  private subsink = new SubSink();
  QueryFilter: string;

  // Column configuration for details
  columnIndexDetail: ColumnIndex[] = [
    { DataField: "Field1", Index: 0 },
    { DataField: "Field2", Index: 1, Visible: false },
    { DataField: "Field3", Index: 2 },
  ];

  // RTL and formatting
  public get isRtlEnabled(): boolean {
    return this.rtlService.IsRtlEnabled;
  }

  public get dateTimeFormat(): string {
    return this.rtlService.DateTimeFormatOnRTL;
  }

  constructor(
    private formBuilder: FormBuilder,
    private rtlService: RTLService,
    private odataService: ODataService,
    private toastrService: ToasterService,
    private translate: TranslateService,
    private errorHandler: ErrorHandlerService
  ) {
    this.initForm();

    // Initialize filters with default date range
    const initFilters = {
      DateField: {
        ge: new Date(this.reportForm.controls.FromDate.value),
        le: new Date(this.reportForm.controls.ToDate.value),
      },
    };

    this.QueryFilter = buildQuery({ filter: initFilters });
  }

  ngOnDestroy(): void {
    this.subsink.unsubscribe();
  }

  // Clear all filters
  ClearFilters = () => {
    this.reportForm.patchValue({
      Field1: [],
      Field2: [],
      Field3: null,
      FromDate: startOfDay(new Date()),
      ToDate: endOfDay(new Date()),
    });
  };

  // Initialize form with default values
  private initForm() {
    this.reportForm = this.formBuilder.group({
      Field1: [[]],
      Field2: [[]],
      Field3: [null],
      FromDate: new FormControl(startOfDay(new Date()), Validators.required),
      ToDate: new FormControl(endOfDay(new Date()), Validators.required),
    });

    // Subscribe to form changes and rebuild query
    this.subsink.add(
      this.reportForm.valueChanges
        .pipe(
          catchError((err) => {
            this.errorHandler.handleError(err);
            return EMPTY;
          })
        )
        .subscribe((x) => {
          this.HandleControlChanges();
        })
    );
  }

  // Build OData query filters based on form values
  private HandleControlChanges(): void {
    // Validate date range
    if (
      this.reportForm.controls.ToDate.value <
      this.reportForm.controls.FromDate.value
    ) {
      this.reportForm.controls.FromDate.setErrors(
        this.translate.instant("COMMON._dateOutOfRangeMessage_")
      );
      this.toastrService.showErrorToast(
        this.translate.instant("COMMON._dateOutOfRangeMessage_")
      );
      return;
    } else {
      this.reportForm.controls.FromDate.enable({ emitEvent: false });
    }

    // Build base filters (required)
    let filters = {
      DateField: {
        ge: new Date(this.reportForm.controls.FromDate.value),
        le: new Date(this.reportForm.controls.ToDate.value),
      },
    };

    // Add optional filters (single value)
    if (this.reportForm.controls.Field1?.value !== null) {
      filters = this.odataService.jsonConcat(
        { Field1Id: this.reportForm.controls.Field1?.value },
        filters
      );
    }

    // Add optional filters (array/in operator)
    if (
      this.odataService.toArray(this.reportForm.controls.Field2?.value ?? [])
        .length > 0
    ) {
      filters = this.odataService.jsonConcat(
        {
          Field2Id: {
            in: this.reportForm.controls.Field2?.value,
          },
        },
        filters
      );
    }

    // Complex filter with OR condition
    if (this.reportForm.controls.Field3?.value !== null) {
      filters = this.odataService.jsonConcat(
        {
          or: [
            {
              Details: {
                any: {
                  MaterialId: this.reportForm.controls.Field3?.value,
                },
              },
            },
            {
              OtherCollection: {
                any: {
                  MaterialId: this.reportForm.controls.Field3?.value,
                },
              },
            },
          ],
        },
        filters
      );
    }

    // Build final OData query string
    this.QueryFilter = buildQuery({ filter: filters });
  }
}
```

### Template Structure

```html
<div class="animated fadeIn delay-0.5s">
  <div class="mainContainer !flex-none">
    <h5 class="animated fadeInDown delay-1.5s">
      {{ 'REPORT._ReportTitle_' | translate }}
    </h5>
    <div class="make-border dx-card">
      <!-- Filter Form -->
      <form
        [formGroup]="reportForm"
        *ngIf="reportForm"
        class="form"
        autocomplete="off"
      >
        <div class="flex flex-row flex-wrap justify-center">
          <!-- Filter Fields -->
          <div
            class="!flex-none w-full sm:w-full md:w-4/12 lg:w-2/12 xl:w-2/12 p-2"
          >
            <log-lookup-component
              formControlName="Field1"
              [isRtlEnabled]="isRtlEnabled"
              [isRequired]="false"
            >
            </log-lookup-component>
          </div>

          <!-- Date Fields -->
          <div class="w-full sm:w-full md:w-4/12 lg:w-2/12 xl:w-2/12 p-2">
            <dx-date-box
              type="datetime"
              [displayFormat]="dateTimeFormat"
              formControlName="FromDate"
              [showClearButton]="true"
              [useMaskBehavior]="true"
              labelMode="floating"
              [label]="('COMMON._FromDate_' | translate) + ' * '"
            ></dx-date-box>
          </div>

          <div class="w-full sm:w-full md:w-6/12 lg:w-2/12 xl:w-2/12 p-2">
            <dx-date-box
              type="datetime"
              formControlName="ToDate"
              [displayFormat]="dateTimeFormat"
              [showClearButton]="true"
              [useMaskBehavior]="true"
              labelMode="floating"
              [label]="('COMMON._ToDate_' | translate) + ' * '"
            ></dx-date-box>
          </div>
        </div>
      </form>
    </div>

    <!-- Report Grid -->
    <report-grid
      [ApplicationId]="12"
      [keyGrid]="keyGrid"
      [DetailPropertyNames]="[
        'Details/SubDetails',
        'OtherCollection/SubCollection'
      ]"
      [ReportName]="reportName"
      [DisabledPreview]="!reportForm.valid"
      [QueryFilter]="QueryFilter"
      [ClearFilterClicked]="ClearFilters"
      [moneyColumns]="[{ DataField: 'TotalAmount' }]"
      [orderby]="['DateField', 'DateField']"
      endpoint="reportendpoint"
      [detailColumnIndex]="[columnIndexDetail]"
    >
    </report-grid>
  </div>
</div>
```

### Key Features

1. **Simple Filter Building**: Uses `buildQuery` from `odata-query`
2. **Form Value Changes**: Automatically rebuilds query on form changes
3. **Date Range Validation**: Built-in validation for date ranges
4. **Optional Nested Details**: Use `DetailPropertyNames` for navigation properties
5. **Column Configuration**: Use `ColumnIndex` array for column ordering/visibility
6. **Money Columns**: Specify money columns for proper formatting

---

## Pattern 2: Master-Detail Report with Row Expansion

**Use Case**: Reports with expandable master rows showing multiple detail tabs.

**Identifying Characteristics**:

- Uses `report-grid-multi-request` component
- Multiple OData endpoints (header + multiple details)
- Detail tabs shown when row is expanded
- Detail data loaded on row expansion
- Support for hyperlinks in detail grids

### Module Structure

```typescript
import { NgModule } from "@angular/core";
import { RouterModule } from "@angular/router";
import { AccflexSharedModule } from "src/app/shared/accflex-shared.module";
import { LogisticsSharedModule } from "../../shared/logistics-shared.module";
import { ReportShellComponent } from "./report-shell/report-shell.component";

@NgModule({
  declarations: [ReportShellComponent],
  imports: [
    AccflexSharedModule,
    LogisticsSharedModule,
    RouterModule.forChild([{ path: "", component: ReportShellComponent }]),
  ],
  providers: [], // Usually no additional providers needed
})
export class ReportModule {}
```

### Component Structure

```typescript
import { Component, OnDestroy, OnInit } from "@angular/core";
import { FormBuilder, FormGroup, Validators } from "@angular/forms";
import { TranslateService } from "@ngx-translate/core";
import { Router } from "@angular/router";
import { endOfDay, startOfDay } from "date-fns";
import { EMPTY, catchError, combineLatest, map } from "rxjs";
import { ColumnIndex } from "src/app/shared/components/report-grid/ColumnIndex";
import { MoneyColumn } from "src/app/shared/components/report-grid/MoneyColumn";
import { DetailHyperLinkColumn } from "src/app/shared/components/report-grid/PreviewArgs";
import { RTLService } from "src/app/shared/services/RTL/rtl.service";
import { AccflexToasterService } from "src/app/shared/services/accflex-toaster/accflex-toaster.service";
import { ODataService } from "src/app/shared/services/ODataServices/odata-service";

@Component({
  selector: "log-report-shell",
  templateUrl: "./report-shell.component.html",
})
export class ReportShellComponent implements OnInit, OnDestroy {
  // Grid configuration
  keyGrid = "MasterKeyField";
  Details: string[] = ["DetailTab1", "DetailTab2"]; // Detail tab names
  odataRequesturl: string;
  reportForm: FormGroup;
  reportName = "Report Name";

  // Query options - passed to endpoints
  HeaderQueryOptions: object = new Object();
  DetailTab1QueryOptions: object = new Object();
  DetailTab2QueryOptions: object = new Object();

  // Filters for OData
  headerFilters: object = new Object();
  detailFilters: object[] = []; // Array of filters for each detail tab

  // Hyperlink configuration for detail grid
  detailHyperLinkColumn: DetailHyperLinkColumn = {
    TabIndex: 0, // Which detail tab
    Column: "ColumnName", // Column name to make clickable
  };

  get IsRtlEnabled(): boolean {
    return this.rtlService.IsRtlEnabled;
  }

  get dateTimeFormat(): string {
    return this.rtlService.DateTimeFormatOnRTL;
  }

  // Data for filtering when no selection made
  allRecords: RecordDto[] = [];

  // Header columns configuration
  columnsIndex: ColumnIndex[] = [
    { DataField: "Field1", Index: 0, Visible: this.IsRtlEnabled },
    {
      DataField: "Field1SecondLanguage",
      Index: 0,
      Visible: !this.IsRtlEnabled,
    },
    { DataField: "Field2", Index: 1 },
    { DataField: "TotalAmount", Index: 2 },
  ];

  // Detail columns configuration (one array per detail tab)
  DetailColumnIndex: ColumnIndex[][] = [
    [
      // Tab 1 columns
      { DataField: "DetailField1", Index: 0 },
      { DataField: "DetailField2", Index: 1 },
    ],
    [
      // Tab 2 columns
      { DataField: "SummaryField1", Index: 0 },
      { DataField: "SummaryField2", Index: 1 },
    ],
  ];

  // Money columns for proper formatting
  MoneyColumns: MoneyColumn[] = [
    { DataField: "TotalDebit" },
    { DataField: "TotalCredit" },
    { DataField: "TotalBalance" },
  ];

  private initForm() {
    this.reportForm = this.formBuilder.group({
      ToDate: [endOfDay(new Date()), Validators.required],
      FromDate: [startOfDay(new Date()), Validators.required],
      ShowOption1: [false],
      ShowOption2: [true],
      FilterIds: [[]],
    });

    this.setDefaultValues();
  }

  private GenerateRequest() {
    this.headerFilters = new Object();

    // Date validation
    if (
      this.reportForm.controls.ToDate.value <
      this.reportForm.controls.FromDate.value
    ) {
      this.reportForm.controls.FromDate.setErrors(
        this.translate.instant("COMMON._dateOutOfRangeMessage_")
      );
      this.accflexToasterService.showErrorToast(
        this.translate.instant("COMMON._dateOutOfRangeMessage_")
      );
      return;
    } else {
      this.reportForm.controls.FromDate.enable({
        emitEvent: false,
      });
    }

    // Build header filters
    if (
      this.odataService.toArray(this.reportForm.controls.FilterIds?.value ?? [])
        .length > 0
    ) {
      this.headerFilters = this.odataService.jsonConcat(
        {
          FilterId: {
            in: this.reportForm.controls.FilterIds?.value,
          },
        },
        this.headerFilters
      );
    } else {
      // If no filter selected, use all records
      this.headerFilters = this.odataService.jsonConcat(
        { FilterId: { in: this.allRecords.flatMap((x) => x.Id) } },
        this.headerFilters
      );
    }

    this.initializeQueryOptions();
  }

  // Initialize query options passed to all endpoints
  initializeQueryOptions() {
    this.HeaderQueryOptions["FromDate"] = startOfDay(
      new Date(this.reportForm.value.FromDate)
    );
    this.HeaderQueryOptions["toDate"] = endOfDay(
      new Date(this.reportForm.value.ToDate)
    );
    this.HeaderQueryOptions["ShowOption1"] = this.reportForm.value.ShowOption1;
    this.HeaderQueryOptions["ShowOption2"] = this.reportForm.value.ShowOption2;

    // Share same options across all endpoints
    this.DetailTab1QueryOptions = this.DetailTab2QueryOptions =
      this.HeaderQueryOptions;
  }

  ngOnInit(): void {}

  ClearFilters = () => {
    this.reportForm.patchValue({
      ToDate: endOfDay(new Date()),
      FromDate: startOfDay(new Date()),
      ShowOption1: false,
      ShowOption2: true,
      FilterIds: [],
    });
  };

  ngOnDestroy(): void {}

  // Handle hyperlink clicks in detail grid
  HyperLinkClicked(row: any) {
    if (row?.EntityTypeId && row?.EntityId) {
      if (row.EntityTypeId === 1) {
        this.navigateToDocument(`document1?id=${row?.EntityId}`);
      }
      if (row.EntityTypeId === 2) {
        this.navigateToDocument(`document2?id=${row?.EntityId}`);
      }
    }
  }

  navigateToDocument = (link: string) => {
    const currentPage = this.router.url.split("/").pop();
    this.router.navigate([]).then((result) => {
      window.open(window.location.href.replace(currentPage, link), "_blank");
    });
  };

  // Load all records for default filtering
  private getAllRecordsForDefaultFiltering() {
    combineLatest([this.staticResourcesService.records$])
      .pipe(map(([records]) => records.filter((r) => r.IsActive)))
      .subscribe({
        next: (res) => {
          this.allRecords = res;
          this.GenerateRequest();
        },
      });
  }

  // Called when a master row is expanded
  onDetailExpand = (e) => {
    const row = e?.parentRowData;
    this.detailFilters.length = 0;

    // Build filters for Tab 1
    let tab1Filter = new Object();
    if (row?.AccountId) {
      tab1Filter = this.odataService.jsonConcat(
        { AccountId: { in: [row?.AccountId] } },
        tab1Filter
      );
    }
    if (row?.CurrencyId) {
      tab1Filter = this.odataService.jsonConcat(
        { CurrencyId: row?.CurrencyId },
        tab1Filter
      );
    }

    // Build filters for Tab 2
    let tab2Filter = new Object();
    if (row?.SummaryId) {
      tab2Filter = this.odataService.jsonConcat(
        { SummaryId: row?.SummaryId },
        tab2Filter
      );
    }

    // Add filters in order matching Details array
    this.detailFilters.push(tab1Filter);
    this.detailFilters.push(tab2Filter);
  };

  setDefaultValues(): void {
    this.staticResourcesService.user.subscribe((user) => {
      if (user?.DefaultValue) {
        this.reportForm.controls.FilterIds?.setValue([user?.DefaultValue]);
      }
    });
  }

  constructor(
    private formBuilder: FormBuilder,
    public rtlService: RTLService,
    public translate: TranslateService,
    public accflexToasterService: AccflexToasterService,
    public odataService: ODataService,
    private staticResourcesService: StaticResourcesService,
    private router: Router
  ) {
    this.initForm();
    this.getAllRecordsForDefaultFiltering();
    this.GenerateRequest();

    this.reportForm.valueChanges
      .pipe(
        catchError((err) => {
          return EMPTY;
        })
      )
      .subscribe((x) => {
        this.GenerateRequest();
      });
  }
}
```

### Template Structure

```html
<accflex-floating-icons [title]="'REPORT._Title_' | translate">
</accflex-floating-icons>
<div class="animated fadeIn delay-0.5s">
  <div class="mainContainer">
    <div class="make-border">
      <!-- Filter Form -->
      <form
        [formGroup]="reportForm"
        *ngIf="reportForm"
        class="form"
        autocomplete="off"
      >
        <div class="flex flex-row flex-wrap justify-center">
          <!-- Filter Fields -->
          <div class="w-full sm:w-full md:w-1/4 lg:w-3/12 xl:w-1/4 p-2">
            <log-lookup-component formControlName="FilterIds">
            </log-lookup-component>
          </div>

          <!-- Date Fields -->
          <div class="w-full sm:w-full md:w-1/4 lg:w-3/12 xl:w-1/4 p-2">
            <dx-date-box
              type="datetime"
              [displayFormat]="dateTimeFormat"
              formControlName="FromDate"
              [showClearButton]="true"
              [useMaskBehavior]="true"
              labelMode="floating"
              [label]="('COMMON._FromDate_' | translate) + ' * '"
            ></dx-date-box>
          </div>

          <div class="w-full sm:w-full md:w-1/4 lg:w-3/12 xl:w-1/4 p-2">
            <dx-date-box
              type="datetime"
              formControlName="ToDate"
              [displayFormat]="dateTimeFormat"
              [showClearButton]="true"
              [useMaskBehavior]="true"
              labelMode="floating"
              [label]="('COMMON._ToDate_' | translate) + ' * '"
            ></dx-date-box>
          </div>

          <!-- Checkboxes for options -->
          <div class="w-full sm:w-full md:w-1/4 lg:w-3/12 xl:w-1/4 p-2">
            <dx-check-box
              formControlName="ShowOption1"
              [text]="'REPORT._ShowOption1_' | translate"
              [elementAttr]="{ class: 'mt-1' }"
              [rtlEnabled]="IsRtlEnabled"
            ></dx-check-box>
          </div>
        </div>
      </form>

      <!-- Report Grid -->
      <div>
        <report-grid-multi-request
          [ApplicationId]="12"
          headerendpoint="HeaderEndpoint"
          [keyGrid]="keyGrid"
          [details]="Details"
          [detailendpoints]="['DetailEndpoint1', 'DetailEndpoint2']"
          [ReportName]="reportName"
          [DisabledPreview]="!reportForm.valid"
          [headerqueryoptions]="HeaderQueryOptions"
          [detailorderby]="[['DateField']]"
          [detailqueryoptions]="[DetailTab1QueryOptions, DetailTab2QueryOptions]"
          [ClearFilterClicked]="ClearFilters"
          [columnIndex]="columnsIndex"
          [detailColumnIndex]="DetailColumnIndex"
          [headerfilter]="headerFilters"
          (HyperLinkColumnClicked)="HyperLinkClicked($event)"
          [detailHyperLinkColumn]="detailHyperLinkColumn"
          [moneyColumns]="MoneyColumns"
          (onDetailExpand)="onDetailExpand($event)"
          [detailfilter]="detailFilters"
        >
        </report-grid-multi-request>
      </div>
    </div>
  </div>
</div>
```

### Key Features

1. **Multiple Endpoints**: Separate endpoints for header and each detail tab
2. **Query Options**: Passed as parameters (not in URL query string)
3. **Detail Filters**: Built dynamically when row is expanded
4. **Detail Tabs**: Multiple tabs shown in expanded row
5. **Hyperlinks**: Clickable columns in detail grids
6. **Default Filtering**: Load all records for filtering when no selection

---

## Pattern 3: Advanced Report with Popup Details and Column Service

**Use Case**: Complex reports with multi-level drill-down popups and dynamic column configuration.

**Identifying Characteristics**:

- Uses `accflex-report-grid-multi-request-with-popups` component
- Dedicated column service for managing columns
- Detail data shown in modal popups (not inline expansion)
- Multi-level drill-down support
- Complex column configurations

### Module Structure

```typescript
import { NgModule } from "@angular/core";
import { ReportComponent } from "./report/report.component";
import { AccflexSharedModule } from "src/app/shared/accflex-shared.module";
import { LogisticsSharedModule } from "../../shared/logistics-shared.module";
import { RouterModule } from "@angular/router";
import { ReportColumnsService } from "./report/report-columns-service";

@NgModule({
  declarations: [ReportComponent],
  imports: [
    AccflexSharedModule,
    LogisticsSharedModule,
    RouterModule.forChild([{ path: "", component: ReportComponent }]),
  ],
  providers: [ReportColumnsService], // Column service
})
export class ReportModule {}
```

### Column Service

```typescript
import { Injectable } from "@angular/core";
import { TranslateService } from "@ngx-translate/core";
import { Column } from "devextreme/ui/data_grid";
import { RTLService } from "src/app/shared/services/RTL/rtl.service";

@Injectable()
export class ReportColumnsService {
  constructor(
    private rtlService: RTLService,
    private translate: TranslateService
  ) {}

  get IsRtlEnabled(): boolean {
    return this.rtlService.IsRtlEnabled;
  }

  // Header columns
  get headerColumns(): Column[] {
    return [
      {
        dataField: "Field1",
        caption: this.translate.instant("REPORT._Field1_"),
        alignment: "center",
        allowSorting: false,
      },
      {
        dataField: this.IsRtlEnabled ? "Field2Ar" : "Field2En",
        caption: this.translate.instant("REPORT._Field2_"),
        alignment: "center",
        allowSorting: false,
      },
      {
        dataField: "Amount",
        caption: this.translate.instant("REPORT._Amount_"),
        alignment: "center",
        dataType: "number",
        format: { type: "fixedPoint", precision: 2 },
      },
      {
        // Drill-down column
        name: "categories",
        caption: this.translate.instant("REPORT._Categories_"),
        cellTemplate: "detailTemplate",
        alignment: "center",
        allowSorting: false,
      },
    ];
  }

  // Detail level 1 columns
  get categoryColumns(): Column[] {
    return [
      {
        dataField: "CategoryName",
        caption: this.translate.instant("REPORT._CategoryName_"),
        alignment: "center",
      },
      {
        dataField: "Quantity",
        caption: this.translate.instant("REPORT._Quantity_"),
        alignment: "center",
        dataType: "number",
      },
      {
        // Drill-down to next level
        name: "materials",
        caption: this.translate.instant("REPORT._Materials_"),
        cellTemplate: "detailTemplate",
        alignment: "center",
      },
    ];
  }

  // Detail level 2 columns
  get materialsColumns(): Column[] {
    return [
      {
        dataField: "MaterialCode",
        caption: this.translate.instant("REPORT._MaterialCode_"),
        alignment: "center",
      },
      {
        dataField: "MaterialName",
        caption: this.translate.instant("REPORT._MaterialName_"),
        alignment: "center",
      },
      {
        // Drill-down to characteristics
        name: "characteristics",
        caption: this.translate.instant("REPORT._Characteristics_"),
        cellTemplate: "detailTemplate",
        alignment: "center",
      },
    ];
  }

  // Detail level 3 columns
  get characteristicsColumns(): Column[] {
    return [
      {
        dataField: "CharacteristicName",
        caption: this.translate.instant("REPORT._CharacteristicName_"),
        alignment: "center",
      },
      {
        dataField: "Value",
        caption: this.translate.instant("REPORT._Value_"),
        alignment: "center",
      },
    ];
  }
}
```

### Component Structure

```typescript
import { Component, OnInit } from "@angular/core";
import { FormControl, FormGroup } from "@angular/forms";
import { TranslateService } from "@ngx-translate/core";
import { endOfDay, startOfDay } from "date-fns";
import { Column } from "devextreme/ui/data_grid";
import { ToasterService } from "projects/general-accounting/src/app/shared/services/toaster/toaster.service";
import { EMPTY, catchError } from "rxjs";
import { DetailClickedEventArgs } from "src/app/shared/components/report-grid-multi-request-with-popups/detail-clicked-event";
import { PreviewArgs } from "src/app/shared/components/report-grid/PreviewArgs";
import { ODataService } from "src/app/shared/services/ODataServices/odata-service";
import { RTLService } from "src/app/shared/services/RTL/rtl.service";
import { ReportColumnsService } from "./report-columns-service";
import { ReportVM } from "./report-view-model";

@Component({
  selector: "log-report",
  templateUrl: "./report.component.html",
})
export class ReportComponent implements OnInit {
  initalFormValues: ReportVM;

  get IsRtlEnabled(): boolean {
    return this.rtlService.IsRtlEnabled;
  }

  get dateTimeFormat(): string {
    return this.rtlService.DateTimeFormatOnRTL;
  }

  reportForm: FormGroup<ReportVM>;

  // Query options for different levels
  HeaderQueryOptions: object = new Object();
  headerFilters = new Object();

  detailQueryOption: object = new Object();
  detailFilters = new Object();

  characteristicsQueryOption: object = new Object();
  characteristicsFilters = new Object();

  ngOnInit(): void {}

  // Handle detail cell clicks for drill-down
  onDetailClicked = (e: DetailClickedEventArgs) => {
    // Level 1: Categories
    if (e?.event?.column.name === "categories") {
      const detialKeyGrid: string = "Id";
      const currenctDetailEndPoint = "CategoryEndpoint";
      const detailColumns = this.columnsService.categoryColumns;
      const detailTitle = this.translate.instant("REPORT._Categories_");

      this.detailQueryOption["ParentId"] = e.row?.data?.Id;
      this.detailQueryOption["fromDate"] =
        this.reportForm?.controls.FromDate?.value;
      this.detailQueryOption["toDate"] =
        this.reportForm?.controls.ToDate?.value;

      e.DetailEndPoint = currenctDetailEndPoint;
      e.DetailQueryOptions = this.detailQueryOption;
      e.OrderByDetailInput = [];
      e.autoExpandGroups = false;
      e.detailColumns = detailColumns;
      e.detailFilters = this.detailFilters;
      e.detailTitle = detailTitle;
      e.detailHyperLinkColumn = null;
      e.detialKeyGrid = detialKeyGrid;
    }
    // Level 2: Materials
    else if (e?.event?.column.name === "materials") {
      const detialKeyGrid: string = "MaterialId";
      const currenctDetailEndPoint = "MaterialEndpoint";
      const detailColumns = this.columnsService.materialsColumns;
      const detailTitle = this.translate.instant("REPORT._Materials_");

      this.detailQueryOption["CategoryId"] = e.row?.data?.CategoryId;
      this.detailQueryOption["fromDate"] =
        this.reportForm?.controls.FromDate?.value;
      this.detailQueryOption["toDate"] =
        this.reportForm?.controls.ToDate?.value;

      e.DetailEndPoint = currenctDetailEndPoint;
      e.DetailQueryOptions = this.detailQueryOption;
      e.detailColumns = detailColumns;
      e.detailFilters = this.detailFilters;
      e.detailTitle = detailTitle;
      e.detialKeyGrid = detialKeyGrid;
    }
    // Level 3: Characteristics
    else if (e?.event?.column.name === "characteristics") {
      const detialKeyGrid: string = "CharacteristicId";
      const currenctDetailEndPoint = "CharacteristicEndpoint";
      const detailColumns = this.columnsService.characteristicsColumns;
      const detailTitle = this.translate.instant("REPORT._Characteristics_");

      this.characteristicsQueryOption["MaterialId"] = e.row?.data?.MaterialId;
      this.characteristicsQueryOption["fromDate"] =
        this.reportForm?.controls.FromDate?.value;
      this.characteristicsQueryOption["toDate"] =
        this.reportForm?.controls.ToDate?.value;

      e.DetailEndPoint = currenctDetailEndPoint;
      e.DetailQueryOptions = this.characteristicsQueryOption;
      e.detailColumns = detailColumns;
      e.detailFilters = this.characteristicsFilters;
      e.detailTitle = detailTitle;
      e.detialKeyGrid = detialKeyGrid;
    }

    // Handle export - merge all query options
    if (e.exporting) {
      const mergedQueryOption = {
        ...this.detailQueryOption,
        ...this.characteristicsQueryOption,
      };
      e.DetailQueryOptions = mergedQueryOption;

      const mergedFilters = {
        ...this.detailFilters,
        ...this.characteristicsFilters,
      };
      e.detailFilters = mergedFilters;
    }
  };

  ClearFilters = () => {
    this.reportForm.patchValue(
      {
        Filters: structuredClone([]),
        FromDate: startOfDay(new Date()),
        ToDate: endOfDay(new Date()),
      },
      { emitEvent: false }
    );
  };

  private initalizeFormGroup() {
    this.reportForm = new FormGroup<ReportVM>({
      Filters: new FormControl([], { nonNullable: false }),
      FromDate: new FormControl(startOfDay(new Date()), {
        nonNullable: false,
      }),
      ToDate: new FormControl(endOfDay(new Date()), {
        nonNullable: false,
      }),
    });

    this.initalFormValues = this.reportForm.value as ReportVM;
    this.setDefaultValues();
  }

  OnPreviewClick(e: PreviewArgs) {
    // Validate before showing report
    if (
      !this.reportForm.controls?.ToDate?.value ||
      !this.reportForm.controls?.FromDate?.value
    ) {
      this.toastr.showErrorToast(
        this.translate.instant("COMMON._dateOutOfRangeMessage_")
      );
      e.Canceled = true;
      return;
    }

    if (
      this.reportForm.controls.ToDate.value <
      this.reportForm.controls.FromDate.value
    ) {
      this.reportForm.controls.FromDate.setErrors(
        this.translate.instant("COMMON._dateOutOfRangeMessage_")
      );
      this.toastr.showErrorToast(
        this.translate.instant("COMMON._dateOutOfRangeMessage_")
      );
      e.Canceled = true;
      return;
    }
  }

  private generateRequest() {
    this.headerFilters = new Object();
    this.UpdateQueryOptions();
  }

  UpdateQueryOptions(): void {
    this.HeaderQueryOptions["filterIds"] =
      this.reportForm?.controls.Filters?.value?.join();
    this.HeaderQueryOptions["fromDate"] =
      this.reportForm?.controls.FromDate?.value;
    this.HeaderQueryOptions["toDate"] = this.reportForm?.controls.ToDate?.value;
  }

  get headerColumns(): Column[] {
    return this.columnsService.headerColumns;
  }

  setDefaultValues(): void {
    this.staticResourcesService.user.subscribe((user) => {
      if (user?.DefaultValue) {
        this.reportForm.controls.Filters?.setValue([user?.DefaultValue]);
      }
    });
  }

  constructor(
    private rtlService: RTLService,
    private columnsService: ReportColumnsService,
    public staticResourcesService: StaticResourcesService,
    private translate: TranslateService,
    private toastr: ToasterService,
    private odataService: ODataService
  ) {
    this.initalizeFormGroup();
    this.generateRequest();

    this.reportForm.valueChanges
      .pipe(
        catchError((err) => {
          return EMPTY;
        })
      )
      .subscribe((x) => {
        this.generateRequest();
      });
  }
}
```

### Template Structure

```html
<accflex-floating-icons [title]="'REPORT._Title_' | translate">
</accflex-floating-icons>
<div class="animated fadeIn delay-0.5s">
  <div class="mainContainer">
    <div class="make-border">
      <!-- Filter Form -->
      <form
        [formGroup]="reportForm"
        *ngIf="reportForm"
        class="form"
        autocomplete="off"
      >
        <div class="flex flex-row flex-wrap justify-center">
          <!-- Filter Fields -->
          <div class="w-full sm:w-full md:w-4/12 lg:w-4/12 xl:w-4/12 p-2">
            <log-checklist-component
              formControlName="Filters"
              [isRequired]="false"
            >
            </log-checklist-component>
          </div>

          <!-- Date Fields -->
          <div class="w-full sm:w-full md:w-2/12 lg:w-2/12 xl:w-2/12 p-2">
            <dx-date-box
              type="datetime"
              [displayFormat]="dateTimeFormat"
              formControlName="FromDate"
              [showClearButton]="false"
              [useMaskBehavior]="true"
              labelMode="floating"
              [label]="('COMMON._FromDate_' | translate) + ' * '"
            ></dx-date-box>
          </div>

          <div class="w-full sm:w-full md:w-2/12 lg:w-2/12 xl:w-2/12 p-2">
            <dx-date-box
              type="datetime"
              formControlName="ToDate"
              [displayFormat]="dateTimeFormat"
              [showClearButton]="false"
              [useMaskBehavior]="true"
              labelMode="floating"
              [label]="('COMMON._ToDate_' | translate) + ' * '"
            ></dx-date-box>
          </div>
        </div>
      </form>

      <!-- Report Grid with Popups -->
      <div>
        <accflex-report-grid-multi-request-with-popups
          [ApplicationId]="12"
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
      </div>
    </div>
  </div>
</div>
```

### Key Features

1. **Column Service**: Centralized column configuration management
2. **Popup Details**: Details shown in modal popups instead of inline
3. **Multi-Level Drill-Down**: Support for 3+ levels of detail
4. **Dynamic Column Configuration**: Columns configured based on column name
5. **Export Support**: Merge all query options for export
6. **Strongly Typed Forms**: Uses FormGroup with type parameter

---

## Pattern Comparison Table

| Feature                  | Pattern 1: Simple OData       | Pattern 2: Master-Detail         | Pattern 3: Popup Details                |
| ------------------------ | ----------------------------- | -------------------------------- | --------------------------------------- |
| **Component**            | `report-grid`                 | `report-grid-multi-request`      | `report-grid-multi-request-with-popups` |
| **Filter Building**      | `buildQuery` from odata-query | ODataService methods             | ODataService methods                    |
| **Detail Display**       | Nested inline                 | Expandable rows with tabs        | Modal popups                            |
| **Number of Endpoints**  | 1                             | 1 header + N details             | 1 header + dynamic details              |
| **Column Configuration** | ColumnIndex arrays            | ColumnIndex arrays               | Column service                          |
| **Drill-Down Levels**    | 1-2                           | 2 (master + details)             | 3+                                      |
| **Query Options**        | In URL (query string)         | Passed as parameters             | Passed as parameters                    |
| **Use Case**             | Simple list reports           | Account statements, transactions | Complex analytical reports              |
| **Complexity**           | Low                           | Medium                           | High                                    |

---

## Common Patterns Across All Types

### 1. Date Range Validation

```typescript
if (
  this.reportForm.controls.ToDate.value <
  this.reportForm.controls.FromDate.value
) {
  this.reportForm.controls.FromDate.setErrors(
    this.translate.instant("COMMON._dateOutOfRangeMessage_")
  );
  this.toastrService.showErrorToast(
    this.translate.instant("COMMON._dateOutOfRangeMessage_")
  );
  return;
}
```

### 2. Form Value Changes Subscription

```typescript
this.reportForm.valueChanges
  .pipe(
    catchError((err) => {
      return EMPTY;
    })
  )
  .subscribe((x) => {
    this.regenerateFilters();
  });
```

### 3. Clear Filters

```typescript
ClearFilters = () => {
  this.reportForm.patchValue({
    Field1: [],
    Field2: null,
    FromDate: startOfDay(new Date()),
    ToDate: endOfDay(new Date()),
  });
};
```

### 4. RTL Support

```typescript
public get isRtlEnabled(): boolean {
  return this.rtlService.IsRtlEnabled;
}

public get dateTimeFormat(): string {
  return this.rtlService.DateTimeFormatOnRTL;
}
```

### 5. Column Index Configuration

```typescript
columnIndex: ColumnIndex[] = [
  { DataField: "Field1", Index: 0 },
  { DataField: "Field2", Index: 1, Visible: false },
  { DataField: this.IsRtlEnabled ? "FieldAr" : "FieldEn", Index: 2 },
];
```

---

## Required Imports

### Pattern 1 (Simple OData)

```typescript
import buildQuery from "odata-query";
import { ColumnIndex } from "src/app/shared/components/report-grid/ColumnIndex";
```

### Pattern 2 (Master-Detail)

```typescript
import { ColumnIndex } from "src/app/shared/components/report-grid/ColumnIndex";
import { MoneyColumn } from "src/app/shared/components/report-grid/MoneyColumn";
import { DetailHyperLinkColumn } from "src/app/shared/components/report-grid/PreviewArgs";
import { ODataService } from "src/app/shared/services/ODataServices/odata-service";
```

### Pattern 3 (Popup Details)

```typescript
import { Column } from "devextreme/ui/data_grid";
import { DetailClickedEventArgs } from "src/app/shared/components/report-grid-multi-request-with-popups/detail-clicked-event";
import { PreviewArgs } from "src/app/shared/components/report-grid/PreviewArgs";
import { ODataService } from "src/app/shared/services/ODataServices/odata-service";
```

### Common Imports

```typescript
import { Component, OnDestroy, OnInit } from "@angular/core";
import {
  FormBuilder,
  FormControl,
  FormGroup,
  Validators,
} from "@angular/forms";
import { TranslateService } from "@ngx-translate/core";
import { endOfDay, startOfDay } from "date-fns";
import { EMPTY, catchError } from "rxjs";
import { RTLService } from "src/app/shared/services/RTL/rtl.service";
import { AccflexToasterService } from "src/app/shared/services/accflex-toaster/accflex-toaster.service";
import { SubSink } from "subsink";
```

---

## Best Practices

1. **Always validate date ranges** before generating requests
2. **Use SubSink** for subscription management
3. **Subscribe to form valueChanges** to auto-regenerate filters
4. **Provide ClearFilters function** for user convenience
5. **Use date-fns** for date operations (startOfDay, endOfDay)
6. **Support RTL** by checking rtlService
7. **Handle errors** with catchError operator
8. **Set default values** from user preferences
9. **Use column service** for complex column configurations (Pattern 3)
10. **Validate form** before showing report (DisabledPreview)
