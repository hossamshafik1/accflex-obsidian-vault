# 🧱 Jira Breakdown (Recommended Split)

## 🔹 Story 1: PWA Enablement & Offline Detection

**Description:**  
Enable offline access to Monitoring Concrete Products screen and detect connectivity state.

**Tasks:**

- Configure service worker
- Cache static assets
- Implement online/offline detection
- UI indicator (Offline banner)

**Estimate:**  
👉 3–5 Story Points

---

## 🔹 Story 2: Local Storage for Concrete Products (IndexedDB)

**Description:**  
Implement local database for storing offline-created concrete product records.

**Tasks:**

- Design local schema
- CRUD operations (create/edit/delete local records)
- Add sync status field
- Store client-generated ID

**Estimate:**  
👉 5–8 Story Points

---

## 🔹 Story 3: Offline UI Enhancements (Monitoring Screen)

**Description:**  
Adapt UI to support offline behavior and local records.

**Tasks:**

- Show local records in grid
- Display sync status per row
- Disable actions that require backend
- Prevent linking unsynced rows

**Estimate:**  
👉 5–8 Story Points

---

## 🔹 Story 4: Manual Sync Mechanism (Frontend)

**Description:**  
Allow user to manually sync local records to backend.

**Tasks:**

- Sync button
- Batch upload logic
- Status updates (Syncing, Synced, Failed)
- Retry failed rows

**Estimate:**  
👉 5–8 Story Points

---

## 🔹 Story 5: Backend Sync API (Core)

**Description:**  
Create API to receive and process offline-created records.

**Tasks:**

- Endpoint: `/offline/concrete-products/sync`
- Accept batch payload
- Validate records
- Insert into DB
- Return response per record

**Estimate:**  
👉 8–13 Story Points

---

## 🔹 Story 6: Idempotency & Duplicate Prevention

**Description:**  
Ensure no duplicate records are created during retry.

**Tasks:**

- Use clientGeneratedId
- Add unique constraint/check
- Handle repeated requests safely

**Estimate:**  
👉 3–5 Story Points

---

## 🔹 Story 7: Sync Result Handling & Mapping

**Description:**  
Map local records to server records after sync.

**Tasks:**

- Return serverId per record
- Store mapping locally
- Update UI accordingly

**Estimate:**  
👉 3–5 Story Points

---

## 🔹 Story 8: Sales Order Linkage Integration

**Description:**  
Allow only synced records to be linked to sales order.

**Tasks:**

- Filter selectable rows
- Use serverId for linkage
- Prevent linking unsynced rows

**Estimate:**  
👉 5–8 Story Points

---

# 📊 Rough Total Estimation

|Area|Story Points|
|---|---|
|Frontend (PWA + UI + Sync)|18–29 SP|
|Backend (API + validation + idempotency)|11–18 SP|
|**Total**|**29–47 SP**|

👉 Roughly:

- **2–3 sprints** for a small team
- **1–2 sprints** for a strong team

---

# ⚠️ Key Risks (Call this out in refinement)

- IndexedDB complexity & browser differences
- Sync failure edge cases
- Duplicate handling correctness
- User confusion about sync state
- Large batch sync performance

---

# 🧠 Product Owner Insight (Important)

You made a **very strong scoping decision**:

- limiting to **create-only offline**
- deferring cached sales order usage

This reduces complexity by **~40–60%** and makes the feature:

- faster to deliver
- safer to release
- easier to maintain