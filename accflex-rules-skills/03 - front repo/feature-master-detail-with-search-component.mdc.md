---
description: "Use when implementing or editing Master Data or Transactional Data feature modules that have a search form component: one shell (skeleton + CRUD floating icons), search-form component, edit form, DataService and FormService, and FormState-driven show/hide (NormalFirstLoaded, AddingNew, Editing, Printing, AfterError, AfterSave). Applies to modules like customer, invoice, production-order. See accflex-erp.mdc for feature types."
globs:
  - "**/*-shell/*.ts"
  - "**/*-shell/*.html"
  - "**/*-search-form*/*.ts"
  - "**/*-search-form*/*.html"
alwaysApply: false
---

#### **Module Structure**

```folders
Customer/
├── components/              # Presentational components
│   ├── customer-form-component 
│   └── search-form-component
├── container/               # Container components
│   └── customer-shell/ #hold the skeleton of module UI
├── shared/                  # Module-specific services
│   ├── customer-data.service.ts
│   └── customer-form.service.ts
└── customer.module.ts
```

- Shell component hold Main CRUD buttons along with all components of the module and show/hide components depend on Form state

```html
<div class="animated fadeIn delay-0.5s">
  <accflex-floating-icons
    [formState]="FeatureFormService.formState$ | async"
    (OnCreate)="FeatureFormService.InitializeForCreate()"
    (OnSubmit)="FeatureFormService.OnSubmit()"
    (OnCancel)="FeatureFormService.ClearState()"
    (OnDelete)="FeatureFormService.ShowDeletePopup()"
    (OnPrint)="OnPrint()"
    [title]="'Feature._Heading_' | translate"
  >
  </accflex-floating-icons>
  <div class="mainContainer">
    <div class="make-border">
      <div class="flex flex-col flex-1" [class.hiddednCotent]= "(FeatureFormService.formState$ | async) == FormState.Editing ||
          (FeatureFormService.formState$ | async) == FormState.AddingNew">
        <log-feature-search-form #searchFormRef></log-feature-search-form>
      </div>

      <div [class.hiddednCotent]="(FeatureFormService.formState$ | async) == FormState.NormalFirstoaded">
        <div class="flex flex-wrap flex-row mb-4">
          <div class="w-full p-1" style="min-height: 50vh">
            <log-feature-form> </log-feature-form>
          </div>
        </div>
      </div>
    </div>
  </div>
<div>
<accflex-concurency-confirm-popup
  #concurencyPopUpRef
  [isConfirmPopupVisible]="FeatureFormService.isConcurrencyVisiable"
  [confirmPopupBody]="'Feature._ConCurencyConfirmPopupBody_' | translate"
  [isRtlEnabled]="FeatureFormService.isRtlEnabled"
  (whenButtonPressed)="FeatureFormService.OnConcurencyPopUpButtonPressed($event)"
>
</accflex-concurency-confirm-popup>

<log-lgx-print
  [ApplicationId]="applicationId"
  #printPurchaseReturn
  [HeaderId]="FeatureFormService.featureForm.get('EntityId').value"
  [ReportName]="FeatureFormService.reportName"
>
</log-lgx-print>  
```

- Main CRUD buttons:
  - Create: to allow create new entity (hided if Module has no creation functionality like configurations )
  - Save: to Save both new entity or updated entity
  - Cancel: reset the changed data and return the form to initial state
  - Delete: allowed only on previously saved entity after select it from search to delete it  (hided if Module has no delete functionality like configurations )
  - Print:  allowed only on previously saved entity after select it from search to print it  (hided if Module has no print functionality like configurations ).
  - Feature module may contain other special function buttons.
- **Form states**
  - NormalFirstLoaded: the initial state when form open from menu link, it show the search components and hide others
  - CreatingNew (AddingNew): when user click on New button hides search components and show form components input controls empty and enabled
  - Editing: when user search for entity and choose it to edit, hides search components and show form components input controls loaded with entity data and enabled
  - Printing: show shared popup to choose print layout then print viewer displayed
  - AfterError: when user click on Save button and API return error, a error toaster displayed and form is disabled
  - AfterSave: when user click on Save button and API return Ok, a success toaster displayed and form is disabled
	  