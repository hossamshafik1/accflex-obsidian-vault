
## Summary
Phase 1 will introduce an offline-capable operational flow for Monitoring Concrete Products that allows users to create new concrete-product/truck records while disconnected, store them locally, and manually sync them later when connectivity returns. The feature will not reuse the current sales-order-driven creation path as-is; instead, it will add a parallel offline/manual record lifecycle that becomes eligible for sales-order linkage only after successful sync.

This plan keeps scope intentionally narrow:
- Offline support is create-only for new records
- Local users may edit/delete unsynced local rows only
- Existing online server rows remain read-only/offline-unavailable
- Sales-order linkage is deferred until after sync succeeds
- Conflict resolution and offline sales-order usage remain out of scope for Phase 1

## Key Technical Decisions
- Treat this as two capabilities, not one:
  - PWA enablement for app availability, caching, and connectivity awareness
  - Offline draft + manual sync architecture for business data persistence
- Keep the current online “Add Trucks” flow intact for sales-order-linked operations.
- Introduce a separate offline/manual record type and lifecycle with explicit statuses:
  - `Pending Sync`
  - `Syncing`
  - `Synced`
  - `Sync Failed`
  - `Linked`
- Use client-generated IDs as the cross-layer identity for retry-safe syncing and duplicate prevention.
- Allow only synced records to participate in downstream sales-order linkage.

## Public Changes and Functional Additions
- Frontend:
  - Add offline/manual entry action in Monitoring Concrete Products
  - Add sync status visibility in the opening grid
  - Add manual `Sync` action
  - Add online/offline indicator
  - Add local-only management rules for unsynced rows
- Backend:
  - Add a dedicated sync API for offline-created concrete-product records
  - Add duplicate prevention/idempotency behavior keyed by client-generated ID
  - Return per-record sync results including server-side identity mapping
  - Add linkage support that accepts only synced server-backed rows
- Data contract level:
  - New offline-sync payload model for manual concrete-product records
  - New sync-result contract containing local ID, sync status, server ID, and validation error when applicable

## Development Ticket Breakdown
1. **T1: PWA Foundation and Connectivity Awareness**
   - Purpose: Enable PWA support for the logistics app and expose reliable online/offline state to the Monitoring screen.
   - Outcome: App shell remains available in low-signal scenarios; users see connection state clearly.
   - Estimate: `3-5 SP`
   - Rough time: `3-5 working days`

2. **T2: Offline Local Record Model and Storage**
   - Purpose: Define the local offline record structure and persist offline-created monitoring records on device.
   - Outcome: Users can create, edit, view, and delete unsynced local records safely.
   - Estimate: `5-8 SP`
   - Rough time: `4-6 working days`

3. **T3: Monitoring Screen Offline UX**
   - Purpose: Extend Monitoring Concrete Products UI to show local rows, sync statuses, and offline-safe actions.
   - Outcome: One combined user experience for online rows plus local pending rows, with clear state boundaries.
   - Estimate: `5-8 SP`
   - Rough time: `4-6 working days`

4. **T4: Manual Sync Workflow on Frontend**
   - Purpose: Add user-triggered sync flow for pending/failed rows and reflect row-level results in the UI.
   - Outcome: Users can manually retry and monitor synchronization progress.
   - Estimate: `5-8 SP`
   - Rough time: `4-6 working days`

5. **T5: Backend Offline Sync API**
   - Purpose: Create a dedicated backend endpoint and processing flow for offline-created records.
   - Outcome: Accepted records are persisted server-side and returned with row-level outcomes.
   - Estimate: `8-13 SP`
   - Rough time: `6-10 working days`

6. **T6: Idempotency and Duplicate Prevention**
   - Purpose: Ensure repeated sync attempts do not create duplicate records.
   - Outcome: Retry-safe sync behavior across network retries and user re-submission.
   - Estimate: `3-5 SP`
   - Rough time: `2-4 working days`

7. **T7: Sync Mapping and Post-Sync State Management**
   - Purpose: Persist local-to-server identity mapping and transition rows into synced state correctly.
   - Outcome: Synced rows become server-backed and ready for downstream linkage.
   - Estimate: `3-5 SP`
   - Rough time: `2-4 working days`

8. **T8: Sales Order Linkage Eligibility**
   - Purpose: Allow only synced records to be selected and linked into the sales-order cycle.
   - Outcome: Business integrity is preserved by preventing unsynced/local-only rows from being linked.
   - Estimate: `5-8 SP`
   - Rough time: `4-6 working days`

## Overall Estimation
- Frontend total: `18-29 SP`
- Backend total: `11-18 SP`
- End-to-end total: `29-47 SP`

Rough delivery outlook:
- Strong focused team: `1-2 sprints`
- Small or shared-capacity team: `2-3 sprints`

## Test and Validation Scope
- Offline create works with no connection and survives page refresh/app reopen
- Unsynced local records can be edited/deleted locally only
- Sync is blocked while still offline
- Sync returns per-row success/failure and preserves failed rows for retry
- Retry does not create duplicate server records
- Synced rows receive server mapping and become linkable
- Unsynced rows remain ineligible for sales-order linkage
- Existing online sales-order-based truck flow remains unchanged

## Assumptions and Defaults
- Phase 1 covers offline creation of new monitoring concrete-product records only.
- The offline-created records follow a separate business path from the current sales-order-driven `Add Truck` flow until sync is completed.
- Sales-order linkage happens only after sync and uses server-backed identity, not local-only identity.
- The plan intentionally avoids implementation-level decisions such as exact storage library, DTO fields, database schema, or endpoint naming details; those will be specified in the implementation design phase.
