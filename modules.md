# FlexPOS Phase 1 — Task 2: Module Boundaries & API Contracts (Windows Offline‑First)

This document defines **module boundaries**, **data ownership**, **inter-module dependencies**, and **high-level API contracts** for FlexPOS (offline-first POS SaaS for Pakistani SMBs, Windows client + cloud).

## Design rules (apply to all modules)

- **Single responsibility**: each module owns a distinct domain.
- **Ownership = write authority**: only the owning module can create/update/delete its domain data. Other modules must use the owner’s contract.
- **Read models are allowed**: modules may keep denormalized, read-only projections for performance/offline UX.
- **No implicit coupling**: cross-module interactions occur through explicit contracts (calls) and/or domain events.
- **Prefer one-way dependencies**: domain modules depend on foundational services (Identity, Audit, Sync, Backup) but not the reverse.
- **Upgrade-safe migrations**: modules must version their persisted data and sync payloads; backward-compatible readers where feasible.

---

## 1) Module overview table (12+ core modules)

| # | Module | Primary responsibility | Runs where | Owns data (examples) | Depends on (typical) |
|---:|---|---|---|---|---|
| 1 | **Identity & Access (IAM)** | Users, roles, permissions, sessions | Local + Cloud | Users, roles, permission grants, device enrollment | (none) / platform |
| 2 | **Business Setup** | Tenant/shop setup, tax & invoice settings, fiscal configs | Local + Cloud | Business profile, branches, counters, tax profiles, receipt templates | IAM, Audit, Sync |
| 3 | **Product & Inventory** | Catalog + stock state, adjustments, transfers, batches/expiry | Local + Cloud | Products, SKUs, categories, stock ledger, warehouses | Business Setup, Audit, Sync |
| 4 | **Sales & POS** | Basket → sale finalization, pricing, discounts, receipts, returns | Local + Cloud | Sales invoices/receipts, payments, returns, POS sessions | Business Setup, Product & Inventory, Customers, IAM, Audit, Sync |
| 5 | **Customers** | Customer records, balances, credit limits, segments | Local + Cloud | Customers, contact info, customer credit profile | Business Setup, Audit, Sync |
| 6 | **Suppliers & Purchases** | Supplier records, purchase orders, goods receipt, supplier invoices | Local + Cloud | Suppliers, POs, GRNs, supplier invoices | Business Setup, Product & Inventory, Accounting, IAM, Audit, Sync |
| 7 | **Accounting & Ledgers** | Chart of accounts, journal entries, receivables/payables, cash/bank | Local + Cloud | Accounts, journals, ledger postings, AR/AP balances | Business Setup, IAM, Audit, Sync |
| 8 | **Employees, Attendance & Payroll** | Staff, shifts, attendance, payroll runs | Local + Cloud | Employees, attendance logs, payroll periods, payslips | Business Setup, IAM, Audit, Sync |
| 9 | **QR Code & Scanning** | Barcode/QR generation + scanning workflows | Local | Scan sessions, device scanner config (minimal) | Product & Inventory, Sales & POS |
| 10 | **Reports** | Read-only analytics: sales, inventory, tax, payroll, AR/AP | Local + Cloud | Report definitions + cached aggregates (read-only) | All domain modules (read-only), Audit |
| 11 | **Backup, Restore & Data Safety** | Local backups, restore, integrity checks, retention | Local | Backup manifests, encryption keys (device-scoped) | IAM, Audit |
| 12 | **Sync & Replication Gateway** | Offline-first sync: change capture, conflict resolution, upload/download | Local + Cloud | Sync state, replication cursor, outbox/inbox, conflict records | IAM, Audit; domain modules provide adapters |
| 13 | **Subscription, Licensing & Expiry** | Plan enforcement, device activation, grace periods, feature flags | Local + Cloud | Subscriptions, licenses, entitlements, expiry/grace policies | IAM, Business Setup, Sync, Audit |
| 14 | **SaaS Admin Dashboard** | Multi-tenant admin: onboarding, support tools, fraud/risk, plan mgmt | Cloud | Tenants, subscriptions admin actions, support tickets metadata | IAM, Subscription, Audit, Reports (cloud) |

Notes:
- **Domain modules**: Business Setup, Product & Inventory, Sales & POS, Customers, Suppliers & Purchases, Accounting, Employees & Payroll.
- **Foundational/platform modules**: IAM, Audit (as a contract), Sync, Backup/Restore, Subscription/Licensing.
- **UI shells are not modules** here; modules are domain/service boundaries.

---

## 2) Detailed module responsibilities (scope, entities, operations, ownership)

For each module:
- **Owns** = authoritative source for writes.
- **References** = stores identifiers and optional cached read models.

### 2.1 Identity & Access (IAM)

**Responsibilities / scope**
- Authentication (local sign-in + cloud sign-in), session management.
- Authorization: roles, permissions, policies (e.g., cashier vs manager).
- Device enrollment/registration (Windows device as a POS terminal).

**Core domain entities (conceptual)**
- User, Role, Permission, Policy, Session, Device, DeviceEnrollment

**Key operations**
- Create/disable user, assign role(s).
- Issue/refresh/revoke session tokens.
- Check permission for an operation (policy evaluation).

**Data ownership**
- Owns: users/roles/permissions, sessions, device enrollment.
- References: business/branch identifiers for scoping.

**Dependencies**
- None (base). Uses OS crypto/secure storage.

**External integrations**
- Optional: SMS/email OTP provider (cloud), SSO later.

---

### 2.2 Business Setup Module

**Responsibilities / scope**
- Tenant setup: business profile, branches/outlets, counters, tax settings.
- Invoice numbering rules, receipt templates, currency/locale/timezone.
- “System settings” that other modules must interpret but not modify.

**Core domain entities**
- Business, Branch, Counter, TaxProfile, InvoiceSequence, ReceiptTemplate, Settings

**Key operations**
- Initialize business, add branch/counter.
- Update tax and invoice settings.
- Publish immutable “effective settings” snapshots used by POS transactions.

**Data ownership**
- Owns: business identity/config and operational settings.
- References: subscription entitlements (read-only) for feature gating.

**Dependencies**
- IAM (permissions), Audit, Sync.

**External integrations**
- None required (optional cloud onboarding).

---

### 2.3 Product & Inventory Management

**Responsibilities / scope**
- Product master data (SKU, barcode, pricing metadata, variants).
- Inventory state via **stock ledger** (adjustments, receipts, issues, transfers).
- Batch/expiry (important for pharmacy/grocery), low-stock thresholds.

**Core domain entities**
- Product, Category, Brand, PriceRule (metadata), Warehouse/Store, StockItem, StockLedgerEntry, Batch, ReorderRule

**Key operations**
- Create/update product, assign barcode/QR.
- Receive stock (from purchases/adjustments), transfer stock, adjust stock.
- Provide availability/pricing inputs to Sales & POS (read-only views).

**Data ownership**
- Owns: product catalog + stock ledger.
- References: suppliers/customers only by ID; does not store supplier/customer details.

**Dependencies**
- Business Setup (branch/warehouse context), IAM (who can adjust), Audit, Sync.

**External integrations**
- Optional: barcode label printers.

---

### 2.4 Sales & POS Module

**Responsibilities / scope**
- POS session lifecycle: opening/closing cash drawer, shift handover.
- Basket building, discounts, taxes, finalization to **Sale**.
- Returns/refunds and sale voiding (policy-based).
- Receipt printing.

**Core domain entities**
- POSSession, Basket, BasketLine, Sale, SaleLine, Payment, Return, Discount, Receipt

**Key operations**
- Start/close POS session.
- Add/remove items, compute totals.
- Finalize sale (atomic): create sale, capture payment(s), emit inventory decrement intent.
- Process return (atomic): create return, emit inventory increment intent.

**Data ownership**
- Owns: sales/returns/payments and POS session records.
- References: product IDs (and cached snapshots of price/name), customer ID, tax profile snapshot.

**Dependencies**
- Business Setup (tax/invoice rules), Product & Inventory (availability + post-sale stock movement), Customers (optional), IAM, Audit, Sync, Subscription (feature enforcement).

**External integrations**
- Receipt printers, cash drawers, payment providers (future), FBR integration (future/optional).

---

### 2.5 Customers Module

**Responsibilities / scope**
- Customer profiles, credit settings, contact details.
- Customer “balance view” is derived from Accounting but can be cached.

**Core domain entities**
- Customer, ContactMethod, Segment, CreditProfile

**Key operations**
- Create/update customer.
- Set credit limits / allowed payment terms.
- Merge/dedupe customers.

**Data ownership**
- Owns: customer profiles.
- References: accounting balance (read-only), sales history (read-only).

**Dependencies**
- Business Setup, IAM, Audit, Sync.

**External integrations**
- Optional: SMS marketing provider.

---

### 2.6 Suppliers & Purchase Module

**Responsibilities / scope**
- Supplier profiles (lightweight) and purchasing lifecycle.
- Purchase orders, goods receipt notes, supplier invoices.
- Emits inventory receipt intents and accounting posting intents.

**Core domain entities**
- Supplier, PurchaseOrder, PurchaseOrderLine, GoodsReceipt (GRN), SupplierInvoice, PurchaseReturn

**Key operations**
- Create PO, receive goods (GRN).
- Post supplier invoice (creates payable intent).
- Purchase returns to supplier.

**Data ownership**
- Owns: purchasing documents + supplier profiles (if supplier profiles are kept separate from “contacts”).
- References: product IDs, warehouses, accounting account IDs.

**Dependencies**
- Business Setup, Product & Inventory, Accounting & Ledgers, IAM, Audit, Sync.

**External integrations**
- Optional: supplier EDI/email export.

---

### 2.7 Accounting & Ledgers

**Responsibilities / scope**
- Financial source of truth: accounts, journal entries, postings, AR/AP.
- Posting rules for sales/purchases/payroll (owned by Accounting).
- Cash management and end-of-day reconciliation (can be shared with POS but owned here).

**Core domain entities**
- Account, ChartOfAccounts, JournalEntry, LedgerPosting, Voucher, Receivable, Payable, CashDrawerReconciliation

**Key operations**
- Post journal entry (manual or from intents).
- Maintain customer/supplier balances.
- Period close (soft), audit exports.

**Data ownership**
- Owns: chart of accounts, journals, ledger postings, AR/AP.
- References: customer/supplier IDs and sales/purchase document IDs.

**Dependencies**
- Business Setup, IAM, Audit, Sync.

**External integrations**
- Optional: export to accounting software; bank feed (future).

---

### 2.8 Employees, Attendance & Payroll

**Responsibilities / scope**
- Employee profiles, roles (HR roles distinct from IAM roles).
- Attendance capture (clock in/out), shift schedules.
- Payroll calculations and payroll runs.

**Core domain entities**
- Employee, AttendanceEvent, Shift, Timesheet, PayrollRun, Payslip, Deduction, Allowance

**Key operations**
- Create/terminate employee.
- Clock in/out, approve timesheets.
- Run payroll (atomic within payroll run) and emit accounting posting intent.

**Data ownership**
- Owns: HR and payroll records.
- References: IAM user IDs (optional link), accounting account IDs.

**Dependencies**
- Business Setup, IAM, Accounting & Ledgers, Audit, Sync.

**External integrations**
- Optional: biometric devices/time clocks.

---

### 2.9 QR Code & Scanning

**Responsibilities / scope**
- Provide scanning workflows: barcode/QR decode, camera/wedge scanner integration.
- Generate QR/barcodes for products/invoices if needed.
- Keep this module **hardware/UI facing**; it does not own product or sales logic.

**Core domain entities**
- ScanSession, ScannerDeviceProfile, ScanResult

**Key operations**
- Start scan session, decode payload.
- Map scan payload → product lookup request (via Product & Inventory).

**Data ownership**
- Owns: scanner configuration and scan sessions (local only).
- References: product IDs.

**Dependencies**
- Product & Inventory, Sales & POS.

**External integrations**
- Scanner SDKs, Windows camera APIs.

---

### 2.10 Reports Module

**Responsibilities / scope**
- Read-only reporting and analytics.
- Defines report queries and cached aggregates.
- Must not write domain state; only creates report artifacts.

**Core domain entities**
- ReportDefinition, ReportRun, ReportResult, CachedAggregate

**Key operations**
- Generate sales/inventory/profit/tax reports.
- Export (PDF/CSV), schedule runs (cloud).

**Data ownership**
- Owns: report definitions and cached aggregates.
- References: read-only data from all other modules.

**Dependencies**
- Read-only dependencies on domain modules; Audit for tracking report access.

**External integrations**
- PDF rendering, Excel/CSV export libraries.

---

### 2.11 Backup, Restore & Data Safety

**Responsibilities / scope**
- Local backups (full + incremental), restore, integrity verification.
- Encryption at rest for backup artifacts.
- “Safe restore” rules: restore must not violate module versioning.

**Core domain entities**
- BackupJob, BackupArtifact, BackupManifest, RestoreJob, IntegrityCheck

**Key operations**
- Create backup (atomic snapshot per module, coordinated).
- Restore backup (with version checks and migration hooks).
- Verify integrity.

**Data ownership**
- Owns: backup artifacts and manifests.
- Does **not** own business data; it orchestrates module snapshot/restore.

**Dependencies**
- IAM (authorize restore), Audit.

**External integrations**
- Optional: cloud storage providers (later), Windows file system APIs.

---

### 2.12 Sync & Replication Gateway

**Responsibilities / scope**
- Offline-first replication between local store and cloud.
- Change capture (outbox), inbound application (inbox), retries.
- Conflict detection/resolution and visibility.

**Core domain entities**
- SyncCursor, OutboxEvent, InboxEvent, ConflictRecord, ReplicationPolicy

**Key operations**
- Enqueue domain changes (append-only events) for upload.
- Pull remote changes, apply via module adapters.
- Detect conflicts and resolve automatically or via manual workflow.

**Data ownership**
- Owns: sync metadata and conflict records.
- Does not own domain entities; modules own their data and expose adapters.

**Dependencies**
- IAM (identity + tokens), Audit.

**External integrations**
- Cloud API, background scheduler, network status.

---

### 2.13 Subscription, Licensing & Expiry

**Responsibilities / scope**
- Subscription lifecycle (plan, billing status), device activation limits.
- Local enforcement (grace periods, read-only mode on expiry).
- Feature flags/entitlements for modules.

**Core domain entities**
- Subscription, Plan, Entitlement, LicenseKey, DeviceActivation, GracePeriod

**Key operations**
- Validate license / refresh entitlement snapshot.
- Enforce feature gates (e.g., max users, max branches).
- Transition to read-only mode if expired (policy).

**Data ownership**
- Owns: subscription/licensing state.
- References: business IDs, device IDs.

**Dependencies**
- IAM, Business Setup (tenant context), Sync, Audit.

**External integrations**
- Payment provider/billing system (cloud), license server.

---

### 2.14 SaaS Admin Dashboard (Cloud)

**Responsibilities / scope**
- Back-office admin for the SaaS operator: tenant onboarding, subscription management.
- Support tooling: view sync health, device activations, audit trails.
- No direct write access to tenant domain data except via controlled admin operations.

**Core domain entities**
- TenantAdminProfile, SupportCase, AdminAction, TenantHealthSnapshot

**Key operations**
- Create/suspend tenant, change plan, reset device activation.
- View audit logs, sync status.

**Data ownership**
- Owns: admin/support metadata and admin actions.
- References: tenant/business IDs, subscription IDs.

**Dependencies**
- IAM (admin roles), Subscription, Audit, Reports.

**External integrations**
- Customer support tools (optional), billing provider.

---

## 3) Module dependency graph

### 3.1 One-way dependency strategy

- **Foundational modules** (IAM, Audit contract, Sync, Backup, Subscription) may be depended upon by domain modules.
- **Domain modules** should not depend on Reports; Reports depends on domain modules (read-only).
- **Sync** depends only on IAM + Audit and uses **adapters** supplied by each module to avoid Sync → Domain compile-time coupling where possible.

### 3.2 Dependency diagram (Mermaid)

```mermaid
flowchart TD
  IAM[Identity & Access]
  AUD[Audit (contract)]
  SYNC[Sync & Replication]
  SUB[Subscription & Licensing]
  BAK[Backup/Restore]

  BS[Business Setup]
  PI[Product & Inventory]
  POS[Sales & POS]
  CUS[Customers]
  SUP[Suppliers & Purchases]
  ACC[Accounting & Ledgers]
  HR[Employees/Payroll]
  QR[QR & Scanning]
  REP[Reports]
  ADMIN[SaaS Admin Dashboard]

  BS --> IAM
  BS --> AUD
  BS --> SYNC
  BS --> SUB

  PI --> BS
  PI --> AUD
  PI --> SYNC

  CUS --> BS
  CUS --> AUD
  CUS --> SYNC

  ACC --> BS
  ACC --> IAM
  ACC --> AUD
  ACC --> SYNC

  SUP --> BS
  SUP --> PI
  SUP --> ACC
  SUP --> IAM
  SUP --> AUD
  SUP --> SYNC

  HR --> BS
  HR --> ACC
  HR --> IAM
  HR --> AUD
  HR --> SYNC

  POS --> BS
  POS --> PI
  POS --> CUS
  POS --> IAM
  POS --> AUD
  POS --> SYNC
  POS --> SUB

  QR --> PI
  QR --> POS

  REP --> BS
  REP --> PI
  REP --> POS
  REP --> CUS
  REP --> SUP
  REP --> ACC
  REP --> HR
  REP --> AUD

  BAK --> IAM
  BAK --> AUD
  BAK --> BS
  BAK --> PI
  BAK --> POS
  BAK --> CUS
  BAK --> SUP
  BAK --> ACC
  BAK --> HR

  ADMIN --> IAM
  ADMIN --> SUB
  ADMIN --> AUD
  ADMIN --> REP
```

### 3.3 Circular dependencies and resolution

Potential cycles to watch:
- **Sales & POS ↔ Accounting**: POS needs “customer balance” and posting results; Accounting needs sales to post journals.
  - **Resolution**: POS emits a **PostingIntent**; Accounting posts and returns a **PostingReceipt** asynchronously. POS can show provisional status.
- **Purchases ↔ Inventory ↔ Accounting**: GRN updates stock; invoice posts payable.
  - **Resolution**: Purchases is the orchestrator: emits **InventoryReceiptIntent** to Inventory and **PayableIntent** to Accounting. Inventory/Accounting do not call back synchronously except to validate.

### 3.4 Initialization order requirements (Windows client)

Recommended startup order (so that permissions + gating + sync are available early):
1. IAM (local session/device identity)
2. Subscription/Licensing (entitlement snapshot)
3. Business Setup (load effective settings)
4. Sync (establish sync state, start background pump)
5. Domain modules: Product/Inventory, Customers, Accounting, Suppliers, HR, Sales/POS
6. Reports (build read-only projections)
7. Backup/Restore scheduler (non-blocking)
8. QR/Scanning (on-demand hardware init)

---

## 4) High-level API contracts (per module)

Contracts are expressed as conceptual operations. They are **not** endpoint specs and **not** schemas.

### Contract conventions

- All commands return either:
  - `Success(result)` with identifiers and/or snapshots, or
  - `Failure(error)` with a typed reason.
- Errors are stable and versioned: `ValidationError`, `NotFound`, `Conflict`, `Unauthorized`, `Forbidden`, `RateLimited`, `PreconditionFailed`, `DependencyUnavailable`.
- Auth context is always explicit: `ActorContext { userId, roles, branchId, deviceId }`.
- For offline-first: commands support `clientRequestId` for idempotency.

---

### 4.1 IAM — contract

**Exposes**
- `Authenticate(credentials|pin|offlineToken) -> Session`
- `RefreshSession(sessionToken) -> Session`
- `RevokeSession(sessionId) -> void`
- `CreateUser(userDraft) -> UserId`
- `AssignRoles(userId, roleIds) -> void`
- `Authorize(actor, permission, scope) -> Allowed|Denied`

**Success/failure scenarios**
- Success: session issued, user created.
- Failure: `Unauthorized` (bad credentials), `Forbidden` (no admin rights), `Conflict` (duplicate username).

**Auth requirements**
- User management requires admin permissions.

**Rate limiting**
- Cloud sign-in endpoints: limit by IP/device; local is not rate limited but should throttle brute force.

---

### 4.2 Business Setup — contract

**Exposes**
- `InitializeBusiness(businessDraft) -> BusinessId`
- `UpdateSettings(settingsPatch, effectiveFrom?) -> SettingsVersion`
- `GetEffectiveSettings(atTime?) -> SettingsSnapshot`
- `CreateBranch(branchDraft) -> BranchId`

**Failure**
- `ValidationError` (invalid tax settings), `Forbidden` (no permission), `PreconditionFailed` (subscription disallows multiple branches).

**Auth**
- Manager/admin only.

**Rate limiting**
- Low; protect cloud onboarding operations.

---

### 4.3 Product & Inventory — contract

**Exposes**
- `CreateProduct(productDraft) -> ProductId`
- `UpdateProduct(productId, patch) -> ProductSnapshot`
- `GetProduct(productId|barcode) -> ProductSnapshot`
- `RecordStockMovement(movementCommand) -> StockMovementReceipt`
- `GetAvailability(query) -> AvailabilityView`

**Input/output (conceptual)**
- `movementCommand`: { type: Receipt|Issue|Transfer|Adjust, lines[], reason, sourceDocumentRef }
- `StockMovementReceipt`: { movementId, postedAt, resultingOnHandSummary }

**Failure**
- `Conflict` (concurrency/version mismatch), `ValidationError` (negative stock not allowed), `NotFound` (unknown product).

**Auth**
- Inventory adjustments require elevated permission.

**Rate limiting**
- Cloud bulk sync endpoints should rate limit per device/tenant.

---

### 4.4 Sales & POS — contract

**Exposes**
- `OpenPOSSession(openCommand) -> POSSessionId`
- `FinalizeSale(saleCommand) -> SaleReceipt`
- `ProcessReturn(returnCommand) -> ReturnReceipt`
- `VoidSale(voidCommand) -> VoidReceipt`
- `GetSale(saleId) -> SaleSnapshot`

**Success/failure**
- Success: receipt issued with invoice number and totals.
- Failure: `Forbidden` (void without manager), `ValidationError` (invalid payments), `DependencyUnavailable` (printer offline is non-fatal; sale still succeeds but prints later).

**Auth**
- Cashier can finalize; manager needed for void/override discounts above threshold.

**Rate limiting**
- Not relevant locally; cloud reconciliation endpoints rate-limited.

---

### 4.5 Customers — contract

**Exposes**
- `CreateCustomer(customerDraft) -> CustomerId`
- `UpdateCustomer(customerId, patch) -> CustomerSnapshot`
- `GetCustomer(customerId|phone) -> CustomerSnapshot`
- `SetCreditProfile(customerId, creditPatch) -> void`

**Failure**
- `Conflict` (merge conflicts), `ValidationError`.

**Auth**
- Customer write requires permission; read may be broader.

---

### 4.6 Suppliers & Purchases — contract

**Exposes**
- `CreateSupplier(supplierDraft) -> SupplierId`
- `CreatePurchaseOrder(poDraft) -> PurchaseOrderId`
- `ReceiveGoods(grnCommand) -> GoodsReceiptId`
- `PostSupplierInvoice(invoiceCommand) -> SupplierInvoiceId`
- `CreatePurchaseReturn(returnCommand) -> PurchaseReturnId`

**Failure**
- `PreconditionFailed` (cannot invoice before receipt if policy), `Conflict` (double posting).

**Auth**
- Purchasing roles only.

---

### 4.7 Accounting & Ledgers — contract

**Exposes**
- `PostJournal(journalCommand) -> JournalEntryId`
- `PostFromIntent(postingIntent) -> PostingReceipt`
- `GetBalance(accountId|partyId, asOf?) -> BalanceView`
- `ClosePeriod(periodId) -> CloseReceipt`

**Failure**
- `ValidationError` (unbalanced entry), `Forbidden` (close period), `Conflict` (period already closed).

**Auth**
- Accountant/manager.

---

### 4.8 Employees/Attendance/Payroll — contract

**Exposes**
- `CreateEmployee(employeeDraft) -> EmployeeId`
- `RecordAttendance(attendanceEvent) -> AttendanceReceipt`
- `RunPayroll(payrollCommand) -> PayrollRunId`

**Failure**
- `ValidationError` (overlapping shifts), `Forbidden` (run payroll).

**Auth**
- HR/manager.

---

### 4.9 QR Code & Scanning — contract

**Exposes**
- `StartScan(sessionOptions) -> ScanSessionId`
- `Decode(scanInput) -> ScanResult`
- `GenerateCode(payload, format) -> Image/Bytes`

**Failure**
- `ValidationError` (unknown format), `DependencyUnavailable` (camera unavailable).

**Auth**
- Inherits caller’s context; no separate privileges.

---

### 4.10 Reports — contract

**Exposes**
- `RunReport(reportId, parameters) -> ReportResultRef`
- `ExportReport(reportResultRef, format) -> FileRef`
- `ListReports(filter) -> ReportDefinition[]`

**Failure**
- `Forbidden` (sensitive financial reports), `DependencyUnavailable` (missing projections triggers regeneration).

**Auth**
- Report-level permissions (e.g., profit report manager-only).

---

### 4.11 Backup/Restore — contract

**Exposes**
- `CreateBackup(backupCommand) -> BackupArtifactRef`
- `RestoreBackup(restoreCommand) -> RestoreReceipt`
- `VerifyIntegrity(target) -> IntegrityReport`

**Failure**
- `Forbidden` (restore), `PreconditionFailed` (version mismatch), `ValidationError` (corrupt backup).

**Auth**
- Owner/admin only.

---

### 4.12 Sync & Replication — contract

**Exposes**
- `StartSync(syncMode) -> SyncSessionRef`
- `GetSyncStatus() -> SyncStatusView`
- `ResolveConflict(conflictId, resolution) -> ConflictResolutionReceipt`

**Failure**
- `Unauthorized` (token expired), `DependencyUnavailable` (network), `RateLimited` (cloud).

**Auth**
- Requires valid device + tenant session.

**Rate limiting**
- Cloud: per-tenant + per-device quotas, exponential backoff.

---

### 4.13 Subscription/Licensing — contract

**Exposes**
- `RefreshEntitlements() -> EntitlementSnapshot`
- `ValidateLocalEntitlement(feature) -> Allowed|Denied(reason)`
- `ActivateDevice(activationRequest) -> ActivationReceipt`

**Failure**
- `Forbidden` (activation limit), `PreconditionFailed` (expired beyond grace).

**Auth**
- Admin for plan changes/activation.

---

### 4.14 SaaS Admin Dashboard — contract (cloud)

**Exposes**
- `CreateTenant(tenantDraft) -> TenantId`
- `SuspendTenant(tenantId, reason) -> void`
- `ChangePlan(tenantId, planChange) -> SubscriptionId`
- `ViewTenantHealth(tenantId) -> TenantHealthSnapshot`

**Failure**
- `Forbidden` (admin-only), `NotFound`.

**Auth**
- SaaS operator roles only.

---

## 5) Module responsibility matrix

### 5.1 Data domain ownership (authoritative writes)

| Data domain | Owner module | Other modules (read-only/reference) | Write access pattern |
|---|---|---|---|
| Business profile, branches, counters, tax settings | Business Setup | All | Write only via Business Setup commands |
| Users/roles/permissions/sessions/devices | IAM | All | Write only via IAM |
| Products/SKUs/barcodes/pricing metadata | Product & Inventory | Sales, Purchases, Reports, QR | Write only via Product & Inventory |
| Stock movements/on-hand ledger | Product & Inventory | Sales, Purchases, Reports | Append-only movements via Product & Inventory |
| Sales, returns, POS sessions | Sales & POS | Accounting, Reports | Write only via Sales & POS |
| Customers | Customers | Sales, Accounting, Reports | Write only via Customers |
| Suppliers, POs, GRNs, supplier invoices | Suppliers & Purchases | Inventory, Accounting, Reports | Write only via Suppliers & Purchases |
| Chart of accounts, journals, AR/AP | Accounting & Ledgers | Sales, Purchases, Reports, Customers | Write only via Accounting |
| Employees, attendance, payroll | Employees/Payroll | Accounting, Reports | Write only via HR/Payroll |
| Reports definitions & cached aggregates | Reports | All | Write only via Reports |
| Backup artifacts/manifests | Backup/Restore | All | Orchestrated snapshots; modules supply exporters |
| Sync metadata/conflicts | Sync | All | Write only via Sync |
| Subscription, entitlements, licensing | Subscription | All | Write only via Subscription |
| SaaS operator admin actions | SaaS Admin Dashboard | Audit/Reports | Write only via Admin |

### 5.2 Sensitive operations and permissions (examples)

| Operation | Owning module | Who can perform | Notes |
|---|---|---|---|
| Void a finalized sale | Sales & POS | Manager+ | Requires audit + reason code |
| Stock adjustment (non-receipt) | Product & Inventory | Inventory manager+ | Must be ledgered + audited |
| Restore backup | Backup/Restore | Owner/admin | Must lock app + verify versions |
| Close accounting period | Accounting | Accountant/owner | Prevent back-dated postings without override |
| Change tax/invoice rules | Business Setup | Owner/admin | New settings version; old sales keep snapshot |
| Run payroll | HR/Payroll | HR manager/owner | Emits accounting intent |
| Device activation | Subscription | Owner/admin | Enforced both cloud and local |

### 5.3 “Who can trigger what” (cross-module command triggers)

- Sales & POS can **request** stock decrements via Product & Inventory, but cannot edit stock directly.
- Purchases can **request** stock receipts via Product & Inventory and **request** payable postings via Accounting.
- HR/Payroll can **request** payroll expense postings via Accounting.
- Reports can **query** everyone, but cannot mutate anyone.

---

## 6) Transaction boundaries

### 6.1 Guiding approach

- **Atomicity is per owning module**: a module must keep its own invariants using local transactions (e.g., within SQLite/local store).
- **Cross-module consistency is “transactional-outbox + idempotent consumers”**:
  - The initiating module commits its own state and writes an **outbox event** in the same local transaction.
  - Sync/dispatcher delivers events to dependent modules.
  - Consumers apply changes idempotently using `clientRequestId` / `eventId`.
- Use **sagas/process managers** for multi-step workflows (purchase receiving, end-of-day close).

### 6.2 Operations that must be atomic (examples)

| Operation | Atomic within | Notes |
|---|---|---|
| Finalize sale (create sale + payments + invoice number reservation) | Sales & POS | Receipt printed can be asynchronous; sale persistence is the boundary |
| Apply stock movement (ledger entry + derived on-hand update) | Product & Inventory | Must prevent partial on-hand updates |
| Post a journal entry (balanced postings) | Accounting | Must be all-or-nothing |
| Run payroll (create payroll run + payslips) | HR/Payroll | Accounting intent emitted after commit |
| Restore backup | Backup/Restore orchestration | Requires global app lock; module-by-module restore with verification |

### 6.3 Cross-module consistency requirements

- **Sale → Inventory**: Inventory decrement must match finalized sales lines.
  - Strategy: POS emits `SaleFinalized` event with immutable line snapshots. Inventory consumes and posts stock issue.
  - If inventory posting fails (e.g., negative stock rule), POS sale remains valid but marked `StockPostingStatus = Pending/Failed` and requires manager resolution.

- **Sale/Purchase/Payroll → Accounting**: financial postings must trace to source documents.
  - Strategy: Accounting consumes intents and records `sourceDocumentRef` for auditability.

### 6.4 Rollback & error handling at boundaries

- No distributed rollback. Instead:
  - **Compensating actions**: e.g., a return compensates a sale’s inventory and accounting impact.
  - **Status tracking**: initiating module tracks downstream posting status.
  - **Retry with idempotency**: consumers must handle duplicates safely.
  - **Manual resolution queue** for irreconcilable conflicts.

---

## 7) Audit trail & logging requirements

### 7.1 Audit trail principles

- Audit is **append-only** and tamper-evident (hash chaining optional).
- Every sensitive or financial operation emits an audit record.
- Audit record includes:
  - `who`: userId, role(s), deviceId
  - `when`: timestamp (local + server), timezone
  - `where`: businessId, branchId, counterId
  - `what`: action type, entity type, entity id
  - `change`: before/after summary (redact secrets)
  - `why`: reason code for voids/adjustments

### 7.2 Modules requiring audit logging

| Module | Must audit | Examples |
|---|---|---|
| IAM | Yes | login/logout, user creation, role changes |
| Business Setup | Yes | tax/invoice changes, branch creation |
| Product & Inventory | Yes | adjustments, transfers, price changes |
| Sales & POS | Yes | finalize sale, void, refunds, discount overrides |
| Customers | Yes | credit limit changes, merges |
| Suppliers & Purchases | Yes | GRN, supplier invoice posting, returns |
| Accounting & Ledgers | Yes (critical) | journal posting, period close, edits/reversals |
| HR/Payroll | Yes | payroll runs, attendance edits |
| Backup/Restore | Yes (critical) | backup creation, restore attempts |
| Sync | Yes | conflict resolutions, device sync disable |
| Subscription | Yes | plan changes, device activations |
| Reports | Access logging | viewing/exporting sensitive financial reports |
| SaaS Admin | Yes (critical) | tenant suspend, plan overrides, support access |

### 7.3 Module-level audit contract (conceptual)

Each module emits:
- `AuditEvent { eventId, actorContext, action, entityRef, summary, before?, after?, reason?, correlationId }`

Audit module (or shared service) exposes:
- `WriteAudit(event) -> void`
- `QueryAudit(filter) -> AuditEvent[]` (restricted)

---

## 8) Offline sync strategy per module

### 8.1 Classification

- **Sync-critical (bidirectional)**: Business Setup, Products/Inventory, Sales/POS, Customers, Suppliers/Purchases, Accounting, HR/Payroll, Subscription.
- **Local-only**: QR/Scanning config/sessions, Backup artifacts.
- **Derived/read-only**: Reports (rebuildable), cached read models in any module.

### 8.2 Sync rules by module

| Module | Sync direction | Conflict strategy | Notes |
|---|---|---|---|
| IAM | Mixed | Prefer server authority for identity; local offline pin/cache | Device enrollment is authoritative in cloud |
| Business Setup | Bi-directional | Versioned settings; **last-write-wins** with admin-only writes | Settings snapshots used by POS are immutable per sale |
| Product & Inventory | Bi-directional | Stock via **append-only ledger**; reconcile by ordering + idempotency | Avoid editing historical stock movements |
| Sales & POS | Mostly upload | Sales are **immutable** after finalize; conflicts rare | Returns are new documents, not edits |
| Customers | Bi-directional | Merge conflicts via deterministic merge + manual review | Phone duplicates resolved by policy |
| Suppliers & Purchases | Bi-directional | Purchase docs are immutable after posting; returns as new docs | |
| Accounting | Bi-directional (with guardrails) | Journals append-only; corrections via reversal entries | Period close rules must be consistent cloud/local |
| HR/Payroll | Bi-directional | Payroll runs immutable; corrections via adjustments | Attendance edits audited |
| Reports | Pull/build | Recompute | Do not sync heavy aggregates unless cloud-generated |
| Backup/Restore | Local only | N/A | Optional later cloud backup |
| Sync | N/A | Owns conflict state | Provides conflict UI hooks |
| Subscription | Primarily pull | Server authority; local cache + grace period | Local enforcement uses signed snapshots |
| SaaS Admin | Cloud only | N/A | |

### 8.3 Conflict resolution at module boundaries

- Conflicts are resolved **within the owning module**, not by callers.
- When a conflict impacts other modules (e.g., product merge affects sales references), the owner emits a **MappingChanged** event so dependents can update their read models.

---

## 9) Scalability & concurrency

### 9.1 Concurrency sources

- **Local (single device)**: multiple UI threads, background sync, report generation, backup.
- **Multi-device**: multiple POS terminals for same business syncing to cloud.

### 9.2 Recommended locking strategy

- Prefer **optimistic concurrency** with version tokens on mutable entities.
- Use **append-only ledgers** for high-contention domains (stock, accounting postings) to reduce write conflicts.
- Local persistence uses short-lived transactions; avoid long locks.

### 9.3 Module-level performance targets (guidance)

| Module | Target latency (local) | Notes |
|---|---:|---|
| Sales & POS | P95 < 100ms for basket ops; finalize < 300ms | Printing/network are async |
| Product lookup/scan | P95 < 50ms | Must feel instant at checkout |
| Inventory posting | P95 < 200ms | Ledger append + on-hand update |
| Reports | Interactive < 2s for common reports | Heavy reports async with progress |
| Sync | Background; avoid UI jank | Backoff + batching |
| Backup | Background; throttled IO | Must not block POS finalize |

### 9.4 Resource limits / safeguards

- Reports and Backup must be **throttled** when POS is active.
- Sync batching limits per run; cap payload sizes; compress.
- Audit log growth: retention policies + archiving.

---

## Appendix: Cross-module communication patterns

- **Synchronous calls**: validations and read views (e.g., POS asks Inventory for availability).
- **Asynchronous events**: postings and downstream effects (sale finalized → inventory/accounting).
- **Idempotency**: all commands and events carry `clientRequestId`/`eventId`.
- **Correlation IDs**: a single sale or purchase has one correlation id across audit, sync, and accounting.
