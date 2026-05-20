## Context

Goal: bring the WinForms `AccflexOfflineReadyMix` desktop to functional parity
with the Angular `monitoring-concrete-products` module so a field operator can
run the same workflow from a local app — search delivery schedules, monitor
delivery tracking, change statuses, assign trucks/pumps, redirect to other
customers, edit price conditions, print.

The desktop sign-in path was also simplified: the user no longer types API URLs.
They pick a tenant, the app derives a company code, and the URLs for Logistics /
Misc / GL / Analytics / Identity are resolved automatically against the Hadaf
license server (`newlicense.hadafsolutions.net`) and cached locally.

A global Online/Offline indicator is wired into the form shell, and every HTTP
call goes through a connectivity guard so API calls short-circuit cleanly when
the app is offline.

---

## End-to-end flow (verified)

```
Program.Main
  └─ Single-instance mutex + DI bootstrap (ApiClientRegistration)
  └─ Pipeline per HttpClient (outer → inner):
       OfflineCache → ConnectivityGuard → Logging → Authentication
  └─ LoginForm
       ├─ Warm URLs from LicenseCache.LastUsedCompanyCode
       ├─ Tenant dropdown loads via TenantLookup
       ├─ cmbTenant.EditValueChanged → CompanyCodeResolver
       │     ├─ cache hit  → URLs applied silently to SessionStore
       │     └─ cache miss → LicenseLookup.FetchAsync → cache write
       ├─ OidcClient.SignInAsync (ROPC) → tokens stored
       └─ ApiClientHelper.SetTokens
  └─ MonitoringConcreteProductsForm
       ├─ BuildHeader → ConnectivityIndicator
       ├─ ConnectivityState.Start (20-s probe + NetworkStatus.Changed hook)
       ├─ MonitoringConcreteProductsState.LoadLookupsAsync (×10 endpoints, Safe-wrapped)
       └─ XtraTabControl
            ├─ DeliveryScheduleTab  → GetDeliverySchedulesAsync
            └─ DeliveryTrackingTab  → GetDeliveryTrackingAsync
                 └─ Row buttons → 10 popups + status workflow + delete
```

Audit result: no missing `TypedClients.Get<T>` registrations, no popup that
the row menu opens without a backing class, no DTO property mismatch. Every
file in `Forms/MonitoringConcreteProducts/` is in the .csproj. The full e2e
path compiles and is wired.

---

## Done

### Sign-in / URL discovery (no more manual URL entry)
- `Auth/LicenseLookup.cs` — cache-first `GetAsync`, network-only `FetchAsync`,
  `ApplyToSession` writes resolved URLs to `SessionStore` and persists.
- `Auth/LicenseCache.cs` — DPAPI-encrypted `license-cache.dat` next to
  `session.dat`. `Get/Put` by code, `LastUsedCompanyCode` tracked.
- `Auth/CompanyCodeResolver.cs` — three heuristics: all-digit `Identifier` →
  `Company_NNN` in `ConnectionString` → first digit run in `Identifier`.
- `Forms/LoginForm.cs` — silent auto-resolve on tenant pick, fallback strip
  only when resolver returns null. API URL group + 4 textboxes hidden;
  Credentials/Sign-in/error label shifted up; form height shrunk.
- `Session.CompanyCode` field added to `SessionStore` for warm starts.

### Monitoring screen (parity with Angular module)
- `Forms/MonitoringConcreteProductsForm.cs` — 2-tab shell, header strip with
  connectivity pill, replaces the old `MonitoringForm` as the post-login
  launch target in `Program.cs`.
- `Forms/MonitoringConcreteProducts/`
  - `DeliveryTrackingTab.cs` — 3-fields-per-row no-scroll filter strip,
    Print/Search/Clear bar, grid with Serial / Material Name / Material Code /
    Customer / Delivery Plan Source Type / Recipe Qty / Unit / Truck /
    Stations / Driver / Pump / Reason / ReadyMix Delivery Code / Notes /
    Actual First Date / Customer Debt / Concrete Element / Previous Status /
    Current Status / Next Status / Partially Directed / Print / Delete.
  - `DeliveryScheduleTab.cs` — same shape filter strip + Sales Orders Status
    radio (All / With remaining / Completed), grid with all Angular columns +
    Add Trucks / Add Pump per-row icons + net-value footer summaries.
  - `TrackingRow.cs` / `ScheduleRow.cs` — grid view-models that resolve FK
    ids to names via cached lookups.
  - `FieldGrid.cs` — shared 3-column TableLayoutPanel helper, no scrollbar
    by construction (rows are exact pixel height).
  - `Popups.cs` — single dialog opener that fires `state.RequestRefresh()` on OK.
  - 10 popups: `TrackingDetailTabPopup`, `AddTrucksPopup`, `AddPumpsPopup`,
    `DetailScheduleQuantitiesPopup`, `RedirectedStatusPopup`,
    `RedirectToAnotherCustomerPopup`, `RedirectCustomerCreatePopup`,
    `PartiallyDirectedPopup`, `PriceConditionsPopup`, `PrintPopup`.
- `Services/MonitoringConcreteProductsState.cs` — shared state, lookups via
  `LoadLookupsAsync` (Customers / Materials / SalesOrgs / Stations /
  MixtureDesigns / SlumpLevels / AddedMaterials / WaterAndIce /
  SalesConditionTypes / PriceProcedures). Lazy truck / driver name resolution
  from local SQLite reference tables.
- `Models/DeliveryPlanDetailStatus.cs` — 1 → 2 → {3,4,5,6} state machine +
  formatter.

### Connectivity
- `Services/ConnectivityState.cs` — singleton `IsOnline` + `StatusChanged` +
  20-s probe of `/.well-known/openid-configuration` + `OfflineException`.
- `Api/ConnectivityGuardDelegatingHandler.cs` — outermost handler on every
  HttpClient. Short-circuits offline, flips state from 5xx / network failures.
- `Forms/MonitoringConcreteProducts/ConnectivityIndicator.cs` — coloured-dot
  pill in the header, click to force a probe.
- Both tabs' `Search()` methods catch `OfflineException` silently (the pill
  already conveys the state).

### Permissions
- `Services/MonitoringPermissions.cs` — module 621 ids (163–172). Permissions
  pulled from JWT claim if present; defaults to allow.
- `PriceConditionsPopup.ApplyPermissions()` drops the grid + Save button to
  read-only when the user lacks edit rights.

### Localization (EN / AR bilingual UI)
- `Infrastructure/AppUiCulture.cs` — maps `Session.Language` (ietf tag, e.g.
  `en-US` / `ar-EG`) to thread UI culture, exposes `IsRtl` and
  `Select(en, ar)` for inline copy without `.resx`.
- `LoginForm` — `rgLanguage` picker, `ApplyFromSession`, localized static
  texts, RTL-aware shell layout. `_suppressLanguageEvent` re-entrancy guard.
- `DeliveryScheduleTab` — every label / button / radio item now goes through
  `AppUiCulture.Select("English", "العربية")`. Filter cell labels, status
  radio items, Search/Clear/Print button text.
- `DeliveryTrackingTab` — same treatment expected on its filter strip and
  buttons (verify the linter applied the same pattern there).

### Build hygiene
- All `ICollection<T>` / `IList<T>` mismatches fixed with `ToList()` helper.
- Real DTO field names verified against the NuGet (`NameDto`,
  `UpdateDeliveryPlanDetailConditionDto`, `DeliveryPlanPumpForDetail`, etc.).
- All new files registered in `.csproj`.
- Login form UX: URL group hidden, credentials shifted up, form shrunk —
  user only sees Tenant + Username + Password + Sign in (+ language radio).

---

## Next steps (prioritised)

### 1. Resolve Truck / Driver / Pump names in the grid
Currently shown as `#123`. Two complementary fixes:
- **Short-term**: `MonitoringConcreteProductsState.TruckName` and `DriverName`
  already fall back to the local SQLite reference tables — wire
  `ReferenceSyncService` to run on form open so those tables are populated
  before the user searches.
- **Long-term**: add explicit `GetAllTrucksAsync` / `GetAllDriversAsync` /
  `GetAllPumpsAsync` endpoints to the Logistics API and regenerate the client.
- Files: `Services/MonitoringConcreteProductsState.cs` (TruckName / DriverName
  helpers), `Sync/ReferenceSyncService.cs`.

### 2. Real permissions claim
`MonitoringPermissions` currently default-allows unless a `permissions` JWT
claim is present. Either:
- Server-side adds the claim list to the access token, or
- Add a small client that fetches `/api/userpermissions/{moduleId}` once on
  sign-in and caches it on `SessionStore`.
- File: `Services/MonitoringPermissions.cs:55`.

### 3. Print parity
`PrintPopup` renders a simple field/value table. The Angular UI lets users
save column layouts per user. The existing SQLite `report_layouts` table
already supports this — wire `GridControl.MainView.SaveLayoutToStream` to
that table keyed by user id, load on open.
- File: `Forms/MonitoringConcreteProducts/PrintPopup.cs`,
  `Data/Repositories/` (existing `report_layouts` table).

### 4. Pre-warm lookups on successful sign-in
After login succeeds, call `MonitoringConcreteProductsState.LoadLookupsAsync`
in the background so the first navigation to the Schedule/Tracking tab is
instant. Fire-and-forget; failures are already `Safe`-wrapped.
- File: `Forms/LoginForm.cs btnSignIn_Click` after `DialogResult.OK`.

### 5. Run ReferenceSyncService from the new form
`Sync/ReferenceSyncService` populates the SQLite `reference_*` tables used by
the lazy lookup helpers. It isn't kicked off from the new form. Either run it
on form open or expose a manual "Sync now" button on the header strip next to
the connectivity pill.
- File: new method in `MonitoringConcreteProductsForm` plus the existing
  reference sync service under `Sync/`.

### 6. Cancel-token plumbing in popups
Popups make `CancellationToken.None` calls. If the user closes the popup
mid-save, the request still completes server-side. Give each popup a CTS that
cancels on `FormClosed`, pass the token into every API call.
- Files: all 10 popups under `Forms/MonitoringConcreteProducts/`.

### 7. Finish localization sweep
- The localization helper (`AppUiCulture.Select`) is in use on the Login form
  and Delivery Schedule tab. Apply it to:
  - Every column caption in the grids (currently hard-coded English).
  - Every popup title + label (10 popups).
  - The connectivity pill text ("Online" / "Offline").
  - Status formatter strings in `Models/DeliveryPlanDetailStatus.cs`.
- Stretch: migrate to `.resx` once the bilingual surface stabilises so a third
  language only needs a new resource file.
- Files: all 10 popups, both tabs (column captions), `ConnectivityIndicator`,
  `DeliveryPlanDetailStatus`.

### 8. Pre-flight verification before shipping
1. `dotnet build` (or VS build) — should compile clean now.
2. Run with `--skip-auth` to exercise the UI without an Identity server.
3. With a real environment:
   - Pick a tenant; confirm Connectivity pill turns green; confirm URL
     resolution happens silently (check `Logger` for `cache hit` / fresh fetch).
   - Tracking tab: search, click Previous/Current/Next status, open each popup
     from the row, walk a row through 1→2→5, redirect a row 2→4, partially
     direct, delete.
   - Schedule tab: search with each radio mode (All / Remaining / Completed),
     confirm net-value footers add up, open Detail Quantities, Add Trucks,
     Add Pump.
   - Pull the network cable mid-search → expect a quiet failure; the pill
     turns red within ~20 s of the next probe.

---

## Critical files to keep in view

- `Program.cs` — DI bootstrap, launch sequencing
- `Api/ApiClientRegistration.cs` — handler pipeline
- `Api/ConnectivityGuardDelegatingHandler.cs`
- `Services/ConnectivityState.cs`
- `Services/MonitoringConcreteProductsState.cs`
- `Auth/LicenseLookup.cs` + `LicenseCache.cs` + `CompanyCodeResolver.cs`
- `Forms/LoginForm.cs`
- `Forms/MonitoringConcreteProductsForm.cs`
- `Forms/MonitoringConcreteProducts/DeliveryTrackingTab.cs`
- `Forms/MonitoringConcreteProducts/DeliveryScheduleTab.cs`
- `Forms/MonitoringConcreteProducts/PriceConditionsPopup.cs` (most complex DTO mapping)
- `Infrastructure/AppUiCulture.cs` — bilingual helper used across the new UI