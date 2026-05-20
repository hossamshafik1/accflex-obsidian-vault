---
description: "Use when implementing or editing Angular feature form services: CRUD form state, FormGroup/BehaviorSubject patterns, delete/concurrency popups, URL state, loading/toast, SubSink, and validation. Applies to Master, Transactional, and Condition/Reference Data features that have a FormService."
globs:
  - "**/*-form.service.ts"
alwaysApply: false
---
# Angular Form Service Architecture - Technical Specifications

## Overview

This document defines the standard architecture pattern for Angular form services used across the application. All form services should follow these specifications to ensure consistency, maintainability, and predictable behavior.

## 1. Service Structure

### 1.1 Service Decorator and Lifecycle

```typescript
@Injectable()
export class [Feature]FormService implements OnDestroy {
  // Implementation
  
  ngOnDestroy(): void {
    this.sub.unsubscribe();
  }
}
```

CRITICAL: All services with subscriptions MUST implement OnDestroy to prevent memory leaks.

### 1.2 Required Properties

- **Subscription Management**: `public sub = new SubSink();`
- **Form Instance**: `[feature]Form: FormGroup<[Feature]ViewModel>;`
- **Initial Values**: `initialValues: [Feature]ViewModel;`
- **Empty Collections**: Arrays for resetting lists (e.g., `emptyDetails: []`)

## 2. State Management

### 2.1 Form State

```typescript
public formState$: BehaviorSubject<FormState> = new BehaviorSubject<FormState>(
  FormState.NormalFirstoaded
);
```

**Required States:**

- `NormalFirstoaded`: Initial/view mode
- `AddingNew`: Create mode
- `Editing`: Update mode

**State Management Method:**

```typescript
updateFormState = (state: FormState): void => {
  this.formState$.next(state);
};
```

### 2.2 Popup Visibility States

All services must implement these BehaviorSubjects:

```typescript
public deleteConfirmationPopupVisibility$: BehaviorSubject<boolean> = 
  new BehaviorSubject<boolean>(false);

public concurrencyPopupVisibility$: BehaviorSubject<boolean> = 
  new BehaviorSubject<boolean>(false);
```

**Visibility Control Methods:**

```typescript
changeDeleteConfirmationPopupVisibility = (state: boolean): void => 
  this.deleteConfirmationPopupVisibility$.next(state);

changeConcurrencyPopupVisibility = (state: boolean): void => 
  this.concurrencyPopupVisibility$.next(state);
```

## 3. CRUD Operations

### 3.1 Create Operation

```typescript
create[Feature](): void {
  // Validation
  if (!this.[feature]Form.valid) {
    this.toasterService.showErrorToast(
      this.translateService.instant('[FEATURE]._DataRequired_')
    );
    return;
  }

  const createCommand = new Create[Feature]Command();
  createCommand.init(this.[feature]Form.getRawValue());
  
  this.loadingHandlerService.showLoading();
  
  this.dataService.create[Feature](createCommand).subscribe({
    next: (res) => {
      this.toasterService.showSuccessToast(this.createDoneMessage);
      this.clearState();
      this.loadingHandlerService.hideLoading();
    },
    error: (err) => {
      this.loadingHandlerService.hideLoading();
      // Handle errors
    }
  });
}
```

### 3.2 Read Operation

```typescript
loadFeatureById(id: number): void {
  this.loadingHandlerService.showLoading();
  
  this.sub.sink = this.dataService.getFeatureById(id).pipe(
    catchError(error => {
      this.handleError(error);
      return throwError(() => error);
    }),
    finalize(() => this.loadingHandlerService.hideLoading())
  ).subscribe({
    next: (dto) => {
      this.updateFeatureForm(dto);
      this.updateFormState(FormState.Editing);
      this.navigateToFeature(id);
    }
  });
}
```

### 3.3 Update Operation

```typescript
updateFeature(): void {
  // Validation
  if (!this.featureForm.valid) {
    this.toasterService.showErrorToast(
      this.translateService.instant('FEATURE._DataRequired_')
    );
    return;
  }

  const updateCommand = new UpdateFeatureCommand();
  updateCommand.init(this.featureForm.getRawValue());
  
  this.loadingHandlerService.showLoading();
  
  this.sub.sink = this.dataService.updateFeature(updateCommand).pipe(
    catchError(error => {
      if (error.status === HTTP_STATUS.BAD_REQUEST && 
          Object.keys(error?.errors).includes(ERROR_KEYS.TIMESTAMP)) {
        this.currentTimeStamp = error.errors.TimeStamp[0];
        this.currentConcurrencyType = ConcurrencyType.Update;
        this.changeConcurrencyPopupVisibility(true);
      } else {
        this.handleError(error);
      }
      return throwError(() => error);
    }),
    finalize(() => this.loadingHandlerService.hideLoading())
  ).subscribe({
    next: (res) => {
      this.toasterService.showSuccessToast(this.editDoneMessage);
      this.clearState();
    }
  });
}
```

### 3.4 Delete Operation

```typescript
deleteFeature(): void {
  const deleteCommand = new DeleteFeatureCommand();
  deleteCommand.FeatureId = this.featureForm.controls.FeatureId.value;
  deleteCommand.TimeStamp = this.featureForm.controls.TimeStamp.value;
  
  this.loadingHandlerService.showLoading();
  
  this.sub.sink = this.dataService.deleteFeature(deleteCommand).pipe(
    catchError(error => {
      if (error.status === HTTP_STATUS.BAD_REQUEST && 
          Object.keys(error?.errors).includes(ERROR_KEYS.TIMESTAMP)) {
        this.currentTimeStamp = error.errors.TimeStamp[0];
        this.currentConcurrencyType = ConcurrencyType.Delete;
        this.changeDeleteConfirmationPopupVisibility(false);
        this.changeConcurrencyPopupVisibility(true);
      } else {
        this.handleError(error);
      }
      return throwError(() => error);
    }),
    finalize(() => this.loadingHandlerService.hideLoading())
  ).subscribe({
    next: (res) => {
      this.toasterService.showSuccessToast(this.deleteDoneMessage);
      this.clearState();
      this.changeDeleteConfirmationPopupVisibility(false);
    }
  });
}
```

## 4. Form Initialization

### 4.1 Form Group Initialization

```typescript
initializeFormGroup(): void {
  this.[feature]Form = new FormGroup<[Feature]ViewModel>({
    [Feature]Id: new FormControl(null, { nonNullable: false }),
    // ... other controls with appropriate validators
    TimeStamp: new FormControl(null, { nonNullable: false }),
    DetailList: new FormControl(structuredClone(this.emptyDetails), {
      nonNullable: false
    })
  });
  
  this.initialValues = this.[feature]Form.getRawValue() as [Feature]ViewModel;
}
```

**Must be called in constructor:**

```typescript
constructor(
  // dependencies
) {
  this.initializeFormGroup();
}
```

### 4.2 Form Update Method

```typescript
private update[Feature]Form(dto: [Feature]Dto): void {
  this.[feature]Form.patchValue(dto, { emitEvent: false });
  
  // Handle nested objects/arrays separately
  this.[feature]Form.patchValue({
    DetailList: structuredClone(dto.DetailList)
  }, { emitEvent: false });
}
```

## 5. Form Reset and Cleanup

### 5.1 Reset Form Values

```typescript
private resetFormValues(): void {
  this.featureForm.patchValue(this.initialValues, {
    emitEvent: false
  });
  this.featureForm.patchValue({
    DetailList: structuredClone(this.emptyDetails)
  }, { emitEvent: false });
}
```

### 5.2 Reset State

```typescript
reset(): void {
  this.formState$.next(FormState.NormalFirstoaded);
  this.resetFormValues();
}
```

### 5.3 Clear State (Complete Cleanup)

```typescript
clearState(): void {
  this.reset();
  this.removeURLState();
}
```

### 5.4 Initialize for Create

```typescript
initializeForCreate(): void {
  this.updateFormState(FormState.AddingNew);
  this.resetFormValues();
}
```

## 6. Form Submission

### 6.1 Standard Submit Handler

```typescript
onSubmit = (): void => {
  if (this.formState$.value == FormState.AddingNew) {
    this.create[Feature]();
  } else if (this.formState$.value == FormState.Editing) {
    this.update[Feature]();
  }
};
```

## 7. Delete Confirmation Flow

### 7.1 Required Methods

```typescript
showDeletePopup(): void {
  this.changeDeleteConfirmationPopupVisibility(true);
}

hideDeletePopup(): void {
  this.changeDeleteConfirmationPopupVisibility(false);
}

confirmDelete(event: ConfirmationEvent): void {
  if (event.confirmed) {
    this.deleteFeature();
  }
  this.changeDeleteConfirmationPopupVisibility(false);
}
```

CRITICAL: Define proper types for event parameters:

```typescript
interface ConfirmationEvent {
  confirmed: boolean;
  data?: unknown;
}
```

## 8. Optimistic Concurrency Control

### 8.1 Required Properties

```typescript
currentTimeStamp: string;
currentConcurrencyType: ConcurrencyType;
```

### 8.2 Concurrency Handler

```typescript
onConcurrencyPopUpButtonPressed(event: PopUpButtonEvent): void {
  if (event?.Button === PopUpButtonState.Yes) {
    // Update with new timestamp
    this.featureForm.patchValue({
      TimeStamp: this.currentTimeStamp
    }, { emitEvent: false });
    
    // Retry operation based on type
    if (this.currentConcurrencyType === ConcurrencyType.Delete) {
      this.confirmDelete({ confirmed: true });
    } else if (this.currentConcurrencyType === ConcurrencyType.Update) {
      this.updateFeature();
    }
  } else if (event?.Button === PopUpButtonState.No) {
    // Reload current data
    if (this.featureForm.controls.FeatureId?.value) {
      this.loadFeatureById(this.featureForm.controls.FeatureId.value);
    }
  }
  this.changeConcurrencyPopupVisibility(false);
}
```

CRITICAL: Define event type:

```typescript
interface PopUpButtonEvent {
  Button: PopUpButtonState;
}
```

### 8.3 Error Handling Pattern

All update/delete operations must include:

```typescript
catchError(error => {
  if (error.status === HTTP_STATUS.BAD_REQUEST && 
      Object.keys(error?.errors).includes(ERROR_KEYS.TIMESTAMP)) {
    this.currentTimeStamp = error.errors.TimeStamp[0];
    this.currentConcurrencyType = ConcurrencyType.Update;
    this.changeConcurrencyPopupVisibility(true);
  } else {
    this.handleError(error);
  }
  return throwError(() => error);
})
```

## 9. URL State Management

### 9.1 Push State (After Load)

```typescript
private navigateToFeature(id: number, ...additionalParams: Record<string, any>[]): void {
  this.router.navigate([], {
    relativeTo: this.activatedRoute,
    queryParams: { id, ...additionalParams },
    queryParamsHandling: 'merge',
    replaceUrl: false
  });
}
```

❌ WRONG - Do NOT use History API:

```typescript
// DON'T DO THIS
private pushState(id: number): void {
  history.pushState(null, document.title, this.router.url.split('?')[0] + `?id=${id}`);
}
```

### 9.2 Remove State (On Reset)

```typescript
private removeURLState(): void {
  this.router.navigate([], {
    relativeTo: this.activatedRoute,
    queryParams: {},
    replaceUrl: true
  });
}
```

❌ WRONG - Do NOT use History API:

```typescript
// DON'T DO THIS
private removeURLState(): void {
  history.replaceState(null, document.title, this.router.url.split('?')[0]);
}
```

### 9.3 Handle Redirection (On Init)

```typescript
handleRedirection(): void {
  const redirect = this.activatedRoute.snapshot.queryParams as {
    id: number;
  };
  
  if (!isNaN(redirect.id)) {
    this.sub.sink = this.dataService.isDataSourcesLoaded$.subscribe({
      next: (isLoaded) => {
        if (isLoaded) {
          this.loadFeatureById(redirect.id);
        }
      }
    });
  }
}
```

## 10. Internationalization

### 10.1 Required Message Properties

```typescript
createDoneMessage = this.translateService.instant('COMMON._CreateDone_');
editDoneMessage = this.translateService.instant('COMMON._EditDone_');
deleteDoneMessage = this.translateService.instant('COMMON._DeletDone_');
```

### 10.2 Translation Keys Convention

- Success messages: `COMMON._CreateDone_`, `COMMON._EditDone_`, `COMMON._DeletDone_`
- Validation messages: `[FEATURE]._DataRequired_`
- Error messages: `[FEATURE]._[SpecificError]_`

## 11. Loading State Management

### 11.1 Standard Pattern

```typescript
// Before async operation
this.loadingHandlerService.showLoading();

// In success/error callbacks
this.loadingHandlerService.hideLoading();
```

### 11.2 Always Hide on Completion

Ensure `hideLoading()` is called in both `next` and `error` callbacks.

## 12. Toast Notifications

### 12.1 Success Notifications

```typescript
this.toasterService.showSuccessToast(this.createDoneMessage);
```

### 12.2 Error Notifications

```typescript
this.toasterService.showErrorToast(
  this.translateService.instant('[FEATURE]._ErrorKey_')
);
```

### 12.3 Info Notifications

```typescript
this.toasterService.showInfoToast(message);
```

## 13. Dependency Injection

### 13.1 Required Services

All form services must inject:

```typescript
constructor(
  private translateService: TranslateService,
  private loadingHandlerService: LoadingHandlerService,
  private toasterService: ToasterService, // or AccflexToasterService
  private router: Router,
  private activatedRoute: ActivatedRoute,
  private dataService: [Feature]DataService,
  // Feature-specific services
) {
  this.initializeFormGroup();
}
```

### 13.2 Optional Common Services

- `RTLService`: For RTL support
- `PermissionService`: For authorization checks
- `StaticResourcesService`: For shared resources
- `SignalRService`: For real-time updates

## 14. Subscription Management

### 14.1 Using SubSink

```typescript
public sub = new SubSink();
```

### 14.2 Subscription Pattern

```typescript
this.sub.sink = this.dataService.someMethod().subscribe({
  next: (res) => {
    // Handle response
  },
  error: (err) => {
    // Handle error
  }
});
```

### 14.3 Cleanup

SubSink automatically unsubscribes when component is destroyed. No manual cleanup needed.

## 15. Data Cloning

### 15.1 Use structuredClone for Deep Copying

```typescript
// When patching arrays/objects
this.[feature]Form.patchValue({
  DetailList: structuredClone(dto.DetailList)
}, { emitEvent: false });
```

### 15.2 Avoid Reference Issues

Always clone when:

- Assigning initial values
- Resetting form arrays
- Updating nested objects

## 16. Validation

### 16.1 Form Validation Check

```typescript
if (!this.[feature]Form.valid) {
  this.toasterService.showErrorToast(
    this.translateService.instant('[FEATURE]._DataRequired_')
  );
  return;
}
```

### 16.2 Business Logic Validation

```typescript
if ((this.[feature]Form.controls?.DetailList?.value ?? []).length == 0) {
  this.toasterService.showErrorToast(
    this.translateService.instant('[FEATURE]._DetailRequired_')
  );
  return;
}
```

## 17. Error Handling (NEW SECTION)

### 17.1 Centralized Error Handler

```typescript
private handleError(error: HttpErrorResponse): void {
  console.error('Service error:', error);
  
  let errorMessage: string;
  
  if (error.error instanceof ErrorEvent) {
    // Client-side error
    errorMessage = this.translateService.instant('ERRORS._ClientError_');
  } else {
    // Server-side error
    switch (error.status) {
      case HTTP_STATUS.BAD_REQUEST:
        errorMessage = this.translateService.instant('ERRORS._BadRequest_');
        break;
      case HTTP_STATUS.UNAUTHORIZED:
        errorMessage = this.translateService.instant('ERRORS._Unauthorized_');
        break;
      case HTTP_STATUS.NOT_FOUND:
        errorMessage = this.translateService.instant('ERRORS._NotFound_');
        break;
      default:
        errorMessage = this.translateService.instant('ERRORS._ServerError_');
    }
  }
  
  this.toasterService.showErrorToast(errorMessage);
}
```

### 17.2 Using Error Handler in Operations

```typescript
this.sub.sink = this.dataService.createFeature(command).pipe(
  catchError(error => {
    this.handleError(error);
    return throwError(() => error);
  }),
  finalize(() => this.loadingHandlerService.hideLoading())
).subscribe({
  next: (res) => {
    this.toasterService.showSuccessToast(this.createDoneMessage);
    this.clearState();
  }
});
```

## 18. Best Practices

### 18.1 Form Control Access

```typescript
// Use optional chaining for safety
this.[feature]Form?.controls?.[ControlName]?.value
```

### 18.2 Error Handling

- Always provide user feedback for errors
- Log errors for debugging: `console.error('Error:', err);`
- Handle concurrency conflicts gracefully

### 18.3 Type Safety

- Use strongly typed FormGroups with view models
- Define interfaces for all DTOs and commands
- Use type assertions carefully: `as [Type]`

### 18.4 Immutability

- Don't mutate form values directly
- Use `patchValue()` for updates
- Clone objects before modification

### 18.5 Performance

- Use `{ emitEvent: false }` when patching values that don't need to trigger subscriptions
- Unsubscribe from observables properly (use SubSink)
- Avoid unnecessary API calls

## 19. Permissions

### 19.1 Permission Check Pattern

```typescript
hasPermissionForType = (typeId: number): boolean => {
  switch (typeId) {
    case Type.Option1:
      return this.permissionService.create[Feature]Option1();
    case Type.Option2:
      return this.permissionService.create[Feature]Option2();
    default:
      return false;
  }
};
```

## 20. Testing Considerations

### 20.1 Testable Methods

All public methods should be unit testable:

- CRUD operations
- State management
- Validation logic
- Permission checks

### 19.2 Mock Dependencies

Ensure all injected services can be mocked:

- Use interfaces where possible
- Avoid tight coupling to concrete implementations
