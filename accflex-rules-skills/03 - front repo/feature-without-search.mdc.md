---
description: "Use when implementing or editing Configuration Data / master-detail feature modules that have NO search: one shell, list (or tree), and form; full dataset loaded; DataService only (no FormService). Applies to settings, lookups, categories, and similar master data. Do not use for features that need search, pagination, or server-side filtering."
alwaysApply: false
---

# Feature Module Without Search Pattern (Master-Detail Pattern)

## Overview

This pattern is for simpler feature modules that display a list of items (often in a tree structure) with a form for create/edit operations. There is no search functionality - the complete dataset is loaded and displayed. Commonly used for master data like categories, settings, lookups, etc.

## Module Structure

### Module File (`feature.module.ts`)

```typescript
import { NgModule } from "@angular/core";
import { RouterModule } from "@angular/router";
import { AccflexSharedModule } from "src/app/shared/accflex-shared.module";
import { LogisticsSharedModule } from "../../shared/logistics-shared.module";
import { FeatureShellComponent } from "./containers/feature-shell/feature-shell.component";
import { FeatureFormComponent } from "./components/feature-form/feature-form.component";
import { FeatureListComponent } from "./components/feature-list/feature-list.component";
import { FeatureDataService } from "./shared/feature-data.service";

@NgModule({
  declarations: [
    FeatureShellComponent,
    FeatureFormComponent,
    FeatureListComponent,
  ],
  imports: [
    LogisticsSharedModule,
    AccflexSharedModule,
    RouterModule.forChild([{ path: "", component: FeatureShellComponent }]),
  ],
  providers: [FeatureDataService],
})
export class FeatureModule {}
```

#### Key Characteristics:

- **3 Components**: Shell (container), Form, List
- **1 Service**: DataService for API operations and state management
- **Simple Routing**: Direct route to shell component
- **No FormService**: State management handled directly in shell component or data service

## Component Structure

### 1. Shell Component (`feature-shell.component.ts`)

**Purpose**: Orchestrates all operations, manages state, handles CRUD operations

**Key Characteristics**:

- ViewChild references to list and form components
- Manages form state (create, edit, view)
- Handles all CRUD operations
- Manages loading and error states
- Controls form enable/disable state
- Handles permissions (add, edit, delete)
- Manages concurrency conflicts

**Structure**:

```typescript
import {
  AfterViewInit,
  Component,
  OnDestroy,
  OnInit,
  ViewChild,
} from "@angular/core";
import { TranslateService } from "@ngx-translate/core";
import { Observable } from "rxjs";
import { SubSink } from "subsink";
import { FormState } from "src/app/models";
import { CommandState } from "src/app/models/CommandState.model";
import { PopUpButtonState } from "src/app/models/PopUpButtonState.model";
import { GlobalLanguageService } from "src/app/core/services/global-language/global-language.service";
import { ConcurencyConfirmPopupComponent } from "src/app/shared/components/concurency-confirm-popup/concurency-confirm-popup.component";
import { AccflexToasterService } from "src/app/shared/services/accflex-toaster/accflex-toaster.service";
import { CleanObjectService } from "src/app/shared/services/clean-object/clean-object.service";
import { ErrorHandlerService } from "src/app/shared/services/error-handler/error-handler.service";
import { LoadingHandlerService } from "src/app/shared/services/loading-handler/loading-handler.service";
import { FeatureFormComponent } from "../../components/feature-form/feature-form.component";
import { FeatureListComponent } from "../../components/feature-list/feature-list.component";
import { FeatureDataService } from "../../shared/feature-data.service";

@Component({
  selector: "log-feature-shell",
  templateUrl: "./feature-shell.component.html",
  providers: [], // Add additional providers if needed
})
export class FeatureShellComponent implements OnInit, OnDestroy, AfterViewInit {
  @ViewChild("listRef") listRef: FeatureListComponent;
  @ViewChild("formRef") formRef: FeatureFormComponent;
  @ViewChild("concurencyPopUpRef")
  concurencyPopUpRef: ConcurencyConfirmPopupComponent;

  // State properties
  isRtlEnabled = true;
  isFormDisabled = true;
  isConfirmDeletePopupVisible = false;
  isConfirmSavePopupVisible = false;
  hasAddPermission = false;
  hasEditPermission = false;

  formState$: Observable<FormState>;
  stateForSubmit: FormState;

  // Lookup/dropdown data
  lookupData1: Type1[] = [];
  lookupData2: Type2[] = [];

  private subs = new SubSink();

  ngOnInit(): void {
    this.getPermission();
    this.getRtl();
    this.getFormState();
    this.loadLookupData();
  }

  ngAfterViewInit(): void {
    this.disableForm();
    this.getDataList();
  }

  ngOnDestroy(): void {
    this.subs.unsubscribe();
    this.clearState();
  }

  // ==================== Initialization ====================

  getPermission = (): void => {
    this.hasAddPermission = this.permissionsHandleService.userAllowAdd();
    this.hasEditPermission = this.permissionsHandleService.userAllowEdit();
  };

  getRtl = (): void => {
    this.globalLanguageService.getLang().subscribe((lang) => {
      if (lang === null || lang === "ar-EG") {
        this.isRtlEnabled = true;
      } else {
        this.isRtlEnabled = false;
      }
    });
  };

  getFormState = (): Observable<FormState> | void => {
    this.formState$ = this.dataService.formState$;
    this.subs.sink = this.dataService.formState$.subscribe(
      (state) => (this.stateForSubmit = state)
    );
  };

  loadLookupData = (): void => {
    // Load all necessary lookup/dropdown data
    this.getLookupData1();
    this.getLookupData2();
  };

  // ==================== Data Loading ====================

  getDataList = (): void => {
    this.subs.sink = this.dataService.getListWithNotification().subscribe(
      (res) => {
        this.initializeList(res);
        this.hideLoading();
      },
      (err) => {
        this.hideLoading();
        this.handleError(err);
      }
    );
  };

  getLookupData1 = (): void => {
    this.subs.sink = this.dataService.getLookupData1().subscribe(
      (res) => {
        this.lookupData1 = res;
        this.hideLoading();
      },
      (err) => {
        this.hideLoading();
        this.handleError(err);
      }
    );
  };

  initializeList = (data: any[]): void => {
    this.listRef.dataSource = data;
  };

  // ==================== Create Operations ====================

  initializeForCreate = (): void => {
    this.dataService.initializeForm(FormState.CreatingNew);
    this.clearErrors();
    this.resetForm();
    this.enableForm();
  };

  AddNewItem = async (e: any): Promise<void> => {
    if (!this.hasAddPermission) {
      this.accflexToasterService.showInfoToast(
        this.translate.instant("COMMON._YouDontHaveAddPermision_")
      );
      return;
    }
    this.dataService.initializeForm(FormState.CreatingNew);
    this.resetForm();
    this.enableForm();
    // Load initial data for new item if needed
    await this.getNewItemData(e?.Id);
  };

  getNewItemData = (parentId?: number): void => {
    this.subs.sink = this.dataService.getNewItemByParentId(parentId).subscribe(
      (res) => {
        this.formRef.Form.patchValue(res);
        this.hideLoading();
      },
      (err) => {
        this.hideLoading();
        this.handleError(err);
      }
    );
  };

  createItem = (): void => {
    let body = this.formRef.Form.value;
    body = this.cleanObjectService.cleanNullAndEmptyString(body);

    this.subs.sink = this.dataService.create(body).subscribe(
      () => {
        this.hideLoading();
        this.getDataList();
        this.accflexToasterService.showSuccessToast(
          this.translate.instant("COMMON._CreateDone_")
        );
        this.clearState();
      },
      (err) => {
        this.hideLoading();
        this.handleError(err);
      }
    );
  };

  // ==================== Read/Edit Operations ====================

  OpenItem = async (e: any): Promise<void> => {
    await this.getItemById(e.Id);
    this.enableForm();
    this.dataService.initializeForm(FormState.Editing);
  };

  getItemById = (id: number): void => {
    this.subs.sink = this.dataService.getById(id).subscribe(
      (res) => {
        this.formRef.Form.patchValue(res);
        this.hideLoading();
      },
      (err) => {
        this.hideLoading();
        this.handleError(err);
      }
    );
  };

  // ==================== Update Operations ====================

  updateItem = (): void => {
    let body = this.formRef.Form.value;
    body = this.cleanObjectService.cleanNullAndEmptyString(body);

    this.subs.sink = this.dataService.update(body.Id, body).subscribe(
      () => {
        this.hideLoading();
        this.getDataList();
        this.accflexToasterService.showSuccessToast(
          this.translate.instant("COMMON._UpdatedDone_")
        );
        this.clearState();
      },
      (err) => {
        // Handle concurrency conflict
        if (
          err.status === 400 &&
          Object.keys(err?.error?.errors).includes("TimeStamp")
        ) {
          this.formRef.Form.patchValue({
            TimeStamp: err.error.errors.TimeStamp[0],
          });
          this.concurencyPopUpRef.showPopup(true, CommandState.Update);
        } else {
          this.handleError(err);
          this.hideLoading();
        }
      }
    );
  };

  // ==================== Delete Operations ====================

  deleteItem = (): void => {
    const body = this.formRef.Form.value;
    this.subs.sink = this.dataService.delete(body.Id, body.TimeStamp).subscribe(
      () => {
        this.hideLoading();
        this.getDataList();
        this.accflexToasterService.showSuccessToast(
          this.translate.instant("COMMON._DeleteDone_")
        );
        this.clearState();
        this.disableForm();
      },
      (err) => {
        // Handle concurrency conflict
        if (
          err.status === 400 &&
          Object.keys(err?.error?.errors).includes("TimeStamp")
        ) {
          this.formRef.Form.patchValue({
            TimeStamp: err.error.errors.TimeStamp[0],
          });
          this.concurencyPopUpRef.showPopup(true, CommandState.Delete);
        } else {
          this.handleError(err);
          this.hideLoading();
        }
      }
    );
  };

  // ==================== Form Submission ====================

  OnSubmit = (e): void => {
    e?.preventDefault();
    if (!this.isFormValid()) {
      return;
    }

    switch (this.stateForSubmit) {
      case FormState.CreatingNew:
        this.createItem();
        break;
      case FormState.Editing:
        this.updateItem();
        break;
      default:
        break;
    }
  };

  isFormValid = (): boolean => {
    const { value } = this.formRef.Form;
    // Add validation logic
    if (!value.RequiredField1 || !value.RequiredField2) {
      return false;
    }
    return true;
  };

  // ==================== Popup Handlers ====================

  whenDeletePopupConfirm = (e): void => {
    this.hideDeletePopup();
    if (e) {
      this.deleteItem();
    }
  };

  whenSavePopupConfirm = (e): void => {
    this.hideSavePopup();
    if (e) {
      this.createItem();
    }
  };

  showDeletePopup = (): boolean => (this.isConfirmDeletePopupVisible = true);
  hideDeletePopup = (): boolean => (this.isConfirmDeletePopupVisible = false);

  showSavePopup = (): boolean => (this.isConfirmSavePopupVisible = true);
  hideSavePopup = (): boolean => (this.isConfirmSavePopupVisible = false);

  // ==================== Concurrency Handler ====================

  concurencyPopUpButtonPressed(e: any): void {
    if (e?.Button === PopUpButtonState.Yes) {
      if (e?.command === CommandState.Update) {
        this.OnSubmit(null);
      }
      if (e?.command === CommandState.Delete) {
        this.deleteItem();
      }
    } else {
      const body = this.formRef.Form.value;
      this.OpenItem(body);
    }
  }

  // ==================== Utility Methods ====================

  handleError = (error: any): void =>
    this.errorHandlerService.handleError(error);
  clearErrors = (): void => this.errorHandlerService.clearErrors();

  showLoading = (): void => this.loadingHandlerService.showLoading();
  hideLoading = (): void => this.loadingHandlerService.hideLoading();

  resetForm = (): void => this.formRef.renderForm();

  disableForm = (): boolean => (this.isFormDisabled = true);
  enableForm = (): boolean => (this.isFormDisabled = false);

  clearState = (): void => {
    this.dataService.initializeForm(FormState.NormalFirstoaded);
    this.resetForm();
    this.disableForm();
    this.clearDxValidators();
  };

  clearDxValidators = (): void => {
    setTimeout(() => {
      this.formRef.clearDxValidators();
    }, 200);
  };

  constructor(
    private dataService: FeatureDataService,
    private loadingHandlerService: LoadingHandlerService,
    private errorHandlerService: ErrorHandlerService,
    private translate: TranslateService,
    private accflexToasterService: AccflexToasterService,
    private globalLanguageService: GlobalLanguageService,
    private cleanObjectService: CleanObjectService,
    private permissionsHandleService: LogisticsPermissionsHandleService
  ) {}
}
```

### 2. List Component (`feature-list.component.ts`)

**Purpose**: Displays data in a list or tree structure

**Key Characteristics**:

- Uses DevExtreme TreeList or DataGrid
- Emits events for user actions (open, add, delete, etc.)
- Receives data via @Input
- Supports selection tracking
- Minimal logic - mostly presentation

**Structure**:

```typescript
import {
  Component,
  Input,
  ViewChild,
  Output,
  EventEmitter,
} from "@angular/core";
import { DxTreeListComponent } from "devextreme-angular";

@Component({
  selector: "log-feature-list",
  templateUrl: "./feature-list.component.html",
})
export class FeatureListComponent {
  @ViewChild(DxTreeListComponent, { static: false })
  treeListRef: DxTreeListComponent;

  // Event emitters for user actions
  @Output() OpenItem: EventEmitter<any> = new EventEmitter<any>();
  @Output() AddNewItem: EventEmitter<any> = new EventEmitter<any>();
  @Output() DeleteItem: EventEmitter<any> = new EventEmitter<any>();
  @Output() ApplyItem: EventEmitter<any> = new EventEmitter<any>();

  // Input properties
  @Input() isRtlEnabled = true;
  @Input() hasAddPermission = true;
  @Input() hasEditPermission = true;
  @Input() dataSource = [];

  selectedItem: any;

  onSelectionChanged(e) {
    this.syncSelection(e.component);
  }

  syncSelection(treeView) {
    const selected = treeView?.getSelectedNodeKeys();
    if (selected?.length) {
      const target = this.dataSource.find((i) => i.Id === selected[0]);
      this.selectedItem = target;
    }
  }

  treeViewContentReady(e) {
    this.syncSelection(e.component);
  }

  onOpenClick = (): void => {
    if (this.selectedItem) {
      this.OpenItem.emit(this.selectedItem);
    }
  };

  onAddClick = (): void => {
    this.AddNewItem.emit(this.selectedItem);
  };

  onDeleteClick = (): void => {
    if (this.selectedItem) {
      this.DeleteItem.emit(this.selectedItem);
    }
  };
}
```

**Template Structure** (Tree List):

```html
<div class="flex flex-col h-full">
  <dx-tree-list
    #treeListRef
    [dataSource]="dataSource"
    keyExpr="Id"
    parentIdExpr="ParentId"
    [rtlEnabled]="isRtlEnabled"
    [showBorders]="true"
    [columnAutoWidth]="true"
    [autoExpandAll]="true"
    [selectedRowKeys]="[selectedItem?.Id]"
    (onSelectionChanged)="onSelectionChanged($event)"
    (onContentReady)="treeViewContentReady($event)"
  >
    <dxo-selection mode="single"></dxo-selection>
    <dxo-search-panel [visible]="true"></dxo-search-panel>

    <dxi-column
      dataField="Code"
      [caption]="'MODULE._Code_' | translate"
    ></dxi-column>
    <dxi-column
      dataField="Name"
      [caption]="'MODULE._Name_' | translate"
    ></dxi-column>

    <!-- Action buttons column -->
    <dxi-column type="buttons" [caption]="'COMMON._Actions_' | translate">
      <dxi-button
        icon="edit"
        hint="Edit"
        [visible]="hasEditPermission"
        [onClick]="onOpenClick"
      ></dxi-button>
      <dxi-button
        icon="add"
        hint="Add"
        [visible]="hasAddPermission"
        [onClick]="onAddClick"
      ></dxi-button>
      <dxi-button
        icon="trash"
        hint="Delete"
        [visible]="hasEditPermission"
        [onClick]="onDeleteClick"
      ></dxi-button>
    </dxi-column>
  </dx-tree-list>
</div>
```

### 3. Form Component (`feature-form.component.ts`)

**Purpose**: Handles form rendering and basic form logic

**Key Characteristics**:

- Contains reactive form definition
- Receives data via @Input
- Emits events for field changes
- Handles field-level enable/disable logic
- Minimal business logic
- ViewChildren for DxValidator components

**Structure**:

```typescript
import { FormBuilder, FormGroup } from "@angular/forms";
import {
  Component,
  OnInit,
  Input,
  Output,
  EventEmitter,
  QueryList,
  ViewChildren,
} from "@angular/core";
import { DxValidatorComponent } from "devextreme-angular";
import { FormState } from "src/app/models";

@Component({
  selector: "log-feature-form",
  templateUrl: "./feature-form.component.html",
  styleUrls: ["./feature-form.component.css"],
})
export class FeatureFormComponent implements OnInit {
  @ViewChildren(DxValidatorComponent)
  validatorViewChildren: QueryList<DxValidatorComponent>;

  // Input properties
  @Input() isRtlEnabled = true;
  @Input() isFormDisabled = true;
  @Input() formState: FormState;
  @Input() lookupData1: any[] = [];
  @Input() lookupData2: any[] = [];

  // Output events
  @Output() fieldChanged: EventEmitter<any> = new EventEmitter<any>();

  // Form
  Form: FormGroup;

  // Field-specific flags
  isField1Disabled = false;
  isField2Disabled = false;

  ngOnInit(): void {
    this.renderForm();
  }

  renderForm = (): void => {
    this.Form = this.fb.group({
      Id: null,
      Code: "",
      Name: "",
      NameSecondLanguage: "",
      Description: "",
      Field1: "",
      Field2: "",
      IsActive: true,
      TimeStamp: "",
      CreatedBy: "",
      CreatedOn: "",
      LastModifiedBy: "",
      LastModifiedOn: "",
    });

    // Set initial field states
    this.disableField1();
    this.disableField2();
  };

  // Field change handlers
  onField1Changed = (e): void => {
    this.fieldChanged.emit({ field: "Field1", value: e.value });
    // Add field-specific logic
  };

  onField2Changed = (e): void => {
    this.fieldChanged.emit({ field: "Field2", value: e.value });
    // Add field-specific logic
  };

  // Field enable/disable methods
  disableField1 = (): void => {
    this.isField1Disabled = true;
    this.Form.patchValue({ Field1: null });
  };

  enableField1 = (): boolean => (this.isField1Disabled = false);

  disableField2 = (): void => {
    this.isField2Disabled = true;
    this.Form.patchValue({ Field2: null });
  };

  enableField2 = (): boolean => (this.isField2Disabled = false);

  // Clear DevExtreme validators
  clearDxValidators = (): void => {
    this.validatorViewChildren.toArray().map((ref) => {
      ref.instance.reset();
    });
  };

  constructor(protected fb: FormBuilder) {}
}
```

### 4. Data Service (`feature-data.service.ts`)

**Purpose**: Handles API calls and form state management

**Key Characteristics**:

- Extends base service class
- BehaviorSubject for form state
- BehaviorSubject for data with notifications
- CRUD operation methods
- Observable streams for reactive updates

**Structure**:

```typescript
import { Injectable } from "@angular/core";
import { BehaviorSubject, Observable } from "rxjs";
import { LogisticsBaseService } from "../../../shared/services/base-service/logistics-base.service";
import { FormState } from "src/app/models";
import {
  FeatureClient,
  FeatureDetailsDto,
  FeatureForListDto,
  CreateFeatureCommand,
  UpdateFeatureCommand,
  NewFeatureInfoDto,
} from "logistics.api.client";

@Injectable()
export class FeatureDataService extends LogisticsBaseService {
  url = `${this.apiUrl}/Feature`;

  // BehaviorSubjects for state management
  formState$ = new BehaviorSubject<FormState>(FormState.NormalFirstoaded);
  data$ = new BehaviorSubject<FeatureForListDto[]>([]);

  // Form state management
  initializeForm = (state: FormState) => this.formState$.next(state);
  clearForm = () => this.formState$.next(FormState.NormalFirstoaded);
  getFormState = (): Observable<any> => this.formState$.asObservable();

  // Data with notification pattern
  dataWithNotification() {
    return this.featureClient.getAll().subscribe((data) => {
      this.data$.next(data);
    });
  }

  getListWithNotification(): Observable<FeatureForListDto[]> {
    this.dataWithNotification();
    return this.data$.asObservable();
  }

  // CRUD Operations
  getById = (id: number): Observable<FeatureDetailsDto> =>
    this.featureClient.getById(id);

  getNewItemByParentId = (parentId?: number): Observable<NewFeatureInfoDto> =>
    this.featureClient.getNewByParentId(parentId);

  create = (command: CreateFeatureCommand): Observable<FeatureDetailsDto> =>
    this.featureClient.post(command);

  update = (
    id: number,
    command: UpdateFeatureCommand
  ): Observable<FeatureDetailsDto> => this.featureClient.put(id, command);

  delete = (id: number, timeStamp: string): Observable<void> =>
    this.featureClient.delete(id, timeStamp);

  // Additional operations
  getLookupData1 = (): Observable<any[]> => this.featureClient.getLookupData1();

  getLookupData2 = (): Observable<any[]> => this.featureClient.getLookupData2();

  constructor(private featureClient: FeatureClient) {
    super();
  }
}
```

## Key Patterns and Best Practices

### 1. State Management

- **Form State**: Use BehaviorSubject in data service
- **Data State**: Use BehaviorSubject for reactive data updates
- **Component State**: Store UI state in shell component

### 2. Data Flow

```
Shell Component (Orchestrator)
    ↓
    ├─→ List Component (Display)
    ├─→ Form Component (Input)
    └─→ Data Service (API)
```

### 3. Event Flow

```
List/Form Component (User Action)
    → Event Emitter
    → Shell Component (Handler)
    → Data Service (API Call)
    → Shell Component (Update State)
    → List Component (Refresh Display)
```

### 4. Subscription Management

- Always use SubSink
- Unsubscribe in ngOnDestroy
- Clean up state on destroy

### 5. Concurrency Handling

```typescript
if (
  err.status === 400 &&
  Object.keys(err?.error?.errors).includes("TimeStamp")
) {
  this.formRef.Form.patchValue({
    TimeStamp: err.error.errors.TimeStamp[0],
  });
  this.concurencyPopUpRef.showPopup(true, CommandState.Update);
}
```

### 6. Permission Checks

```typescript
if (!this.hasAddPermission) {
  this.accflexToasterService.showInfoToast(
    this.translate.instant("COMMON._YouDontHaveAddPermision_")
  );
  return;
}
```

### 7. Form Validation

- Use DevExtreme validators in template
- Add custom validation in `isFormValid()` method
- Clear validators when resetting form

### 8. Data Cleaning

```typescript
let body = this.formRef.Form.value;
body = this.cleanObjectService.cleanNullAndEmptyString(body);
```

## Required Services/Imports

```typescript
import { SubSink } from "subsink";
import { TranslateService } from "@ngx-translate/core";
import { Observable, BehaviorSubject } from "rxjs";
import { FormState } from "src/app/models";
import { CommandState } from "src/app/models/CommandState.model";
import { PopUpButtonState } from "src/app/models/PopUpButtonState.model";
import { GlobalLanguageService } from "src/app/core/services/global-language/global-language.service";
import { AccflexToasterService } from "src/app/shared/services/accflex-toaster/accflex-toaster.service";
import { CleanObjectService } from "src/app/shared/services/clean-object/clean-object.service";
import { ErrorHandlerService } from "src/app/shared/services/error-handler/error-handler.service";
import { LoadingHandlerService } from "src/app/shared/services/loading-handler/loading-handler.service";
import { ConcurencyConfirmPopupComponent } from "src/app/shared/components/concurency-confirm-popup/concurency-confirm-popup.component";
```

## Typical Shell Component Template

```html
<div class="flex flex-row h-full">
  <!-- List Section (Left) -->
  <div class="w-1/3 p-2 border-r">
    <log-feature-list
      #listRef
      [isRtlEnabled]="isRtlEnabled"
      [hasAddPermission]="hasAddPermission"
      [hasEditPermission]="hasEditPermission"
      (OpenItem)="OpenItem($event)"
      (AddNewItem)="AddNewItem($event)"
      (DeleteItem)="showDeletePopup()"
    >
    </log-feature-list>
  </div>

  <!-- Form Section (Right) -->
  <div class="w-2/3 p-2">
    <log-feature-form
      #formRef
      [isRtlEnabled]="isRtlEnabled"
      [isFormDisabled]="isFormDisabled"
      [formState]="stateForSubmit"
      [lookupData1]="lookupData1"
      [lookupData2]="lookupData2"
      (fieldChanged)="onFieldChanged($event)"
    >
    </log-feature-form>

    <!-- Form Buttons -->
    <div class="button-container" *ngIf="!isFormDisabled">
      <dx-button
        [text]="'COMMON._Save_' | translate"
        type="success"
        [disabled]="!formRef.Form.valid"
        (onClick)="OnSubmit($event)"
      ></dx-button>
      <dx-button
        [text]="'COMMON._Cancel_' | translate"
        type="danger"
        (onClick)="clearState()"
      ></dx-button>
    </div>
  </div>
</div>

<!-- Confirmation Popups -->
<accflex-popup-confirm
  [isVisible]="isConfirmDeletePopupVisible"
  [message]="'COMMON._AreYouSureDelete_' | translate"
  (OnClose)="whenDeletePopupConfirm($event)"
>
</accflex-popup-confirm>

<!-- Concurrency Popup -->
<concurency-confirm-popup
  #concurencyPopUpRef
  (OnButtonPressed)="concurencyPopUpButtonPressed($event)"
>
</concurency-confirm-popup>
```

## Folder Structure

```
feature/
├── components/
│   ├── feature-form/
│   │   ├── feature-form.component.ts
│   │   ├── feature-form.component.html
│   │   └── feature-form.component.css
│   └── feature-list/
│       ├── feature-list.component.ts
│       └── feature-list.component.html
├── containers/
│   └── feature-shell/
│       ├── feature-shell.component.ts
│       └── feature-shell.component.html
├── shared/
│   └── feature-data.service.ts
└── feature.module.ts
```

## When to Use This Pattern

Use this pattern when:

- ✅ Working with master data (categories, types, lookups)
- ✅ Dataset is small enough to load entirely
- ✅ Hierarchical data structure (tree)
- ✅ No complex search requirements
- ✅ Simple CRUD operations
- ✅ Master-detail UI layout

Don't use this pattern when:

- ❌ Large datasets requiring pagination
- ❌ Complex search functionality needed
- ❌ Multiple search criteria
- ❌ Server-side filtering/sorting required

## Key Differences from Search Pattern

| Feature              | Without Search       | With Search           |
| -------------------- | -------------------- | --------------------- |
| Data Loading         | Load all at once     | Paginated             |
| Search Functionality | None                 | Search form + grid    |
| Component Count      | 3 (Shell/Form/List)  | 4+ (adds SearchShell) |
| Service Complexity   | Simple DataService   | Form + Data Service   |
| State Management     | Shell or DataService | FormService           |
| UI Layout            | Master-Detail        | Search → Detail       |
| CustomStore          | Not needed           | Required              |
| Remote Operations    | No                   | Yes                   |
