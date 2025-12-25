# FlexPOS Module Boundaries & API Contracts

**Version:** 1.0  
**Last Updated:** 2024  
**Status:** Phase 1 - Design Specification  
**Related Documents:** [architecture.md](./architecture.md)

## Table of Contents

1. [Overview](#overview)
2. [Module Boundaries](#module-boundaries)
3. [Tauri IPC Contracts](#tauri-ipc-contracts)
4. [NestJS REST API Contracts](#nestjs-rest-api-contracts)
5. [Database Repository Contracts](#database-repository-contracts)
6. [Hardware Driver Contracts](#hardware-driver-contracts)
7. [Supabase Cloud Integration Contracts](#supabase-cloud-integration-contracts)
8. [Shared Type Definitions](#shared-type-definitions)
9. [Error Handling Standards](#error-handling-standards)
10. [State Management Contracts](#state-management-contracts)

---

## Overview

This document defines the **contracts** (interfaces, types, and protocols) between all modules in the FlexPOS system. These contracts serve as the **single source of truth** for integration between components.

### Design Principles

1. **Type Safety**: All contracts defined in TypeScript with strict typing
2. **Versioning**: All APIs support versioning for backward compatibility
3. **Validation**: Input validation at every boundary
4. **Error Consistency**: Standardized error responses across all layers
5. **Documentation**: Self-documenting through TypeScript types
6. **Idempotency**: All write operations are idempotent where possible

### Communication Layers

```
┌──────────────────────────────────────────────────────────────┐
│                    COMMUNICATION FLOW                         │
└──────────────────────────────────────────────────────────────┘

React Frontend
     │
     │ (A) Tauri IPC Commands & Events
     │     Type-safe, serialized JSON
     ▼
Tauri Backend (Rust)
     │
     │ (B) REST API / WebSocket
     │     JSON over HTTP
     ▼
NestJS Backend Service
     │
     │ (C) Repository Pattern
     │     TypeScript interfaces
     ▼
SQLite Database
     │
     │ (D) Hardware APIs
     │     Driver-specific protocols
     ▼
Hardware Devices

     │
     │ (E) Supabase Client SDK
     │     REST API / WebSocket
     ▼
Supabase Cloud
```

---

## Module Boundaries

### 1. Sales Module

**Responsibility**: Handle all sales transactions, payments, receipts, and returns

**Bounded Context**: Sales transactions, cart management, checkout flow

**Dependencies**:
- Inventory Module (stock validation and updates)
- Customer Module (customer lookup and loyalty)
- Hardware Module (receipt printing)
- Auth Module (user permissions)

**Public Interface**:
```typescript
interface ISalesModule {
  // Cart operations
  createCart(): Promise<Cart>;
  addItemToCart(cartId: string, item: CartItem): Promise<Cart>;
  removeItemFromCart(cartId: string, itemId: string): Promise<Cart>;
  updateCartItemQuantity(cartId: string, itemId: string, quantity: number): Promise<Cart>;
  applyDiscount(cartId: string, discount: Discount): Promise<Cart>;
  clearCart(cartId: string): Promise<void>;
  
  // Checkout operations
  calculateTotal(cartId: string): Promise<SaleTotal>;
  checkout(cartId: string, payments: Payment[]): Promise<Sale>;
  
  // Sale operations
  getSale(saleId: string): Promise<Sale>;
  voidSale(saleId: string, reason: string): Promise<Sale>;
  processSaleReturn(saleId: string, items: ReturnItem[]): Promise<Return>;
  
  // Receipt operations
  printReceipt(saleId: string): Promise<PrintResult>;
  emailReceipt(saleId: string, email: string): Promise<void>;
  
  // Query operations
  getSalesByDateRange(startDate: Date, endDate: Date): Promise<Sale[]>;
  getSalesByUser(userId: string): Promise<Sale[]>;
  getDailySummary(date: Date): Promise<SalesSummary>;
}
```

**Data Ownership**:
- `sales` table (full ownership)
- `sale_items` table (full ownership)
- `payments` table (full ownership)
- `returns` table (full ownership)
- `receipts` table (full ownership)

---

### 2. Inventory Module

**Responsibility**: Product catalog, stock management, categories, suppliers

**Bounded Context**: Product master data, inventory levels, stock movements

**Dependencies**:
- Sync Module (data synchronization)
- Auth Module (user permissions)

**Public Interface**:
```typescript
interface IInventoryModule {
  // Product operations
  createProduct(product: CreateProductDto): Promise<Product>;
  updateProduct(productId: string, updates: UpdateProductDto): Promise<Product>;
  deleteProduct(productId: string): Promise<void>;
  getProduct(productId: string): Promise<Product>;
  getProductByBarcode(barcode: string): Promise<Product | null>;
  searchProducts(query: string, filters?: ProductFilters): Promise<Product[]>;
  
  // Category operations
  createCategory(category: CreateCategoryDto): Promise<Category>;
  updateCategory(categoryId: string, updates: UpdateCategoryDto): Promise<Category>;
  deleteCategory(categoryId: string): Promise<void>;
  getCategories(): Promise<Category[]>;
  
  // Stock operations
  adjustStock(productId: string, adjustment: StockAdjustment): Promise<StockMovement>;
  getStockLevel(productId: string): Promise<number>;
  getLowStockProducts(threshold?: number): Promise<Product[]>;
  recordStockMovement(movement: CreateStockMovementDto): Promise<StockMovement>;
  
  // Supplier operations
  createSupplier(supplier: CreateSupplierDto): Promise<Supplier>;
  updateSupplier(supplierId: string, updates: UpdateSupplierDto): Promise<Supplier>;
  getSuppliers(): Promise<Supplier[]>;
  
  // Purchase orders
  createPurchaseOrder(po: CreatePurchaseOrderDto): Promise<PurchaseOrder>;
  receivePurchaseOrder(poId: string, items: ReceivedItem[]): Promise<PurchaseOrder>;
}
```

**Data Ownership**:
- `products` table (full ownership)
- `categories` table (full ownership)
- `stock_movements` table (full ownership)
- `suppliers` table (full ownership)
- `purchase_orders` table (full ownership)

---

### 3. Customer Module

**Responsibility**: Customer management, loyalty programs, customer accounts

**Bounded Context**: Customer data, loyalty points, customer transactions

**Dependencies**:
- Sync Module (data synchronization)
- Auth Module (user permissions)

**Public Interface**:
```typescript
interface ICustomerModule {
  // Customer operations
  createCustomer(customer: CreateCustomerDto): Promise<Customer>;
  updateCustomer(customerId: string, updates: UpdateCustomerDto): Promise<Customer>;
  deleteCustomer(customerId: string): Promise<void>;
  getCustomer(customerId: string): Promise<Customer>;
  searchCustomers(query: string): Promise<Customer[]>;
  getCustomerByPhone(phone: string): Promise<Customer | null>;
  
  // Loyalty operations
  getLoyaltyPoints(customerId: string): Promise<number>;
  addLoyaltyPoints(customerId: string, points: number, reason: string): Promise<void>;
  redeemLoyaltyPoints(customerId: string, points: number): Promise<void>;
  getLoyaltyTransactions(customerId: string): Promise<LoyaltyTransaction[]>;
  
  // Customer account operations
  getCustomerAccount(customerId: string): Promise<CustomerAccount>;
  recordCredit(customerId: string, amount: number, reason: string): Promise<void>;
  recordPayment(customerId: string, amount: number): Promise<void>;
  getCustomerBalance(customerId: string): Promise<number>;
}
```

**Data Ownership**:
- `customers` table (full ownership)
- `customer_accounts` table (full ownership)
- `loyalty_transactions` table (full ownership)

---

### 4. Auth Module

**Responsibility**: Authentication, authorization, user management, permissions

**Bounded Context**: User accounts, roles, permissions, sessions

**Dependencies**:
- License Module (license validation)
- Sync Module (user data sync)

**Public Interface**:
```typescript
interface IAuthModule {
  // Authentication
  login(username: string, password: string): Promise<AuthResult>;
  loginWithBiometric(userId: string, biometricData: BiometricData): Promise<AuthResult>;
  logout(userId: string): Promise<void>;
  refreshToken(refreshToken: string): Promise<AuthResult>;
  validateSession(sessionToken: string): Promise<Session>;
  
  // User management
  createUser(user: CreateUserDto): Promise<User>;
  updateUser(userId: string, updates: UpdateUserDto): Promise<User>;
  deleteUser(userId: string): Promise<void>;
  getUser(userId: string): Promise<User>;
  getUsers(): Promise<User[]>;
  changePassword(userId: string, oldPassword: string, newPassword: string): Promise<void>;
  resetPassword(userId: string, newPassword: string): Promise<void>;
  
  // Role & permission management
  assignRole(userId: string, roleId: string): Promise<void>;
  getRoles(): Promise<Role[]>;
  getPermissions(userId: string): Promise<Permission[]>;
  checkPermission(userId: string, permission: string): Promise<boolean>;
  
  // Biometric management
  enrollBiometric(userId: string, biometricData: BiometricData): Promise<void>;
  deleteBiometric(userId: string): Promise<void>;
}
```

**Data Ownership**:
- `users` table (full ownership)
- `roles` table (full ownership)
- `permissions` table (full ownership)
- `sessions` table (full ownership)
- `biometric_templates` table (full ownership, local-only)

---

### 5. Sync Module

**Responsibility**: Cloud synchronization, conflict resolution, sync queue management

**Bounded Context**: Sync operations, conflict detection, offline queue

**Dependencies**:
- Supabase Client (cloud communication)
- All data modules (for syncing their data)

**Public Interface**:
```typescript
interface ISyncModule {
  // Sync operations
  syncNow(): Promise<SyncResult>;
  getSyncStatus(): Promise<SyncStatus>;
  getSyncQueue(): Promise<SyncQueueItem[]>;
  retrySyncItem(itemId: string): Promise<void>;
  clearSyncQueue(): Promise<void>;
  
  // Connection management
  isOnline(): Promise<boolean>;
  getConnectionStatus(): Promise<ConnectionStatus>;
  
  // Conflict resolution
  getConflicts(): Promise<Conflict[]>;
  resolveConflict(conflictId: string, resolution: ConflictResolution): Promise<void>;
  
  // Sync configuration
  getSyncConfig(): Promise<SyncConfig>;
  updateSyncConfig(config: Partial<SyncConfig>): Promise<SyncConfig>;
  
  // Event subscriptions
  onSyncStart(callback: () => void): Unsubscribe;
  onSyncComplete(callback: (result: SyncResult) => void): Unsubscribe;
  onSyncError(callback: (error: Error) => void): Unsubscribe;
  onConflictDetected(callback: (conflict: Conflict) => void): Unsubscribe;
}
```

**Data Ownership**:
- `sync_queue` table (full ownership)
- `sync_log` table (full ownership)
- `conflicts` table (full ownership)

---

### 6. Hardware Module

**Responsibility**: Hardware device integration (printers, scanners, biometric)

**Bounded Context**: Hardware communication, device configuration

**Dependencies**:
- Config Module (device configuration)
- Auth Module (biometric authentication)

**Public Interface**:
```typescript
interface IHardwareModule {
  // Printer operations
  listPrinters(): Promise<Printer[]>;
  setDefaultPrinter(printerId: string): Promise<void>;
  printReceipt(receipt: Receipt, printerId?: string): Promise<PrintResult>;
  printReport(report: Report, printerId?: string): Promise<PrintResult>;
  testPrint(printerId: string): Promise<PrintResult>;
  getPrinterStatus(printerId: string): Promise<PrinterStatus>;
  
  // Barcode scanner operations
  listScanners(): Promise<Scanner[]>;
  enableScanner(scannerId: string): Promise<void>;
  disableScanner(scannerId: string): Promise<void>;
  onBarcodeScanned(callback: (barcode: string) => void): Unsubscribe;
  
  // Biometric device operations
  listBiometricDevices(): Promise<BiometricDevice[]>;
  captureFingerprint(deviceId?: string): Promise<BiometricData>;
  verifyFingerprint(template: BiometricTemplate, data: BiometricData): Promise<boolean>;
  
  // Cash drawer operations
  openCashDrawer(): Promise<void>;
  getCashDrawerStatus(): Promise<CashDrawerStatus>;
  
  // Device configuration
  configureDevice(deviceId: string, config: DeviceConfig): Promise<void>;
  getDeviceConfig(deviceId: string): Promise<DeviceConfig>;
}
```

**Data Ownership**:
- `hardware_config` table (full ownership, local-only)
- `biometric_templates` table (shared with Auth Module, local-only)

---

### 7. Backup Module

**Responsibility**: Automated backups, restore operations, data export/import

**Bounded Context**: Database backups, data integrity

**Dependencies**:
- Database Service (backup operations)
- Supabase Client (cloud backup upload)

**Public Interface**:
```typescript
interface IBackupModule {
  // Backup operations
  createBackup(name?: string): Promise<Backup>;
  listBackups(): Promise<Backup[]>;
  deleteBackup(backupId: string): Promise<void>;
  verifyBackup(backupId: string): Promise<boolean>;
  
  // Restore operations
  restoreBackup(backupId: string): Promise<void>;
  
  // Cloud backup operations
  uploadBackupToCloud(backupId: string): Promise<void>;
  downloadBackupFromCloud(cloudBackupId: string): Promise<Backup>;
  listCloudBackups(): Promise<CloudBackup[]>;
  
  // Export/import operations
  exportData(format: 'json' | 'csv', tables?: string[]): Promise<ExportResult>;
  importData(file: File): Promise<ImportResult>;
  
  // Scheduling
  getBackupSchedule(): Promise<BackupSchedule>;
  updateBackupSchedule(schedule: BackupSchedule): Promise<void>;
}
```

**Data Ownership**:
- `backups_log` table (full ownership)
- File system backups (full ownership)

---

### 8. License Module

**Responsibility**: License validation, feature flags, subscription management

**Bounded Context**: License state, feature access, grace period

**Dependencies**:
- Supabase Client (license validation API)
- Config Module (license storage)

**Public Interface**:
```typescript
interface ILicenseModule {
  // License validation
  validateLicense(): Promise<LicenseStatus>;
  activateLicense(licenseKey: string): Promise<ActivationResult>;
  deactivateLicense(): Promise<void>;
  
  // License information
  getLicenseInfo(): Promise<LicenseInfo>;
  getLicenseStatus(): Promise<LicenseStatus>;
  getDaysRemaining(): Promise<number>;
  getGracePeriodStatus(): Promise<GracePeriodStatus>;
  
  // Feature flags
  isFeatureEnabled(feature: string): Promise<boolean>;
  getEnabledFeatures(): Promise<string[]>;
  
  // Events
  onLicenseExpiring(callback: (daysRemaining: number) => void): Unsubscribe;
  onLicenseExpired(callback: () => void): Unsubscribe;
  onGracePeriodEntered(callback: () => void): Unsubscribe;
}
```

**Data Ownership**:
- `license_cache` table (full ownership, local-only)
- Windows Registry (license key storage)

---

### 9. Reports Module

**Responsibility**: Analytics, reporting, dashboard data aggregation

**Bounded Context**: Business intelligence, sales reports, inventory reports

**Dependencies**:
- Sales Module (sales data)
- Inventory Module (inventory data)
- Customer Module (customer data)

**Public Interface**:
```typescript
interface IReportsModule {
  // Sales reports
  getDailySalesReport(date: Date): Promise<DailySalesReport>;
  getSalesReportByDateRange(startDate: Date, endDate: Date): Promise<SalesReport>;
  getSalesReportByProduct(productId: string, dateRange: DateRange): Promise<ProductSalesReport>;
  getSalesReportByUser(userId: string, dateRange: DateRange): Promise<UserSalesReport>;
  getTopSellingProducts(limit: number, dateRange: DateRange): Promise<ProductSales[]>;
  
  // Inventory reports
  getInventoryReport(): Promise<InventoryReport>;
  getLowStockReport(): Promise<LowStockReport>;
  getStockMovementReport(dateRange: DateRange): Promise<StockMovementReport>;
  
  // Customer reports
  getCustomerReport(): Promise<CustomerReport>;
  getTopCustomers(limit: number, dateRange: DateRange): Promise<CustomerSales[]>;
  
  // Financial reports
  getPaymentMethodReport(dateRange: DateRange): Promise<PaymentMethodReport>;
  getTaxReport(dateRange: DateRange): Promise<TaxReport>;
  getProfitReport(dateRange: DateRange): Promise<ProfitReport>;
  
  // Dashboard data
  getDashboardData(date: Date): Promise<DashboardData>;
  
  // Export reports
  exportReport(reportType: string, format: 'pdf' | 'csv' | 'excel', params: any): Promise<File>;
}
```

**Data Ownership**:
- No direct table ownership (aggregates data from other modules)
- `reports_cache` table (for cached computations)

---

### 10. Config Module

**Responsibility**: Application configuration, store settings, system preferences

**Bounded Context**: Configuration management, system settings

**Dependencies**: None (foundational module)

**Public Interface**:
```typescript
interface IConfigModule {
  // Store configuration
  getStoreConfig(): Promise<StoreConfig>;
  updateStoreConfig(config: Partial<StoreConfig>): Promise<StoreConfig>;
  
  // System configuration
  getSystemConfig(): Promise<SystemConfig>;
  updateSystemConfig(config: Partial<SystemConfig>): Promise<SystemConfig>;
  
  // Tax configuration
  getTaxRates(): Promise<TaxRate[]>;
  updateTaxRates(rates: TaxRate[]): Promise<void>;
  
  // Receipt configuration
  getReceiptTemplate(): Promise<ReceiptTemplate>;
  updateReceiptTemplate(template: ReceiptTemplate): Promise<void>;
  
  // Currency configuration
  getCurrencyConfig(): Promise<CurrencyConfig>;
  updateCurrencyConfig(config: CurrencyConfig): Promise<void>;
  
  // Feature flags (local overrides)
  getFeatureFlags(): Promise<Record<string, boolean>>;
  setFeatureFlag(flag: string, enabled: boolean): Promise<void>;
}
```

**Data Ownership**:
- `config` table (full ownership)
- Configuration files in file system

---

## Tauri IPC Contracts

### Overview

Tauri uses a command-based IPC system where the React frontend invokes Rust commands and receives responses. All communication is serialized as JSON.

### Command Pattern

```typescript
// Frontend invokes command
const result = await invoke<ResponseType>('command_name', { 
  param1: value1,
  param2: value2 
});

// Tauri backend handles command
#[tauri::command]
async fn command_name(param1: Type1, param2: Type2) -> Result<ResponseType, String> {
  // Implementation
}
```

### Sales Commands

```typescript
// ============================================
// CART COMMANDS
// ============================================

/**
 * Create a new shopping cart
 */
invoke<Cart>('create_cart', {});

/**
 * Add item to cart
 */
invoke<Cart>('add_to_cart', {
  cartId: string;
  productId: string;
  quantity: number;
  price?: number; // Optional override
});

/**
 * Remove item from cart
 */
invoke<Cart>('remove_from_cart', {
  cartId: string;
  itemId: string;
});

/**
 * Update cart item quantity
 */
invoke<Cart>('update_cart_item_quantity', {
  cartId: string;
  itemId: string;
  quantity: number;
});

/**
 * Apply discount to cart
 */
invoke<Cart>('apply_discount', {
  cartId: string;
  discount: {
    type: 'percentage' | 'fixed';
    value: number;
    reason?: string;
  };
});

/**
 * Clear cart
 */
invoke<void>('clear_cart', {
  cartId: string;
});

// ============================================
// CHECKOUT COMMANDS
// ============================================

/**
 * Calculate cart total (with tax and discounts)
 */
invoke<SaleTotal>('calculate_total', {
  cartId: string;
});

/**
 * Complete checkout
 */
invoke<Sale>('checkout', {
  cartId: string;
  customerId?: string;
  payments: Array<{
    method: 'cash' | 'card' | 'mobile' | 'credit';
    amount: number;
    reference?: string;
  }>;
  printReceipt?: boolean;
});

// ============================================
// SALE COMMANDS
// ============================================

/**
 * Get sale by ID
 */
invoke<Sale>('get_sale', {
  saleId: string;
});

/**
 * Void sale (manager permission required)
 */
invoke<Sale>('void_sale', {
  saleId: string;
  reason: string;
  managerId: string; // Manager authorization
});

/**
 * Process return
 */
invoke<Return>('process_return', {
  saleId: string;
  items: Array<{
    saleItemId: string;
    quantity: number;
    reason: string;
  }>;
  refundMethod: 'cash' | 'card' | 'credit';
});

// ============================================
// RECEIPT COMMANDS
// ============================================

/**
 * Print receipt
 */
invoke<PrintResult>('print_receipt', {
  saleId: string;
  printerId?: string; // Optional specific printer
});

/**
 * Email receipt
 */
invoke<void>('email_receipt', {
  saleId: string;
  email: string;
});

/**
 * Get receipt preview
 */
invoke<ReceiptPreview>('get_receipt_preview', {
  saleId: string;
});
```

### Inventory Commands

```typescript
// ============================================
// PRODUCT COMMANDS
// ============================================

/**
 * Create product
 */
invoke<Product>('create_product', {
  name: string;
  sku: string;
  barcode?: string;
  categoryId: string;
  price: number;
  cost?: number;
  stock: number;
  lowStockThreshold?: number;
  description?: string;
  imageUrl?: string;
});

/**
 * Update product
 */
invoke<Product>('update_product', {
  productId: string;
  updates: Partial<{
    name: string;
    sku: string;
    barcode: string;
    categoryId: string;
    price: number;
    cost: number;
    stock: number;
    lowStockThreshold: number;
    description: string;
    imageUrl: string;
    active: boolean;
  }>;
});

/**
 * Delete product
 */
invoke<void>('delete_product', {
  productId: string;
});

/**
 * Get product by ID
 */
invoke<Product>('get_product', {
  productId: string;
});

/**
 * Get product by barcode
 */
invoke<Product | null>('get_product_by_barcode', {
  barcode: string;
});

/**
 * Search products
 */
invoke<Product[]>('search_products', {
  query: string;
  filters?: {
    categoryId?: string;
    minPrice?: number;
    maxPrice?: number;
    inStock?: boolean;
  };
  limit?: number;
  offset?: number;
});

// ============================================
// STOCK COMMANDS
// ============================================

/**
 * Adjust stock
 */
invoke<StockMovement>('adjust_stock', {
  productId: string;
  quantity: number; // Positive = add, negative = subtract
  reason: string;
  reference?: string;
});

/**
 * Get stock level
 */
invoke<number>('get_stock_level', {
  productId: string;
});

/**
 * Get low stock products
 */
invoke<Product[]>('get_low_stock_products', {
  threshold?: number;
});

// ============================================
// CATEGORY COMMANDS
// ============================================

/**
 * Create category
 */
invoke<Category>('create_category', {
  name: string;
  description?: string;
  parentCategoryId?: string;
});

/**
 * Get categories
 */
invoke<Category[]>('get_categories', {});

/**
 * Update category
 */
invoke<Category>('update_category', {
  categoryId: string;
  updates: Partial<{
    name: string;
    description: string;
    parentCategoryId: string;
  }>;
});
```

### Customer Commands

```typescript
/**
 * Create customer
 */
invoke<Customer>('create_customer', {
  name: string;
  phone: string;
  email?: string;
  address?: string;
  dateOfBirth?: string;
});

/**
 * Search customers
 */
invoke<Customer[]>('search_customers', {
  query: string;
  limit?: number;
});

/**
 * Get customer by phone
 */
invoke<Customer | null>('get_customer_by_phone', {
  phone: string;
});

/**
 * Update customer
 */
invoke<Customer>('update_customer', {
  customerId: string;
  updates: Partial<Customer>;
});

/**
 * Get loyalty points
 */
invoke<number>('get_loyalty_points', {
  customerId: string;
});

/**
 * Add loyalty points
 */
invoke<void>('add_loyalty_points', {
  customerId: string;
  points: number;
  reason: string;
});

/**
 * Redeem loyalty points
 */
invoke<void>('redeem_loyalty_points', {
  customerId: string;
  points: number;
});
```

### Auth Commands

```typescript
/**
 * Login
 */
invoke<AuthResult>('login', {
  username: string;
  password: string;
});

/**
 * Login with biometric
 */
invoke<AuthResult>('login_biometric', {
  userId: string;
  biometricData: string; // Base64 encoded
});

/**
 * Logout
 */
invoke<void>('logout', {
  userId: string;
});

/**
 * Validate session
 */
invoke<Session>('validate_session', {
  sessionToken: string;
});

/**
 * Change password
 */
invoke<void>('change_password', {
  userId: string;
  oldPassword: string;
  newPassword: string;
});

/**
 * Check permission
 */
invoke<boolean>('check_permission', {
  userId: string;
  permission: string;
});

/**
 * Get current user
 */
invoke<User>('get_current_user', {});
```

### Sync Commands

```typescript
/**
 * Trigger manual sync
 */
invoke<SyncResult>('sync_now', {});

/**
 * Get sync status
 */
invoke<SyncStatus>('get_sync_status', {});

/**
 * Get sync queue items
 */
invoke<SyncQueueItem[]>('get_sync_queue', {});

/**
 * Check online status
 */
invoke<boolean>('is_online', {});

/**
 * Get conflicts
 */
invoke<Conflict[]>('get_conflicts', {});

/**
 * Resolve conflict
 */
invoke<void>('resolve_conflict', {
  conflictId: string;
  resolution: 'local' | 'remote' | 'merge';
  mergedData?: any;
});
```

### Hardware Commands

```typescript
/**
 * List printers
 */
invoke<Printer[]>('list_printers', {});

/**
 * Print receipt
 */
invoke<PrintResult>('print_receipt', {
  saleId: string;
  printerId?: string;
});

/**
 * Test print
 */
invoke<PrintResult>('test_print', {
  printerId: string;
});

/**
 * Get printer status
 */
invoke<PrinterStatus>('get_printer_status', {
  printerId: string;
});

/**
 * Open cash drawer
 */
invoke<void>('open_cash_drawer', {});

/**
 * Enable scanner
 */
invoke<void>('enable_scanner', {
  scannerId: string;
});

/**
 * Disable scanner
 */
invoke<void>('disable_scanner', {
  scannerId: string;
});

/**
 * Capture fingerprint
 */
invoke<BiometricData>('capture_fingerprint', {
  deviceId?: string;
});
```

### Backup Commands

```typescript
/**
 * Create backup
 */
invoke<Backup>('create_backup', {
  name?: string;
});

/**
 * List backups
 */
invoke<Backup[]>('list_backups', {});

/**
 * Restore backup
 */
invoke<void>('restore_backup', {
  backupId: string;
});

/**
 * Upload backup to cloud
 */
invoke<void>('upload_backup_to_cloud', {
  backupId: string;
});

/**
 * Export data
 */
invoke<ExportResult>('export_data', {
  format: 'json' | 'csv';
  tables?: string[];
});
```

### License Commands

```typescript
/**
 * Activate license
 */
invoke<ActivationResult>('activate_license', {
  licenseKey: string;
});

/**
 * Get license status
 */
invoke<LicenseStatus>('get_license_status', {});

/**
 * Validate license
 */
invoke<LicenseStatus>('validate_license', {});

/**
 * Get grace period status
 */
invoke<GracePeriodStatus>('get_grace_period_status', {});

/**
 * Check if feature is enabled
 */
invoke<boolean>('is_feature_enabled', {
  feature: string;
});
```

### Reports Commands

```typescript
/**
 * Get daily sales report
 */
invoke<DailySalesReport>('get_daily_sales_report', {
  date: string; // ISO date string
});

/**
 * Get sales report by date range
 */
invoke<SalesReport>('get_sales_report', {
  startDate: string;
  endDate: string;
});

/**
 * Get dashboard data
 */
invoke<DashboardData>('get_dashboard_data', {
  date: string;
});

/**
 * Get top selling products
 */
invoke<ProductSales[]>('get_top_selling_products', {
  limit: number;
  startDate: string;
  endDate: string;
});

/**
 * Export report
 */
invoke<string>('export_report', { // Returns file path
  reportType: string;
  format: 'pdf' | 'csv' | 'excel';
  params: any;
});
```

### Config Commands

```typescript
/**
 * Get store config
 */
invoke<StoreConfig>('get_store_config', {});

/**
 * Update store config
 */
invoke<StoreConfig>('update_store_config', {
  config: Partial<StoreConfig>;
});

/**
 * Get system config
 */
invoke<SystemConfig>('get_system_config', {});

/**
 * Update system config
 */
invoke<SystemConfig>('update_system_config', {
  config: Partial<SystemConfig>;
});

/**
 * Get receipt template
 */
invoke<ReceiptTemplate>('get_receipt_template', {});

/**
 * Update receipt template
 */
invoke<void>('update_receipt_template', {
  template: ReceiptTemplate;
});
```

### Tauri Events (Backend → Frontend)

```typescript
// Event listener pattern
import { listen } from '@tauri-apps/api/event';

// Barcode scanned event
listen<string>('barcode-scanned', (event) => {
  const barcode = event.payload;
  // Handle barcode
});

// Sync started event
listen<void>('sync-started', () => {
  // Update UI
});

// Sync completed event
listen<SyncResult>('sync-completed', (event) => {
  const result = event.payload;
  // Show notification
});

// Sync error event
listen<Error>('sync-error', (event) => {
  const error = event.payload;
  // Show error message
});

// Conflict detected event
listen<Conflict>('conflict-detected', (event) => {
  const conflict = event.payload;
  // Prompt user for resolution
});

// License expiring event
listen<number>('license-expiring', (event) => {
  const daysRemaining = event.payload;
  // Show warning
});

// License expired event
listen<void>('license-expired', () => {
  // Block access or show message
});

// Printer status changed
listen<PrinterStatus>('printer-status-changed', (event) => {
  // Update UI
});

// Low stock alert
listen<Product>('low-stock-alert', (event) => {
  const product = event.payload;
  // Show notification
});
```

---

## NestJS REST API Contracts

### Base URL

```
http://localhost:3000/api/v1
```

### Authentication

All API requests (except `/auth/login`) require a JWT token in the Authorization header:

```
Authorization: Bearer <jwt_token>
```

### Standard Response Format

#### Success Response
```typescript
{
  success: true;
  data: T; // Response data
  meta?: {
    page?: number;
    limit?: number;
    total?: number;
  };
}
```

#### Error Response
```typescript
{
  success: false;
  error: {
    code: string;        // Error code (e.g., 'PRODUCT_NOT_FOUND')
    message: string;     // Human-readable error message
    details?: any;       // Additional error details
  };
}
```

### Sales Endpoints

```typescript
// ============================================
// CART ENDPOINTS
// ============================================

/**
 * POST /api/v1/sales/cart
 * Create a new cart
 */
Request: {}
Response: {
  success: true;
  data: Cart;
}

/**
 * POST /api/v1/sales/cart/:cartId/items
 * Add item to cart
 */
Request: {
  productId: string;
  quantity: number;
  price?: number;
}
Response: {
  success: true;
  data: Cart;
}

/**
 * DELETE /api/v1/sales/cart/:cartId/items/:itemId
 * Remove item from cart
 */
Response: {
  success: true;
  data: Cart;
}

/**
 * PATCH /api/v1/sales/cart/:cartId/items/:itemId
 * Update cart item
 */
Request: {
  quantity: number;
}
Response: {
  success: true;
  data: Cart;
}

/**
 * POST /api/v1/sales/cart/:cartId/discount
 * Apply discount
 */
Request: {
  type: 'percentage' | 'fixed';
  value: number;
  reason?: string;
}
Response: {
  success: true;
  data: Cart;
}

/**
 * DELETE /api/v1/sales/cart/:cartId
 * Clear cart
 */
Response: {
  success: true;
  data: null;
}

// ============================================
// CHECKOUT ENDPOINTS
// ============================================

/**
 * POST /api/v1/sales/cart/:cartId/calculate
 * Calculate cart total
 */
Response: {
  success: true;
  data: SaleTotal;
}

/**
 * POST /api/v1/sales/checkout
 * Complete checkout
 */
Request: {
  cartId: string;
  customerId?: string;
  payments: Array<{
    method: PaymentMethod;
    amount: number;
    reference?: string;
  }>;
}
Response: {
  success: true;
  data: Sale;
}

// ============================================
// SALE ENDPOINTS
// ============================================

/**
 * GET /api/v1/sales/:saleId
 * Get sale by ID
 */
Response: {
  success: true;
  data: Sale;
}

/**
 * GET /api/v1/sales
 * Get sales with filters
 */
Query Params: {
  startDate?: string;
  endDate?: string;
  userId?: string;
  page?: number;
  limit?: number;
}
Response: {
  success: true;
  data: Sale[];
  meta: {
    page: number;
    limit: number;
    total: number;
  };
}

/**
 * POST /api/v1/sales/:saleId/void
 * Void a sale
 */
Request: {
  reason: string;
  managerId: string;
}
Response: {
  success: true;
  data: Sale;
}

/**
 * POST /api/v1/sales/:saleId/return
 * Process return
 */
Request: {
  items: Array<{
    saleItemId: string;
    quantity: number;
    reason: string;
  }>;
  refundMethod: PaymentMethod;
}
Response: {
  success: true;
  data: Return;
}

/**
 * GET /api/v1/sales/summary/daily
 * Get daily summary
 */
Query Params: {
  date: string; // ISO date
}
Response: {
  success: true;
  data: SalesSummary;
}
```

### Inventory Endpoints

```typescript
// ============================================
// PRODUCT ENDPOINTS
// ============================================

/**
 * POST /api/v1/inventory/products
 * Create product
 */
Request: {
  name: string;
  sku: string;
  barcode?: string;
  categoryId: string;
  price: number;
  cost?: number;
  stock: number;
  lowStockThreshold?: number;
  description?: string;
  imageUrl?: string;
}
Response: {
  success: true;
  data: Product;
}

/**
 * GET /api/v1/inventory/products/:productId
 * Get product by ID
 */
Response: {
  success: true;
  data: Product;
}

/**
 * PATCH /api/v1/inventory/products/:productId
 * Update product
 */
Request: Partial<Product>
Response: {
  success: true;
  data: Product;
}

/**
 * DELETE /api/v1/inventory/products/:productId
 * Delete product
 */
Response: {
  success: true;
  data: null;
}

/**
 * GET /api/v1/inventory/products
 * Search products
 */
Query Params: {
  q?: string;           // Search query
  categoryId?: string;
  minPrice?: number;
  maxPrice?: number;
  inStock?: boolean;
  page?: number;
  limit?: number;
}
Response: {
  success: true;
  data: Product[];
  meta: PaginationMeta;
}

/**
 * GET /api/v1/inventory/products/barcode/:barcode
 * Get product by barcode
 */
Response: {
  success: true;
  data: Product | null;
}

/**
 * GET /api/v1/inventory/products/low-stock
 * Get low stock products
 */
Query Params: {
  threshold?: number;
}
Response: {
  success: true;
  data: Product[];
}

// ============================================
// STOCK ENDPOINTS
// ============================================

/**
 * POST /api/v1/inventory/products/:productId/stock/adjust
 * Adjust stock
 */
Request: {
  quantity: number; // Can be negative
  reason: string;
  reference?: string;
}
Response: {
  success: true;
  data: StockMovement;
}

/**
 * GET /api/v1/inventory/products/:productId/stock
 * Get current stock level
 */
Response: {
  success: true;
  data: {
    productId: string;
    stock: number;
    lastUpdated: string;
  };
}

/**
 * GET /api/v1/inventory/stock-movements
 * Get stock movements
 */
Query Params: {
  productId?: string;
  startDate?: string;
  endDate?: string;
  page?: number;
  limit?: number;
}
Response: {
  success: true;
  data: StockMovement[];
  meta: PaginationMeta;
}

// ============================================
// CATEGORY ENDPOINTS
// ============================================

/**
 * POST /api/v1/inventory/categories
 * Create category
 */
Request: {
  name: string;
  description?: string;
  parentCategoryId?: string;
}
Response: {
  success: true;
  data: Category;
}

/**
 * GET /api/v1/inventory/categories
 * Get all categories
 */
Response: {
  success: true;
  data: Category[];
}

/**
 * PATCH /api/v1/inventory/categories/:categoryId
 * Update category
 */
Request: Partial<Category>
Response: {
  success: true;
  data: Category;
}

/**
 * DELETE /api/v1/inventory/categories/:categoryId
 * Delete category
 */
Response: {
  success: true;
  data: null;
}

// ============================================
// SUPPLIER ENDPOINTS
// ============================================

/**
 * POST /api/v1/inventory/suppliers
 * Create supplier
 */
Request: {
  name: string;
  contactPerson?: string;
  phone?: string;
  email?: string;
  address?: string;
}
Response: {
  success: true;
  data: Supplier;
}

/**
 * GET /api/v1/inventory/suppliers
 * Get all suppliers
 */
Response: {
  success: true;
  data: Supplier[];
}

/**
 * PATCH /api/v1/inventory/suppliers/:supplierId
 * Update supplier
 */
Request: Partial<Supplier>
Response: {
  success: true;
  data: Supplier;
}
```

### Customer Endpoints

```typescript
/**
 * POST /api/v1/customers
 * Create customer
 */
Request: {
  name: string;
  phone: string;
  email?: string;
  address?: string;
  dateOfBirth?: string;
}
Response: {
  success: true;
  data: Customer;
}

/**
 * GET /api/v1/customers/:customerId
 * Get customer by ID
 */
Response: {
  success: true;
  data: Customer;
}

/**
 * PATCH /api/v1/customers/:customerId
 * Update customer
 */
Request: Partial<Customer>
Response: {
  success: true;
  data: Customer;
}

/**
 * GET /api/v1/customers
 * Search customers
 */
Query Params: {
  q?: string;
  page?: number;
  limit?: number;
}
Response: {
  success: true;
  data: Customer[];
  meta: PaginationMeta;
}

/**
 * GET /api/v1/customers/phone/:phone
 * Get customer by phone
 */
Response: {
  success: true;
  data: Customer | null;
}

/**
 * GET /api/v1/customers/:customerId/loyalty
 * Get loyalty points
 */
Response: {
  success: true;
  data: {
    customerId: string;
    points: number;
    tier: string;
  };
}

/**
 * POST /api/v1/customers/:customerId/loyalty/add
 * Add loyalty points
 */
Request: {
  points: number;
  reason: string;
}
Response: {
  success: true;
  data: {
    customerId: string;
    pointsAdded: number;
    totalPoints: number;
  };
}

/**
 * POST /api/v1/customers/:customerId/loyalty/redeem
 * Redeem loyalty points
 */
Request: {
  points: number;
}
Response: {
  success: true;
  data: {
    customerId: string;
    pointsRedeemed: number;
    remainingPoints: number;
  };
}
```

### Auth Endpoints

```typescript
/**
 * POST /api/v1/auth/login
 * User login
 */
Request: {
  username: string;
  password: string;
}
Response: {
  success: true;
  data: {
    user: User;
    accessToken: string;
    refreshToken: string;
    expiresIn: number;
  };
}

/**
 * POST /api/v1/auth/logout
 * User logout
 */
Request: {
  refreshToken: string;
}
Response: {
  success: true;
  data: null;
}

/**
 * POST /api/v1/auth/refresh
 * Refresh access token
 */
Request: {
  refreshToken: string;
}
Response: {
  success: true;
  data: {
    accessToken: string;
    refreshToken: string;
    expiresIn: number;
  };
}

/**
 * POST /api/v1/auth/validate
 * Validate session token
 */
Request: {
  token: string;
}
Response: {
  success: true;
  data: {
    valid: boolean;
    user?: User;
  };
}

/**
 * POST /api/v1/auth/change-password
 * Change password
 */
Request: {
  oldPassword: string;
  newPassword: string;
}
Response: {
  success: true;
  data: null;
}

/**
 * GET /api/v1/auth/me
 * Get current user
 */
Response: {
  success: true;
  data: User;
}
```

### User Management Endpoints

```typescript
/**
 * POST /api/v1/users
 * Create user
 */
Request: {
  username: string;
  password: string;
  fullName: string;
  email?: string;
  roleId: string;
}
Response: {
  success: true;
  data: User;
}

/**
 * GET /api/v1/users
 * Get all users
 */
Response: {
  success: true;
  data: User[];
}

/**
 * GET /api/v1/users/:userId
 * Get user by ID
 */
Response: {
  success: true;
  data: User;
}

/**
 * PATCH /api/v1/users/:userId
 * Update user
 */
Request: Partial<User>
Response: {
  success: true;
  data: User;
}

/**
 * DELETE /api/v1/users/:userId
 * Delete user
 */
Response: {
  success: true;
  data: null;
}

/**
 * GET /api/v1/users/:userId/permissions
 * Get user permissions
 */
Response: {
  success: true;
  data: {
    userId: string;
    roleId: string;
    permissions: string[];
  };
}

/**
 * POST /api/v1/users/:userId/permissions/check
 * Check if user has permission
 */
Request: {
  permission: string;
}
Response: {
  success: true;
  data: {
    hasPermission: boolean;
  };
}
```

### Sync Endpoints

```typescript
/**
 * POST /api/v1/sync/trigger
 * Trigger manual sync
 */
Response: {
  success: true;
  data: SyncResult;
}

/**
 * GET /api/v1/sync/status
 * Get sync status
 */
Response: {
  success: true;
  data: SyncStatus;
}

/**
 * GET /api/v1/sync/queue
 * Get sync queue
 */
Response: {
  success: true;
  data: SyncQueueItem[];
}

/**
 * POST /api/v1/sync/queue/:itemId/retry
 * Retry sync item
 */
Response: {
  success: true;
  data: SyncQueueItem;
}

/**
 * GET /api/v1/sync/conflicts
 * Get conflicts
 */
Response: {
  success: true;
  data: Conflict[];
}

/**
 * POST /api/v1/sync/conflicts/:conflictId/resolve
 * Resolve conflict
 */
Request: {
  resolution: 'local' | 'remote' | 'merge';
  mergedData?: any;
}
Response: {
  success: true;
  data: null;
}

/**
 * GET /api/v1/sync/online
 * Check online status
 */
Response: {
  success: true;
  data: {
    online: boolean;
    lastChecked: string;
  };
}
```

### Reports Endpoints

```typescript
/**
 * GET /api/v1/reports/sales/daily
 * Get daily sales report
 */
Query Params: {
  date: string;
}
Response: {
  success: true;
  data: DailySalesReport;
}

/**
 * GET /api/v1/reports/sales/range
 * Get sales report by date range
 */
Query Params: {
  startDate: string;
  endDate: string;
}
Response: {
  success: true;
  data: SalesReport;
}

/**
 * GET /api/v1/reports/products/top-selling
 * Get top selling products
 */
Query Params: {
  limit: number;
  startDate: string;
  endDate: string;
}
Response: {
  success: true;
  data: ProductSales[];
}

/**
 * GET /api/v1/reports/dashboard
 * Get dashboard data
 */
Query Params: {
  date: string;
}
Response: {
  success: true;
  data: DashboardData;
}

/**
 * GET /api/v1/reports/inventory
 * Get inventory report
 */
Response: {
  success: true;
  data: InventoryReport;
}

/**
 * POST /api/v1/reports/export
 * Export report
 */
Request: {
  reportType: string;
  format: 'pdf' | 'csv' | 'excel';
  params: any;
}
Response: {
  success: true;
  data: {
    filePath: string;
    fileName: string;
  };
}
```

### Hardware Endpoints

```typescript
/**
 * GET /api/v1/hardware/printers
 * List printers
 */
Response: {
  success: true;
  data: Printer[];
}

/**
 * POST /api/v1/hardware/printers/:printerId/test
 * Test print
 */
Response: {
  success: true;
  data: PrintResult;
}

/**
 * GET /api/v1/hardware/printers/:printerId/status
 * Get printer status
 */
Response: {
  success: true;
  data: PrinterStatus;
}

/**
 * POST /api/v1/hardware/cash-drawer/open
 * Open cash drawer
 */
Response: {
  success: true;
  data: null;
}

/**
 * GET /api/v1/hardware/scanners
 * List scanners
 */
Response: {
  success: true;
  data: Scanner[];
}

/**
 * POST /api/v1/hardware/scanners/:scannerId/enable
 * Enable scanner
 */
Response: {
  success: true;
  data: null;
}
```

### Backup Endpoints

```typescript
/**
 * POST /api/v1/backups
 * Create backup
 */
Request: {
  name?: string;
}
Response: {
  success: true;
  data: Backup;
}

/**
 * GET /api/v1/backups
 * List backups
 */
Response: {
  success: true;
  data: Backup[];
}

/**
 * POST /api/v1/backups/:backupId/restore
 * Restore backup
 */
Response: {
  success: true;
  data: null;
}

/**
 * POST /api/v1/backups/:backupId/upload
 * Upload backup to cloud
 */
Response: {
  success: true;
  data: {
    cloudBackupId: string;
    uploadedAt: string;
  };
}

/**
 * POST /api/v1/backups/export
 * Export data
 */
Request: {
  format: 'json' | 'csv';
  tables?: string[];
}
Response: {
  success: true;
  data: {
    filePath: string;
    fileName: string;
  };
}
```

### License Endpoints

```typescript
/**
 * POST /api/v1/license/activate
 * Activate license
 */
Request: {
  licenseKey: string;
}
Response: {
  success: true;
  data: {
    activated: boolean;
    licenseInfo: LicenseInfo;
  };
}

/**
 * GET /api/v1/license/status
 * Get license status
 */
Response: {
  success: true;
  data: LicenseStatus;
}

/**
 * POST /api/v1/license/validate
 * Validate license
 */
Response: {
  success: true;
  data: LicenseStatus;
}

/**
 * GET /api/v1/license/features
 * Get enabled features
 */
Response: {
  success: true;
  data: {
    features: string[];
  };
}

/**
 * GET /api/v1/license/features/:feature
 * Check if feature is enabled
 */
Response: {
  success: true;
  data: {
    feature: string;
    enabled: boolean;
  };
}
```

### Config Endpoints

```typescript
/**
 * GET /api/v1/config/store
 * Get store configuration
 */
Response: {
  success: true;
  data: StoreConfig;
}

/**
 * PATCH /api/v1/config/store
 * Update store configuration
 */
Request: Partial<StoreConfig>
Response: {
  success: true;
  data: StoreConfig;
}

/**
 * GET /api/v1/config/system
 * Get system configuration
 */
Response: {
  success: true;
  data: SystemConfig;
}

/**
 * PATCH /api/v1/config/system
 * Update system configuration
 */
Request: Partial<SystemConfig>
Response: {
  success: true;
  data: SystemConfig;
}

/**
 * GET /api/v1/config/receipt-template
 * Get receipt template
 */
Response: {
  success: true;
  data: ReceiptTemplate;
}

/**
 * PUT /api/v1/config/receipt-template
 * Update receipt template
 */
Request: ReceiptTemplate
Response: {
  success: true;
  data: ReceiptTemplate;
}

/**
 * GET /api/v1/config/tax-rates
 * Get tax rates
 */
Response: {
  success: true;
  data: TaxRate[];
}

/**
 * PUT /api/v1/config/tax-rates
 * Update tax rates
 */
Request: TaxRate[]
Response: {
  success: true;
  data: TaxRate[];
}
```

---

## Database Repository Contracts

### Base Repository Interface

```typescript
interface IRepository<T, ID = string> {
  create(entity: Omit<T, 'id' | 'createdAt' | 'updatedAt'>): Promise<T>;
  update(id: ID, updates: Partial<T>): Promise<T>;
  delete(id: ID): Promise<void>;
  findById(id: ID): Promise<T | null>;
  findAll(options?: FindOptions): Promise<T[]>;
  count(filter?: any): Promise<number>;
}

interface FindOptions {
  where?: any;
  orderBy?: { field: string; direction: 'asc' | 'desc' };
  limit?: number;
  offset?: number;
}
```

### Sales Repository

```typescript
interface ISalesRepository extends IRepository<Sale> {
  findByDateRange(startDate: Date, endDate: Date): Promise<Sale[]>;
  findByUser(userId: string): Promise<Sale[]>;
  findByCustomer(customerId: string): Promise<Sale[]>;
  getDailySummary(date: Date): Promise<SalesSummary>;
  getTotalSales(startDate: Date, endDate: Date): Promise<number>;
  voidSale(saleId: string, reason: string, userId: string): Promise<Sale>;
}

interface ISaleItemRepository extends IRepository<SaleItem> {
  findBySale(saleId: string): Promise<SaleItem[]>;
  findByProduct(productId: string, startDate?: Date, endDate?: Date): Promise<SaleItem[]>;
}

interface IPaymentRepository extends IRepository<Payment> {
  findBySale(saleId: string): Promise<Payment[]>;
  getTotalByMethod(method: PaymentMethod, startDate: Date, endDate: Date): Promise<number>;
}
```

### Product Repository

```typescript
interface IProductRepository extends IRepository<Product> {
  findByBarcode(barcode: string): Promise<Product | null>;
  findBySku(sku: string): Promise<Product | null>;
  search(query: string, filters?: ProductFilters): Promise<Product[]>;
  findByCategory(categoryId: string): Promise<Product[]>;
  findLowStock(threshold?: number): Promise<Product[]>;
  updateStock(productId: string, quantity: number): Promise<Product>;
  bulkUpdatePrices(updates: Array<{ id: string; price: number }>): Promise<void>;
}
```

### Customer Repository

```typescript
interface ICustomerRepository extends IRepository<Customer> {
  findByPhone(phone: string): Promise<Customer | null>;
  findByEmail(email: string): Promise<Customer | null>;
  search(query: string): Promise<Customer[]>;
  getTopCustomers(limit: number, startDate: Date, endDate: Date): Promise<Customer[]>;
}
```

### User Repository

```typescript
interface IUserRepository extends IRepository<User> {
  findByUsername(username: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  updatePassword(userId: string, passwordHash: string): Promise<void>;
  updateLastLogin(userId: string): Promise<void>;
}
```

### Sync Queue Repository

```typescript
interface ISyncQueueRepository extends IRepository<SyncQueueItem> {
  findPending(limit?: number): Promise<SyncQueueItem[]>;
  markAsSynced(itemId: string): Promise<void>;
  markAsFailed(itemId: string, error: string): Promise<void>;
  incrementRetryCount(itemId: string): Promise<void>;
  clearSynced(olderThan?: Date): Promise<void>;
}
```

---

## Hardware Driver Contracts

### Printer Driver Interface

```typescript
interface IPrinterDriver {
  // Connection management
  connect(config: PrinterConfig): Promise<void>;
  disconnect(): Promise<void>;
  isConnected(): boolean;
  
  // Printing operations
  print(commands: PrintCommand[]): Promise<PrintResult>;
  printRawText(text: string): Promise<PrintResult>;
  printReceipt(receipt: Receipt): Promise<PrintResult>;
  
  // Printer control
  getStatus(): Promise<PrinterStatus>;
  cut(): Promise<void>;
  feedPaper(lines: number): Promise<void>;
  openCashDrawer(): Promise<void>;
  
  // Configuration
  getCapabilities(): PrinterCapabilities;
  setConfig(config: Partial<PrinterConfig>): Promise<void>;
}

interface PrintCommand {
  type: 'text' | 'barcode' | 'qrcode' | 'image' | 'cut' | 'feed';
  data: any;
  style?: TextStyle;
}

interface TextStyle {
  bold?: boolean;
  underline?: boolean;
  align?: 'left' | 'center' | 'right';
  size?: 'normal' | 'large' | 'xlarge';
  font?: 'A' | 'B';
}

interface PrinterStatus {
  online: boolean;
  paperPresent: boolean;
  coverClosed: boolean;
  error?: string;
}

interface PrinterCapabilities {
  maxWidth: number;
  supportsBarcodes: boolean;
  supportsQRCodes: boolean;
  supportsImages: boolean;
  supportsCashDrawer: boolean;
}

interface PrintResult {
  success: boolean;
  error?: string;
  timestamp: Date;
}
```

### Scanner Driver Interface

```typescript
interface IScannerDriver {
  // Connection management
  connect(config: ScannerConfig): Promise<void>;
  disconnect(): Promise<void>;
  isConnected(): boolean;
  
  // Scanning operations
  startScanning(): Promise<void>;
  stopScanning(): Promise<void>;
  isScanning(): boolean;
  
  // Event handling
  onScan(callback: (barcode: string, type: BarcodeType) => void): Unsubscribe;
  
  // Configuration
  setConfig(config: Partial<ScannerConfig>): Promise<void>;
  getConfig(): ScannerConfig;
}

type BarcodeType = 'EAN13' | 'EAN8' | 'UPC' | 'CODE39' | 'CODE128' | 'QR' | 'UNKNOWN';

interface ScannerConfig {
  port?: string;
  baudRate?: number;
  enabledTypes?: BarcodeType[];
  beep?: boolean;
  autoStart?: boolean;
}
```

### Biometric Driver Interface

```typescript
interface IBiometricDriver {
  // Connection management
  connect(deviceId?: string): Promise<void>;
  disconnect(): Promise<void>;
  isConnected(): boolean;
  
  // Capture operations
  captureFingerprint(timeout?: number): Promise<BiometricData>;
  
  // Verification operations
  verifyFingerprint(template: BiometricTemplate, data: BiometricData): Promise<boolean>;
  matchFingerprint(data: BiometricData, templates: BiometricTemplate[]): Promise<string | null>;
  
  // Template management
  extractTemplate(data: BiometricData): Promise<BiometricTemplate>;
  
  // Device info
  getDeviceInfo(): Promise<BiometricDeviceInfo>;
  getStatus(): Promise<BiometricDeviceStatus>;
}

interface BiometricData {
  rawData: Buffer;
  quality: number; // 0-100
  capturedAt: Date;
}

interface BiometricTemplate {
  id: string;
  userId: string;
  template: Buffer;
  quality: number;
  createdAt: Date;
}

interface BiometricDeviceInfo {
  deviceId: string;
  manufacturer: string;
  model: string;
  serialNumber: string;
}

interface BiometricDeviceStatus {
  connected: boolean;
  ready: boolean;
  error?: string;
}
```

---

## Supabase Cloud Integration Contracts

### Supabase Client Configuration

```typescript
interface SupabaseConfig {
  url: string;
  anonKey: string;
  serviceRoleKey?: string; // For admin operations
}

const supabase = createClient<Database>(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_ANON_KEY
);
```

### Database Tables (Cloud)

```typescript
// Supabase schema mirrors local schema (for synced tables)
interface Database {
  public: {
    Tables: {
      sales: {
        Row: Sale;
        Insert: Omit<Sale, 'id' | 'createdAt' | 'updatedAt'>;
        Update: Partial<Sale>;
      };
      products: {
        Row: Product;
        Insert: Omit<Product, 'id' | 'createdAt' | 'updatedAt'>;
        Update: Partial<Product>;
      };
      customers: {
        Row: Customer;
        Insert: Omit<Customer, 'id' | 'createdAt' | 'updatedAt'>;
        Update: Partial<Customer>;
      };
      // ... other tables
    };
  };
}
```

### Auth Integration

```typescript
interface SupabaseAuthService {
  // License validation (admin-only endpoint)
  validateLicense(licenseKey: string, machineId: string): Promise<LicenseValidationResult>;
  
  // API key authentication for desktop client
  authenticateWithApiKey(apiKey: string): Promise<{ user: User; session: Session }>;
}

interface LicenseValidationResult {
  valid: boolean;
  licenseInfo?: {
    licenseKey: string;
    storeId: string;
    storeName: string;
    plan: 'basic' | 'pro' | 'enterprise';
    features: string[];
    expiresAt: Date;
    maxTerminals: number;
  };
  error?: string;
}
```

### Sync Service

```typescript
interface SupabaseSyncService {
  // Batch upload operations
  uploadSales(sales: Sale[]): Promise<SyncResult>;
  uploadProducts(products: Product[]): Promise<SyncResult>;
  uploadCustomers(customers: Customer[]): Promise<SyncResult>;
  
  // Batch download operations
  downloadProducts(lastSyncAt: Date): Promise<Product[]>;
  downloadCustomers(lastSyncAt: Date): Promise<Customer[]>;
  
  // Realtime subscriptions
  subscribeToProductChanges(callback: (change: RealtimeChange<Product>) => void): RealtimeChannel;
  subscribeToCustomerChanges(callback: (change: RealtimeChange<Customer>) => void): RealtimeChannel;
}

interface RealtimeChange<T> {
  eventType: 'INSERT' | 'UPDATE' | 'DELETE';
  table: string;
  new: T | null;
  old: T | null;
  timestamp: Date;
}

interface SyncResult {
  success: boolean;
  uploaded: number;
  failed: number;
  errors?: Array<{ id: string; error: string }>;
}
```

### Storage Service

```typescript
interface SupabaseStorageService {
  // Backup operations
  uploadBackup(backupFile: File, metadata: BackupMetadata): Promise<string>;
  downloadBackup(backupId: string): Promise<Blob>;
  listBackups(storeId: string): Promise<CloudBackup[]>;
  deleteBackup(backupId: string): Promise<void>;
  
  // Asset operations (product images, etc.)
  uploadAsset(file: File, path: string): Promise<string>;
  getAssetUrl(path: string): Promise<string>;
  deleteAsset(path: string): Promise<void>;
}

interface BackupMetadata {
  storeId: string;
  backupName: string;
  size: number;
  checksum: string;
  createdAt: Date;
}
```

---

## Shared Type Definitions

### Core Business Types

```typescript
// ============================================
// SALES TYPES
// ============================================

interface Cart {
  id: string;
  items: CartItem[];
  subtotal: number;
  discount: number;
  tax: number;
  total: number;
  createdAt: Date;
  updatedAt: Date;
}

interface CartItem {
  id: string;
  productId: string;
  product: Product;
  quantity: number;
  price: number;
  discount: number;
  total: number;
}

interface Sale {
  id: string;
  storeId: string;
  terminalId: string;
  userId: string;
  customerId?: string;
  items: SaleItem[];
  payments: Payment[];
  subtotal: number;
  discount: number;
  tax: number;
  total: number;
  status: SaleStatus;
  voidReason?: string;
  createdAt: Date;
  updatedAt: Date;
  syncedAt?: Date;
}

type SaleStatus = 'completed' | 'voided' | 'returned';

interface SaleItem {
  id: string;
  saleId: string;
  productId: string;
  productName: string;
  productSku: string;
  quantity: number;
  price: number;
  discount: number;
  tax: number;
  total: number;
}

interface Payment {
  id: string;
  saleId: string;
  method: PaymentMethod;
  amount: number;
  reference?: string;
  createdAt: Date;
}

type PaymentMethod = 'cash' | 'card' | 'mobile' | 'credit' | 'other';

interface SaleTotal {
  subtotal: number;
  discount: number;
  taxBreakdown: TaxBreakdown[];
  totalTax: number;
  total: number;
  change?: number; // If payment > total
}

interface TaxBreakdown {
  name: string;
  rate: number;
  amount: number;
}

interface Return {
  id: string;
  saleId: string;
  items: ReturnItem[];
  refundMethod: PaymentMethod;
  refundAmount: number;
  reason: string;
  createdAt: Date;
}

interface ReturnItem {
  saleItemId: string;
  quantity: number;
  reason: string;
}

interface SalesSummary {
  date: Date;
  totalSales: number;
  totalTransactions: number;
  averageTransactionValue: number;
  totalDiscount: number;
  totalTax: number;
  paymentBreakdown: PaymentBreakdown[];
}

interface PaymentBreakdown {
  method: PaymentMethod;
  count: number;
  total: number;
}

// ============================================
// INVENTORY TYPES
// ============================================

interface Product {
  id: string;
  storeId: string;
  name: string;
  sku: string;
  barcode?: string;
  categoryId: string;
  category?: Category;
  price: number;
  cost?: number;
  stock: number;
  lowStockThreshold?: number;
  description?: string;
  imageUrl?: string;
  active: boolean;
  createdAt: Date;
  updatedAt: Date;
  syncedAt?: Date;
}

interface ProductFilters {
  categoryId?: string;
  minPrice?: number;
  maxPrice?: number;
  inStock?: boolean;
}

interface Category {
  id: string;
  name: string;
  description?: string;
  parentCategoryId?: string;
  createdAt: Date;
  updatedAt: Date;
}

interface StockMovement {
  id: string;
  productId: string;
  quantity: number; // Positive = add, negative = subtract
  reason: string;
  reference?: string;
  userId: string;
  createdAt: Date;
}

interface Supplier {
  id: string;
  name: string;
  contactPerson?: string;
  phone?: string;
  email?: string;
  address?: string;
  createdAt: Date;
  updatedAt: Date;
}

interface PurchaseOrder {
  id: string;
  supplierId: string;
  items: PurchaseOrderItem[];
  status: POStatus;
  total: number;
  receivedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

type POStatus = 'pending' | 'received' | 'cancelled';

interface PurchaseOrderItem {
  id: string;
  productId: string;
  quantity: number;
  cost: number;
  receivedQuantity: number;
}

// ============================================
// CUSTOMER TYPES
// ============================================

interface Customer {
  id: string;
  storeId: string;
  name: string;
  phone: string;
  email?: string;
  address?: string;
  dateOfBirth?: Date;
  loyaltyPoints: number;
  loyaltyTier: LoyaltyTier;
  totalSpent: number;
  totalVisits: number;
  createdAt: Date;
  updatedAt: Date;
  syncedAt?: Date;
}

type LoyaltyTier = 'bronze' | 'silver' | 'gold' | 'platinum';

interface CustomerAccount {
  customerId: string;
  creditLimit: number;
  balance: number;
  lastPaymentAt?: Date;
}

interface LoyaltyTransaction {
  id: string;
  customerId: string;
  points: number;
  type: 'earn' | 'redeem';
  reason: string;
  saleId?: string;
  createdAt: Date;
}

// ============================================
// USER & AUTH TYPES
// ============================================

interface User {
  id: string;
  username: string;
  fullName: string;
  email?: string;
  roleId: string;
  role?: Role;
  active: boolean;
  lastLoginAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

interface Role {
  id: string;
  name: string;
  description?: string;
  permissions: string[];
}

interface Permission {
  id: string;
  name: string;
  description?: string;
  resource: string;
  action: string;
}

interface Session {
  id: string;
  userId: string;
  token: string;
  expiresAt: Date;
  createdAt: Date;
}

interface AuthResult {
  user: User;
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
}

// ============================================
// SYNC TYPES
// ============================================

interface SyncQueueItem {
  id: string;
  entityType: string;
  entityId: string;
  operation: SyncOperation;
  payload: string; // JSON string
  priority: number;
  syncStatus: SyncStatus;
  retryCount: number;
  lastRetryAt?: Date;
  errorMessage?: string;
  createdAt: Date;
}

type SyncOperation = 'CREATE' | 'UPDATE' | 'DELETE';
type SyncStatus = 'pending' | 'syncing' | 'synced' | 'failed';

interface SyncResult {
  success: boolean;
  itemsSynced: number;
  itemsFailed: number;
  errors?: Array<{ id: string; error: string }>;
  duration: number;
  timestamp: Date;
}

interface SyncStatusInfo {
  lastSyncAt?: Date;
  nextSyncAt?: Date;
  isSyncing: boolean;
  pendingItems: number;
  failedItems: number;
}

interface Conflict {
  id: string;
  entityType: string;
  entityId: string;
  localVersion: string;
  remoteVersion: string;
  baseVersion?: string;
  resolutionStatus: 'pending' | 'resolved';
  createdAt: Date;
}

interface ConflictResolution {
  strategy: 'local' | 'remote' | 'merge';
  mergedData?: any;
}

interface ConnectionStatus {
  online: boolean;
  lastChecked: Date;
  supabaseReachable: boolean;
  latency?: number;
}

interface SyncConfig {
  autoSync: boolean;
  syncInterval: number; // seconds
  batchSize: number;
  maxRetries: number;
  retryBackoff: number; // seconds
}

// ============================================
// HARDWARE TYPES
// ============================================

interface Printer {
  id: string;
  name: string;
  type: 'thermal' | 'impact' | 'laser';
  connection: 'usb' | 'serial' | 'network';
  port?: string;
  ipAddress?: string;
  isDefault: boolean;
}

interface Receipt {
  saleId: string;
  storeName: string;
  storeAddress: string;
  storePhone: string;
  terminalId: string;
  cashierName: string;
  items: ReceiptItem[];
  subtotal: number;
  discount: number;
  tax: number;
  total: number;
  payments: ReceiptPayment[];
  change?: number;
  customerName?: string;
  loyaltyPoints?: number;
  footer?: string;
  timestamp: Date;
}

interface ReceiptItem {
  name: string;
  quantity: number;
  price: number;
  total: number;
}

interface ReceiptPayment {
  method: string;
  amount: number;
}

interface Scanner {
  id: string;
  name: string;
  type: 'usb' | 'serial' | 'bluetooth';
  port?: string;
  enabled: boolean;
}

interface BiometricDevice {
  id: string;
  name: string;
  manufacturer: string;
  model: string;
  connected: boolean;
}

interface CashDrawerStatus {
  open: boolean;
  lastOpenedAt?: Date;
}

interface DeviceConfig {
  deviceId: string;
  deviceType: 'printer' | 'scanner' | 'biometric' | 'cash_drawer';
  config: any;
}

// ============================================
// BACKUP TYPES
// ============================================

interface Backup {
  id: string;
  name: string;
  filePath: string;
  size: number;
  checksum: string;
  verified: boolean;
  type: 'auto' | 'manual' | 'pre-migration';
  createdAt: Date;
}

interface CloudBackup {
  id: string;
  storeId: string;
  name: string;
  size: number;
  checksum: string;
  uploadedAt: Date;
}

interface BackupSchedule {
  enabled: boolean;
  frequency: 'daily' | 'weekly';
  time: string; // HH:MM format
  retentionDays: number;
  uploadToCloud: boolean;
}

interface ExportResult {
  filePath: string;
  fileName: string;
  format: 'json' | 'csv';
  recordCount: number;
  size: number;
}

interface ImportResult {
  success: boolean;
  recordsImported: number;
  recordsFailed: number;
  errors?: Array<{ row: number; error: string }>;
}

// ============================================
// LICENSE TYPES
// ============================================

interface LicenseInfo {
  licenseKey: string;
  storeId: string;
  storeName: string;
  plan: 'basic' | 'pro' | 'enterprise';
  features: string[];
  activatedAt: Date;
  expiresAt: Date;
  maxTerminals: number;
}

interface LicenseStatus {
  valid: boolean;
  mode: 'online' | 'offline' | 'expired';
  licenseInfo?: LicenseInfo;
  gracePeriod?: GracePeriodStatus;
  lastValidatedAt: Date;
}

interface GracePeriodStatus {
  active: boolean;
  daysRemaining: number;
  expiresAt: Date;
  warning?: string;
}

interface ActivationResult {
  success: boolean;
  licenseInfo?: LicenseInfo;
  error?: string;
}

// ============================================
// REPORT TYPES
// ============================================

interface DailySalesReport {
  date: Date;
  totalSales: number;
  totalTransactions: number;
  averageTransactionValue: number;
  totalDiscount: number;
  totalTax: number;
  paymentBreakdown: PaymentBreakdown[];
  hourlySales: HourlySales[];
}

interface HourlySales {
  hour: number;
  sales: number;
  transactions: number;
}

interface SalesReport {
  startDate: Date;
  endDate: Date;
  totalSales: number;
  totalTransactions: number;
  averageTransactionValue: number;
  totalDiscount: number;
  totalTax: number;
  dailyBreakdown: DailySalesReport[];
}

interface ProductSales {
  productId: string;
  productName: string;
  quantitySold: number;
  totalSales: number;
  averagePrice: number;
}

interface CustomerSales {
  customerId: string;
  customerName: string;
  totalSpent: number;
  totalVisits: number;
  averageTransactionValue: number;
}

interface InventoryReport {
  totalProducts: number;
  totalValue: number;
  lowStockCount: number;
  outOfStockCount: number;
  categories: CategoryInventory[];
}

interface CategoryInventory {
  categoryId: string;
  categoryName: string;
  productCount: number;
  totalValue: number;
}

interface DashboardData {
  date: Date;
  todaySales: number;
  todayTransactions: number;
  yesterdaySales: number;
  monthToDateSales: number;
  lowStockCount: number;
  pendingSyncItems: number;
  topSellingProducts: ProductSales[];
  recentSales: Sale[];
}

// ============================================
// CONFIG TYPES
// ============================================

interface StoreConfig {
  storeId: string;
  storeName: string;
  address: string;
  phone: string;
  email?: string;
  taxId?: string;
  currency: string;
  timezone: string;
  locale: string;
}

interface SystemConfig {
  autoBackup: boolean;
  backupTime: string;
  backupRetentionDays: number;
  lowStockThreshold: number;
  allowNegativeStock: boolean;
  requireCustomerForSale: boolean;
  printReceiptAutomatically: boolean;
  defaultPaymentMethod: PaymentMethod;
  theme: 'light' | 'dark';
  language: string;
}

interface TaxRate {
  id: string;
  name: string;
  rate: number;
  type: 'percentage' | 'fixed';
  active: boolean;
}

interface ReceiptTemplate {
  header: string;
  footer: string;
  showLogo: boolean;
  logoUrl?: string;
  showBarcode: boolean;
  fontSize: 'small' | 'medium' | 'large';
}

interface CurrencyConfig {
  code: string; // PKR
  symbol: string; // ₨
  decimalPlaces: number;
  thousandsSeparator: string;
  decimalSeparator: string;
}

// ============================================
// UTILITY TYPES
// ============================================

interface PaginationMeta {
  page: number;
  limit: number;
  total: number;
  totalPages: number;
}

interface DateRange {
  startDate: Date;
  endDate: Date;
}

type Unsubscribe = () => void;
```

---

## Error Handling Standards

### Error Codes

```typescript
// Standardized error codes
enum ErrorCode {
  // Authentication & Authorization
  UNAUTHORIZED = 'UNAUTHORIZED',
  FORBIDDEN = 'FORBIDDEN',
  INVALID_CREDENTIALS = 'INVALID_CREDENTIALS',
  SESSION_EXPIRED = 'SESSION_EXPIRED',
  PERMISSION_DENIED = 'PERMISSION_DENIED',
  
  // Validation
  VALIDATION_ERROR = 'VALIDATION_ERROR',
  INVALID_INPUT = 'INVALID_INPUT',
  MISSING_REQUIRED_FIELD = 'MISSING_REQUIRED_FIELD',
  
  // Resource
  RESOURCE_NOT_FOUND = 'RESOURCE_NOT_FOUND',
  RESOURCE_ALREADY_EXISTS = 'RESOURCE_ALREADY_EXISTS',
  RESOURCE_CONFLICT = 'RESOURCE_CONFLICT',
  
  // Business Logic
  INSUFFICIENT_STOCK = 'INSUFFICIENT_STOCK',
  INVALID_QUANTITY = 'INVALID_QUANTITY',
  SALE_ALREADY_VOIDED = 'SALE_ALREADY_VOIDED',
  CANNOT_RETURN_VOIDED_SALE = 'CANNOT_RETURN_VOIDED_SALE',
  PAYMENT_TOTAL_MISMATCH = 'PAYMENT_TOTAL_MISMATCH',
  INSUFFICIENT_LOYALTY_POINTS = 'INSUFFICIENT_LOYALTY_POINTS',
  
  // Hardware
  PRINTER_NOT_FOUND = 'PRINTER_NOT_FOUND',
  PRINTER_OFFLINE = 'PRINTER_OFFLINE',
  PRINTER_ERROR = 'PRINTER_ERROR',
  SCANNER_NOT_FOUND = 'SCANNER_NOT_FOUND',
  BIOMETRIC_DEVICE_ERROR = 'BIOMETRIC_DEVICE_ERROR',
  
  // License
  LICENSE_INVALID = 'LICENSE_INVALID',
  LICENSE_EXPIRED = 'LICENSE_EXPIRED',
  GRACE_PERIOD_EXPIRED = 'GRACE_PERIOD_EXPIRED',
  FEATURE_NOT_ENABLED = 'FEATURE_NOT_ENABLED',
  
  // Sync
  SYNC_FAILED = 'SYNC_FAILED',
  NETWORK_ERROR = 'NETWORK_ERROR',
  CONFLICT_DETECTED = 'CONFLICT_DETECTED',
  
  // Database
  DATABASE_ERROR = 'DATABASE_ERROR',
  TRANSACTION_FAILED = 'TRANSACTION_FAILED',
  
  // System
  INTERNAL_ERROR = 'INTERNAL_ERROR',
  SERVICE_UNAVAILABLE = 'SERVICE_UNAVAILABLE',
}
```

### Error Response Format

```typescript
interface ErrorResponse {
  success: false;
  error: {
    code: ErrorCode;
    message: string;
    details?: any;
    field?: string; // For validation errors
    timestamp: string;
    requestId?: string; // For tracing
  };
}

// Example error responses
const examples = {
  validation: {
    success: false,
    error: {
      code: 'VALIDATION_ERROR',
      message: 'Invalid product data',
      details: {
        name: 'Product name is required',
        price: 'Price must be greater than 0',
      },
      timestamp: '2024-01-01T10:00:00Z',
    },
  },
  
  notFound: {
    success: false,
    error: {
      code: 'RESOURCE_NOT_FOUND',
      message: 'Product not found',
      details: { productId: 'abc123' },
      timestamp: '2024-01-01T10:00:00Z',
    },
  },
  
  insufficientStock: {
    success: false,
    error: {
      code: 'INSUFFICIENT_STOCK',
      message: 'Not enough stock available',
      details: {
        productId: 'abc123',
        requestedQuantity: 10,
        availableStock: 5,
      },
      timestamp: '2024-01-01T10:00:00Z',
    },
  },
  
  licenseExpired: {
    success: false,
    error: {
      code: 'LICENSE_EXPIRED',
      message: 'License has expired. Please renew your subscription.',
      details: {
        expiredAt: '2024-01-01T00:00:00Z',
        gracePeriodExpired: true,
      },
      timestamp: '2024-01-01T10:00:00Z',
    },
  },
};
```

### Exception Hierarchy

```typescript
// Base exception class
class AppException extends Error {
  constructor(
    public code: ErrorCode,
    public message: string,
    public details?: any,
    public httpStatus: number = 400
  ) {
    super(message);
    this.name = 'AppException';
  }
}

// Specific exception classes
class ValidationException extends AppException {
  constructor(message: string, details?: any) {
    super(ErrorCode.VALIDATION_ERROR, message, details, 400);
    this.name = 'ValidationException';
  }
}

class NotFoundException extends AppException {
  constructor(resource: string, identifier: string) {
    super(
      ErrorCode.RESOURCE_NOT_FOUND,
      `${resource} not found`,
      { identifier },
      404
    );
    this.name = 'NotFoundException';
  }
}

class UnauthorizedException extends AppException {
  constructor(message: string = 'Unauthorized') {
    super(ErrorCode.UNAUTHORIZED, message, undefined, 401);
    this.name = 'UnauthorizedException';
  }
}

class ForbiddenException extends AppException {
  constructor(message: string = 'Forbidden') {
    super(ErrorCode.FORBIDDEN, message, undefined, 403);
    this.name = 'ForbiddenException';
  }
}

class BusinessLogicException extends AppException {
  constructor(code: ErrorCode, message: string, details?: any) {
    super(code, message, details, 422);
    this.name = 'BusinessLogicException';
  }
}

class HardwareException extends AppException {
  constructor(code: ErrorCode, message: string, details?: any) {
    super(code, message, details, 503);
    this.name = 'HardwareException';
  }
}
```

---

## State Management Contracts

### Frontend State Structure

```typescript
// Global state (Zustand store)
interface AppState {
  // Auth state
  auth: {
    user: User | null;
    session: Session | null;
    isAuthenticated: boolean;
    permissions: string[];
  };
  
  // Current cart state
  cart: {
    currentCart: Cart | null;
    isLoading: boolean;
  };
  
  // Sync state
  sync: {
    status: SyncStatusInfo;
    isOnline: boolean;
    pendingCount: number;
  };
  
  // License state
  license: {
    status: LicenseStatus;
    gracePeriod: GracePeriodStatus | null;
  };
  
  // Hardware state
  hardware: {
    printers: Printer[];
    scanners: Scanner[];
    defaultPrinter: string | null;
  };
  
  // UI state
  ui: {
    theme: 'light' | 'dark';
    sidebarOpen: boolean;
    notifications: Notification[];
  };
}

// Actions
interface AppActions {
  // Auth actions
  login: (username: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  checkSession: () => Promise<boolean>;
  
  // Cart actions
  createCart: () => Promise<void>;
  addToCart: (productId: string, quantity: number) => Promise<void>;
  removeFromCart: (itemId: string) => Promise<void>;
  updateCartItem: (itemId: string, quantity: number) => Promise<void>;
  clearCart: () => Promise<void>;
  checkout: (payments: Payment[]) => Promise<Sale>;
  
  // Sync actions
  syncNow: () => Promise<void>;
  updateSyncStatus: () => Promise<void>;
  
  // License actions
  checkLicense: () => Promise<void>;
  activateLicense: (key: string) => Promise<void>;
  
  // UI actions
  showNotification: (notification: Notification) => void;
  dismissNotification: (id: string) => void;
  toggleSidebar: () => void;
  setTheme: (theme: 'light' | 'dark') => void;
}

// Combined store type
type AppStore = AppState & AppActions;
```

### React Query Keys

```typescript
// Query key factory for consistent cache management
const queryKeys = {
  // Products
  products: {
    all: ['products'] as const,
    lists: () => [...queryKeys.products.all, 'list'] as const,
    list: (filters: ProductFilters) => [...queryKeys.products.lists(), filters] as const,
    details: () => [...queryKeys.products.all, 'detail'] as const,
    detail: (id: string) => [...queryKeys.products.details(), id] as const,
    search: (query: string) => [...queryKeys.products.all, 'search', query] as const,
  },
  
  // Sales
  sales: {
    all: ['sales'] as const,
    lists: () => [...queryKeys.sales.all, 'list'] as const,
    list: (filters: { startDate?: Date; endDate?: Date }) => 
      [...queryKeys.sales.lists(), filters] as const,
    details: () => [...queryKeys.sales.all, 'detail'] as const,
    detail: (id: string) => [...queryKeys.sales.details(), id] as const,
  },
  
  // Customers
  customers: {
    all: ['customers'] as const,
    lists: () => [...queryKeys.customers.all, 'list'] as const,
    list: (query?: string) => [...queryKeys.customers.lists(), query] as const,
    details: () => [...queryKeys.customers.all, 'detail'] as const,
    detail: (id: string) => [...queryKeys.customers.details(), id] as const,
  },
  
  // Reports
  reports: {
    all: ['reports'] as const,
    daily: (date: Date) => [...queryKeys.reports.all, 'daily', date] as const,
    range: (startDate: Date, endDate: Date) => 
      [...queryKeys.reports.all, 'range', startDate, endDate] as const,
    dashboard: (date: Date) => [...queryKeys.reports.all, 'dashboard', date] as const,
  },
  
  // Config
  config: {
    all: ['config'] as const,
    store: () => [...queryKeys.config.all, 'store'] as const,
    system: () => [...queryKeys.config.all, 'system'] as const,
  },
  
  // Sync
  sync: {
    all: ['sync'] as const,
    status: () => [...queryKeys.sync.all, 'status'] as const,
    queue: () => [...queryKeys.sync.all, 'queue'] as const,
  },
};
```

---

## Conclusion

This API contracts document defines the **complete interface specification** for all modules in the FlexPOS system. These contracts serve as:

1. **Implementation Guide**: Clear specifications for developers implementing each module
2. **Integration Reference**: Type-safe contracts for communication between layers
3. **Testing Foundation**: Well-defined inputs and outputs for unit and integration tests
4. **Documentation**: Self-documenting TypeScript types and interfaces

### Next Steps

1. **Review & Approve** this contracts document
2. **Generate OpenAPI Spec** from NestJS REST API contracts
3. **Create mock implementations** for rapid frontend development
4. **Proceed to Phase 1 Task 3**: Data Flow Diagrams (Critical Flows)

### Key Benefits

- ✅ **Type Safety**: End-to-end TypeScript typing across all layers
- ✅ **Clear Boundaries**: Well-defined module responsibilities
- ✅ **Consistent APIs**: Standardized request/response formats
- ✅ **Error Handling**: Comprehensive error codes and responses
- ✅ **Testability**: Clear contracts enable easy mocking and testing
- ✅ **Maintainability**: Single source of truth for interfaces
