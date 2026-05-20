

## Context

The customer's primary need for `monitoring-concrete-products` in the logistics sub-app is the two server-rendered prints (**"Delivery Plan Detail Print"** and **"DeliveryTrackingReport"**), and field sites can lose internet for extended periods. The PM has pivoted from the in-progress PWA approach to a **lightweight Windows desktop companion** that works with zero connectivity.

Design must mirror the existing `AccflexCloudDesigner` app's conventions exactly: **WinForms + .NET Framework 4.6.2 + DevExpress XtraReports v21.2 + WixSharp MSI + `accflex-<app>://` deep-link OIDC handoff**. The desktop app embeds the *existing* Angular `monitoring-concrete-products` route in **WebView2** (no new Angular workspace project) so the substantial offline work already committed in `projects/logistics/.../monitoring-concrete-products/offline/` is reused verbatim. The two server-rendered reports render locally via the XtraReports runtime already licensed for the Designer.

Intended outcome: a field user launches the desktop app once from the ERP web (via `accflex-monitoring://` protocol + MSI download fallback), syncs lookups and report layouts, then operates fully offline — creating delivery schedules, adding trucks, printing both reports — and syncs back when reconnected.

---

## Architecture at a Glance

```
┌────────────────────────── AccFlexMonitoring.exe (WinForms, .NET 4.6.2) ──────────────────────────┐
│                                                                                                     │
│  Program.cs ──► parse accflex-monitoring:// args ──► SessionStore (DPAPI on disk)                  │
│        │                                                                                            │
│        ▼                                                                                            │
│  MainForm (WebView2 control)                                                                        │
│        │                                                                                            │
│        │  SetVirtualHostNameToFolderMapping → https://desktop.accflex.local/ → wwwroot\             │
│        │  AddScriptToExecuteOnDocumentCreatedAsync → seeds window.__ACCFLEX_DESKTOP_CONTEXT__       │
│        │  AddHostObjectToScript("accflexHost", DesktopHost)                                         │
│        │                                                                                            │
│        ▼                                                                                            │
│  wwwroot\  (the existing logistics ng build output, shipped with the MSI)                          │
│        │                                                                                            │
│        ▼                                                                                            │
│  Angular app (monitoring-concrete-products module)                                                  │
│    • Dexie cache (already exists)                                                                   │
│    • window.chrome.webview.postMessage({kind:'print', reportName, layoutId, data}) on print        │
│                                                                                                     │
│  DesktopHost (ApiClientHelper, ReportRenderer)                                                      │
│    • Bearer refresh against IdP                                                                     │
│    • XtraReport.LoadLayoutFromXml + JsonDataSource + ShowRibbonPreview/Print/ExportToPdf           │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. New .NET Solution

**Location:** `D:\Hadaf Work TFS\AccFlex_ERP\AccFlexMonitoringDesktop\`  
**Pattern source:** clone `D:\Hadaf Work TFS\AccFlex_ERP\AccflexCloudDesigner\` project/folder structure verbatim, then adapt.

### Projects

| Project                              | Target                  | Purpose                               |
| ------------------------------------ | ----------------------- | ------------------------------------- |
| `AccFlexMonitoringDesktop`           | `net462`, `WinExe`, x64 | Host + WebView2 + XtraReports runtime |
| `AccFlexMonitoringDesktop.Installer` | WixSharp console exe    | Generates MSI at build time           |


### Files to create

| File                                                                       | Source pattern                                                                                                                 | Notes                                                                                                                                                                                                                                                                                                                                              |
| -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AccFlexMonitoringDesktop\Program.cs`                                      | `AccflexCloudDesigner\AccFlexDesigner\Program.cs` (lines 27–41 registry self-registration; version-check branch lines 119–186) | Parses `accflex-monitoring://` args; `versioncheck` branch reused verbatim                                                                                                                                                                                                                                                                         |
| `AccFlexMonitoringDesktop\ReportHelper.cs`                                 | `AccflexCloudDesigner\AccFlexDesigner\ReportHelper.cs` (param parser lines 452–485)                                            | Rename to `ProtocolArgs.cs`; same parsing logic                                                                                                                                                                                                                                                                                                    |
| `AccFlexMonitoringDesktop\ApiClientHelper.cs`                              | `AccflexCloudDesigner\AccFlexDesigner\ApiClientHelper.cs`                                                                      | Static HttpClient + Bearer header; extend with `RefreshAccessTokenAsync()` against IdP                                                                                                                                                                                                                                                             |
| `AccFlexMonitoringDesktop\SessionStore.cs`                                 | **NEW**                                                                                                                        | DPAPI wrapper: `ProtectedData.Protect/Unprotect` → `%LOCALAPPDATA%\Accflex\MonitoringDesktop\session.dat`. Stores refresh_token, id_token, userId, tenantId, apiUrls, language, expiresAt. **Divergence from Designer (which is in-memory-only) is required because the companion must outlive the browser session.** Access token stays RAM-only. |
| `AccFlexMonitoringDesktop\MainForm.cs(.Designer.cs)`                       | **NEW**                                                                                                                        | Fullscreen WebView2 host. Wire `SetVirtualHostNameToFolderMapping`, inject `window.__ACCFLEX_DESKTOP_CONTEXT__` via `AddScriptToExecuteOnDocumentCreatedAsync`, register `DesktopHost` via `AddHostObjectToScript`. Handle `WebMessageReceived` for print/sync requests.                                                                           |
| `AccFlexMonitoringDesktop\Bridge\DesktopHost.cs`                           | **NEW**                                                                                                                        | `[ComVisible(true)]`. Methods: `PrintReport(string layoutXml, string dataJson)`, `PreviewReport(...)`, `ExportPdf(string outPath, ...)`, `GetAccessToken()`, `RefreshSession()`, `PersistSession(...)`, `IsOnline()`.                                                                                                                              |
| `AccFlexMonitoringDesktop\Reporting\ReportRenderer.cs`                     | **NEW**                                                                                                                        | `XtraReport.LoadLayoutFromXml(stream)`, `report.DataSource = new DevExpress.DataAccess.Json.JsonDataSource(...)`, `ShowRibbonPreview()` / `Print()` / `ExportToPdf()`.                                                                                                                                                                             |
| `AccFlexMonitoringDesktop\wwwroot\`                                        | **build artifact**                                                                                                             | Populated by a post-build step copying `dist/accflex-erp/` (the main Angular build) into the MSI payload.                                                                                                                                                                                                                                          |
| `AccFlexMonitoringDesktop.Installer\Program.cs`                            | `AccflexCloudDesigner\AccFlexDesigner.Installer\Program.cs`                                                                    | Change: installer dir → `%ProgramFiles%\Accflex\MonitoringDesktop`; protocol → `accflex-monitoring`; new product GUID (both MSIs must coexist); bundle WebView2 Evergreen Bootstrapper.                                                                                                                                                            |
| `AccFlexMonitoringDesktop.Installer\Assets\MicrosoftEdgeWebview2Setup.exe` | download                                                                                                                       | Embedded bootstrapper — runs silently if runtime absent.                                                                                                                                                                                                                                                                                           |

---

## 2. Angular-Side Changes (main workspace — no new project)

Reuses the existing `accflex-erp` build; the desktop just serves `dist/accflex-erp/` from its local virtual host and deep-links into `/apps/:companyId/logistics/monitoring-concrete-products`.

### 2a. New files

| File | Purpose |
|---|---|
| `src/app/shared/services/accflex-monitoring-desktop/accflex-monitoring-desktop.service.ts` | Mirror of `accflex-advanced-designer.service.ts`. Builds `accflex-monitoring://open?...` URL (adds `refresh_token`, `id_token`, `tenantId`, `expiresAt`, `logisticsApiUrl`, `miscApiUrl`) and routes to `/monitoring-desktop-launcher`. |
| `src/app/shared/components/monitoring-desktop-launcher/*` | Mirror of `designer-launcher` (same protocol+fallback-download mechanic, different installer filename). |
| `src/app/core/services/desktop-host/desktop-host.service.ts` | Detects `window.chrome?.webview` + `window.__ACCFLEX_DESKTOP_CONTEXT__`. Exposes `isDesktopHost`, `getContext()`, `postMessage(msg)`, `onMessage$`. Used by OIDC bootstrap and the print integration. |
| `projects/logistics/src/app/modules/monitoring-concrete-products/offline/offline-print.service.ts` | Collects the exact JSON data contract for each of the 2 reports from Dexie, then `desktopHost.postMessage({kind:'print', reportName, layoutId, data})`. Also handles a "capture" mode that snapshots the *server*'s JSON payload once while online so we can replay offline with an identical shape. |

### 2b. Files to modify

| File | Change |
|---|---|
| `src/app/core/services/accflex-oidc.service.ts` | On bootstrap, if `desktopHostService.isDesktopHost`, seed `currentUser` from `window.__ACCFLEX_DESKTOP_CONTEXT__` and skip the OIDC redirect dance. Refresh via `desktopHost.RefreshSession()` bridge call instead of direct IdP call. |
| `src/app/shared/services/utils.ts` (`configStorageGetter`) | Extend to read URL/context API urls when running in desktop host — so `configStorage.apiUrls` is already populated before any API client constructs. |
| `src/app/app-routing.module.ts` | Add `/monitoring-desktop-launcher` route. |
| `projects/logistics/src/app/modules/monitoring-concrete-products/offline/offline-db.ts` | Bump schema to **v3**. Add table `reportLayouts: '++id, [reportName+userId+tenantId], layoutId, lastUpdated'` (stores the XtraReports XML string per user per report). |
| `projects/logistics/src/app/modules/monitoring-concrete-products/offline/offline-db.service.ts` | Add `getReportLayout(reportName, userId, tenantId)` / `saveReportLayout(entry)` / `listReportLayouts()`. |
| `projects/logistics/src/app/modules/monitoring-concrete-products/offline/reference-data-cache.service.ts` | Add `syncReportLayouts()` — calls `ReportLayoutClient.getLayout(...)` for the 2 ReportNames scoped to the current user, stores XML. |
| `projects/logistics/src/app/modules/monitoring-concrete-products/offline/offline-sync.service.ts` | Replace the `of(0)` stubs at **lines 113 & 118**: `pushScheduleToServer` → `MonitoringConcreteProductsClient.createDeliverySchedule(record.payload)` (confirm exact generated method name at implementation time); `pushTruckToServer` → `DeliveryPlanClient.addTruck({...record.payload, deliveryPlanDetailId: record.serverParentId})`. Schedule before trucks (trucks may reference offline-only parents). On 409/400: set `syncStatus=SyncFailed` + `syncError`, keep record. |
| `projects/logistics/src/app/modules/monitoring-concrete-products/componants/add-trucks-pupop/add-trucks-pupop.component.ts` | `OnSubmitThenPrint()` now delegates to `OfflinePrintService.print('Delivery Plan Detail Print', deliveryPlanDetailId)` when `desktopHostService.isDesktopHost || !networkStatusService.isOnline`. Otherwise keeps existing `printRef.ProcessPrint()`. |
| `projects/logistics/src/app/modules/monitoring-concrete-products/componants/delivery-tracking/delivery-tracking.component.ts` **(line 432)** | Same branching pattern for both `"Delivery Plan Detail Print"` and `"DeliveryTrackingReport"`. |
| `projects/logistics/src/app/modules/monitoring-concrete-products/offline/components/sync-status-badge/*` | Add a "Sync Now" button that invokes `OfflineSyncService.syncAll()` + `ReferenceDataCacheService.refreshAll()` + `syncReportLayouts()`. Surface failed records with retry/discard actions. |
| `src/assets/configs/config.json` | Add `accflexMonitoringDesktop: { installerVersion: "1.0.0.0" }`. |
| `src/assets/installer/monitoring-desktop/.gitkeep` | Drop-zone for CI to publish `AccFlexMonitoringDesktop-{version}.msi`. |

### 2c. Reused as-is (no changes)

- `offline/offline-db.service.ts` (CRUD wrapper), `offline-entry.service.ts`, `offline-delivery-schedule.service.ts`, `offline-delivery-schedule-popup/`, `models/`
- `src/app/core/services/pwa/network-status.service.ts`
- `AccflexSharedModule`, `LogisticsSharedModule`

---

## 3. Deep-Link & Auth Flow (definitive)

1. In the ERP, a button on the monitoring module (or sidebar) calls `accflexMonitoringDesktop.open()`.
2. URL built: `accflex-monitoring://open?access_token=…&refresh_token=…&id_token=…&miscApiUrl=…&logisticsApiUrl=…&analyticsApiUrl=…&language=ar-EG&userId=…&tenantId=…&expiresAt=…`.
3. Browser routes to `/monitoring-desktop-launcher` which runs the same blur-event detection as `designer-launcher`; if the protocol doesn't resolve within ~2 s, prompts the MSI download from `/assets/installer/monitoring-desktop/AccFlexMonitoringDesktop-{version}.msi`.
4. Windows invokes `AccFlexMonitoringDesktop.exe "<protocol-url>"`. `Program.cs` parses args → `SessionStore.Save(...)` via DPAPI → opens `MainForm`.
5. `MainForm` initializes WebView2, maps `wwwroot\` to `https://desktop.accflex.local/`, pre-injects `window.__ACCFLEX_DESKTOP_CONTEXT__` with tokens/URLs/profile, then navigates to `https://desktop.accflex.local/apps/{companyIdFromProfile}/logistics/monitoring-concrete-products`.
6. Angular OIDC service sees the desktop context, seeds `currentUser`, skips the redirect dance, and the monitoring module boots straight in.
7. Refresh: a timer in `DesktopHost` refreshes access token via IdP using refresh token; updated token re-injected into WebView with `ExecuteScriptAsync`.
8. Re-launch after close: `MainForm` loads `SessionStore`, refreshes silently if online, or enters "fully offline" mode if not. If the refresh token itself is expired, show a dialog asking the user to re-launch from the ERP web.

---

## 4. Offline Print Rendering (definitive)

- **Layout cache**: when online, `ReferenceDataCacheService.syncReportLayouts()` calls `ReportLayoutClient` (already in `logistics.api.client`) to fetch the DevExpress XML for `ApplicationId=12 + ReportName IN ('Delivery Plan Detail Print','DeliveryTrackingReport')` for the current user, storing XML strings in the new Dexie `reportLayouts` table.
- **Data contract capture**: because the server-side DataSource shape is not currently documented on the front end, the first phase of implementation includes a one-time "capture" mode (flag-gated) that logs the exact JSON payload the server uses when rendering each report. That payload becomes the contract `OfflinePrintService` assembles from Dexie going forward.
- **Print hand-off**: `OfflinePrintService.print(reportName, headerId)` reads layout XML + assembles data → `window.chrome.webview.postMessage({ kind:'print', reportName, layoutId, data })`.
- **Render**: `DesktopHost` handler → `ReportRenderer.Render(xml, json)` → `XtraReport.LoadLayoutFromXml` + `JsonDataSource` bound to `Json` data member → `ShowRibbonPreview()` for user preview/print; fallback `ExportToPdf` for silent save.
- **Online path unchanged**: when online and not in desktop host, the existing `log-lgx-print` / `log-report-viewer` flow runs server-side as today. No regression risk for web-only users.

Risk: any server-side report layout that references subqueries, server-evaluated calculated fields, or stored procs will not render offline. Mitigation for v1: confirm both target layouts bind to a flat JSON DataSource; if not, simplify those two layouts server-side (one-time cleanup) before shipping.

---

## 5. Local Storage

Dexie only. All caching (lookups, report layouts, pending mutations) stays in IndexedDB inside WebView2. No SQLite on the .NET side — the host never queries data, only receives JSON via `postMessage`. This preserves the committed offline work untouched.

---

## 6. Milestones (sequential)

1. **.NET skeleton boots** — clone Designer structure, WebView2 loads a local `wwwroot\index.html`. `accflex-monitoring://` protocol registered.
2. **Session + context injection** — DPAPI persistence; `window.__ACCFLEX_DESKTOP_CONTEXT__` injection; Angular OIDC reads it and bootstraps.
3. **Launcher + installer** — `monitoring-desktop-launcher` component + WixSharp MSI that bundles WebView2 Evergreen Bootstrapper + Angular `dist/accflex-erp/` payload.
4. **Layout + data sync** — `reportLayouts` Dexie table, `syncReportLayouts()`, data-contract capture mode.
5. **Offline print bridge** — `OfflinePrintService` → `DesktopHost.PrintReport` → `ReportRenderer`. Both reports print offline.
6. **Sync Now (green path)** — fill the `of(0)` stubs in `offline-sync.service.ts` lines 113 & 118; wire "Sync Now" button in `sync-status-badge`.
7. **Failure UX** — surface SyncFailed records with server error + retry/discard actions.
8. **Token refresh + re-launch flows** — DPAPI silent refresh; expired-refresh-token prompt.
9. **UAT** per §7.

---

## 7. Verification

Run on a clean Windows 10/11 x64 VM with no WebView2 runtime preinstalled.

1. Install `AccFlexMonitoringDesktop-{version}.msi`. Confirm `HKCR\accflex-monitoring` registered; WebView2 runtime installed silently; `%LOCALAPPDATA%\Accflex\MonitoringDesktop` not yet present.
2. In the ERP web, open a monitoring-concrete-products page → click "Open Offline Monitoring" → browser prompts to open `accflex-monitoring://`.
3. App window opens, Angular boots straight into `/apps/{companyId}/logistics/monitoring-concrete-products`. Confirm `session.dat` now present (encrypted).
4. Click **Sync Now** in the sync-status-badge. Confirm in DevTools (WebView2 exposes `Ctrl+Shift+I`) → Application → IndexedDB: all 9 reference-data caches populated and 2 `reportLayouts` entries present.
5. **Disconnect network.** Confirm `NetworkStatusService` emits offline.
6. Create a new delivery schedule + add trucks through the existing forms. Records appear in `offline-delivery-schedule-popup` with status Pending.
7. Trigger "Delivery Plan Detail Print" from `add-trucks-pupop` → XtraReports ribbon preview opens in-process → print/export to PDF succeeds.
8. Trigger "DeliveryTrackingReport" from `delivery-tracking` → same.
9. Close app, reopen (simulate multi-day offline). `SessionStore` loads, data and layouts still present, both prints still work.
10. Reconnect network. Click Sync Now. Confirm pending records push to server (cross-check via web ERP), local copies vanish, badge clears.
11. Conflict test: pre-create a duplicate on server, push from desktop → 409 → record stays SyncFailed with server error visible; "discard local" removes it.
12. Token-expiry test: set a short refresh-token lifetime on a test IdP, let it expire offline, reopen → confirm "re-launch via web" prompt.
13. Coexistence test: install both `AccFlexDesigner` and `AccFlexMonitoringDesktop` MSIs. Confirm both `HKCR\accflex-designer` and `HKCR\accflex-monitoring` present and independently launchable.

---

## 8. Risks & Open Questions

| # | Item | Mitigation |
|---|---|---|
| 1 | Generated API client method names for `createDeliverySchedule` / `addTruck` not verified | Confirm at implementation time; if missing, backend team to add commands before §6 step 6 |
| 2 | Report layouts may reference server-only data sources (SPs, subqueries) | Implementation step §6.4 explicitly audits the two layouts; one-time server-side layout cleanup if needed |
| 3 | Angular `dist/accflex-erp/` bundle size inflates MSI | Accepted tradeoff per user direction ("same technique as advanced-designer" → don't carve a new Angular project). Monitor MSI size; revisit with a slim project only if >100 MB |
| 4 | DPAPI refresh-token storage diverges from Designer's in-memory-only policy | Required for multi-day offline. Documented in the plan; security review recommended before release |
| 5 | WebView2 runtime bundler adds ~120 MB to the MSI | Accepted per user choice (bundle). Split installer into web + offline variants later if size becomes a distribution issue |
| 6 | Per-user report layouts not present in Dexie for a new user on first launch | First-run flow requires an online Sync Now before offline use; "Sync Now" button is prominent in `sync-status-badge` |
| 7 | Desktop host detection in Angular could produce different behavior than web-only users | Gate behind `DesktopHostService.isDesktopHost`; unit-test both branches of each modified component |

---

## Critical File Paths (for quick reference)

### Pattern sources (read-only, clone structure)
- `D:\Hadaf Work TFS\AccFlex_ERP\AccflexCloudDesigner\AccFlexDesigner\Program.cs`
- `D:\Hadaf Work TFS\AccFlex_ERP\AccflexCloudDesigner\AccFlexDesigner\ReportHelper.cs`
- `D:\Hadaf Work TFS\AccFlex_ERP\AccflexCloudDesigner\AccFlexDesigner\ApiClientHelper.cs`
- `D:\Hadaf Work TFS\AccFlex_ERP\AccflexCloudDesigner\AccFlexDesigner.Installer\Program.cs`
- `D:\Hadaf Work TFS\AccFlex_Cloud_Front\accflex-erp\src\app\shared\services\accflex-advanced-designer\accflex-advanced-designer.service.ts`
- `D:\Hadaf Work TFS\AccFlex_Cloud_Front\accflex-erp\src\app\shared\components\designer-launcher\*`

### To be created (new)
- `D:\Hadaf Work TFS\AccFlex_ERP\AccFlexMonitoringDesktop\` (entire solution)
- `accflex-erp\src\app\shared\services\accflex-monitoring-desktop\accflex-monitoring-desktop.service.ts`
- `accflex-erp\src\app\shared\components\monitoring-desktop-launcher\*`
- `accflex-erp\src\app\core\services\desktop-host\desktop-host.service.ts`
- `accflex-erp\projects\logistics\src\app\modules\monitoring-concrete-products\offline\offline-print.service.ts`

### To be modified
- `accflex-erp\src\app\core\services\accflex-oidc.service.ts`
- `accflex-erp\src\app\shared\services\utils.ts`
- `accflex-erp\src\app\app-routing.module.ts`
- `accflex-erp\src\assets\configs\config.json`
- `accflex-erp\projects\logistics\src\app\modules\monitoring-concrete-products\offline\offline-db.ts`
- `accflex-erp\projects\logistics\src\app\modules\monitoring-concrete-products\offline\offline-db.service.ts`
- `accflex-erp\projects\logistics\src\app\modules\monitoring-concrete-products\offline\reference-data-cache.service.ts`
- `accflex-erp\projects\logistics\src\app\modules\monitoring-concrete-products\offline\offline-sync.service.ts` **(lines 113 & 118)**
- `accflex-erp\projects\logistics\src\app\modules\monitoring-concrete-products\componants\add-trucks-pupop\add-trucks-pupop.component.ts`
- `accflex-erp\projects\logistics\src\app\modules\monitoring-concrete-products\componants\delivery-tracking\delivery-tracking.component.ts` **(line 432)**
- `accflex-erp\projects\logistics\src\app\modules\monitoring-concrete-products\offline\components\sync-status-badge\*`