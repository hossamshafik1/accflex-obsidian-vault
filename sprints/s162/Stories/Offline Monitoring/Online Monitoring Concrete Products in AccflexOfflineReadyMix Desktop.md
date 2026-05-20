
## Context

The Angular client at `projects/logistics/src/app/modules/monitoring-concrete-products` is the production-grade monitoring screen for ready-mix delivery (Module 621). It has two tabs (Delivery Schedule Search + Delivery Tracking), ~25 API endpoints, and ten popups for status workflow, redirects, partial deliveries, price conditions, and printing.

The WinForms/DevExpress desktop app `AccflexOfflineReadyMix` (.NET 4.7.2) was built as an offline-first companion with a thin `MonitoringForm` that hits only the analytics OData endpoint and caches everything locally in SQLite. The user wants this desktop to become a fully **online** replacement for the Angular screen — same tabs, same grids, same popups, same APIs — so field staff on Windows can run the entire monitoring workflow without a browser.

The OAuth2 client `accflex_offlinereadymix` already exists in Identity (commits 8bee543, 552f77c) with scopes `logisticsapi miscapi analyticsreaderapi generalledgerapi IdentityServerApi` — no scope changes are required. The desktop already has token acquisition, refresh, typed HttpClient registration, and DPAPI session storage. Per the user's decision: offline code stays on disk but is bypassed at runtime; the new screen replaces the current `MonitoringForm`; all popups are in scope (full Angular parity).

## Scope Confirmation

- **No Identity changes needed.** `accflex_offlinereadymix` already holds every scope this screen will touch. Verified at `Identity/AccessControl.API/Helpers/Config.cs:1455-1485` and `AccflexOfflineReadyMix/Auth/OidcClient.cs:31`.
- **No new SQLite migration.** Offline schema (`Data/Migrations/V1Schema.cs`) is left untouched; the new screen does live API calls only.
- **No MVVM framework introduced.** Continue with code-behind WinForms + DevExpress, matching the existing form style.

## Target Architecture

### New form: `MonitoringConcreteProductsForm` (replaces `MonitoringForm`)

`Forms/MonitoringConcreteProductsForm.cs` — `XtraForm` with a `XtraTabControl` containing two pages mirroring `MonitoringConcreteProductsShellComponent`:

- **Tab 1 — Delivery Schedule Search** (mirrors `delivery-schedule-search.component.ts`)
- **Tab 2 — Delivery Tracking** (mirrors `delivery-tracking.component.ts`)

`Program.cs:121-192` is updated so the post-login launch points at the new form instead of `MonitoringForm`. The old `MonitoringForm.cs` and `SyncForm.cs` are kept on disk (per user decision) but no longer referenced from the launcher.

### API client layer

Add typed clients under `Api/Logistics/` that mirror the Angular `*Client` services one-for-one. Each is registered through the existing reflection sweep in `Api/ApiClientRegistration.cs:40-64` by implementing `ILogisticsClientMarker` (already supported). Base URL comes from `SessionStore.Current.LogisticsApiUrl`; auth handler is already wired.

Required clients and the endpoints each must expose:

| Client                             | Endpoints (method + purpose)                                                                                                            |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `MonitoringConcreteProductsClient` | `GET DeliveryTracking` (14 filters), `GET DeliverySchedules` (8 filters), `PUT DeliveryPlanFromSalesOrder`, `DELETE DeliveryPlanDetail` |
| `CustomerClient`                   | `GET AllSalesCustomer`, `GET CustomerDeliveryAddressDto`, `GET CustomerIndebtedness`                                                    |
| `StationClient`                    | `GET AllStations`                                                                                                                       |
| `ConcreteRecipeClient`             | `GET AllMixtureDesign`, `GET AllSlumpLevel`, `GET AllAddedMaterials`, `GET AllWaterAndIce`                                              |
| `ConditionTypeClient`              | `GET SalesConditionTypeList`                                                                                                            |
| `PriceProcedureClient`             | `GET PriceProcedureByType`                                                                                                              |
| `DeliveryPlanClient`               | `GET SalesOrdersForRedirectDeliveryPlans`, `GET MaterialsFromSalesOrderForDeliveryPlan`                                                 |
| `SalesOrderClient`                 | `GET BySalesOrderId`, `PUT UpdateSalesOrderDetailConditions`                                                                            |
| `BillOfMaterialClient`             | `GET AllRawMaterialToBomList`                                                                                                           |
| `GeneralLedgerSharedClient`        | `GET Currencies`, `GET AccountsForPriceProcedure` (uses `GeneralLedgerApiUrl`)                                                          |

URLs are taken verbatim from the Angular generated client (`accflex-erp/projects/*/clients`) — confirm the exact route strings during implementation by reading those files.

### Service & state layer

`Services/MonitoringConcreteProductsDataService.cs` — non-UI helper that holds the cached lookup lists (customers, materials, drivers, trucks, pumps, units, price-procedure list, condition-type list, inventory-management list, project list, sales-org list, stations) and exposes async loaders. Lookups are fetched on form open and reused across both tabs, matching the Angular pattern (18 BehaviorSubjects → C# fields + `event`s).

`Services/MonitoringPermissionsService.cs` — fetches the user permissions blob already cached in localStorage on the web (Module 621, IDs 163–172 for Discount/Tax/Freight/Price edit rights). On desktop we read it from the `users_me` endpoint or the JWT claims and gate the price-condition popup buttons accordingly.

### Grids

Use `GridControl` + `GridView` with banded layout for both tabs. Column inventory (must match Angular):

- **Schedule tab grid**: Material, Customer, Sales Org, Delivery Schedule Date, Quantity, Unit, Status, Total Required/Received/Remaining, Detail-Quantities button, Add-Trucks button, Add-Pumps button, Slump Level (combo), Added Materials (checklist), Water/Ice (checklist).
- **Tracking tab grid**: Serial, Material Name/Code, Customer, Source Type, Quantity, Unit, Truck, Station, Driver, Pump, Reason, Delivery Plan Code, Delivery Plan Date, Sales Order Notes, Customer Indebtedness, Mixture Designs (combo), Status (Previous/Current/Next), Partially-Directed, Print, Delete, Info.

Search panels use DevExpress `LayoutControl` with the same required-marker semantics.

### Popups (all in scope)

Each becomes its own `XtraForm` under `Forms/MonitoringConcreteProducts/`:

1. `DetailScheduleQuantitiesForm.cs`
2. `AddTrucksForm.cs` (replaces existing offline `AddTrucksDialog.cs`)
3. `AddPumpsForm.cs`
4. `DeliveryTrackingDetailTabForm.cs` — time fields (ExPlant, StartDischarge, EndDischarge, OnSite, ReturnedPlant)
5. `RedirectToAnotherCustomerForm.cs`
6. `RedirectCustomerCreateDataForm.cs`
7. `RedirectedStatusForm.cs`
8. `PartiallyDirectedForm.cs` — split delivered/redirected/scrapped quantities
9. `DeliveryPlanDetailPriceConditionForm.cs` — Discount/Tax/Freight/Price conditions
10. `DeliveryPlanDetailPrintForm.cs` — XtraReports viewer + layout designer; layout cache reused from existing `report_layouts` SQLite table.

Status workflow (1→2→5; 2→3/4/6) is enforced in `MonitoringConcreteProductsDataService.AdvanceStatus()` exactly as the Angular service does.

### DTOs

Add C# DTOs under `Models/MonitoringConcreteProducts/` mirroring the TS interfaces (`DeliveryTrackingDto`, `DeliveryScheduleForListDto`, `PartiallyDirectedViewModel`, `DeliveryPlanDetailDto`, etc.). Generate by reading the Angular generated-clients output and translating field-by-field. Use `Newtonsoft.Json` (already referenced) with camelCase property naming.

## Files to add

- `Forms/MonitoringConcreteProductsForm.cs/.Designer.cs/.resx`
- `Forms/MonitoringConcreteProducts/<10 popup forms>.cs/.Designer.cs/.resx`
- `Api/Logistics/<10 typed clients>.cs`
- `Api/GeneralLedger/GeneralLedgerSharedClient.cs`
- `Services/MonitoringConcreteProductsDataService.cs`
- `Services/MonitoringPermissionsService.cs`
- `Models/MonitoringConcreteProducts/*.cs` (DTOs)

## Files to modify

- `Program.cs:121-192` — point the post-login launch at `MonitoringConcreteProductsForm`.
- `Api/ApiClientRegistration.cs` — no logic change; new clients are picked up automatically by the marker-interface sweep. Verify `GeneralLedgerApiUrl` base address is wired.
- `Auth/SessionStore.cs` — confirm `LogisticsApiUrl` and `GeneralLedgerApiUrl` are populated post-login; add a fast-fail check in the new form if either is empty.
- `AccflexOfflineReadyMix.csproj` — include the new files.

## Files left untouched (per "bypass, don't remove")

- `Data/LocalDb.cs`, `Data/Migrations/V1Schema.cs`, all `Data/Repositories/*`
- `Sync/*`
- `Forms/MonitoringForm.cs`, `Forms/SyncForm.cs`, `Forms/AddTrucksDialog.cs`
- `Api/MonitoringApi.cs` (legacy analytics-OData wrapper)

## Verification

End-to-end manual test path:

1. **Identity**: launch `AccessControl.API` (`https://localhost:5000`); confirm `accflex_offlinereadymix` token request returns `access_token` containing `logisticsapi`, `miscapi`, `analyticsreaderapi`, `generalledgerapi` scopes (decode JWT at jwt.io).
2. **Logistics & GL APIs**: start both APIs locally on their existing ports.
3. **Desktop login**: run `AccflexOfflineReadyMix.exe`, sign in via `LoginForm`. Confirm `SessionStore.Current.LogisticsApiUrl` / `GeneralLedgerApiUrl` are set (log line at form load).
4. **Schedule tab**: enter Material + SalesOrg + Customer + date range → Search → grid populates from `GET DeliverySchedules`. Open Detail Quantities, Add Trucks, Add Pumps popups; each loads its lookup and persists via the matching `PUT`.
5. **Tracking tab**: same search → grid populates from `GET DeliveryTracking`. Walk a row through status 1→2→5; then a second row 2→4 (Redirect) → choose existing customer → confirm `PUT DeliveryPlanFromSalesOrder` lands; then a third row 2→partial-direct → split quantities → confirm save.
6. **Price conditions**: open the conditions popup on a redirected row, edit Discount/Tax, save, confirm `PUT UpdateSalesOrderDetailConditions` returns 200 and the grid refreshes.
7. **Print**: open the print popup, render the report, switch to the layout designer, save a layout, reopen and confirm layout persists (via the existing `report_layouts` SQLite table — this is the one place we intentionally reuse local storage for UX, not for data).
8. **Cross-check with Angular**: run the same search in the Angular client side-by-side and confirm the grid row count and totals match for the same filter set.

Network sanity: use Fiddler/Charles against the desktop process to confirm every HTTP call carries `Authorization: Bearer …` and that 401s trigger the existing refresh-token flow in `AuthenticationDelegatingHandler`.

## Out of scope

- Removing offline code, SQLite, sync, or PBKDF2 login (user opted to bypass, not delete).
- Identity scope/client changes (already in place).
- Mobile, web, or non-monitoring desktop screens.
- Any AI/LLM-driven endpoints — the Angular module does not call any AI endpoints; the user's mention of "AI calls" appears to refer to API calls (verified during exploration).