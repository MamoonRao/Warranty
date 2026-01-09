# FlexPOS System Architecture

**Version:** 1.0  
**Last Updated:** 2024  
**Status:** Phase 1 - Architecture Design  
**Target Platform:** Windows 10/11  

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [High-Level Architecture](#high-level-architecture)
3. [Layered Architecture](#layered-architecture)
4. [Component Overview](#component-overview)
5. [Offline-First Architecture Pattern](#offline-first-architecture-pattern)
6. [Data Isolation & Boundaries](#data-isolation--boundaries)
7. [Technology Stack](#technology-stack)
8. [Performance Characteristics](#performance-characteristics)
9. [Architectural Principles Validation](#architectural-principles-validation)
10. [Deployment Architecture](#deployment-architecture)
11. [Security Architecture](#security-architecture)
12. [Scalability & Future Considerations](#scalability--future-considerations)

---

## Executive Summary

FlexPOS is an **offline-first** Point of Sale (POS) SaaS solution designed specifically for Pakistani Small and Medium Businesses (SMBs) running on Windows platforms. The system prioritizes local-first operation, ensuring zero data loss and sub-second transaction performance regardless of internet connectivity.

### Core Design Principles

- **Offline-First**: Internet connectivity is optional; all core POS operations work without internet
- **Zero Data Loss**: Transaction-safe operations with automated conflict resolution
- **Enterprise Reliability**: Windows service-level stability with automatic recovery
- **Performance-First**: <1 second transaction completion time
- **Clean Architecture**: Strict separation of concerns across all layers
- **Security by Design**: Local data encryption with SQLCipher, secure cloud sync

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FLEXPOS SYSTEM ARCHITECTURE                          │
│                           (Windows 10/11 Platform)                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                              USER INTERFACE LAYER                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Tauri Desktop Application                         │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │              React Frontend (TypeScript)                      │  │   │
│  │  │  • POS Terminal UI      • Inventory Management               │  │   │
│  │  │  • Sales Reports        • Customer Management                │  │   │
│  │  │  • Settings & Config    • User Management                    │  │   │
│  │  └──────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ IPC (Inter-Process Communication)
                                    │ Tauri Commands & Events
┌───────────────────────────────────┴─────────────────────────────────────────┐
│                         APPLICATION LAYER (Rust)                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      Tauri Backend (Rust)                            │   │
│  │  • Window Management        • Hardware Abstraction                  │   │
│  │  • System Tray Service      • OS Integration                        │   │
│  │  • Auto-start Configuration • File System Access                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ REST API / WebSocket
                                    │ http://localhost:3000
┌───────────────────────────────────┴─────────────────────────────────────────┐
│                      BUSINESS LOGIC LAYER (NestJS)                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │               Local NestJS Backend Service (Node.js)                 │   │
│  │                                                                       │   │
│  │  ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────┐   │   │
│  │  │  Sales Module   │  │ Inventory Module │  │  Sync Module    │   │   │
│  │  │  • Transactions │  │ • Stock Mgmt     │  │ • Queue Manager │   │   │
│  │  │  • Receipts     │  │ • Categories     │  │ • Conflict Res. │   │   │
│  │  │  • Discounts    │  │ • Suppliers      │  │ • Retry Logic   │   │   │
│  │  └─────────────────┘  └──────────────────┘  └─────────────────┘   │   │
│  │                                                                       │   │
│  │  ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────┐   │   │
│  │  │   Auth Module   │  │ Hardware Module  │  │  Backup Module  │   │   │
│  │  │  • Local Auth   │  │ • Printer Driver │  │ • Auto Backup   │   │   │
│  │  │  • Permissions  │  │ • Scanner Driver │  │ • Export/Import │   │   │
│  │  │  • License Val. │  │ • Biometric API  │  │ • Restore       │   │   │
│  │  └─────────────────┘  └──────────────────┘  └─────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ ORM (TypeORM / Prisma)
                                    │ Repository Pattern
┌───────────────────────────────────┴─────────────────────────────────────────┐
│                              DOMAIN LAYER                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      Core Business Entities                          │   │
│  │  • Sale           • Product         • Customer                      │   │
│  │  • SaleItem       • Category        • User                          │   │
│  │  • Payment        • Supplier        • Store                         │   │
│  │  • Invoice        • StockMovement   • License                       │   │
│  │                                                                       │   │
│  │                    Business Rules & Validations                      │   │
│  │  • Price Calculation    • Stock Validation                          │   │
│  │  • Discount Rules       • Tax Computation                           │   │
│  │  • License Constraints  • Permission Checks                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ Data Access Layer
                                    │
┌───────────────────────────────────┴─────────────────────────────────────────┐
│                          INFRASTRUCTURE LAYER                                │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌─────────────────┐  │
│  │  SQLite Database     │  │  File System         │  │  Hardware I/O   │  │
│  │  ┌────────────────┐  │  │  • Local Backups     │  │  • COM Ports    │  │
│  │  │  Encrypted DB  │  │  │  • Config Files      │  │  • USB Devices  │  │
│  │  │  (SQLCipher)   │  │  │  • Log Files         │  │  • Network      │  │
│  │  └────────────────┘  │  │  • Export Data       │  │    Printers     │  │
│  │  • Transactions DB   │  │  • Receipt Templates │  │  • Biometric    │  │
│  │  • Master Data DB    │  │  • Assets/Images     │  │    Readers      │  │
│  │  • Sync Queue DB     │  │                       │  │                 │  │
│  └──────────────────────┘  └──────────────────────┘  └─────────────────┘  │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                      Windows OS Integration                         │    │
│  │  • Windows Service API    • Registry Access                        │    │
│  │  • Event Log             • Windows Authentication                  │    │
│  │  • Task Scheduler        • File System Security                    │    │
│  └────────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ HTTPS / WebSocket (When Online)
                                    │
┌───────────────────────────────────┴─────────────────────────────────────────┐
│                         SAAS INTEGRATION LAYER                               │
│                              (Cloud Services)                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      Supabase Cloud Platform                         │   │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │   │
│  │  │  PostgreSQL DB   │  │  Authentication  │  │  Storage Bucket │  │   │
│  │  │  • Central Data  │  │  • OAuth/JWT     │  │  • Backups      │  │   │
│  │  │  • Analytics     │  │  • API Keys      │  │  • Reports      │  │   │
│  │  │  • Multi-tenant  │  │  • License API   │  │  • Assets       │  │   │
│  │  └──────────────────┘  └──────────────────┘  └─────────────────┘  │   │
│  │                                                                       │   │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │   │
│  │  │  Realtime Sync   │  │  Edge Functions  │  │  Admin Portal   │  │   │
│  │  │  • Change Feeds  │  │  • Webhooks      │  │  • License Mgmt │  │   │
│  │  │  • Pub/Sub       │  │  • Data Process  │  │  • Monitoring   │  │   │
│  │  └──────────────────┘  └──────────────────┘  └─────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘

              [Internet Connection Required - Async, Non-Blocking]
```

---

## Layered Architecture

### Architecture Overview

FlexPOS follows a **Clean Architecture** pattern with strict dependency rules:
- **Dependencies flow inward** (outer layers depend on inner layers, never the reverse)
- **Domain layer is independent** of all infrastructure concerns
- **Business logic is isolated** from UI and external services

```
┌──────────────────────────────────────────────────────────────┐
│                    LAYER DEPENDENCY FLOW                      │
│                                                                │
│   UI Layer (Tauri + React)                                   │
│        │                                                       │
│        ├──→ Application Layer (Rust)                         │
│        │        │                                             │
│        │        ├──→ Business Logic Layer (NestJS)           │
│        │        │        │                                    │
│        │        │        ├──→ Domain Layer (Entities)        │
│        │        │        │        ↑                           │
│        │        │        └────────┘                           │
│        │        │                 │                           │
│        │        └─────→ Infrastructure Layer ←────────────────┤
│        │                         │                            │
│        └─────────────────────────┘                            │
│                                                                │
│   SaaS Integration Layer (Supabase) ←── Async Sync Only      │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Layer Descriptions

#### 1. UI Layer (Presentation)

**Technologies:** Tauri (Rust) + React (TypeScript) + TailwindCSS

**Responsibilities:**
- Render user interfaces for all POS operations
- Handle user input and validation
- Display real-time feedback and updates
- Manage UI state and navigation
- Call application layer commands via Tauri IPC

**Key Components:**
- POS Terminal Screen (main sales interface)
- Inventory Management UI
- Reports & Analytics Dashboard
- Customer Management
- Settings & Configuration
- User Management & Permissions

**Design Constraints:**
- **NO business logic in UI components**
- All calculations and validations delegated to backend
- UI is purely presentational and event-driven
- Optimistic UI updates with rollback capability

#### 2. Application Layer (Desktop Application Container)

**Technologies:** Tauri (Rust)

**Responsibilities:**
- Manage desktop application lifecycle
- Provide IPC bridge between React frontend and NestJS backend
- Handle system tray integration
- Manage window state and multi-window support
- Provide hardware abstraction for React frontend
- Handle OS-level integrations (auto-start, notifications)
- Manage application updates and versioning

**Key Features:**
- System tray always-on service
- Auto-start on Windows boot
- Window management (minimize to tray, multi-monitor)
- Native file dialogs and OS integration
- Secure IPC with command validation

**Design Principles:**
- Thin application layer (orchestration only)
- Delegates all business logic to NestJS backend
- Provides secure, type-safe IPC boundaries

#### 3. Business Logic Layer (Backend Service)

**Technologies:** NestJS (Node.js + TypeScript)

**Responsibilities:**
- Implement all business rules and workflows
- Process transactions and maintain consistency
- Manage offline sync queues
- Coordinate hardware integrations
- Enforce security and permissions
- Handle backup and restore operations
- Manage local database migrations

**Core Modules:**

##### Sales Module
- Transaction processing (sale creation, modification, void)
- Receipt generation and printing
- Payment processing (cash, card, mixed payments)
- Discount and promotion application
- Tax calculation (GST, sales tax)
- Return and refund processing

##### Inventory Module
- Stock management (add, update, adjust)
- Low stock alerts
- Product categorization
- Supplier management
- Purchase orders
- Stock movement tracking

##### Sync Module
- Queue management (CRUD operations pending sync)
- Conflict detection and resolution
- Retry logic with exponential backoff
- Batch upload optimization
- Connection status monitoring
- Sync status reporting

##### Auth Module
- Local authentication (offline login)
- Permission enforcement (role-based access control)
- License validation (online check + 7-day grace period)
- User session management
- Multi-user support

##### Hardware Module
- Thermal printer drivers (ESC/POS commands)
- Barcode scanner integration (USB/Serial)
- Biometric device integration (fingerprint readers)
- Cash drawer control
- Customer display support

##### Backup Module
- Automated daily backups
- Manual backup/export
- Restore operations
- Backup encryption
- Cloud backup upload (when online)

**Design Principles:**
- Transaction-safe operations (ACID compliance)
- Idempotent operations for sync reliability
- Clear module boundaries with dependency injection
- Comprehensive error handling and logging

#### 4. Domain Layer (Core Business Logic)

**Technologies:** TypeScript (Plain Objects & Classes)

**Responsibilities:**
- Define core business entities
- Implement business rules and invariants
- Provide domain services and calculations
- Enforce data integrity constraints

**Core Entities:**

```typescript
// Sales Domain
- Sale (transaction header)
- SaleItem (line item)
- Payment (payment record)
- Receipt (printed receipt)
- Return (return transaction)

// Inventory Domain
- Product (item for sale)
- Category (product classification)
- Supplier (vendor)
- PurchaseOrder (stock order)
- StockMovement (inventory change)

// Customer Domain
- Customer (buyer information)
- CustomerAccount (credit/loyalty)

// User Domain
- User (system user)
- Role (permission set)
- Permission (access right)

// Store Domain
- Store (location/outlet)
- Terminal (POS device)
- License (subscription)

// Sync Domain
- SyncQueue (pending changes)
- SyncLog (sync history)
- ConflictResolution (merge rules)
```

**Business Rules Examples:**
- Sale total must equal sum of items minus discounts plus tax
- Stock cannot go negative (configurable)
- Void transactions require manager approval
- Discounts cannot exceed maximum threshold
- License must be valid or within 7-day grace period

**Design Principles:**
- **Framework-agnostic** (no NestJS decorators)
- **Pure business logic** (no infrastructure dependencies)
- **Testable** (unit tests without mocks)
- **Immutable where possible** (value objects)

#### 5. Infrastructure Layer (Technical Implementation)

**Technologies:** SQLite + SQLCipher, Node.js File System, Windows APIs

**Responsibilities:**
- Persist data to local database
- Manage file system operations
- Interface with hardware devices
- Integrate with Windows OS services
- Implement logging and monitoring

**Key Components:**

##### SQLite Database (Encrypted)
- **Technology:** SQLite 3.x with SQLCipher 4.x
- **Encryption:** AES-256 encryption at rest
- **Location:** `%APPDATA%/FlexPOS/data/flexpos.db`
- **Backup Location:** `%APPDATA%/FlexPOS/backups/`
- **Journals:** Write-Ahead Logging (WAL) for concurrent access

**Database Schema:**
```
Main Database Tables:
├── sales
├── sale_items
├── payments
├── products
├── categories
├── customers
├── users
├── suppliers
├── stock_movements
├── sync_queue
├── sync_log
└── system_config
```

##### File System Storage
```
%APPDATA%/FlexPOS/
├── data/
│   ├── flexpos.db (main database)
│   └── flexpos.db-wal (write-ahead log)
├── backups/
│   ├── auto/ (automated backups)
│   └── manual/ (user-initiated backups)
├── logs/
│   ├── app.log
│   ├── sync.log
│   └── error.log
├── config/
│   ├── app.config.json
│   ├── printer.config.json
│   └── hardware.config.json
├── receipts/
│   └── templates/ (receipt templates)
└── exports/
    └── reports/ (exported reports)
```

##### Hardware Interfaces
- **Thermal Printers:** ESC/POS protocol over USB/Serial/Network
- **Barcode Scanners:** Keyboard wedge or USB HID
- **Biometric Devices:** Vendor SDKs (e.g., ZKTeco, Suprema)
- **Cash Drawers:** Pulse signal via printer or standalone
- **Customer Displays:** Serial or USB display protocol

##### Windows OS Integration
- **Windows Service:** Background service for always-on operation
- **Event Log:** Application logging to Windows Event Viewer
- **Registry:** License keys and system configuration
- **Task Scheduler:** Automated backup scheduling
- **Windows Authentication:** Optional domain integration

**Design Principles:**
- All database operations wrapped in transactions
- Automatic retry with exponential backoff
- Graceful degradation (e.g., print to file if printer offline)
- Comprehensive error logging

#### 6. SaaS Integration Layer (Cloud Services)

**Technologies:** Supabase (PostgreSQL, Auth, Storage, Realtime)

**Responsibilities:**
- Centralized multi-tenant data storage
- License validation and subscription management
- Admin portal backend
- Analytics and reporting aggregation
- Remote backup storage
- Cross-store data sync (for multi-location businesses)

**Supabase Services:**

##### PostgreSQL Database (Cloud)
- Central store for synced data
- Multi-tenant with Row-Level Security (RLS)
- Aggregated analytics and reports
- License and subscription records

##### Authentication Service
- OAuth 2.0 / JWT authentication
- API key management for desktop clients
- License validation API
- User management for admin portal

##### Storage Bucket
- Cloud backup storage
- Report archives
- Receipt template library
- Product images (synced to local cache)

##### Realtime Service
- Real-time sync notifications
- Multi-user conflict detection
- Live dashboard updates for admin portal

##### Edge Functions
- Webhook receivers for payment gateways
- Data transformation and validation
- Scheduled jobs (e.g., license expiry notifications)
- Custom business logic for cloud operations

**Design Principles:**
- **Async and non-blocking** (never blocks local operations)
- **Optional dependency** (system works without cloud)
- **Secure communication** (HTTPS, API keys, JWT)
- **Efficient sync** (delta sync, batching, compression)

---

## Component Overview

### 1. Desktop Application Container (Tauri + React)

**Purpose:** Cross-platform desktop application shell providing native OS integration and a modern web-based UI.

**Technology Justification:**
- **Tauri:** Lightweight (vs. Electron), better performance, smaller binary size, native OS integration
- **React:** Rich ecosystem, component reusability, developer familiarity
- **TypeScript:** Type safety, better IDE support, maintainability

**Key Features:**
- Native window management
- System tray integration (run in background)
- Auto-start capability
- Native file dialogs
- Hardware access via Rust
- Update mechanism (self-updating binary)

**Communication:**
- Frontend → Backend: Tauri IPC commands (type-safe)
- Backend → Frontend: Tauri events (async pub/sub)

**Configuration:**
```json
{
  "productName": "FlexPOS",
  "identifier": "com.flexpos.desktop",
  "windows": {
    "allowedCommands": ["execute_sale", "sync_data", "print_receipt", ...],
    "minimizeToTray": true,
    "autoStart": true
  }
}
```

### 2. Local NestJS Backend Service

**Purpose:** Core business logic execution, local data management, and hardware coordination.

**Technology Justification:**
- **NestJS:** Enterprise-grade architecture, dependency injection, modular design
- **TypeScript:** Type safety, matches frontend language, better refactoring
- **Node.js:** Rich ecosystem for hardware drivers, good performance for I/O

**Architecture:**
```
NestJS Application
├── Modules (feature-based)
│   ├── SalesModule
│   ├── InventoryModule
│   ├── SyncModule
│   ├── AuthModule
│   ├── HardwareModule
│   └── BackupModule
├── Shared Services
│   ├── DatabaseService
│   ├── LoggerService
│   ├── ConfigService
│   └── QueueService
└── Guards & Interceptors
    ├── AuthGuard (permission checks)
    ├── LicenseGuard (license validation)
    └── LoggingInterceptor (audit trail)
```

**Startup Behavior:**
- Launched automatically by Tauri on application start
- Runs on `localhost:3000` (randomized port if conflict)
- Single instance via port lock file
- Graceful shutdown on application exit

**API Design:**
- RESTful API for CRUD operations
- WebSocket for real-time updates (e.g., sync status)
- JSON request/response format
- Standard HTTP status codes
- Comprehensive error responses

### 3. SQLite Database (Encrypted with SQLCipher)

**Purpose:** Local, embedded, encrypted database for offline-first data persistence.

**Technology Justification:**
- **SQLite:** Embedded, zero-configuration, ACID-compliant, mature
- **SQLCipher:** Transparent AES-256 encryption, minimal performance overhead
- **WAL Mode:** Concurrent read/write, crash-safe, better performance

**Database Configuration:**
```sql
-- Encryption key derived from machine ID + license key
PRAGMA key = 'x''...'';  -- 256-bit key
PRAGMA cipher_page_size = 4096;
PRAGMA kdf_iter = 64000;
PRAGMA cipher_hmac_algorithm = HMAC_SHA256;
PRAGMA cipher_kdf_algorithm = PBKDF2_HMAC_SHA256;

-- Performance optimizations
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA temp_store = MEMORY;
PRAGMA cache_size = -64000;  -- 64MB cache
```

**Backup Strategy:**
- Automated daily backups (full database copy)
- Backup on every schema migration
- Manual backup on user request
- Backup retention: 30 days local, 90 days cloud
- Backup verification (checksum validation)

**Migration Strategy:**
- Schema versioning (migration files)
- Forward-only migrations
- Automatic migration on application update
- Pre-migration backup (safety net)
- Rollback capability for failed migrations

### 4. Supabase Cloud Services

**Purpose:** Cloud backend for sync, licensing, admin portal, and analytics.

**Technology Justification:**
- **Supabase:** Open-source, PostgreSQL-based, comprehensive feature set
- **PostgreSQL:** Robust, scalable, excellent for multi-tenant SaaS
- **Realtime:** Built-in change data capture for sync
- **Row-Level Security:** Native multi-tenant data isolation

**Architecture:**
```
Supabase Project
├── PostgreSQL Database
│   ├── public schema (central tables)
│   ├── RLS policies (tenant isolation)
│   └── Functions & triggers
├── Auth Service
│   ├── JWT-based authentication
│   └── API key management
├── Storage Buckets
│   ├── backups/ (encrypted backups)
│   ├── reports/ (generated reports)
│   └── assets/ (product images)
├── Realtime Service
│   └── Change notifications
└── Edge Functions
    ├── license-validation
    ├── sync-processor
    └── webhook-handler
```

**Connection Handling:**
- Connection pooling (max 10 connections per client)
- Automatic reconnection with exponential backoff
- Circuit breaker pattern (stop retrying after threshold)
- Graceful fallback to offline mode

### 5. Hardware Interfaces

#### Thermal Printer Integration

**Supported Protocols:**
- ESC/POS (most common)
- Star CUPS
- ZPL (for label printers)

**Connection Types:**
- USB (preferred)
- Serial (COM ports)
- Network (Ethernet/Wi-Fi)

**Features:**
- Automatic printer detection
- Print queue management
- Fallback to PDF if printer offline
- Custom receipt templates
- Logo printing support
- Barcode/QR code printing

**Driver Architecture:**
```typescript
interface PrinterDriver {
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  print(receipt: Receipt): Promise<PrintResult>;
  getStatus(): PrinterStatus;
}

class ESCPOSDriver implements PrinterDriver {
  // ESC/POS command generation
}

class PDFDriver implements PrinterDriver {
  // Fallback to PDF export
}
```

#### Barcode Scanner Integration

**Supported Types:**
- USB HID (keyboard wedge)
- USB Serial
- Bluetooth scanners

**Features:**
- Automatic product lookup
- Quantity scanning (e.g., `*5*` prefix for quantity 5)
- Multi-scan mode (continuous scanning)
- Scan validation and feedback

#### Biometric Device Integration

**Supported Devices:**
- ZKTeco fingerprint readers
- Suprema BioMini
- Generic USB fingerprint sensors

**Features:**
- Employee clock-in/clock-out
- Manager approval via fingerprint
- User login via biometric
- Fallback to PIN code

**Security:**
- Biometric templates stored locally (never synced)
- Template encryption
- Secure enclave integration (if available)

### 6. File System (Local Storage)

**Purpose:** Configuration, logs, backups, and temporary files.

**Directory Structure:**
```
%APPDATA%/FlexPOS/
├── data/              # Database files
├── backups/           # Database backups
│   ├── auto/          # Automated backups
│   │   └── flexpos_YYYYMMDD_HHMMSS.db.enc
│   └── manual/        # User-initiated backups
├── logs/              # Application logs
│   ├── app.log        # General application log
│   ├── sync.log       # Sync operation log
│   ├── error.log      # Error log
│   └── audit.log      # Audit trail
├── config/            # Configuration files
│   ├── app.config.json
│   ├── printer.config.json
│   ├── hardware.config.json
│   └── network.config.json
├── receipts/          # Receipt templates
│   ├── default.tmpl
│   └── custom.tmpl
├── exports/           # Exported data
│   ├── sales_YYYYMMDD.csv
│   └── inventory_YYYYMMDD.xlsx
└── temp/              # Temporary files
    └── .lock          # Application lock file
```

**Log Management:**
- Log rotation (daily, max 10 files)
- Log levels: DEBUG, INFO, WARN, ERROR, FATAL
- Structured logging (JSON format)
- Log compression after rotation
- Sensitive data redaction

**Configuration Management:**
- JSON configuration files
- Hot-reload capability (no restart needed)
- Configuration validation on load
- Default configuration fallback
- Configuration backup before update

### 7. Windows OS Integration

**Purpose:** Deep integration with Windows OS for enterprise reliability.

**Features:**

#### Windows Service
- Background service for always-on operation
- Automatic start on system boot
- Automatic restart on crash
- Clean shutdown on system shutdown
- Service control (start, stop, restart)

#### Event Log Integration
- Application events logged to Windows Event Viewer
- Standard event types (Information, Warning, Error)
- Event ID mapping for monitoring
- Integration with enterprise monitoring tools

#### Registry Integration
- License key storage (encrypted)
- Machine ID generation
- Application configuration
- Auto-start configuration
- Uninstall information

#### Task Scheduler
- Daily backup task
- Weekly database optimization
- Monthly log cleanup
- Sync retry scheduling

#### Windows Authentication
- Optional domain authentication
- Windows user context
- Single sign-on (SSO) capability
- Active Directory integration (future)

---

## Offline-First Architecture Pattern

### Philosophy

FlexPOS is designed with the principle: **"The internet is optional, not required."**

All core POS operations must work flawlessly without an internet connection. Cloud sync is a background enhancement, not a dependency.

### Local-First Data Storage

```
┌──────────────────────────────────────────────────────────────┐
│                    DATA FLOW ARCHITECTURE                     │
└──────────────────────────────────────────────────────────────┘

USER ACTION (e.g., Complete Sale)
     │
     ▼
┌─────────────────────────────────────────────┐
│         React Frontend (UI)                  │
│  • Optimistic UI Update                     │
│  • Show "Saving..." indicator               │
└──────────────┬──────────────────────────────┘
               │ Tauri IPC
               ▼
┌─────────────────────────────────────────────┐
│         Tauri Application Layer              │
│  • Validate request                         │
│  • Forward to backend                       │
└──────────────┬──────────────────────────────┘
               │ HTTP/REST
               ▼
┌─────────────────────────────────────────────┐
│         NestJS Business Logic                │
│  • Validate business rules                  │
│  • Calculate totals, tax, etc.              │
│  • Begin transaction                        │
└──────────────┬──────────────────────────────┘
               │ Repository Pattern
               ▼
┌─────────────────────────────────────────────┐
│         SQLite Database (Local)              │
│  ✓ Transaction committed                    │
│  ✓ Receipt printed                          │
│  ✓ UI updated with success                  │
└──────────────┬──────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────┐
│         Sync Queue (Background)              │  ← NON-BLOCKING
│  • Add operation to sync queue              │
│  • Mark for cloud sync (when online)        │
└──────────────┬──────────────────────────────┘
               │ (Async, when internet available)
               ▼
┌─────────────────────────────────────────────┐
│         Supabase Cloud (When Online)         │
│  • Sync data to cloud                       │
│  • Resolve conflicts (if any)               │
│  • Update sync status                       │
└─────────────────────────────────────────────┘

CRITICAL: Sale is COMPLETE after local DB commit
          Cloud sync is a background enhancement
```

### Async Sync Queue Architecture

**Sync Queue Table Schema:**
```sql
CREATE TABLE sync_queue (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  entity_type TEXT NOT NULL,           -- 'sale', 'product', 'customer', etc.
  entity_id TEXT NOT NULL,             -- Local ID of the entity
  operation TEXT NOT NULL,             -- 'CREATE', 'UPDATE', 'DELETE'
  payload TEXT NOT NULL,               -- JSON payload of changes
  created_at TEXT NOT NULL,            -- Timestamp of local change
  retry_count INTEGER DEFAULT 0,       -- Number of retry attempts
  last_retry_at TEXT,                  -- Timestamp of last retry
  sync_status TEXT DEFAULT 'pending',  -- 'pending', 'syncing', 'synced', 'failed'
  error_message TEXT,                  -- Error details if failed
  priority INTEGER DEFAULT 5           -- 1-10, higher = more urgent
);

CREATE INDEX idx_sync_queue_status ON sync_queue(sync_status, priority DESC, created_at ASC);
```

**Sync Queue Processing:**

```typescript
class SyncQueueService {
  private syncInterval = 30000; // 30 seconds
  private maxRetries = 10;
  private batchSize = 50;

  async processSyncQueue(): Promise<void> {
    // Check internet connectivity
    if (!await this.isOnline()) {
      this.logger.debug('Offline - skipping sync');
      return;
    }

    // Check license validity (cloud check)
    if (!await this.isLicenseValid()) {
      this.logger.warn('License invalid - sync disabled');
      return;
    }

    // Fetch pending items (highest priority first)
    const pendingItems = await this.fetchPendingItems(this.batchSize);

    for (const item of pendingItems) {
      try {
        await this.syncItem(item);
        await this.markAsSynced(item.id);
      } catch (error) {
        await this.handleSyncError(item, error);
      }
    }
  }

  private async handleSyncError(item: SyncQueueItem, error: Error): Promise<void> {
    const retryCount = item.retry_count + 1;

    if (retryCount >= this.maxRetries) {
      // Mark as permanently failed
      await this.markAsFailed(item.id, error.message);
      await this.notifyAdmin(item, error);
    } else {
      // Schedule retry with exponential backoff
      const nextRetry = this.calculateBackoff(retryCount);
      await this.scheduleRetry(item.id, nextRetry, retryCount);
    }
  }

  private calculateBackoff(retryCount: number): number {
    // Exponential backoff: 1min, 2min, 4min, 8min, 16min, 32min, 1hr, 2hr, 4hr, 8hr
    const baseDelay = 60000; // 1 minute
    return Math.min(baseDelay * Math.pow(2, retryCount - 1), 8 * 60 * 60 * 1000);
  }
}
```

### Conflict Resolution Strategy

**Conflict Types:**

1. **No Conflict:** Remote data unchanged since last sync
2. **Remote Update:** Remote data changed, no local changes
3. **Local Update:** Local changes not yet synced
4. **Concurrent Update:** Both local and remote changed the same entity

**Resolution Rules:**

```typescript
enum ConflictResolution {
  LOCAL_WINS = 'local_wins',           // Local changes take precedence
  REMOTE_WINS = 'remote_wins',         // Remote changes take precedence
  LAST_WRITE_WINS = 'last_write_wins', // Most recent timestamp wins
  MERGE = 'merge',                     // Intelligent merge
  MANUAL = 'manual'                    // Requires human intervention
}

// Conflict resolution rules per entity type
const CONFLICT_RULES: Record<string, ConflictResolution> = {
  sale: ConflictResolution.LOCAL_WINS,              // Sales are immutable after creation
  product: ConflictResolution.LAST_WRITE_WINS,      // Product updates can be merged by timestamp
  customer: ConflictResolution.MERGE,               // Customer data can be intelligently merged
  inventory: ConflictResolution.LOCAL_WINS,         // Local stock adjustments are authoritative
  user: ConflictResolution.REMOTE_WINS,             // User management from admin portal
};
```

**Conflict Detection:**
```typescript
interface ConflictDetection {
  localVersion: string;      // Local data version (hash or timestamp)
  remoteVersion: string;     // Remote data version
  lastSyncVersion: string;   // Version at last successful sync
}

function detectConflict(detection: ConflictDetection): boolean {
  // Conflict if both local and remote changed since last sync
  return detection.localVersion !== detection.lastSyncVersion &&
         detection.remoteVersion !== detection.lastSyncVersion;
}
```

**Merge Strategy (for mergeable entities):**
```typescript
function mergeCustomer(local: Customer, remote: Customer, base: Customer): Customer {
  return {
    id: local.id,
    // Non-conflicting fields
    name: local.name !== base.name ? local.name : remote.name,
    phone: local.phone !== base.phone ? local.phone : remote.phone,
    email: local.email !== base.email ? local.email : remote.email,
    
    // Additive fields (union)
    tags: [...new Set([...local.tags, ...remote.tags])],
    
    // Numerical fields (sum of changes)
    loyaltyPoints: base.loyaltyPoints + 
                   (local.loyaltyPoints - base.loyaltyPoints) +
                   (remote.loyaltyPoints - base.loyaltyPoints),
    
    // Metadata
    updatedAt: new Date().toISOString(),
    syncedAt: new Date().toISOString(),
  };
}
```

### Cache Management

**Multi-Level Caching:**

```
┌──────────────────────────────────────────────────────────┐
│                    CACHE ARCHITECTURE                     │
└──────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  L1 Cache: In-Memory (Frontend)                         │
│  • UI State (React state/context)                       │
│  • Recent transactions                                  │
│  • Hot products (frequent sales)                        │
│  • TTL: Session duration                                │
│  • Size: ~50MB                                          │
└────────────────────┬────────────────────────────────────┘
                     │ Cache miss
                     ▼
┌─────────────────────────────────────────────────────────┐
│  L2 Cache: In-Memory (Backend)                          │
│  • Frequently accessed entities                         │
│  • Computed aggregations                                │
│  • Session data                                         │
│  • TTL: 5-15 minutes (configurable)                     │
│  • Size: ~200MB                                         │
│  • Strategy: LRU (Least Recently Used)                  │
└────────────────────┬────────────────────────────────────┘
                     │ Cache miss
                     ▼
┌─────────────────────────────────────────────────────────┐
│  L3 Cache: SQLite Database                              │
│  • All persistent data                                  │
│  • Full ACID compliance                                 │
│  • Source of truth for offline operation                │
└─────────────────────────────────────────────────────────┘
```

**Cache Invalidation:**
- Time-based expiration (TTL)
- Event-based invalidation (on data update)
- Manual invalidation (user-triggered refresh)
- Cache warming (preload frequently used data)

**Cache Strategies:**
```typescript
// Cache-aside pattern (lazy loading)
async getProduct(id: string): Promise<Product> {
  // Check L2 cache first
  const cached = this.cache.get(`product:${id}`);
  if (cached) return cached;
  
  // Cache miss - fetch from database
  const product = await this.productRepository.findById(id);
  
  // Store in cache
  this.cache.set(`product:${id}`, product, { ttl: 300 }); // 5 min TTL
  
  return product;
}

// Write-through pattern (immediate consistency)
async updateProduct(id: string, updates: Partial<Product>): Promise<Product> {
  // Update database first
  const product = await this.productRepository.update(id, updates);
  
  // Update cache immediately
  this.cache.set(`product:${id}`, product, { ttl: 300 });
  
  // Invalidate related caches
  this.cache.del(`products:category:${product.categoryId}`);
  
  return product;
}
```

### Grace Period Handling (7-Day Offline License)

**License Validation Flow:**

```
┌──────────────────────────────────────────────────────────────┐
│              LICENSE VALIDATION DECISION TREE                 │
└──────────────────────────────────────────────────────────────┘

START: User attempts to use application
  │
  ▼
Is internet available?
  │
  ├─── YES ──→ Call Supabase license API
  │            │
  │            ▼
  │         License valid?
  │            │
  │            ├─── YES ──→ Update local cache (lastChecked = NOW)
  │            │            Allow full access
  │            │
  │            └─── NO ──→ Show "License expired" message
  │                        Allow read-only mode OR block access
  │
  └─── NO ──→ Check local license cache
               │
               ▼
            Local cache exists?
               │
               ├─── YES ──→ Check lastChecked timestamp
               │            │
               │            ▼
               │         lastChecked < 7 days ago?
               │            │
               │            ├─── YES ──→ Allow full access
               │            │            Show "Offline mode" indicator
               │            │            Show days remaining
               │            │
               │            └─── NO ──→ Show "Grace period expired"
               │                        Allow read-only mode
               │                        Require internet to re-validate
               │
               └─── NO ──→ First-time setup requires internet
                           Show "Internet required for activation"
```

**Implementation:**

```typescript
class LicenseService {
  private readonly GRACE_PERIOD_DAYS = 7;
  
  async validateLicense(): Promise<LicenseStatus> {
    const isOnline = await this.isOnline();
    
    if (isOnline) {
      // Online validation
      try {
        const licenseValid = await this.validateWithSupabase();
        
        if (licenseValid) {
          // Cache validation result
          await this.updateLocalCache({
            valid: true,
            lastChecked: new Date(),
            expiresAt: this.calculateExpiry(),
          });
          
          return { valid: true, mode: 'online' };
        } else {
          return { valid: false, reason: 'license_expired' };
        }
      } catch (error) {
        // Network error during online check - fall back to offline
        return this.validateOffline();
      }
    } else {
      // Offline validation
      return this.validateOffline();
    }
  }
  
  private async validateOffline(): Promise<LicenseStatus> {
    const cachedLicense = await this.getLocalCache();
    
    if (!cachedLicense) {
      return {
        valid: false,
        reason: 'no_cached_license',
        message: 'Internet required for initial activation'
      };
    }
    
    const daysSinceCheck = this.daysSince(cachedLicense.lastChecked);
    
    if (daysSinceCheck <= this.GRACE_PERIOD_DAYS) {
      const daysRemaining = this.GRACE_PERIOD_DAYS - daysSinceCheck;
      
      return {
        valid: true,
        mode: 'offline',
        gracePeriod: {
          active: true,
          daysRemaining: daysRemaining,
          warning: daysRemaining <= 2 ? 'Connect to internet soon' : null
        }
      };
    } else {
      return {
        valid: false,
        reason: 'grace_period_expired',
        message: 'Please connect to internet to validate license'
      };
    }
  }
}
```

**User Experience:**

```typescript
// UI Indicators
interface OfflineModeIndicator {
  status: 'online' | 'offline_with_grace' | 'offline_grace_expired';
  message: string;
  daysRemaining?: number;
  urgency: 'none' | 'low' | 'medium' | 'high';
}

// Examples:
// Online:
{ status: 'online', message: 'Connected', urgency: 'none' }

// Offline with grace:
{ status: 'offline_with_grace', message: 'Offline mode - 5 days remaining', daysRemaining: 5, urgency: 'low' }

// Grace expiring soon:
{ status: 'offline_with_grace', message: 'Offline mode - 1 day remaining', daysRemaining: 1, urgency: 'high' }

// Grace expired:
{ status: 'offline_grace_expired', message: 'License check required - connect to internet', urgency: 'high' }
```

---

## Data Isolation & Boundaries

### Data Categories

FlexPOS manages four distinct categories of data, each with different storage, sync, and security requirements:

```
┌──────────────────────────────────────────────────────────────┐
│                   DATA CATEGORY BOUNDARIES                    │
└──────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  1. LOCAL-ONLY DATA (Never Synced)                         │
│  ═══════════════════════════════════════════════════════   │
│  • Biometric templates (fingerprints)                      │
│  • Hardware configurations (printer settings)              │
│  • UI preferences (window size, theme)                     │
│  • Local cache (temporary data)                            │
│  • Encryption keys                                         │
│                                                             │
│  Storage: SQLite + File System                             │
│  Encryption: Yes (AES-256)                                 │
│  Sync: Never                                               │
│  Backup: Local only                                        │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  2. TRANSACTIONAL DATA (Sync When Online)                  │
│  ═══════════════════════════════════════════════════════   │
│  • Sales transactions                                      │
│  • Payments                                                │
│  • Stock movements                                         │
│  • Customer records                                        │
│  • Audit logs                                              │
│                                                             │
│  Storage: SQLite (local) → PostgreSQL (cloud)              │
│  Encryption: Yes (at rest and in transit)                  │
│  Sync: Async (via sync queue)                              │
│  Backup: Local + Cloud                                     │
│  Conflict Resolution: Per-entity rules                     │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  3. MASTER DATA (Bidirectional Sync)                       │
│  ═══════════════════════════════════════════════════════   │
│  • Products & categories                                   │
│  • Suppliers                                               │
│  • Store configuration                                     │
│  • Users & permissions                                     │
│  • Tax rates                                               │
│                                                             │
│  Storage: SQLite (local) ↔ PostgreSQL (cloud)              │
│  Encryption: Yes                                           │
│  Sync: Bidirectional (local changes push, remote pull)     │
│  Backup: Local + Cloud                                     │
│  Conflict Resolution: Last-write-wins or merge             │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  4. CLOUD-ONLY DATA (License & Admin)                      │
│  ═══════════════════════════════════════════════════════   │
│  • License validation records                              │
│  • Subscription billing                                    │
│  • Analytics aggregations                                  │
│  • Admin portal data                                       │
│  • Multi-store orchestration                               │
│                                                             │
│  Storage: PostgreSQL (cloud only)                          │
│  Encryption: Yes                                           │
│  Sync: One-way (cloud → local for license checks)          │
│  Backup: Cloud only                                        │
│  Access: API only (no local storage)                       │
└────────────────────────────────────────────────────────────┘
```

### Boundary Enforcement

**Architectural Boundaries:**

```typescript
// Clear separation via TypeScript types
interface LocalOnlyEntity {
  _syncable: false;
  _cloudBackup: false;
}

interface SyncableEntity {
  _syncable: true;
  _cloudBackup: true;
  id: string;                    // Local ID
  cloudId?: string;              // Cloud ID (after first sync)
  version: number;               // Optimistic locking
  updatedAt: string;             // For conflict detection
  syncStatus: SyncStatus;        // 'pending' | 'synced' | 'conflict'
}

interface CloudOnlyEntity {
  _localStorageProhibited: true;
}

// Example entities
interface BiometricTemplate extends LocalOnlyEntity {
  userId: string;
  template: Buffer;          // Never leaves the device
  deviceId: string;
}

interface Sale extends SyncableEntity {
  storeId: string;
  terminalId: string;
  items: SaleItem[];
  payments: Payment[];
  total: number;
  // ... business fields
}

interface LicenseValidation extends CloudOnlyEntity {
  licenseKey: string;
  validUntil: Date;
  features: string[];
  // Only accessed via API, never stored locally
}
```

**Repository Pattern Enforcement:**

```typescript
// Base repository enforces sync rules
abstract class BaseRepository<T extends { _syncable: boolean }> {
  async save(entity: T): Promise<T> {
    // Save to local database
    const saved = await this.localDb.save(entity);
    
    // Enqueue for sync if syncable
    if (entity._syncable) {
      await this.syncQueue.enqueue({
        entityType: this.entityType,
        entityId: saved.id,
        operation: 'CREATE',
        payload: saved,
      });
    }
    
    return saved;
  }
}

// Local-only repository explicitly prevents sync
class BiometricRepository extends BaseRepository<BiometricTemplate> {
  async save(template: BiometricTemplate): Promise<BiometricTemplate> {
    // Override to ensure no sync
    return this.localDb.save(template); // No sync queue
  }
}
```

### Data Flow Diagrams

#### Local-Only Data Flow
```
User Action (e.g., Save Fingerprint)
     │
     ▼
┌─────────────────────────┐
│   NestJS Backend        │
│   (BiometricService)    │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   SQLite Database       │
│   (biometric_templates) │
└─────────────────────────┘
             │
             X  ← SYNC BLOCKED
         (Never syncs to cloud)
```

#### Transactional Data Flow (Sale)
```
User Action (Complete Sale)
     │
     ▼
┌─────────────────────────┐
│   NestJS Backend        │
│   (SalesService)        │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐     ┌──────────────────┐
│   SQLite Database       │────→│   Sync Queue     │
│   (sales table)         │     │   (pending sync) │
└─────────────────────────┘     └────────┬─────────┘
         ✓                               │
   Transaction complete              (Async)
   Receipt printed                       │
                                         ▼
                                   ┌─────────────────┐
                                   │  Supabase Cloud │
                                   │  (when online)  │
                                   └─────────────────┘
```

#### Master Data Flow (Product Update)
```
Scenario: Product updated in admin portal (cloud)
                                   
┌─────────────────────────┐
│  Admin Portal (Cloud)   │
│  Update product price   │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Supabase PostgreSQL    │
│  Update product record  │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Realtime Notification  │
│  (Change Data Capture)  │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Desktop App (Online)   │
│  Receive change event   │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐     ┌──────────────────────┐
│  Local SQLite           │────→│  Conflict Check      │
│  Check for conflicts    │     │  (if local changes)  │
└─────────────────────────┘     └──────────┬───────────┘
             │                              │
             ▼                              ▼
      No Conflict                      Conflict Detected
             │                              │
             ▼                              ▼
    ┌─────────────────┐           ┌──────────────────┐
    │ Apply Update    │           │ Merge or Manual  │
    │ Update Local DB │           │ Resolution       │
    └─────────────────┘           └──────────────────┘
```

### Security Boundaries

**Data Security by Category:**

| Category | Encryption at Rest | Encryption in Transit | Access Control | Audit Logging |
|----------|-------------------|----------------------|----------------|---------------|
| Local-Only | ✓ (AES-256) | N/A (never transmitted) | Device-level | Local only |
| Transactional | ✓ (SQLCipher) | ✓ (HTTPS/TLS 1.3) | Role-based | Local + Cloud |
| Master Data | ✓ (SQLCipher) | ✓ (HTTPS/TLS 1.3) | Role-based | Local + Cloud |
| Cloud-Only | ✓ (PostgreSQL encryption) | ✓ (HTTPS/TLS 1.3) | Admin-only | Cloud only |

**Access Control Layers:**

```typescript
// Permission-based access control
enum Permission {
  // Sales permissions
  CREATE_SALE = 'sales:create',
  VIEW_SALE = 'sales:view',
  VOID_SALE = 'sales:void',           // Manager only
  
  // Inventory permissions
  ADJUST_STOCK = 'inventory:adjust',
  VIEW_INVENTORY = 'inventory:view',
  MANAGE_PRODUCTS = 'inventory:manage',
  
  // System permissions
  MANAGE_USERS = 'system:users',      // Admin only
  VIEW_REPORTS = 'reports:view',
  EXPORT_DATA = 'system:export',      // Manager only
  
  // Hardware permissions
  CONFIGURE_HARDWARE = 'hardware:config', // Admin only
}

// Role definitions
const ROLES = {
  CASHIER: [
    Permission.CREATE_SALE,
    Permission.VIEW_SALE,
    Permission.VIEW_INVENTORY,
  ],
  MANAGER: [
    ...ROLES.CASHIER,
    Permission.VOID_SALE,
    Permission.ADJUST_STOCK,
    Permission.VIEW_REPORTS,
    Permission.EXPORT_DATA,
  ],
  ADMIN: [
    ...ROLES.MANAGER,
    Permission.MANAGE_USERS,
    Permission.MANAGE_PRODUCTS,
    Permission.CONFIGURE_HARDWARE,
  ],
};

// Guard implementation
@Injectable()
export class PermissionGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const requiredPermission = this.reflector.get<Permission>(
      'permission',
      context.getHandler()
    );
    
    const user = context.switchToHttp().getRequest().user;
    
    return user.permissions.includes(requiredPermission);
  }
}

// Usage in controller
@Post('void')
@RequirePermission(Permission.VOID_SALE)
async voidSale(@Body() data: VoidSaleDto) {
  return this.salesService.voidSale(data);
}
```

---

## Technology Stack

### Frontend Stack

| Layer | Technology | Version | Purpose | Justification |
|-------|-----------|---------|---------|---------------|
| **Desktop Framework** | Tauri | 1.5+ | Native desktop wrapper | Lightweight, secure, native performance |
| **UI Framework** | React | 18+ | Component-based UI | Rich ecosystem, developer familiarity |
| **Language** | TypeScript | 5+ | Type-safe development | Reduces bugs, better IDE support |
| **State Management** | Zustand + React Query | Latest | Client state + server state | Simple, performant, less boilerplate |
| **Styling** | TailwindCSS | 3+ | Utility-first CSS | Rapid development, consistent design |
| **UI Components** | shadcn/ui | Latest | Pre-built components | Accessible, customizable, modern |
| **Forms** | React Hook Form | Latest | Form management | Performant, validation, type-safe |
| **Tables** | TanStack Table | Latest | Data grids | Virtual scrolling, sorting, filtering |
| **Charts** | Recharts | Latest | Data visualization | React-native, responsive |
| **Icons** | Lucide React | Latest | Icon library | Modern, tree-shakeable |

### Backend Stack

| Layer | Technology | Version | Purpose | Justification |
|-------|-----------|---------|---------|---------------|
| **Framework** | NestJS | 10+ | Backend framework | Enterprise architecture, DI, modular |
| **Runtime** | Node.js | 20 LTS | JavaScript runtime | Mature, large ecosystem, good performance |
| **Language** | TypeScript | 5+ | Type-safe backend | Consistency with frontend, safety |
| **Database** | SQLite | 3.42+ | Embedded database | Zero-config, reliable, fast for local use |
| **Encryption** | SQLCipher | 4.5+ | Database encryption | Transparent AES-256 encryption |
| **ORM** | Prisma | 5+ | Database ORM | Type-safe queries, great DX, migrations |
| **Validation** | class-validator | Latest | Input validation | Declarative, type-safe |
| **Logging** | Winston | Latest | Structured logging | Flexible, multiple transports |
| **Task Scheduling** | @nestjs/schedule | Latest | Cron jobs | Background tasks (sync, backup) |
| **Queues** | Bull (SQLite adapter) | Latest | Job queues | Reliable background processing |

### Infrastructure Stack

| Layer | Technology | Version | Purpose | Justification |
|-------|-----------|---------|---------|---------------|
| **Cloud Platform** | Supabase | Latest | Backend-as-a-Service | PostgreSQL, Auth, Storage, Realtime |
| **Cloud Database** | PostgreSQL | 15+ | Cloud data store | Robust, scalable, excellent for SaaS |
| **Authentication** | Supabase Auth | Latest | Cloud auth | OAuth, JWT, user management |
| **Storage** | Supabase Storage | Latest | File storage | Encrypted backups, assets |
| **Realtime** | Supabase Realtime | Latest | Change data capture | Sync notifications, live updates |
| **API** | Supabase PostgREST | Latest | Auto-generated API | RESTful API from database schema |

### Hardware Integration Stack

| Component | Technology | Purpose | Justification |
|-----------|-----------|---------|---------------|
| **Thermal Printers** | node-escpos | ESC/POS printing | Wide hardware compatibility |
| **Barcode Scanners** | node-serialport | Serial/USB communication | Low-level device access |
| **Biometric Devices** | Vendor SDKs (FFI) | Fingerprint readers | Native library integration |
| **USB Devices** | node-usb | USB device access | Direct USB communication |

### Development Tools

| Category | Tool | Purpose |
|----------|------|---------|
| **Package Manager** | pnpm | Fast, efficient dependency management |
| **Bundler** | Vite | Fast development builds, HMR |
| **Linter** | ESLint | Code quality enforcement |
| **Formatter** | Prettier | Consistent code style |
| **Testing (Unit)** | Vitest | Fast, modern test runner |
| **Testing (E2E)** | Playwright | End-to-end testing |
| **Version Control** | Git | Source control |
| **CI/CD** | GitHub Actions | Automated builds and tests |
| **Documentation** | Markdown | Technical documentation |

### Build & Distribution

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Installer** | NSIS | Windows installer creation |
| **Code Signing** | Windows Authenticode | Trusted application signing |
| **Auto-Update** | Tauri Updater | Silent background updates |
| **Crash Reporting** | Sentry | Error monitoring |

---

## Performance Characteristics

### Performance Targets

| Operation | Target | Maximum Acceptable | Notes |
|-----------|--------|-------------------|-------|
| **POS Transaction** | <500ms | <1000ms | From scan to receipt print |
| **Product Search** | <100ms | <200ms | Search 10,000+ products |
| **Database Query** | <50ms | <100ms | 95th percentile |
| **UI Interaction** | <16ms | <32ms | 60 FPS minimum |
| **Sync Operation** | N/A (background) | <5s per batch | Should not block UI |
| **Application Startup** | <3s | <5s | Cold start on HDD |
| **Receipt Printing** | <2s | <3s | Physical print complete |
| **Backup Creation** | <10s | <30s | 100MB database |

### Performance Optimization Strategies

#### Database Optimizations

```sql
-- Indexes for frequent queries
CREATE INDEX idx_products_barcode ON products(barcode);
CREATE INDEX idx_products_name ON products(name);
CREATE INDEX idx_sales_created_at ON sales(created_at DESC);
CREATE INDEX idx_sale_items_sale_id ON sale_items(sale_id);
CREATE INDEX idx_sync_queue_status ON sync_queue(sync_status, priority DESC);

-- Materialized views for reports (computed periodically)
CREATE TABLE daily_sales_summary AS
  SELECT 
    DATE(created_at) as sale_date,
    COUNT(*) as transaction_count,
    SUM(total) as daily_total,
    AVG(total) as avg_transaction
  FROM sales
  GROUP BY DATE(created_at);

-- Full-text search for products
CREATE VIRTUAL TABLE products_fts USING fts5(
  name, 
  barcode, 
  description,
  content=products
);
```

#### Application-Level Caching

```typescript
// Product cache (hot products)
class ProductCache {
  private cache = new LRU<string, Product>({
    max: 1000,              // Cache up to 1000 products
    ttl: 1000 * 60 * 15,    // 15 minute TTL
    updateAgeOnGet: true,   // Refresh TTL on access
  });
  
  async getProduct(id: string): Promise<Product> {
    // Check cache
    const cached = this.cache.get(id);
    if (cached) return cached;
    
    // Cache miss - fetch from DB
    const product = await this.db.findProduct(id);
    this.cache.set(id, product);
    
    return product;
  }
  
  // Preload hot products on startup
  async warmCache(): Promise<void> {
    const hotProducts = await this.db.query(`
      SELECT p.* FROM products p
      JOIN (
        SELECT product_id, COUNT(*) as sale_count
        FROM sale_items
        WHERE created_at > datetime('now', '-7 days')
        GROUP BY product_id
        ORDER BY sale_count DESC
        LIMIT 100
      ) hot ON p.id = hot.product_id
    `);
    
    hotProducts.forEach(p => this.cache.set(p.id, p));
  }
}
```

#### UI Performance

```typescript
// Virtual scrolling for large product lists
import { useVirtualizer } from '@tanstack/react-virtual';

function ProductList({ products }: { products: Product[] }) {
  const parentRef = useRef<HTMLDivElement>(null);
  
  const virtualizer = useVirtualizer({
    count: products.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60, // Estimated row height
    overscan: 10,           // Render 10 extra items
  });
  
  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <ProductRow 
            key={virtualRow.key}
            product={products[virtualRow.index]}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: virtualRow.size,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          />
        ))}
      </div>
    </div>
  );
}

// Debounced search
function useProductSearch() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 150); // 150ms debounce
  
  const { data: products, isLoading } = useQuery({
    queryKey: ['products', debouncedQuery],
    queryFn: () => api.searchProducts(debouncedQuery),
    staleTime: 1000 * 60 * 5, // Cache for 5 minutes
  });
  
  return { products, isLoading, setQuery };
}
```

#### Sync Performance

```typescript
// Batch sync operations
class SyncOptimizer {
  async syncBatch(items: SyncQueueItem[]): Promise<void> {
    // Group by entity type for batch operations
    const grouped = groupBy(items, 'entity_type');
    
    // Batch insert for each type
    for (const [entityType, entityItems] of Object.entries(grouped)) {
      const payloads = entityItems.map(item => JSON.parse(item.payload));
      
      // Single batch request instead of N individual requests
      await this.supabase
        .from(entityType)
        .upsert(payloads, { onConflict: 'id' });
      
      // Mark all as synced in single transaction
      await this.db.transaction(async (tx) => {
        for (const item of entityItems) {
          await tx.update('sync_queue')
            .set({ sync_status: 'synced', synced_at: new Date() })
            .where('id', item.id);
        }
      });
    }
  }
  
  // Compress large payloads
  compressPayload(payload: any): string {
    const json = JSON.stringify(payload);
    if (json.length > 10000) {
      return gzip(json).toString('base64');
    }
    return json;
  }
}
```

### Resource Management

#### Memory Management

```typescript
// Automatic cleanup of old data
class DataRetentionService {
  @Cron('0 2 * * *') // Run at 2 AM daily
  async cleanupOldData(): Promise<void> {
    const retentionDays = this.config.get('retention.days', 365);
    const cutoffDate = subDays(new Date(), retentionDays);
    
    await this.db.transaction(async (tx) => {
      // Archive old sales to separate table
      await tx.raw(`
        INSERT INTO sales_archive
        SELECT * FROM sales
        WHERE created_at < ?
      `, [cutoffDate]);
      
      // Delete archived sales from main table
      await tx('sales')
        .where('created_at', '<', cutoffDate)
        .delete();
      
      // Vacuum database to reclaim space
      await tx.raw('VACUUM');
    });
    
    this.logger.log(`Archived sales older than ${cutoffDate}`);
  }
}
```

#### Storage Management

```typescript
// Monitor disk space
class StorageMonitor {
  async checkDiskSpace(): Promise<StorageStatus> {
    const dataDir = path.join(app.getPath('appData'), 'FlexPOS');
    const { free, size } = await checkDiskSpace(dataDir);
    
    const dbSize = await this.getDatabaseSize();
    const backupSize = await this.getBackupSize();
    
    return {
      totalDisk: size,
      freeDisk: free,
      databaseSize: dbSize,
      backupSize: backupSize,
      warning: free < 1024 * 1024 * 1024, // Warn if less than 1GB free
    };
  }
  
  @Cron('0 * * * *') // Check hourly
  async monitorStorage(): Promise<void> {
    const status = await this.checkDiskSpace();
    
    if (status.warning) {
      this.logger.warn('Low disk space detected', status);
      
      // Cleanup old backups
      await this.cleanupOldBackups(30); // Keep only 30 days
      
      // Compress logs
      await this.compressOldLogs();
    }
  }
}
```

---

## Architectural Principles Validation

### 1. No Hardcoded Business Logic in UI ✓

**Validation:**

```typescript
// ❌ BAD: Business logic in UI component
function CheckoutButton({ cart }) {
  const calculateTotal = () => {
    return cart.items.reduce((sum, item) => {
      const itemTotal = item.price * item.quantity;
      const discount = item.discount || 0;
      const tax = itemTotal * 0.18; // GST hardcoded in UI - BAD!
      return sum + itemTotal - discount + tax;
    }, 0);
  };
  
  return <Button onClick={() => processCheckout(calculateTotal())}>Checkout</Button>;
}

// ✓ GOOD: UI delegates to backend
function CheckoutButton({ cart }) {
  const { mutate: checkout } = useMutation({
    mutationFn: (cartId) => api.checkout(cartId), // Backend calculates everything
  });
  
  return <Button onClick={() => checkout(cart.id)}>Checkout</Button>;
}
```

**Backend Implementation:**
```typescript
@Injectable()
export class SalesService {
  async checkout(cartId: string): Promise<Sale> {
    // All business logic in backend
    const cart = await this.cartsRepo.findById(cartId);
    
    // Calculate totals (using domain service)
    const calculator = new SaleCalculator(this.taxService, this.discountService);
    const totals = calculator.calculate(cart);
    
    // Create sale with calculated values
    const sale = await this.salesRepo.create({
      ...cart,
      subtotal: totals.subtotal,
      discount: totals.discount,
      tax: totals.tax,
      total: totals.total,
    });
    
    return sale;
  }
}
```

### 2. Transaction-Safe Operations ✓

**Validation:**

```typescript
// All database operations wrapped in transactions
@Injectable()
export class SalesService {
  async completeSale(saleData: CreateSaleDto): Promise<Sale> {
    // Use database transaction for atomicity
    return this.db.transaction(async (tx) => {
      // 1. Create sale record
      const sale = await tx('sales').insert({
        ...saleData,
        status: 'completed',
        created_at: new Date(),
      }).returning('*');
      
      // 2. Create sale items
      await tx('sale_items').insert(
        saleData.items.map(item => ({
          sale_id: sale.id,
          product_id: item.productId,
          quantity: item.quantity,
          price: item.price,
        }))
      );
      
      // 3. Update inventory (deduct stock)
      for (const item of saleData.items) {
        await tx('products')
          .where('id', item.productId)
          .decrement('stock', item.quantity);
      }
      
      // 4. Create payment records
      await tx('payments').insert(
        saleData.payments.map(payment => ({
          sale_id: sale.id,
          method: payment.method,
          amount: payment.amount,
        }))
      );
      
      // 5. Add to sync queue (within same transaction)
      await tx('sync_queue').insert({
        entity_type: 'sale',
        entity_id: sale.id,
        operation: 'CREATE',
        payload: JSON.stringify(sale),
      });
      
      // All or nothing - if any step fails, entire transaction rolls back
      return sale;
    });
  }
}
```

**Idempotency for Sync:**
```typescript
// Sync operations are idempotent (safe to retry)
async syncSale(sale: Sale): Promise<void> {
  // Upsert (update or insert) is idempotent
  await this.supabase
    .from('sales')
    .upsert({
      id: sale.id,              // Same ID every time
      ...sale,
      synced_at: new Date(),
    }, {
      onConflict: 'id',          // If exists, update
      ignoreDuplicates: false,   // Overwrite with latest
    });
  
  // Safe to call multiple times with same data
}
```

### 3. Upgrade-Safe Migrations ✓

**Validation:**

```typescript
// Prisma migration example (forward-only)
// prisma/migrations/20240101_add_customer_loyalty.sql

-- Migration up (forward)
ALTER TABLE customers ADD COLUMN loyalty_points INTEGER DEFAULT 0;
ALTER TABLE customers ADD COLUMN loyalty_tier TEXT DEFAULT 'bronze';

CREATE TABLE loyalty_transactions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  customer_id TEXT NOT NULL,
  points INTEGER NOT NULL,
  type TEXT NOT NULL, -- 'earn' or 'redeem'
  created_at TEXT NOT NULL,
  FOREIGN KEY (customer_id) REFERENCES customers(id)
);

CREATE INDEX idx_loyalty_transactions_customer_id 
  ON loyalty_transactions(customer_id);

-- Version tracking
INSERT INTO schema_migrations (version, applied_at) 
  VALUES ('20240101_add_customer_loyalty', datetime('now'));
```

**Migration Runner:**
```typescript
@Injectable()
export class MigrationService {
  async runMigrations(): Promise<void> {
    // Get current schema version
    const currentVersion = await this.getCurrentVersion();
    
    // Get pending migrations
    const pendingMigrations = this.getMigrationsAfter(currentVersion);
    
    for (const migration of pendingMigrations) {
      this.logger.log(`Running migration: ${migration.name}`);
      
      try {
        // Create backup before migration
        await this.backupService.createBackup(`pre_migration_${migration.name}`);
        
        // Run migration in transaction
        await this.db.transaction(async (tx) => {
          await migration.up(tx);
        });
        
        this.logger.log(`Migration ${migration.name} completed successfully`);
      } catch (error) {
        this.logger.error(`Migration ${migration.name} failed`, error);
        
        // Restore from backup
        await this.backupService.restoreBackup(`pre_migration_${migration.name}`);
        
        throw new Error(`Migration failed: ${migration.name}`);
      }
    }
  }
}
```

### 4. Modular and Scalable Design ✓

**Validation:**

```
FlexPOS Backend Architecture (NestJS Modules)

src/
├── main.ts                          # Application entry point
├── app.module.ts                    # Root module
│
├── modules/                         # Feature modules (isolated)
│   ├── sales/
│   │   ├── sales.module.ts          # Self-contained module
│   │   ├── sales.controller.ts      # HTTP endpoints
│   │   ├── sales.service.ts         # Business logic
│   │   ├── sales.repository.ts      # Data access
│   │   ├── entities/
│   │   │   ├── sale.entity.ts
│   │   │   └── sale-item.entity.ts
│   │   ├── dto/
│   │   │   ├── create-sale.dto.ts
│   │   │   └── update-sale.dto.ts
│   │   └── tests/
│   │       └── sales.service.spec.ts
│   │
│   ├── inventory/
│   │   ├── inventory.module.ts
│   │   ├── products.controller.ts
│   │   ├── products.service.ts
│   │   └── ...
│   │
│   ├── sync/
│   ├── auth/
│   ├── hardware/
│   └── backup/
│
├── shared/                          # Shared services
│   ├── database/
│   │   ├── database.module.ts
│   │   └── database.service.ts
│   ├── logger/
│   ├── config/
│   └── queue/
│
└── domain/                          # Domain logic (framework-agnostic)
    ├── calculators/
    │   ├── sale-calculator.ts       # Pure business logic
    │   └── tax-calculator.ts
    ├── validators/
    └── policies/
```

**Module Isolation:**
```typescript
// Each module is self-contained with clear boundaries
@Module({
  imports: [
    DatabaseModule,    // Shared dependency
    LoggerModule,      // Shared dependency
  ],
  controllers: [SalesController],
  providers: [
    SalesService,
    SalesRepository,
    SaleCalculator,    // Domain service
  ],
  exports: [
    SalesService,      // Export for other modules
  ],
})
export class SalesModule {}

// Other modules import SalesModule if they need sales functionality
@Module({
  imports: [
    SalesModule,       // Import entire module, not individual services
    InventoryModule,
  ],
  // ...
})
export class ReportsModule {}
```

### 5. Zero Data Loss Mechanisms ✓

**Validation:**

**1. WAL Mode (Write-Ahead Logging):**
```typescript
// SQLite configuration for crash safety
await db.exec(`
  PRAGMA journal_mode = WAL;  -- Write-Ahead Logging
  PRAGMA synchronous = FULL;  -- Maximum durability (slower but safest)
  PRAGMA wal_autocheckpoint = 1000; -- Checkpoint every 1000 pages
`);

// WAL ensures:
// - Atomic commits (all or nothing)
// - Crash recovery (can recover from power loss)
// - Concurrent readers during write
```

**2. Automatic Backups:**
```typescript
@Injectable()
export class BackupService {
  @Cron('0 2 * * *') // Daily at 2 AM
  async createAutomaticBackup(): Promise<void> {
    const timestamp = format(new Date(), 'yyyyMMdd_HHmmss');
    const backupPath = path.join(
      this.config.backupDir,
      `auto/flexpos_${timestamp}.db.enc`
    );
    
    // Create backup (hot backup, app keeps running)
    await this.createBackup(backupPath);
    
    // Verify backup integrity
    const isValid = await this.verifyBackup(backupPath);
    if (!isValid) {
      throw new Error('Backup verification failed');
    }
    
    // Upload to cloud (if online)
    if (await this.isOnline()) {
      await this.uploadToCloud(backupPath);
    }
    
    this.logger.log(`Automatic backup created: ${backupPath}`);
  }
  
  private async createBackup(destination: string): Promise<void> {
    // Use SQLite BACKUP API (hot backup, consistent snapshot)
    await this.db.exec(`
      VACUUM INTO '${destination}';
    `);
    
    // Encrypt backup file
    await this.encryptFile(destination);
  }
}
```

**3. Sync Queue Durability:**
```typescript
// All operations added to sync queue within same transaction
async completeSale(sale: Sale): Promise<void> {
  await this.db.transaction(async (tx) => {
    // Save sale
    await tx('sales').insert(sale);
    
    // CRITICAL: Add to sync queue within SAME transaction
    await tx('sync_queue').insert({
      entity_type: 'sale',
      entity_id: sale.id,
      operation: 'CREATE',
      payload: JSON.stringify(sale),
      created_at: new Date(),
      sync_status: 'pending',
    });
    
    // If transaction commits, both sale AND sync queue entry are saved
    // If transaction fails, neither is saved (atomicity)
  });
}

// Sync queue is persistent (survives app restart)
// Even if app crashes before sync, sale will sync on next startup
```

**4. Audit Trail:**
```typescript
// Every critical operation logged
@Injectable()
export class AuditService {
  async log(action: string, userId: string, details: any): Promise<void> {
    await this.db('audit_log').insert({
      action,
      user_id: userId,
      details: JSON.stringify(details),
      created_at: new Date(),
      ip_address: this.getIpAddress(),
      user_agent: this.getUserAgent(),
    });
    
    // Audit log never deleted (permanent record)
    // Can reconstruct all actions if needed
  }
}
```

**5. Conflict Resolution (Never Loses Data):**
```typescript
// Even in conflict, both versions preserved
async resolveConflict(
  local: Entity,
  remote: Entity,
  base: Entity
): Promise<Entity> {
  // Log conflict for manual review
  await this.db('conflicts').insert({
    entity_type: this.entityType,
    entity_id: local.id,
    local_version: JSON.stringify(local),
    remote_version: JSON.stringify(remote),
    base_version: JSON.stringify(base),
    resolution_status: 'pending',
    created_at: new Date(),
  });
  
  // Apply resolution rule
  const resolved = this.applyResolutionRule(local, remote, base);
  
  // Return resolved version (both versions saved in conflicts table)
  return resolved;
}
```

---

## Deployment Architecture

### Single-Store Deployment

```
┌─────────────────────────────────────────────────────────────┐
│                    SINGLE STORE DEPLOYMENT                   │
└─────────────────────────────────────────────────────────────┘

Store Location: "ABC Electronics Shop"

┌─────────────────────────────────────────────────────────────┐
│  Windows PC (Terminal 1)                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  FlexPOS Desktop App                                   │ │
│  │  • Tauri + React UI                                    │ │
│  │  • NestJS Backend (localhost:3000)                     │ │
│  │  • SQLite Database (local)                             │ │
│  │  • Connected Printer (USB)                             │ │
│  │  • Barcode Scanner (USB)                               │ │
│  └────────────────────────────────────────────────────────┘ │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ HTTPS (When Online)
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Supabase Cloud (Optional)                                   │
│  • License validation                                        │
│  • Data backup                                               │
│  • Analytics sync                                            │
└─────────────────────────────────────────────────────────────┘
```

### Multi-Terminal Deployment (Same Store)

```
┌─────────────────────────────────────────────────────────────┐
│              MULTI-TERMINAL DEPLOYMENT (FUTURE)              │
│                   (Phase 2 Enhancement)                      │
└─────────────────────────────────────────────────────────────┘

Store Location: "ABC Electronics Shop" (2 Terminals)

┌──────────────────────────┐    ┌──────────────────────────┐
│  Terminal 1 (Main)       │    │  Terminal 2 (Counter)    │
│  ┌────────────────────┐  │    │  ┌────────────────────┐  │
│  │  FlexPOS App       │  │    │  │  FlexPOS App       │  │
│  │  • Local DB        │  │    │  │  • Local DB        │  │
│  │  • Printer 1       │  │    │  │  • Printer 2       │  │
│  └────────────────────┘  │    │  └────────────────────┘  │
└────────────┬─────────────┘    └────────────┬─────────────┘
             │                                │
             └────────────┬───────────────────┘
                          │ Local Network Sync (Peer-to-Peer)
                          │ OR Cloud Sync (via Supabase Realtime)
                          │
                          ▼
            ┌─────────────────────────────┐
            │  Shared Inventory           │
            │  • Realtime stock updates   │
            │  • Conflict resolution      │
            └─────────────────────────────┘
```

### Multi-Store Deployment (Enterprise)

```
┌─────────────────────────────────────────────────────────────┐
│                 MULTI-STORE DEPLOYMENT                       │
│                  (Enterprise Edition)                        │
└─────────────────────────────────────────────────────────────┘

Organization: "ABC Electronics Chain"

┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│  Store 1 (Downtown)  │  │  Store 2 (Mall)      │  │  Store 3 (Airport)   │
│  ┌────────────────┐  │  │  ┌────────────────┐  │  │  ┌────────────────┐  │
│  │  FlexPOS App   │  │  │  │  FlexPOS App   │  │  │  │  FlexPOS App   │  │
│  │  • Local DB    │  │  │  │  • Local DB    │  │  │  │  • Local DB    │  │
│  │  • Store ID: 1 │  │  │  │  • Store ID: 2 │  │  │  │  • Store ID: 3 │  │
│  └────────────────┘  │  │  └────────────────┘  │  │  └────────────────┘  │
└──────────┬───────────┘  └──────────┬───────────┘  └──────────┬───────────┘
           │                         │                         │
           │                         │                         │
           └─────────────────────────┼─────────────────────────┘
                                     │ HTTPS (When Online)
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Supabase Cloud (Central Hub)                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  PostgreSQL (Multi-Tenant)                             │ │
│  │  ├─ Store 1 data (RLS isolation)                       │ │
│  │  ├─ Store 2 data (RLS isolation)                       │ │
│  │  └─ Store 3 data (RLS isolation)                       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Admin Portal (Web)                                    │ │
│  │  • Cross-store analytics                               │ │
│  │  • Centralized inventory                               │ │
│  │  • User management                                     │ │
│  │  • License management                                  │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Installation Process

```
┌─────────────────────────────────────────────────────────────┐
│                    INSTALLATION FLOW                         │
└─────────────────────────────────────────────────────────────┘

1. Download Installer
   └─→ flexpos-setup-v1.0.0.exe (Windows)

2. Run Installer (NSIS)
   ├─→ Install to: C:\Program Files\FlexPOS\
   ├─→ Create Start Menu shortcut
   ├─→ Create Desktop shortcut
   ├─→ Set auto-start on boot (optional)
   └─→ Install Windows Service (background service)

3. First Launch
   └─→ Activation Wizard
       ├─→ Enter license key (requires internet)
       ├─→ Validate license with Supabase
       ├─→ Generate machine ID
       ├─→ Create local database
       │   └─→ %APPDATA%\FlexPOS\data\flexpos.db
       ├─→ Setup encryption (derived from license + machine ID)
       ├─→ Initialize database schema (run migrations)
       ├─→ Create admin user (local)
       └─→ Download initial data (products, etc.)

4. Hardware Setup
   └─→ Configuration Wizard
       ├─→ Detect thermal printer (USB/Network)
       ├─→ Print test receipt
       ├─→ Configure barcode scanner (USB)
       ├─→ Test scan
       └─→ Optional: Configure biometric device

5. Ready to Use
   └─→ Launch POS Terminal
       └─→ Login with admin credentials
```

### Update Mechanism

```typescript
// Tauri auto-updater configuration
class UpdateService {
  async checkForUpdates(): Promise<void> {
    // Check for updates from Supabase (update manifest)
    const updateInfo = await this.supabase
      .from('app_updates')
      .select('*')
      .eq('platform', 'windows')
      .order('version', { ascending: false })
      .limit(1)
      .single();
    
    if (updateInfo.version > this.currentVersion) {
      // Download update in background
      await this.downloadUpdate(updateInfo.download_url);
      
      // Prompt user to restart and install
      const shouldUpdate = await this.promptUser(
        'Update Available',
        `Version ${updateInfo.version} is available. Restart to install?`
      );
      
      if (shouldUpdate) {
        // Install and restart
        await this.installUpdateAndRestart();
      }
    }
  }
  
  @Cron('0 */6 * * *') // Check every 6 hours
  async autoCheckUpdates(): Promise<void> {
    if (await this.isOnline()) {
      await this.checkForUpdates();
    }
  }
}
```

---

## Security Architecture

### Security Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    SECURITY ARCHITECTURE                     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  1. Application Security                                     │
│  ├─ Code signing (Windows Authenticode)                     │
│  ├─ ASLR (Address Space Layout Randomization)               │
│  ├─ DEP (Data Execution Prevention)                         │
│  └─ No eval() or dynamic code execution                     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  2. Authentication & Authorization                           │
│  ├─ Local authentication (bcrypt hashed passwords)          │
│  ├─ Role-based access control (RBAC)                        │
│  ├─ Permission-based authorization                          │
│  ├─ Session management (JWT tokens, short-lived)            │
│  └─ Biometric authentication (optional)                     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  3. Data Encryption                                          │
│  ├─ Database: AES-256 encryption (SQLCipher)                │
│  ├─ Backups: Encrypted with separate key                    │
│  ├─ Network: HTTPS/TLS 1.3 for cloud sync                   │
│  ├─ Sensitive fields: Additional encryption (e.g., card #)  │
│  └─ Key derivation: PBKDF2 (license key + machine ID)       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  4. Network Security                                         │
│  ├─ HTTPS only (no HTTP)                                    │
│  ├─ Certificate pinning for Supabase                        │
│  ├─ API key rotation                                        │
│  ├─ Rate limiting on API endpoints                          │
│  └─ Firewall-friendly (outbound HTTPS only)                 │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  5. Input Validation                                         │
│  ├─ All inputs validated (frontend + backend)               │
│  ├─ SQL injection protection (parameterized queries)        │
│  ├─ XSS protection (sanitized outputs)                      │
│  ├─ CSRF protection (token-based)                           │
│  └─ File upload validation (type, size, content)            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  6. Audit & Monitoring                                       │
│  ├─ Comprehensive audit logging                             │
│  ├─ Failed login attempts tracked                           │
│  ├─ Critical operations logged                              │
│  ├─ Log integrity (append-only, checksums)                  │
│  └─ Anomaly detection (future)                              │
└─────────────────────────────────────────────────────────────┘
```

### Encryption Implementation

```typescript
// Database encryption configuration
class EncryptionService {
  async initializeDatabase(): Promise<void> {
    // Derive encryption key from license + machine ID
    const licenseKey = await this.getLicenseKey();
    const machineId = await this.getMachineId();
    
    const encryptionKey = await this.deriveKey(licenseKey, machineId);
    
    // Configure SQLCipher
    await this.db.exec(`
      PRAGMA key = "x'${encryptionKey.toString('hex')}'";
      PRAGMA cipher_page_size = 4096;
      PRAGMA kdf_iter = 256000;        -- High iteration count
      PRAGMA cipher_hmac_algorithm = HMAC_SHA512;
      PRAGMA cipher_kdf_algorithm = PBKDF2_HMAC_SHA512;
    `);
    
    // Test encryption (will fail if key wrong)
    await this.db.exec('SELECT count(*) FROM sqlite_master');
  }
  
  private async deriveKey(
    licenseKey: string,
    machineId: string
  ): Promise<Buffer> {
    // PBKDF2 key derivation
    return crypto.pbkdf2Sync(
      licenseKey,           // Password
      machineId,            // Salt
      256000,               // Iterations
      32,                   // Key length (256 bits)
      'sha512'              // Hash algorithm
    );
  }
  
  // Field-level encryption for sensitive data
  async encryptField(plaintext: string): Promise<string> {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(
      'aes-256-gcm',
      this.getFieldEncryptionKey(),
      iv
    );
    
    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    // Return: iv + authTag + encrypted (all hex)
    return iv.toString('hex') + authTag.toString('hex') + encrypted;
  }
}
```

---

## Scalability & Future Considerations

### Phase 1: Single-Store, Single-Terminal (MVP)
- ✓ Offline-first POS operations
- ✓ Local SQLite database
- ✓ Cloud sync (async)
- ✓ License validation
- ✓ Basic reporting

### Phase 2: Multi-Terminal Support (Same Store)
- [ ] Peer-to-peer sync between terminals (local network)
- [ ] Realtime inventory sync via Supabase Realtime
- [ ] Shared customer data across terminals
- [ ] Centralized reporting for store

### Phase 3: Multi-Store Support (Enterprise)
- [ ] Multi-tenant architecture (Supabase RLS)
- [ ] Centralized inventory management
- [ ] Cross-store transfers
- [ ] Admin portal (web-based)
- [ ] Consolidated analytics

### Phase 4: Advanced Features
- [ ] Loyalty program integration
- [ ] E-commerce integration
- [ ] Accounting software integration (QuickBooks, etc.)
- [ ] Advanced analytics (AI-powered insights)
- [ ] Mobile app (React Native)
- [ ] Self-checkout kiosks

### Scalability Considerations

**Database Scalability:**
- SQLite suitable for single-store operations (<1M transactions/year)
- For high-volume stores, can optimize with:
  - Partitioning (archive old data)
  - Materialized views for reports
  - Read replicas (if multi-terminal)

**Cloud Sync Scalability:**
- Supabase PostgreSQL can handle millions of records
- Batch syncing reduces API calls
- Delta sync (only changes) minimizes bandwidth
- Connection pooling for multi-terminal scenarios

**Hardware Scalability:**
- Supports multiple printers (separate queues)
- Multiple scanners (via USB hub)
- Cash drawer per terminal
- Customer display per terminal

---

## Conclusion

This architecture document defines a robust, offline-first Point of Sale system designed for reliability, performance, and scalability. Key architectural decisions:

1. **Offline-First Design**: Local SQLite database ensures POS operations never depend on internet connectivity
2. **Clean Architecture**: Strict separation of concerns with clear boundaries between layers
3. **Zero Data Loss**: Transaction-safe operations, automatic backups, and durable sync queues
4. **Security by Design**: End-to-end encryption, role-based access control, comprehensive audit logging
5. **Enterprise Reliability**: Windows service integration, automatic recovery, graceful degradation
6. **Performance-Optimized**: Multi-level caching, indexed queries, virtual scrolling, <1s transactions
7. **Modular & Scalable**: Feature-based modules enable easy extension and maintenance

This architecture serves as the foundation for subsequent phases:
- **Phase 2**: Database schema design
- **Phase 3-4**: Backend implementation (NestJS + SQLite)
- **Phase 5**: Frontend implementation (Tauri + React)
- **Phase 6**: Hardware integration and deployment

**Next Steps:**
1. Review and approve this architecture document
2. Begin Phase 2: Database schema design based on this architecture
3. Set up development environment matching this tech stack
4. Create proof-of-concept for critical components (SQLCipher, Tauri IPC, hardware drivers)
