---
description: "Use when implementing or editing Organizational Data feature modules where search is a DevExtreme grid on the side of the edit form: search-shell with reactive form, CustomStore, server-side pagination, row double-click and select button. Applies to Company Code, Plant, Sales Org and similar structure-of-enterprise features."
alwaysApply: false
globs:
  - "**/*-search-shell.component.ts"
  - "**/*-search-shell.component.html"
---

# Search Component with Grid Pattern

## Overview

This pattern is for search components that use a DevExtreme grid to display search results with server-side pagination. The component allows users to search for records using various criteria and select items from the grid results.

## Component Structure

### TypeScript Component (`feature-search-shell.component.ts`)

#### Key Characteristics:

- **Search Form**: Reactive form with search criteria fields
- **Grid State**: Manages grid enabled/disabled state
- **CustomStore**: Uses DevExtreme CustomStore for remote operations and server-side pagination
- **Item Selection**: Supports both row double-click and select button
- **Date Range**: Includes date range validation
- **Reset Functionality**: Allows clearing search form and grid

#### Component Template:

```typescript
import { Component, OnInit } from "@angular/core";
import {
  AbstractControlOptions,
  FormBuilder,
  FormGroup,
  Validators,
} from "@angular/forms";
import { TranslateService } from "@ngx-translate/core";
import { endOfDay, startOfDay } from "date-fns";
import { LoadOptions } from "devextreme/data";
import CustomStore from "devextreme/data/custom_store";
import { firstValueFrom } from "rxjs";
import { FormState } from "src/app/models";
import { AccflexValidators } from "src/app/shared/services/global-methods";
import { LoadingHandlerService } from "src/app/shared/services/loading-handler/loading-handler.service";
import { AccflexToasterService } from "src/app/shared/services/accflex-toaster/accflex-toaster.service";

@Component({
  selector: "log-feature-search-shell",
  templateUrl: "./feature-search-shell.component.html",
})
export class FeatureSearchShellComponent implements OnInit {
  // Form and state properties
  searchForm: FormGroup;
  isGridDisabled = true;
  disabledSearchButton = false;
  gridDataSource: any;

  get isRtlEnabled(): boolean {
    return this.featureFormService.isRtlEnabled;
  }

  // Initialize search form with default values
  private renderSearchForm(): void {
    this.searchForm = this.formBuilder.group(
      {
        searchField1: [null],
        searchField2: [null],
        inventoryManagementId: [null],
        startDate: [startOfDay(new Date()), Validators.required],
        endDate: [endOfDay(new Date()), Validators.required],
        includeTime: false,
      },
      {
        validator: AccflexValidators.rangeValidationCustomNames(
          "startDate",
          "endDate"
        ),
      } as AbstractControlOptions
    );

    // Set default values from static resources if needed
    this.staticResourceService
      .getDefaultInventoryManagementId(true)
      .subscribe((id) => {
        this.searchForm.controls.inventoryManagementId?.setValue(id);
      });
  }

  // CustomStore load function for server-side pagination
  loadFunction = (loadOptions: LoadOptions): Promise<any> => {
    const query = this.searchForm.value;
    query.Skip = loadOptions.skip;
    query.Take = loadOptions.take;
    return this.getSearchResult(query);
  };

  // Execute search API call
  getSearchResult(query): Promise<any> {
    this.disabledSearchButton = true;
    const NoResultMsg = this.translate.instant("COMMON._NoResult_");

    return firstValueFrom(
      this.featureClient.search(
        query.searchField1,
        query.searchField2,
        query.inventoryManagementId,
        query.startDate,
        query.endDate,
        query.Skip,
        query.Take
      )
    )
      .then((data: PagedListOfFeatureDto) => {
        this.disabledSearchButton = false;
        if (!data?.TotalCount) {
          this.toaster.showSuccessToast(NoResultMsg);
          this.disableGrid();
          return { data: [], totalCount: 0 };
        } else {
          this.enableGrid();
          return { data: data.Data, totalCount: data.TotalCount };
        }
      })
      .catch((error) => {
        this.toaster.showSuccessToast(NoResultMsg);
        this.disableGrid();
        return { data: [], totalCount: 0 };
      })
      .finally(() => {
        this.hideLoading();
        this.disabledSearchButton = false;
      });
  }

  // Submit search form
  OnSearchSubmit(): void {
    if (!this.searchForm) return;
    if (!this.searchForm.valid) {
      this.toaster.showErrorToast(
        this.translate.instant("COMMON._DataRequired_")
      );
      return;
    }
    this.gridDataSource = new CustomStore({
      key: "Id",
      load: this.loadFunction,
    });
  }

  // Handle select button click
  selectItem = (e): void => {
    if (!e.row?.data?.Id) {
      this.toaster.showSuccessToast(
        this.translate.instant("MODULE._NotAvailable_")
      );
      return;
    }
    this.featureFormService.loadById(e.row.data.Id);
  };

  // Handle row double-click
  onRowDblClick = (e): void => {
    this.featureFormService.loadById(e.data.Id);
  };

  // Grid state management
  hideLoading = (): void => this.loading.hideLoading();
  disableGrid = (): boolean => (this.isGridDisabled = true);
  enableGrid = (): boolean => (this.isGridDisabled = false);

  // Reset functionality
  handleReset = (): void => {
    this.disabledSearchButton = false;
    this.resetSearchForm();
    this.resetGrid();
    this.disableGrid();
  };

  resetSearchForm = (): void => {
    if (this.searchForm?.controls)
      this.searchForm.patchValue({
        searchField1: null,
        searchField2: null,
        inventoryManagementId: null,
        startDate: startOfDay(new Date()),
        endDate: endOfDay(new Date()),
        includeTime: false,
      });
  };

  resetGrid = (): any =>
    (this.gridDataSource = new CustomStore({
      key: "Id",
      load: (options): Promise<any> => {
        return Promise.resolve({ data: [], totalCount: 0 });
      },
    }));

  ngOnInit(): void {
    this.renderSearchForm();
  }

  ngOnDestroy(): void {}

  constructor(
    public featureFormService: FeatureFormService,
    private formBuilder: FormBuilder,
    private toaster: AccflexToasterService,
    private loading: LoadingHandlerService,
    private translate: TranslateService,
    private featureClient: FeatureClient,
    public staticResourceService: LogisticsStaticResourcesService
  ) {
    // Subscribe to form state changes to refresh grid after save
    this.featureFormService.sub.sink =
      this.featureFormService.formState$.subscribe({
        next: (res) => {
          if (res == FormState.NormalFirstLoaded) {
            this.resetGrid();
            this.OnSearchSubmit();
          }
        },
      });
  }
}
```

### HTML Template (`feature-search-shell.component.html`)

#### Template Structure:

```html
<h6 class="animated fadeInDown delay-1.8s">
  {{ 'COMMON._Search_' | translate }}
</h6>

<form class="flex flex-col flex-1" [formGroup]="searchForm" *ngIf="searchForm">
  <!-- Search Fields Section -->
  <div class="flex !flex-none flex-wrap flex-row">
    <!-- Text/Number Search Field -->
    <div class="w-full sm:w-full md:w-1/4 lg:w-1/4 xl:w-1/4 p-1">
      <accflex-number-box
        [label]="'MODULE._FieldLabel_'"
        [minimumValue]="0"
        [control]="searchForm.controls.searchField1"
        [preventFractions]="true"
      >
      </accflex-number-box>
    </div>

    <!-- Lookup Field -->
    <div class="!flex-none w-full sm:w-full md:w-1/4 lg:w-1/4 xl:w-1/4 p-1">
      <log-feature-lookup formControlName="searchField2"> </log-feature-lookup>
    </div>

    <!-- Inventory Management Lookup -->
    <div class="!flex-none w-full sm:w-full md:w-1/4 lg:w-1/4 xl:w-1/4 p-1">
      <log-inventory-management-lookup-new
        formControlName="inventoryManagementId"
        [isActive]="true"
        [label]="'MODULE._InventoryManagement_'"
      >
      </log-inventory-management-lookup-new>
    </div>

    <!-- Start Date Field -->
    <div class="w-full sm:w-full md:w-1/4 lg:w-1/4 xl:w-1/4 p-1">
      <em *ngIf="searchForm.getError('InValidPeriod')"
        >{{ 'COMMON._dateOutOfRangeMessage_' | translate }}</em
      >
      <dx-date-box
        type="datetime"
        [showClearButton]="false"
        [useMaskBehavior]="true"
        formControlName="startDate"
        [max]="searchForm.controls.startDate.value"
        [dateOutOfRangeMessage]="'COMMON._dateOutOfRangeMessage_' | translate"
        width="100%"
        labelMode="floating"
        [label]="('COMMON._FromDate_' | translate) + ' * '"
      >
        <dx-validator>
          <dxi-validation-rule
            type="required"
            [message]="'COMMON._Required_' | translate"
          >
          </dxi-validation-rule>
        </dx-validator>
      </dx-date-box>
    </div>

    <!-- End Date Field -->
    <div class="w-full sm:w-full md:w-1/4 lg:w-1/4 xl:w-1/4 p-1">
      <em *ngIf="searchForm.getError('InValidPeriod')"
        >{{ 'COMMON._dateOutOfRangeMessage_' | translate }}</em
      >
      <dx-date-box
        type="datetime"
        [showClearButton]="false"
        [useMaskBehavior]="true"
        [min]="searchForm.controls.endDate.value"
        [dateOutOfRangeMessage]="'COMMON._dateOutOfRangeMessage_' | translate"
        formControlName="endDate"
        width="100%"
        labelMode="floating"
        [label]="('COMMON._ToDate_' | translate) + ' * '"
      >
        <dx-validator>
          <dxi-validation-rule
            type="required"
            [message]="'COMMON._Required_' | translate"
          >
          </dxi-validation-rule>
        </dx-validator>
      </dx-date-box>
    </div>

    <!-- Search Buttons -->
    <div class="w-full text-center my-4">
      <search-button
        [isRtlEnabled]="isRtlEnabled"
        (OnSubmit)="OnSearchSubmit()"
        [useSubmitBehavior]="true"
        [isDisabled]="!searchForm.valid || disabledSearchButton"
      >
      </search-button>
      <search-cancel-button
        [formState]="2"
        [isRtlEnabled]="isRtlEnabled"
        (OnCancel)="handleReset()"
      >
      </search-cancel-button>
    </div>
  </div>

  <!-- DevExtreme Grid Section -->
  <div
    class="flex flex-1 flex-col w-full sm:w-full md:w-12/12 lg:w-12/12 xl:w-12/12 p-1"
  >
    <dx-data-grid
      [disabled]="isGridDisabled"
      [dataSource]="gridDataSource"
      [remoteOperations]="true"
      [showBorders]="true"
      focusedRowEnabled="true"
      [rtlEnabled]="isRtlEnabled"
      [columnAutoWidth]="true"
      (onRowDblClick)="onRowDblClick($event)"
      [noDataText]="'COMMON._NoData_' | translate"
      width="100%"
    >
      <!-- Search Panel -->
      <dxo-search-panel
        [visible]="true"
        [placeholder]="'COMMON._Search_' | translate"
      >
      </dxo-search-panel>

      <!-- Pagination -->
      <dxo-paging [pageSize]="10"></dxo-paging>
      <dxo-pager
        [showPageSizeSelector]="true"
        [allowedPageSizes]="[5, 10, 20, 50]"
        [showInfo]="false"
      >
      </dxo-pager>

      <!-- Number Column -->
      <dxi-column
        alignment="center"
        dataType="number"
        [allowSorting]="false"
        dataField="FieldNumber"
        [caption]="'MODULE._NumberCaption_' | translate"
      >
      </dxi-column>

      <!-- String Column -->
      <dxi-column
        alignment="center"
        dataType="string"
        [allowSorting]="false"
        dataField="FieldString"
        [caption]="'MODULE._StringCaption_' | translate"
      >
      </dxi-column>

      <!-- Date Column with Format -->
      <dxi-column
        alignment="center"
        dataType="date"
        [allowSorting]="false"
        format="dd/MM/yyyy"
        dataField="FieldDate"
        [caption]="'MODULE._DateCaption_' | translate"
      >
      </dxi-column>

      <!-- Select Button Column -->
      <dxi-column
        class="text-center"
        type="buttons"
        [allowSorting]="false"
        [caption]="'COMMON._Select' | translate"
      >
        <dxi-button
          name="edit"
          icon="fas fa-edit"
          hint="select"
          [onClick]="selectItem"
        >
        </dxi-button>
      </dxi-column>
    </dx-data-grid>
  </div>
</form>
```

## Key Features

### 1. Search Form Setup

- **Reactive Forms**: Uses FormBuilder to create reactive form
- **Date Initialization**: Uses `startOfDay()` and `endOfDay()` from date-fns
- **Date Range Validation**: Custom validator `AccflexValidators.rangeValidationCustomNames()`
- **Default Values**: Loads default values like inventoryManagementId from static resources

### 2. Grid with Remote Operations

- **CustomStore**: Uses DevExtreme CustomStore for server-side operations
- **Load Function**: Implements loadFunction that receives LoadOptions (skip, take)
- **Key Field**: Always specify a unique key field (usually 'Id')
- **Remote Operations**: Set `[remoteOperations]="true"` on dx-data-grid

### 3. Grid State Management

- **isGridDisabled**: Boolean flag to control grid enabled/disabled state
- **Initial State**: Grid starts disabled (`isGridDisabled = true`)
- **Enable on Results**: Grid enabled only after successful search with results
- **Disable on No Results**: Grid disabled when search returns no results

### 4. Item Selection

Two ways to select items from the grid:

1. **Row Double-Click**: `(onRowDblClick)="onRowDblClick($event)"`
2. **Select Button**: Button column with `[onClick]="selectItem"`

Both methods call `featureFormService.loadById(id)` to load the selected item.

### 5. Search and Reset Flow

**Search Flow**:

1. User fills search form
2. User clicks search button
3. `OnSearchSubmit()` validates form
4. Creates new CustomStore with load function
5. Grid automatically calls load function
6. `getSearchResult()` executes API call with pagination
7. Returns results with data array and totalCount
8. Grid displays results

**Reset Flow**:

1. User clicks cancel button
2. `handleReset()` is called
3. Resets search form to default values
4. Resets grid to empty CustomStore
5. Disables grid

### 6. Date Range Validation

```typescript
{
  validator: AccflexValidators.rangeValidationCustomNames('startDate', 'endDate')
} as AbstractControlOptions
```

- Shows error message when date range is invalid
- Error key: `'InValidPeriod'`
- Displays: `'COMMON._dateOutOfRangeMessage_'`

### 7. Form State Subscription

```typescript
this.featureFormService.sub.sink = this.featureFormService.formState$.subscribe(
  {
    next: (res) => {
      if (res == FormState.NormalFirstLoaded) {
        this.resetGrid();
        this.OnSearchSubmit();
      }
    },
  }
);
```

- Listens to form state changes
- When returning to search after save (`NormalFirstLoaded`)
- Automatically refreshes search results

## Required Imports

```typescript
import { Component, OnInit } from "@angular/core";
import {
  AbstractControlOptions,
  FormBuilder,
  FormGroup,
  Validators,
} from "@angular/forms";
import { TranslateService } from "@ngx-translate/core";
import { endOfDay, startOfDay } from "date-fns";
import { LoadOptions } from "devextreme/data";
import CustomStore from "devextreme/data/custom_store";
import { firstValueFrom } from "rxjs";
import { FormState } from "src/app/models";
import { AccflexValidators } from "src/app/shared/services/global-methods";
import { LoadingHandlerService } from "src/app/shared/services/loading-handler/loading-handler.service";
import { AccflexToasterService } from "src/app/shared/services/accflex-toaster/accflex-toaster.service";
```

## Common Grid Column Types

### Number Column

```html
<dxi-column
  alignment="center"
  dataType="number"
  [allowSorting]="false"
  dataField="FieldName"
  [caption]="'MODULE._Caption_' | translate"
>
</dxi-column>
```

### String Column

```html
<dxi-column
  alignment="center"
  dataType="string"
  [allowSorting]="false"
  dataField="FieldName"
  [caption]="'MODULE._Caption_' | translate"
>
</dxi-column>
```

### Date Column

```html
<dxi-column
  alignment="center"
  dataType="date"
  [allowSorting]="false"
  format="dd/MM/yyyy"
  dataField="FieldName"
  [caption]="'MODULE._Caption_' | translate"
>
</dxi-column>
```

### Boolean Column

```html
<dxi-column
  alignment="center"
  dataType="boolean"
  [allowSorting]="false"
  dataField="FieldName"
  [caption]="'MODULE._Caption_' | translate"
>
</dxi-column>
```

## Best Practices

1. **Always use CustomStore** with server-side pagination for large datasets
2. **Set remoteOperations="true"** to enable server-side operations
3. **Disable grid initially** and enable only after successful search
4. **Validate form** before executing search
5. **Use firstValueFrom()** to convert Observable to Promise for CustomStore
6. **Handle errors** and show appropriate toaster messages
7. **Return proper structure** from load function: `{ data: [], totalCount: 0 }`
8. **Disable search button** during API call to prevent multiple requests
9. **Use date-fns** for date operations (startOfDay, endOfDay)
10. **Subscribe to formState$** to refresh results after save/update operations
11. **Always translate** user-facing text
12. **Support RTL** by checking isRtlEnabled from form service

## Common Search Field Types

### Number Box

```html
<accflex-number-box
  [label]="'MODULE._Label_'"
  [minimumValue]="0"
  [control]="searchForm.controls.fieldName"
  [preventFractions]="true"
>
</accflex-number-box>
```

### Custom Lookup Component

```html
<log-feature-lookup formControlName="fieldName"> </log-feature-lookup>
```

### DevExtreme Lookup

```html
<dx-lookup
  [dataSource]="dataSource$ | async"
  [rtlEnabled]="isRtlEnabled"
  valueExpr="Id"
  displayExpr="Name"
  formControlName="fieldName"
  [searchEnabled]="true"
  [showClearButton]="true"
  [showCancelButton]="false"
  [clearButtonText]="'COMMON._Clear_' | translate"
  [searchPlaceholder]="'COMMON._Search_' | translate"
  placeholder=" "
  labelMode="floating"
  [label]="'MODULE._Label_' | translate"
>
</dx-lookup>
```

### Date Box

```html
<dx-date-box
  type="datetime"
  [showClearButton]="false"
  [useMaskBehavior]="true"
  formControlName="dateField"
  width="100%"
  labelMode="floating"
  [label]="'MODULE._Label_' | translate"
>
  <dx-validator>
    <dxi-validation-rule
      type="required"
      [message]="'COMMON._Required_' | translate"
    >
    </dxi-validation-rule>
  </dx-validator>
</dx-date-box>
```

## Responsive Layout Classes

Use Tailwind CSS classes for responsive column widths:

- **Full width**: `w-full`
- **Quarter width**: `sm:w-full md:w-1/4 lg:w-1/4 xl:w-1/4`
- **Half width**: `sm:w-full md:w-1/2 lg:w-1/2 xl:w-1/2`
- **Third width**: `sm:w-full md:w-1/3 lg:w-1/3 xl:w-1/3`

Always add `p-1` for padding between fields.
