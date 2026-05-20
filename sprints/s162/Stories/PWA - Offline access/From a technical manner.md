## PWA Foundation — Technical Breakdown

### The Problem We're Solving

A normal Angular app is just files served by a web server. When the browser can't reach the server (no internet), the user sees Chrome's dinosaur page — the app is completely dead. PWA foundation solves three things:

1. **Installability** — users can "install" the app to their home screen like a native app
2. **Offline shell** — the app's UI skeleton loads even without internet (APIs still fail, but the page renders)
3. **Connectivity awareness** — the app knows in real-time whether it's online or offline, so later steps can route data to IndexedDB instead of the API

---

### Component-by-Component Explanation

#### 1. `manifest.webmanifest` — The App's Identity Card

```
Browser reads this → "Oh, this website is actually an app"
```

**What it does**: A JSON file that tells the browser metadata about the app — name, icons, theme color, how to display it (standalone = no browser chrome).

**Why we need it**: Without this file, the browser treats the app as a regular website. With it:

- Chrome shows the "Install" button in the address bar
- When installed, the app opens in its own window (no URL bar, no tabs — like a native app)
- The OS shows the correct icon and name on the taskbar/home screen

**It does NOT**: cache anything, handle offline, or run any code. It's purely declarative metadata.

---

#### 2. `ngsw-config.json` — Cache Rules for the Service Worker

```
Angular build reads this → generates ngsw.json → ngsw-worker.js uses it at runtime
```

**What it does**: Defines **what** to cache and **how**. It has two sections:

**`assetGroups`** — Static files (built into the app):
```
┌─────────────────────────────────────────────────┐
│ "app" group (prefetch)                          │
│ → index.html, *.js, *.css, manifest, favicon    │
│ → Downloaded IMMEDIATELY on first visit         │
│ → These are the app shell                       │
├─────────────────────────────────────────────────┤
│ "assets" group (lazy)                           │
│ → /assets/**, images, fonts                     │
│ → Downloaded only when first requested          │
│ → Icons, i18n JSON, DevExtreme CSS etc.         │
└─────────────────────────────────────────────────┘
```

**`dataGroups`** — Dynamic URLs (API calls):

```
┌─────────────────────────────────────────────────┐
│ "api-calls" group (freshness, maxSize: 0)       │
│ → /api/**, *.hadafsolutions.net/**              │
│ → PASS-THROUGH: never cached                    │
│ → SW stays out of the way for API traffic       │
└─────────────────────────────────────────────────┘      
```

**Why `maxSize: 0` for APIs?**: Our offline strategy is IndexedDB (Dexie) at the application layer, not SW-level caching. If the SW cached API responses, we'd get stale data and conflicts with the sync logic we'll build in Steps 2-6. So we explicitly tell the SW: "don't touch API calls."

**Why we need it**: Without cache rules, the service worker wouldn't know what to cache. The `ngsw-config.json` is a build-time input — Angular's build process reads it and generates the actual runtime config (`ngsw.json`) with hashed file URLs.

---

#### 3. `ngsw-worker.js` — The Service Worker Itself (Angular's, not ours)

```
Browser ──request──→ Service Worker ──cache hit?──→ return cached file
                                    ──cache miss?──→ forward to network
```

**What it does**: A script that sits between the browser and the network. It's a **proxy** — every HTTP request the browser makes goes through it first. Based on the rules from `ngsw-config.json`, it either:

- Returns a cached response (offline or faster)
- Forwards to the network (API calls, uncached assets)

**Why we need it**: This is what makes offline shell loading work. When the user opens the app without internet:

```
1. Browser requests index.html → SW returns cached copy ✓
2. Browser requests main.js → SW returns cached copy ✓  
3. Browser requests styles.css → SW returns cached copy ✓
4. App renders! (empty, but the UI skeleton is there)
5. App tries API call → SW passes through → network fails → app handles gracefully
```

**We don't write this file** — it comes from `@angular/service-worker` package. Angular's build copies it to dist.

---

#### 4. `provideServiceWorker()` in `app.config.ts` — Registration Logic

```
provideServiceWorker('ngsw-worker.js', {
  enabled: environment.production,          // only in prod
  registrationStrategy: 'registerWhenStable:30000'  // don't compete with app boot
})
```

**What it does**: Tells Angular **when and how** to register the service worker with the browser.

**`enabled: environment.production`** — In dev mode (`ng serve`), the SW is NOT registered. Why?

- Dev server uses live reload — a SW caching files would break hot module replacement
- You'd get stale cached files after code changes
- SW requires HTTPS (except localhost, but it's still confusing in dev)

**`registerWhenStable:30000`** — Wait for Angular to finish bootstrapping (all initial HTTP calls done, no pending timers), OR 30 seconds max, before registering the SW. Why?

- SW registration triggers cache downloads (prefetch group)
- If this happens during app startup, it competes for bandwidth with the app's actual API calls
- Result: slower initial load. So we wait until the app is "stable" first.

---

#### 5. `NetworkStatusService` — Connectivity Detection

```
readonly isOnline$ = new BehaviorSubject<boolean>(navigator.onLine);
// + window 'online'/'offline' event listeners
```

**What it does**: Exposes a single `BehaviorSubject<boolean>` that is:

- `true` when the browser has internet
- `false` when it doesn't
- Updates in real-time when connectivity changes

**Why `BehaviorSubject` (not `Subject` or `ReplaySubject`)?**:

- `BehaviorSubject` always has a current value. When a component subscribes, it immediately gets the current status — no waiting for the next event.

**Why `NgZone.run()`?**:

```
private readonly onlineHandler = () => this.ngZone.run(() => this.isOnline$.next(true));
```

The browser's `online`/`offline` events fire **outside** Angular's zone. Without `NgZone.run()`:

- The `BehaviorSubject` value updates ✓
- But Angular **doesn't know** the value changed → templates using `isOnline$ | async` don't re-render ✗

`NgZone.run()` tells Angular: "something happened, run change detection."

**Why we need it**: This is the foundation for all offline logic in Steps 2-6. The sync service will check `isOnline$` before attempting to sync. The offline form will check it to decide whether to save locally or call the API. The UI will show an offline indicator based on it.

---

#### 6. `PwaUpdateService` — App Version Management

```
Every 6 hours: "Is there a new version on the server?"  → Yes: "A new version is available. Reload to update?"  → No: do nothing
```

**What it does**: Handles the **update lifecycle** of the cached app. Without this, once the SW caches v1 of the app, the user could be stuck on v1 forever (the SW happily serves cached files without checking for updates).

**The flow**:

```
1. App stabilizes after boot2. Every 6 hours: SwUpdate.checkForUpdate()
2. Angular SW compares cached ngsw.json hash vs server ngsw.json hash
3. If different → VERSION_READY event fires
4. PwaUpdateService shows confirm() dialog6. User clicks OK → page reloads → SW activates new version
```

**Why `SwUpdate.isEnabled` guard?**:

```
if (!this.swUpdate.isEnabled) return;
```

In dev mode, no SW is registered → `SwUpdate` methods would throw. This guard makes the service a safe no-op in development.

**Why we need it**: Without it, field workers could run outdated app versions for days/weeks. When you deploy a bug fix or a new feature, this is how they get it — the next time they open the app and have connectivity, they'll be prompted to update.

---

### How They All Fit Together

```
┌────────────────────────────────────────────────────────┐
│                    BROWSER                             │
│                                                        │
│  ┌─────────────┐    ┌──────────────────────┐           │
│  │ manifest    │    │ ngsw-worker.js       │           │
│  │ .webmanifest│    │ (Service Worker)     │           │
│  │             │    │                      │           │
│  │ "This is    │    │ Intercepts requests  │           │
│  │  an app"    │    │ Serves from cache    │           │
│  └─────────────┘    │ Passes API through   │           │
│        ↓            └──────────┬───────────┘           │
│   Install prompt               │                       │
│                                │                       │
│  ┌─────────────────────────────┼──────────────────┐    │
│  │              ANGULAR APP    │                  │    │
│  │                             │                  │    │
│  │  ┌──────────────────┐  ┌────┴─────────────┐    │    │
│  │  │ NetworkStatus    │  │ PwaUpdate        │    │    │
│  │  │ Service          │  │ Service          │    │    │
│  │  │                  │  │                  │    │    │
│  │  │ isOnline$ ═══════╪══╪═→ Steps 2-6      │    │    │
│  │  │ (true/false)     │  │ "New version?"   │    │    │
│  │  └──────────────────┘  └──────────────────┘    │    │
│  │                                                │    │
│  └────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────┘
```

The manifest makes it installable.
The service worker caches the shell. 
`NetworkStatusService` tells the app layer whether it's online. 
`PwaUpdateService` keeps the cached app up-to-date. 

Together they form the foundation that Steps 2-6 build on — the offline form, IndexedDB storage, and sync logic all depend on this infrastructure being in place.