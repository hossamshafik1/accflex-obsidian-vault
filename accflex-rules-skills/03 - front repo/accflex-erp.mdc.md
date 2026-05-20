---
description: AccFlex ERP Angular Application Architecture
globs: *.ts
alwaysApply: true
---

# AccFlex ERP Angular Application Architecture

## **Overall Architecture**

This is a **multi-project Angular workspace** (Angular 18) that implements a **micro-frontend architecture** for an Enterprise Resource Planning (ERP) system. The application uses a **monorepo structure** with multiple independent Angular applications.

### **Main Application Structure**

### **Technology Stack**

- **Angular 18** with standalone components
- **DevExtreme** for UI components and data grids
- **NgRx** for state management
- **SignalR** for real-time communication
- **OIDC** for authentication
- **TypeScript** with strict typing
- **Tailwind CSS** for styling
- **RxJS** for reactive programming

#### **Root Application (`accflex-erp`)**

- **Purpose**: Main shell application that orchestrates all sub-applications
- **Architecture**: Standalone components with lazy-loaded child applications
- **Key Features**:
  - OIDC authentication integration
  - Multi-language support (Arabic/English with RTL)
  - Theme management (DevExtreme themes)
  - Global navigation and routing
  - Concurrency handling
  - Loading states management

## Root Project Structure

```folder
accflex-erp/
├── dist/
├── node_modules/
├── src/
└── projects/
```

## Main Application (`accflex-erp/src/`)

```folder

/accflex-erp/src/
├── environments/
├── assets/
│ ├── configs/
│ ├── dash-i18n/
│ │ └── i18n/
│ ├── dist/
│ │ ├── css/
│ │ ├── img/
│ │ └── js/
│ ├── doc-links/
│ ├── fonts/
│ ├── i18n/
│ ├── icons/
│ └── silent-refresh/
└── app/
```

## Main Application (`accflex-erp/src/app/`)

```
app/
├── messages/
├── models/
│ ├── classes/
│ ├── enums/
│ └── interfaces/
├── core/
│ ├── components/
│ │ ├── accflex-logo/
│ │ ├── approval-home/
│ │ ├── apps-menu/
│ │ ├── company-info/
│ │ ├── config-menu/
│ │ ├── copy-users-layouts/
│ │ ├── drawer-menu/
│ │ ├── home/
│ │ ├── nav/
│ │ ├── page-not-found/
│ │ ├── session-expired/
│ │ ├── setting-menu/
│ │ ├── signin-oidc/
│ │ ├── signout-oidc/
│ │ ├── theme-menu/
│ │ └── use-apps/
│ ├── guards/
│ ├── interceptors/
│ └── services/
│ ├── apps-drawer-menu/
│ ├── global-language/
│ └── i18n-loader/
├── shared/
│ ├── accflex-notifaction/
│ ├── api/
│ ├── components/
│ ├── directives/
│ ├── pips/
│ ├── services/
│ ├── signalR/
│ ├── tour/
│ ├── types/
│ └── utils/
├── accflex-concurrency/
│ ├── components/
│ └── services/
└── favorites-dashboard/
├── components/
├── models/
└── services/
```

## Sub-Applications (`projects/`)

The workspace contains **7 independent Angular applications**:

1. **`logistics`** - Manufacturing and supply chain management (application Id =12)
2. **`general-accounting`** - Financial accounting and GL (application Id =1)
3. **`payroll`** - Human resources and payroll (application Id =4)
4. **`cash-control`** - Treasury and cash management (application Id =7)
5. **`construction`** - Construction project management (application Id =9)
6. **`dashboard`** - System administration and approvals
7. **`property-manager`** - Property management (application Id =15)

### **Sub-Project Architecture (Logistics Example)**

Each sub-project follows a consistent **feature-based module architecture**:

```folder
projects/logistics/
├── src/
│ └── app/
│   ├── core/               # Core functionality
│   │ ├── components/
│   │ │ └── home/
│   │ └── guards/
│   │ └── can-activate/
│   ├── models/             # TypeScript interfaces/models
│   │ ├── classes/
│   │ ├── enums/
│   │ └── interfaces/
│   ├── modules/            # Feature modules
│   ├── shared/             # Shared components/services
│   │ ├── components/
│   │ ├── directives/
│   │ ├── pipes/
│   │ └── services/
│   │   ├── base-service/
│   │   └── shared-service/
│   └── routes.ts               # Routing configuration
└── environments/
```

### **Feature Module Architecture**

- Features Module in ERP has one of these types
  - "Master Data" – relatively static business entities like (Customer, Vendor, Material).
    - **[module rules](mdc:feature-with-search-component.mdc)**
  - "Transactional Data" – day-to-day documents like (Quotation, Sales Order, Invoice).
    - has CRUD operations
    - creation has special flow with search to choose the referencing document
    - has search-form-component
    - has print functionality  with multiple layout
    - **[module rules](mdc:feature-with-search-component.mdc)**
  - "Configuration Data" – how the system works (rules, settings).
    - Opens directly with default data and ability to save it.
    - No search-form-component,No delete or print functionality
    - **[module rules](mdc:feature-without-search.mdc)**
  - "Organizational Data" – structure of the enterprise (Company Code, Plant, Sales Org).
    - No delete or print functionality
    - Search functionality done by devexpress data grid displayed always on side of edit form
    - **[module rules](mdc:feature-with-search-grid.mdc)**
  - "Condition/Reference Data" – flexible business rules (pricing, discounts).
    - has CRUD operations
    - No print functionality
    - No Delete functionality replaced with active period.
    - **[module rules](mdc:feature-with-search-component.mdc)**
  - "Analytical Data" – Reporting of historical/derived data.
    - has query form inputs
    - has DevExtreme grid for result
    - has print functionality with multiple layout

- Features Module follows a consistent **container-presenter pattern** in most cases it has ONE shell component
- Some Feature Modules may has different structure this will be stated explicitly..

### **Key Architectural Patterns**

#### **1. Container-Presenter Pattern**

- **Container**: Handle business logic, data fetching, and state management
- **Presenters**: Focus on UI rendering and user interactions

#### **2. Service Layer Architecture**

- **Data Services**: Handle API communication and data transformation **[Data Service](mdc:feature-data-service.mdc)**
- **Form Services**:
  - Form State Management: Manages form lifecycle and validation
  - CRUD Operations: Create, Read, Update, Delete business logic
  - Data Import: Import from various source types (deliveries, production orders, etc.)
  - Concurrency Handling: Manages concurrent access conflicts
  - Permission Management: Role-based access control
  - Real-time Updates: SignalR integration for live updates
  - **[Form service rule](mdc:feature-form-service-pattern.mdc)**

#### **3. Lazy Loading Strategy**

- Each sub-application is lazy-loaded based on company context
- Route structure: `/apps/:companyID/[application-name]`
- Dynamic imports for optimal bundle splitting

#### **4. Shared Module Pattern**

- **LogisticsSharedModule**: Contains 600+ shared components and services search here for components before create new one.
- **AccflexSharedModule**: Global shared functionality
- Reusable components like lookups, checklists, and CRUD operations

### **Module Communication**

#### **Inter-Module Communication**

- **API Clients**: Auto-generated from OpenAPI specifications
- **Shared Models**: Common interfaces across modules
- **Event Services**: For cross-module communication
- **State Management**: NgRx for complex state sharing

#### **Production Order Module Services**

```typescript
// Data Service - API communication
ProductionOrderDataService

// Form Service - Business logic and form management  
ProductionOrderFormService

// Validators - Custom validation logic
ProductionOrderValidator (multi-provider pattern)
```

### **Routing Architecture**

#### **Multi-Level Routing**

```typescript
// Root level
/apps/:companyIdentifier

// Sub-Application level  
/apps/:companyIdentifier/logistics

// Module level
/apps/:companyIdentifier/logistics/production-order # for create new entity
/apps/:companyIdentifier/logistics/production-order/:id # for update saved entity
```

#### **Route Guards**

- **AccflexAuthGuard**: Authentication verification
- **CanActivateRouteGuard**: Permission-based access control

### **State Management**

#### **Form State Management**

- **Reactive Forms** with custom validators
- **FormState enum** for tracking form lifecycle
- **BehaviorSubject** for real-time state updates

#### **Data Flow**

1. **Container** subscribes to data service
2. **Data Service** fetches from API clients
3. **Form Service** manages form state and validation
4. **Components** react to state changes

### **Development Patterns**

#### **Code Organization**

- **Feature modules** with clear boundaries
- **Shared components** for reusability
- **Type-safe** API clients with generated models
- **Consistent naming** conventions across modules

#### **Performance Optimizations**

- **Lazy loading** of sub-applications
- **OnPush** change detection strategy
- **Bundle splitting** by feature
- **Tree shaking** for unused code elimination

This architecture provides a **scalable, maintainable, and modular** ERP solution that can handle complex business requirements while maintaining code quality and developer productivity.
