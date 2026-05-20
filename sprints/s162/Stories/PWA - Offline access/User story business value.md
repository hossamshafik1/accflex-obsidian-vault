**Title:** Offline creation and manual sync of concrete product records

**User Story:**  
As a monitoring user, I want to create concrete product records while the system is offline and manually sync them when internet connectivity is restored, so that production data is not lost and can later be linked to a sales order.

**Description:**  
When internet connectivity is unavailable, the system shall allow users to enter concrete product data in the Monitoring Concrete Products screen. These records shall be stored locally on the user device. Once internet connectivity is restored, the user can manually trigger a sync process to upload the records to the backend. Successfully synced records become available for selection and linkage to a sales order.

**Scope (Phase 1):**

- Offline mode supports **creating new records only**
    
- No offline editing or deletion of server-side data
    
- No offline usage of sales order data
    
- No conflict resolution required
    

**Acceptance Criteria:**

1. **Offline Data Entry**
    
    - Given the user is offline
    - When the user creates a concrete product record
    - Then the system stores it locally
    - And marks it as `Pending Sync`
2. **Local Record Management**
    - User can view, edit, and delete **unsynced local records only**
3. **Manual Sync**
    - Given internet is available
    - When user clicks "Sync"
    - Then all pending records are sent to backend
    - And each record receives a sync status
4. **Sync Status Handling**
    
    - Records must have statuses:
        - Pending Sync
        - Syncing
        - Synced
        - Sync Failed
5. **Duplicate Prevention**
    
    - System must prevent duplicate records if sync is retried
        
6. **Validation Handling**
    
    - Invalid records must return error messages and remain in `Sync Failed`
        
7. **Sales Order Linkage**
    
    - Only **synced records** can be selected and linked to a sales order
        

**Technical Notes:**

- Use PWA for offline capability and connectivity detection
    
- Use IndexedDB (or equivalent) for local storage
    
- Sync must be idempotent using client-generated IDs
    
- Backend must return mapping (localId → serverId)
    

**Business Value:**

- Eliminates production data loss during outages
    
- Ensures continuity of operations
    
- Maintains ERP data integrity
    
- Enables accurate sales order linkage