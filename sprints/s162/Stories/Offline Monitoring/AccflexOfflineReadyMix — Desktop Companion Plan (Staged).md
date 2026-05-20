
## Context

The previous plan (WebView2 + Angular reuse) is scrapped. The PM now wants a **pure WinForms desktop app** — no Angular inside, no WebView2 host. The app replicates only the most critical slice of `monitoring-concrete-products` (delivery tracking + add trucks) with offline-first SQLite storage, local credential cache for offline login, and XtraReports for the two prints.

The desktop app is a new OIDC client (separate from the web) with its own refresh-token lifecycle, decoupled from the browser session. Auth handshake still uses the `accflex-offlinereadymix://` deep-link protocol to seed first-time login. After that, the desktop refreshes on its own.

Functional reference (read-only, don't modify): `projects\logistics\src\app\modules\monitoring-concrete-products` — we mirror its API contracts and state machine, nothing more.

Install/distribution/deep-link pattern is cloned verbatim from `D:\Hadaf Work TFS\AccFlex_ERP\AccflexCloudDesigner`.

**User-confirmed decisions (do not re-ask):**
- Form 1 scope = delivery tracking grid + add-trucks popup only (pumps/redirects/recipes deferred)
- Local DB = SQLite + Dapper (file at `%LOCALAPPDATA%\Accflex\OfflineReadyMix\local.db`)
- Offline login = PBKDF2 hash stored locally; plaintext never persisted
- Reports = XtraReports with cached XML layouts + JSON data from SQLite

---

## Solution Layout

**Root:** `D:\Hadaf Work TFS\AccFlex_ERP\AccflexOfflineReadyMix\` (sibling of `AccflexCloudDesigner\`)

```
AccflexOfflineReadyMix.sln
├── AccflexOfflineReadyMix/                     (WinForms, net462, WinExe, x64)
│   ├── Program.cs                              — protocol arg parse, single-instance guard
│   ├── Forms/
│   │   ├── LoginForm.cs                        — online/offline sign-in gate
│   │   ├── MonitoringForm.cs                   — Form 1: delivery tracking grid + add-trucks
│   │   └── SyncForm.cs                         — Form 2: sync dashboard
│   ├── Auth/
│   │   ├── OidcClient.cs                       — PKCE/refresh against IdS (desktop client)
│   │   ├── SessionStore.cs                     — DPAPI-wrapped tokens on disk
│   │   └── CredentialStore.cs                  — PBKDF2 hash in SQLite
│   ├── Data/
│   │   ├── LocalDb.cs                          — SQLite connection factory + migrations
│   │   ├── Repositories/                       — DeliveryTrackingRepo, TrucksRepo, LookupsRepo, OutboxRepo, LayoutsRepo
│   │   └── Models/                             — POCOs mirroring logistics DTOs (hand-written, minimal)
│   ├── Sync/
│   │   ├── ReferenceSyncService.cs             — pulls lookups (materials, trucks, drivers, stations, BOMs…)
│   │   ├── DeliverySyncService.cs              — pulls delivery schedules + tracking rows
│   │   ├── LayoutSyncService.cs                — pulls XtraReports XML for the 2 reports
│   │   └── OutboxSyncService.cs                — pushes pending add-truck / status-change commands
│   ├── Api/
│   │   ├── ApiClientHelper.cs                  — cloned from Designer; Bearer + refresh
│   │   └── MonitoringApi.cs                    — thin typed wrapper for the 5 endpoints we call
│   ├── Reporting/
│   │   └── ReportRenderer.cs                   — XtraReport.LoadLayoutFromXml + JsonDataSource
│   └── Infrastructure/
│       ├── NetworkStatus.cs                    — ping-based + NetworkChange event
│       └── Logger.cs                           — Serilog to %LOCALAPPDATA%\...\logs\
│
└── AccflexOfflineReadyMix.Installer/           (WixSharp console, net48)
    ├── Program.cs                              — cloned from AccFlexDesigner.Installer
    └── Assets/
```

---

## Auth Design (definitive)

### Identity-Server registration (backend task, stage 1)
Register a **new OIDC client** in the identity server:

| Field                  | Value                                                             |
| ---------------------- | ----------------------------------------------------------------- |
| ClientId               | `accflex-offlinereadymix`                                         |
| Grant types            | `authorization_code` (PKCE) + `refresh_token`                     |
| Redirect URI           | `http://127.0.0.1:<ephemeral>/callback` (loopback)                |
| Allowed scopes         | `openid profile offline_access logisticsapi miscapi analyticsapi` |
| Refresh token lifetime | 30 days sliding (field users, multi-day offline)                  |
| Access token lifetime  | 1 hour                                                            |
| Require consent        | false                                                             |

This decouples desktop from web: revoking web session doesn't break desktop, and vice versa.

### First-launch handshake (deep link)
1. ERP web has a new button "Open Offline Ready-Mix" (skill `monitoring-concrete-products` page).
2. Button opens `accflex-offlinereadymix://open?tenantId=…&userHint=…&miscApiUrl=…&logisticsApiUrl=…&analyticsApiUrl=…` — **no tokens in the URL**. Web tokens aren't reused.
3. Desktop app launches, parses args, stores API URLs + tenant in SessionStore, then opens `LoginForm`.
4. LoginForm presents username/password. On submit:
   - If online → desktop runs PKCE flow against IdS as `accflex-offlinereadymix` client (hidden loopback browser or direct `/token` ROPC if permitted; PKCE preferred).
   - On success: cache user profile + refresh-token (DPAPI) + PBKDF2 hash of the entered password (for offline login next time).
5. Subsequent launches: LoginForm first checks refresh token → silent refresh online; if offline or refresh failed, fall back to password prompt validated against local PBKDF2 hash.

### Token lifecycle
- Access token: in-memory only.
- Refresh token: DPAPI-encrypted at `%LOCALAPPDATA%\Accflex\OfflineReadyMix\session.dat`.
- Password hash: PBKDF2-SHA256, 100k iterations, per-user salt, stored in SQLite `users` table alongside `userId`, `userName`, `tenantId`.
- Offline unlock: hash match → issue short-lived local "session" (no server token). API calls are blocked offline; only cached reads + queued writes.
- On reconnect: refresh token used silently; if expired, prompt password and exchange via IdS to get a fresh refresh token.

---

## Stages (Milestones)

Each stage is independently validatable. Do not start stage N+1 until stage N passes its exit criteria.

### Stage 1 — Solution skeleton + deep-link + installer shell
**Goal:** app launches from protocol URL, shows an empty `MonitoringForm`, MSI installs cleanly.

- Clone `AccflexCloudDesigner` solution structure into `AccflexOfflineReadyMix\`.
- Create both projects, copy `Program.cs` protocol-arg parsing + single-instance mutex.
- Registry self-registration on first run (dev aid); formal registration done by MSI.
- `Installer\Program.cs` cloned, renamed: product name `Accflex Offline Ready-Mix`, install dir `%ProgramFiles%\Accflex\OfflineReadyMix`, protocol `accflex-offlinereadymix`, new upgrade GUID so it coexists with the Designer MSI.
- Stub empty `MonitoringForm` + `SyncForm` + menu button wiring.

**Exit criteria:** Install MSI on clean Win10 VM → `HKCR\accflex-offlinereadymix` present → pasting `accflex-offlinereadymix://open?tenantId=1` into Run dialog opens the app and the stub form. Also coexists with installed Designer.

---

### Stage 2 — Local SQLite + migrations + session store
**Goal:** on first launch the app creates its DB and persists/restores session.

- `LocalDb.cs` — opens/creates `%LOCALAPPDATA%\Accflex\OfflineReadyMix\local.db` (SQLite), runs embedded schema migrations on startup (versioned `__schema_version` table).
- Initial schema (v1):
  - `users(UserId, UserName, TenantId, PasswordHash, Salt, CreatedAt, LastLoginAt)`
  - `reference_materials`, `reference_trucks`, `reference_drivers`, `reference_stations`, `reference_bom`, `reference_inventory_locations` (mirror logistics lookup DTOs; schema snapshot in next milestone)
  - `delivery_schedules(Id JSON…)`, `delivery_plan_details(Id, …, Status, LocalStatus, ServerTimestamp)`
  - `trucks(Id, DeliveryPlanDetailId, Payload JSON, SyncStatus)` — outbox table
  - `outbox(Id, Kind, Payload JSON, CreatedAt, SyncStatus, Attempts, LastError)`
  - `report_layouts(ReportName, UserId, TenantId, LayoutXml, UpdatedAt)`
  - `app_settings(Key, Value)`
- `SessionStore.cs` — DPAPI read/write for refresh token + API URLs + user context.

**Exit criteria:** app creates `local.db` on first launch; closing/reopening restores API URLs from session. Inspect DB with DB Browser for SQLite → schema matches; no plaintext tokens on disk (DPAPI-encrypted blob only).

---

### Stage 3 — Identity Server desktop client + LoginForm
**Goal:** user can sign in (online PKCE) and the desktop app obtains its own refresh token.

Backend task (must be done before this stage can validate):
- Register `accflex-offlinereadymix` OIDC client per table above.

Frontend/desktop:
- `OidcClient.cs` — PKCE authorization-code flow using loopback listener (e.g. `HttpListener` on `127.0.0.1:<random>`), fallback to ROPC if IdS config allows.
- `LoginForm.cs` — username/password inputs; on submit → `OidcClient.SignInAsync()`.
- On success: save refresh token to `SessionStore`, save PBKDF2 hash of entered password + user profile to `users` table, transition to `MonitoringForm`.
- `NetworkStatus.cs` integrated — login button text switches to "Sign in (offline)" when offline.

**Exit criteria:** online login succeeds → refresh token in DPAPI blob, user row in SQLite with hash (plaintext absent). Kill app, disconnect network, relaunch → login with same password succeeds via PBKDF2 check; wrong password rejected.

---

### Stage 4 — Reference data sync (read-only)
**Goal:** the app can pull and cache all lookups needed by Form 1, working only with cached data after sync.

- `ApiClientHelper.cs` cloned from Designer, extended with refresh-token flow (auto-refresh on 401).
- `MonitoringApi.cs` typed calls:
  - `GET /api/Material/all-with-organization`
  - `GET /api/Station/all`
  - `GET /api/BillOfMaterial/raw-material-to-bom-list`
  - `GET /api/Driver/all`, `/api/Truck/all`, `/api/InventoryLocation/all`
  - `GET /odata/MonitoringConcreteProducts/DeliveryTracking?$filter=…` (paged)
  - (exact method names verified at implementation time against `logistics.api.client` signatures)
- `ReferenceSyncService.cs` + `DeliverySyncService.cs` — upsert into SQLite by primary key; `UpdatedAt` tracked; full replace on first sync, delta (by date filter) on subsequent.
- `SyncForm.cs` v1: "Sync reference data" button + per-table row with last-sync timestamp + record count.

**Exit criteria:** click Sync → all reference tables populated, delivery tracking rows for last 7 days cached. Disconnect network → data still visible in SQLite. No API calls attempted while offline.

---

### Stage 5 — MonitoringForm: delivery tracking grid (read-only)
**Goal:** Form 1 shows the cached delivery tracking grid with filters.

- DevExpress XtraGrid (already licensed via Designer) bound to `delivery_plan_details` joined with lookups.
- Filter panel replicates the Angular search form minimally: date range, material, station, truck, status.
- Status column rendered via the existing state machine enum (Not Started / On Way / Delivered / Rejected variants).
- Double-click row → opens details in same form (split panel) — no editing yet.

**Exit criteria:** offline, grid displays cached rows; filters work locally; status values match the Angular component's enum mapping.

---

### Stage 6 — Add Trucks popup + outbox enqueue
**Goal:** user can assign a truck to a schedule while offline; the command is queued.

- `AddTrucksDialog.cs` — replicates the fields used in `add-trucks-pupop.component.ts`:
  - Truck (from cached `reference_trucks`), Driver (cached), Quantity (capped by truck MaxLoad), BoM (cached), Raw material grid with inventory location, Station filter, Discharge times.
- Validation matches Angular rules (all inventory locations required, quantity ≤ MaxLoad).
- On Save (offline-first): serialize `CreateTruckCommand` payload as JSON → insert into `outbox(Kind='AddTruck', Payload, SyncStatus=Pending)` → also apply optimistic update to local `trucks` table so the delivery row reflects new assignment immediately.
- Status-change transitions (Not Started → On Way → Delivered) — same outbox pattern, `Kind='UpdateDeliveryStatus'`.

**Exit criteria:** offline, add a truck → outbox row created → grid reflects change. Close/reopen app → state persists.

---

### Stage 7 — Outbox sync (push) + conflict UX
**Goal:** queued mutations push to server when reconnected; failures surfaced.

- `OutboxSyncService.cs` — iterates `outbox WHERE SyncStatus=Pending ORDER BY CreatedAt`, dispatches by `Kind`:
  - `AddTruck` → `MonitoringApi.AddTruckAsync(payload)`
  - `UpdateDeliveryStatus` → `MonitoringApi.UpdateDeliveryPlanDetailAsync(payload)`
- On 200: mark `Synced`, attach server-assigned IDs back to local rows.
- On 400/409 (concurrency/validation): mark `Failed`, store `LastError`, increment `Attempts`. Expose in `SyncForm` with **Retry** / **Discard** actions.
- SyncForm now has 3 sections: Reference data (pull), Delivery data (pull), Outbox (push).

**Exit criteria:** reconnect → click Sync → pending outbox entries push; verify via web ERP that server row created. Force a conflict (pre-create duplicate) → row appears Failed with server message; Discard removes it, Retry re-queues it.

---

### Stage 8 — XtraReports offline (the 2 prints)
**Goal:** both `Delivery Plan Detail Print` and `DeliveryTrackingReport` render offline.

- `LayoutSyncService.cs` — when online, fetches layout XML from `ReportLayout` client for ApplicationId=12 + the two ReportNames, scoped to current user; upserts into `report_layouts`.
- Document the exact JSON data contract for each report (one-off capture: run web in dev, intercept payload sent to XtraReport server, save as canonical shape).
- `ReportRenderer.cs`:
  - `Render(reportName, keyId)` — loads XML from SQLite, assembles JSON from local tables matching captured shape, `XtraReport.LoadLayoutFromXml` + `JsonDataSource`, `ShowRibbonPreview()` or `ExportToPdf()`.
- Wire print buttons: in `AddTrucksDialog` (after save → print `Delivery Plan Detail Print`) and grid toolbar (`DeliveryTrackingReport`).

**Exit criteria:** offline, click Print on a saved-then-queued delivery → XtraReports preview opens with correct data → Print/Export PDF works. Both reports validated.

---

### Stage 9 — Token refresh + expiry UX + polish
**Goal:** production-ready session handling.

- Background timer refreshes access token 5 min before expiry.
- If refresh token rejected (expired/revoked) while app open: show modal "Session expired, please sign in again" → LoginForm.
- Detect first-run-after-install vs returning user; guide through "Sync now before going offline" if reference tables empty.
- Logging via Serilog; crash dumps to `%LOCALAPPDATA%\…\logs\`.
- MSI: bundle all DevExpress runtime DLLs; verify no external dependency (WebView2 NOT needed here — different from previous plan).

**Exit criteria:** full UAT per verification section below, on clean VM.

---

## Verification (end-to-end on clean Win10/11 x64 VM)

1. Install `AccflexOfflineReadyMix-{version}.msi`. Confirm registry protocol + install dir + coexists with Designer MSI.
2. From ERP web `monitoring-concrete-products` page → click "Open Offline Ready-Mix" → browser prompts `accflex-offlinereadymix://` → app opens to LoginForm.
3. Sign in with valid credentials → `MonitoringForm` opens. Click `SyncForm` → Sync All → reference + delivery + report layouts populate SQLite (inspect with DB Browser).
4. Disconnect network. Close/reopen app → LoginForm accepts same password via PBKDF2 → MonitoringForm loads cached grid.
5. Create a new truck assignment offline → outbox row appears; optimistic update visible in grid. Print `Delivery Plan Detail Print` → XtraReports preview renders offline.
6. Print `DeliveryTrackingReport` from grid toolbar → preview renders.
7. Change status of a delivery to "On Way" → outbox row created.
8. Reconnect. Click Sync → outbox drains, server rows confirmed via web ERP.
9. Conflict test: pre-create a duplicate truck on server → retry outbox → item marked Failed with server error; Discard removes it.
10. Set short refresh-token lifetime on IdS, leave offline past expiry, reopen → re-login prompt appears.
11. Wrong password offline → rejected without server call.

---

## Files to Create/Modify

### New .NET solution (everything under `D:\Hadaf Work TFS\AccFlex_ERP\AccflexOfflineReadyMix\`)
Per "Solution Layout" above. Seed patterns from Designer:
- `AccflexCloudDesigner\AccFlexDesigner\Program.cs` → protocol parsing + single-instance
- `AccflexCloudDesigner\AccFlexDesigner\ApiClientHelper.cs` → Bearer HttpClient (extend with refresh)
- `AccflexCloudDesigner\AccFlexDesigner.Installer\Program.cs` → MSI build script

### Angular side (minimal — just the launcher)
- **Modify** a single place in the existing monitoring-concrete-products shell: add a toolbar button "Open Offline Ready-Mix" that redirects to a simple launcher route.
- **New** `accflex-erp\src\app\shared\services\accflex-offline-readymix\accflex-offline-readymix.service.ts` — builds `accflex-offlinereadymix://open?tenantId=…&userHint=…&miscApiUrl=…&logisticsApiUrl=…&analyticsApiUrl=…` (no tokens — desktop runs its own PKCE).
- **New** `accflex-erp\src\app\shared\components\offline-readymix-launcher\*` — clone of `designer-launcher`, just different installer filename + protocol.
- **Modify** `src\app\app-routing.module.ts` — add `/offline-readymix-launcher` route.
- **Modify** `src\assets\configs\config.json` — add `accflexOfflineReadyMix: { installerVersion: "1.0.0.0" }`.
- **New** `src\assets\installer\offline-readymix\.gitkeep` — CI drop-zone for MSI.

**Crucially, we do NOT modify** the monitoring-concrete-products module internals (offline sync, Dexie, etc.). The desktop is a standalone replacement for field use; the web keeps its current online-only flow.

### Backend
- Register new OIDC client `accflex-offlinereadymix` in identity server config (handled by backend team before Stage 3 validation).

---

## Open Questions / Risks (tracked, not blocking)

| # | Item | Mitigation |
|---|---|---|
| 1 | Exact method signatures on `MonitoringConcreteProductsClient` (AddTruck, UpdateDeliveryStatus) need confirmation at Stage 6 | Inspect generated C# client in `logistics.api.client` during Stage 4 wiring; adjust API wrappers |
| 2 | Report layout XML may reference server-only data (subqueries, SPs) | Stage 8 starts with layout audit; simplify server-side if needed before offline render works |
| 3 | PKCE loopback flow may be blocked by strict firewalls | Fallback to ROPC grant if IdS allows; document in stage 3 |
| 4 | SQLite native DLL architecture must match `x64` WinExe target | Use `System.Data.SQLite.Core` with correct build; bundle both `x86`/`x64` native DLLs in MSI just in case |
| 5 | Password hash upgrade path if PBKDF2 params change later | Store algorithm version in `users.Salt` prefix; migrate on next successful online login |
| 6 | DPAPI refresh tokens survive OS user profile only — NOT portable across machines | Acceptable for field-device use case; document |