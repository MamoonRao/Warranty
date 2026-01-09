# FlexPOS Phase 1 Task 4: Technology Stack Confirmation & Setup

## Project: Offline-First POS SaaS for Pakistani SMBs (Windows)

---

## 1. APPROVED TECHNOLOGY STACK OVERVIEW

### Technology Stack Summary Table

| Component | Technology | Version | Purpose | Windows Support |
|-----------|-----------|---------|---------|-----------------|
| Desktop UI | Tauri + React | Latest LTS | Native Windows desktop application | ✅ Native |
| Backend | Node.js + NestJS | Node 18+ LTS, NestJS 10+ | Local business logic engine | ✅ Native |
| Local Database | SQLite + SQLCipher | SQLite 3.44+, SQLCipher v4 | Encrypted local data storage | ✅ Native |
| Cloud Services | Supabase | Latest | Licensing, backup sync, admin dashboard | ✅ Full Support |
| Thermal Printer | ESC/POS Protocol | Universal | Receipt printing | ✅ Via Drivers |
| Barcode Scanner | USB HID | Standard | Product scanning | ✅ Native HID |
| Biometric Device | HID-Compatible | Standard | Employee attendance | ✅ Native HID |

---

## 2. DESKTOP APPLICATION STACK: TAURI + REACT (TYPESCRIPT)

### Tauri Configuration

**Version Requirements:**
- Tauri CLI: v1.5+ (latest v2.x recommended)
- Tauri Core: v2.0+
- Requires Rust toolchain for compilation

**Windows Compatibility:**
- Windows 10 (Build 2004) minimum
- Windows 11 fully supported
- Both 32-bit and 64-bit architectures supported
- Native Windows API integration via Tauri bridge

**Key Capabilities:**
- True native Windows application (.exe)
- Direct file system access for local database and backups
- Hardware device communication (USB, HID, serial ports)
- System tray integration for background daemon
- Auto-updater built-in
- Code signing support for Windows executables

**Performance Characteristics:**
- Lightweight memory footprint (50-100MB runtime)
- Fast startup time (<2 seconds)
- Chromium-based renderer (Webview2 on Windows)
- Supports multi-window management
- GPU acceleration available

**Known Limitations:**
- Requires Webview2 runtime on end-user machines (Windows 11 has it built-in, Win10 needs installation)
- File size ~150-200MB per installation (includes runtime)
- Building requires Rust compiler installation on dev machine

**Integration Points:**
- IPC (Inter-Process Communication) with local NestJS backend
- File system access for database and backups
- USB device communication for hardware
- Native Windows APIs via Tauri plugins

### React Configuration

**Version Requirements:**
- React 18.x (latest stable)
- React Router v6 for navigation
- TypeScript 5.x

**Build Configuration:**
- Vite as build tool (faster than Create React App)
- Production build outputs to Tauri's `dist` directory
- Code splitting for optimal load times
- Asset optimization included

**UI Framework:**
- No hardcoded framework preference (Material-UI, shadcn/ui, or custom are all viable)
- Keyboard-first design essential for POS usage
- Touch support for optional touchscreen displays
- Accessibility (WCAG 2.1 AA) required

**State Management:**
- Redux Toolkit or Zustand recommended
- Context API for lightweight state
- Avoid prop drilling with centralized state

**Performance Targets:**
- Initial load: <2 seconds
- Page transitions: <500ms
- Modal opening: <300ms

### TypeScript Configuration

**Version:** 5.x (latest stable)
- Strict mode enabled (`"strict": true`)
- All files must be `.ts` or `.tsx`
- No implicit `any` types
- DOM and Node types installed

**Development Setup:**
```bash
node --version  # v18.17.0 LTS or higher
npm --version   # v9.0+
npx tauri --version  # Latest
rustc --version # Latest
3. LOCAL BACKEND STACK: NODE.JS + NESTJS
Node.js Requirements
Version:

Node.js 18.x LTS (minimum)
Node.js 20.x LTS (recommended)
Must be x64 architecture for Windows (x86 deprecated)
Installation:

Download from nodejs.org
Add to system PATH
Verify: node --version and npm --version
Windows Service Setup:

Backend runs as Windows service (via NSSM or similar)
Auto-starts with Windows
Auto-restart on crash
Logs to Windows Event Viewer
NestJS Configuration
Version: 10.x or higher

Project Structure:

backend/
├── src/
│   ├── business-setup/     # Module 1
│   ├── inventory/          # Module 2
│   ├── sales/              # Module 3
│   ├── customers/          # Module 4
│   ├── suppliers/          # Module 5
│   ├── accounting/         # Module 6
│   ├── payroll/            # Module 7
│   ├── qr-scanning/        # Module 8
│   ├── reports/            # Module 9
│   ├── backup/             # Module 10
│   ├── licensing/          # Module 11
│   ├── database/           # Shared DB module
│   ├── auth/               # Authentication
│   ├── hardware/           # Hardware integration
│   └── main.ts             # Entry point
├── package.json
└── tsconfig.json
Core Configuration:

Dependency Injection via NestJS (decorators)
Module-based architecture (each feature = module)
Global Exception Filters for error handling
Middleware for logging, validation
Guards for authorization checks
API Port:

Default: 3000 (can be changed)
Only accessible locally (127.0.0.1)
No external network exposure (Tauri IPC only)
Database Connection:

SQLite via TypeORM or Prisma
Connection pooling (default: 5-10 connections)
Transaction support for atomic operations
Migrations via TypeORM/Prisma
Logging:

Winston or Pino recommended
Log files in %APPDATA%/FlexPOS/logs/
Daily rotation, 30-day retention
Separate logs for: general, database, errors, audit
Background Jobs:

Bull for job queue
Redis not required (in-memory option available)
Backup jobs scheduled via node-cron
Sync queue manager for offline operations
Development Dependencies
{
  "dependencies": {
    "@nestjs/common": "^10.0.0",
    "@nestjs/core": "^10.0.0",
    "@nestjs/typeorm": "^9.0.0",
    "typeorm": "^0.3.0",
    "sqlite3": "^5.1.0",
    "sqlite-cipher": "^4.5.0",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0",
    "ts-node": "^10.0.0"
  }
}
4. LOCAL DATABASE: SQLITE + SQLCIPHER
SQLite Configuration
Version: 3.44.0 or higher

Lightweight, serverless architecture
File-based (single .db file)
No separate database server needed
ACID compliance for data safety
Location:

%APPDATA%\FlexPOS\data\flexpos.db
Backup Location:

%APPDATA%\FlexPOS\backups\
├── daily/           # Daily backups (7 kept)
├── weekly/          # Weekly backups (4 kept)
├── manual/          # User-initiated backups
└── cloud/           # Google Drive synced backups
SQLCipher Encryption
Purpose: Encrypt the entire SQLite database at rest

Version: SQLCipher v4 (compatible with SQLite 3.x)

Encryption Standard:

AES-256 encryption
64,000 PBKDF2 iterations (default)
Each database has unique encryption key
Key Management:

Master key stored in Windows Credential Manager (secure)
Alternative: Environment variable (less secure, fallback only)
Key never written to disk in plaintext
On app startup: retrieve key from Credential Manager, unlock database
Backup Encryption:

Backup files inherit database encryption
Encrypted backups are portable between machines
Restore requires correct master key
Performance Impact:

~5-10% slower than unencrypted
WAL mode mitigates impact
Acceptable for POS operations
Write-Ahead Logging (WAL) Mode
Purpose: Prevent data corruption during crashes/power failures

Configuration:

PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;  -- Balance between safety and speed
PRAGMA cache_size = -64000;   -- 64MB cache
PRAGMA temp_store = MEMORY;
Benefits:

Readers don't block writers
Better concurrency
Faster transactions
Automatic recovery from crashes
Automatic Cleanup:

WAL checkpoint every 1000 pages
Checkpoints trigger on connection close
Manual checkpoint on app exit
Connection Pooling
Strategy:

Pool size: 5-10 connections (configurable)
Min idle: 2 connections
Connection timeout: 30 seconds
Idle timeout: 5 minutes
TypeORM/Prisma Setup:

// TypeORM example
@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'sqlite',
      database: 'flexpos.db',
      entities: [/* all entities */],
      synchronize: false,  // Use migrations instead
      logging: ['error'],
      poolSize: 5,
      cache: {
        type: 'database',
        alwaysEnabled: true
      }
    })
  ]
})
export class DatabaseModule {}
Transaction Handling
Isolation Levels:

Default: READ_COMMITTED (SQLite default)
SERIALIZABLE for critical operations (sales, payroll)
Transaction timeout: 30 seconds
Atomicity Guarantees:

All-or-nothing: either all operations succeed or all rollback
No partial transactions in database
Automatic rollback on error
Example Critical Operations:

Sales Transaction = atomic
├── Create Sale record
├── Update Inventory
├── Create Ledger entries
├── Print Receipt
└── Trigger Backup
(If any step fails, entire transaction rolls back)
Database Size & Performance
Expected Growth:

10,000 products: ~5-10MB
100,000 transactions: ~20-30MB
Full year of operations: ~200-500MB
Indexing Strategy:

Primary indexes on: id, created_at, product_id, customer_id
Composite indexes for: (user_id, created_at), (product_id, warehouse_id)
Query analyzer to identify missing indexes
Avoid over-indexing (slows writes)
Performance Targets:

Insert: <5ms
Query (with index): <10ms
Batch insert (1000 records): <500ms
Complex report query: <2 seconds
5. CLOUD SERVICES: SUPABASE
Supabase Project Setup
Purpose:

Licensing management (SaaS control)
Admin dashboard (multi-tenant)
Optional: Cloud backup storage
Optional: Cloud sync (future phase)
Setup Steps:

Create Supabase project at supabase.com
Configure authentication method
Create tables for: licenses, businesses, admin users
Generate API keys (public + secret)
Store keys in environment variables
Authentication (JWT-based)
Local Authentication:

Username/password for business users
Stored locally in SQLite
PBKDF2 password hashing
SaaS Admin Authentication:

OAuth via Supabase Auth or Google
JWT token for admin dashboard
Multi-user support (different admins for different clients)
License Validation:

Verify license status with Supabase on startup
Cache validation result locally (grace period: 7 days)
Re-verify when app goes online
Real-Time Capabilities
Not required for MVP, but available for:

Admin notifications (license expiry, unusual activity)
Multi-user sync notifications
Future phase: collaborative features
Google Drive Backup Integration
OAuth Flow:

User clicks "Backup to Google Drive"
Browser opens Google OAuth consent screen
User grants permission
OAuth token stored securely in Credential Manager
Backup file encrypted and uploaded to user's Google Drive
Sync status shown to user
File Structure on Google Drive:

My Drive/
└── FlexPOS Backups/
    ├── Business Name/
    │   ├── daily/
    │   ├── weekly/
    │   └── manual/
Permissions Required:

https://www.googleapis.com/auth/drive.file (create, update, delete only)
NOT full Drive access
API Key Management
Storage:

Public API key: In code (safe, anonymous)
Secret API key: Environment variable or Credential Manager
License keys: Supabase database (encrypted column)
Environment Variables:

SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_PUBLIC_KEY=eyJxxx...
SUPABASE_SECRET_KEY=eyJzzz...  # Server-side only
GOOGLE_OAUTH_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=xxx
6. HARDWARE INTEGRATION
ESC/POS Thermal Printers
Compatible Printers:

Epson TM series (TM-U220, TM-T20, TM-T82, etc.)
Star Micronics series
Zebra ZD series
Any ESC/POS compatible printer
Driver Installation:

Windows uses generic USB or LPT drivers
ESC/POS is a standard protocol (not vendor-specific)
No special driver needed (direct USB communication)
Protocol Specifications:

ESC/POS command set (standard)
80mm or 58mm paper width support
203 DPI resolution typical
Baud rate: 9600 (serial) or USB native
Windows Integration:

Direct USB communication via WinUSB driver
Alternative: Windows Print Spooler (slower but reliable)
Device detection via USB VID:PID matching
Integration Code Location:

backend/src/hardware/printer/
├── printer.service.ts      # Main printer service
├── espos.driver.ts         # ESC/POS protocol handler
├── device.detector.ts      # USB printer detection
└── receipt.formatter.ts    # Receipt template formatting
Fallback Mechanism:

If printer unavailable: Show receipt on screen (PDF option)
Buffer receipt data for later printing
Alert user to connect printer
USB Barcode Scanners
Compatible Devices:

Any USB HID keyboard-emulation scanner
Common in Pakistan: Honeywell, Symbol, Datalogic, etc.
Cost: 500-2000 PKR
HID Protocol:

Scanners emulate keyboard input
No special driver needed (Windows HID driver handles it)
Scanned code appears as keyboard text + Enter
Scanning Protocols Supported:

Code128, Code39, EAN13, EAN8, UPC-A, QR codes
Configurable via scanner buttons (manufacturer-specific)
Integration:

Capture keyboard input in POS screen
Focus on barcode input field automatically
Scan → field filled → Enter key triggers product lookup
Fallback:

Manual product entry if scanner unavailable
Keyboard navigation still works
HID Biometric Attendance Devices
Common Devices in Pakistan Market:

U-Trust Biometric readers (USB, fingerprint, ~2000-5000 PKR)
Essl Time Track (fingerprint + RFID, ~3000-7000 PKR)
ZKTeco USB (fingerprint, ~1500-3000 PKR)
HID Compatibility:

Most devices present as USB HID or serial COM port
Proprietary SDKs available from manufacturers
USB data format varies by manufacturer
Fingerprint Data Storage:

Raw fingerprint templates stored locally (encrypted)
NOT raw fingerprint images
Employee ID linked to template
Templates encrypted in SQLite
Attendance Workflow:

Employee places finger on device
Device captures fingerprint template
Device sends employee ID + timestamp to USB interface
App receives via COM port or HID
Record written to attendance table with timestamp
Manual override option available (supervisor PIN required)
Device Integration:

backend/src/hardware/biometric/
├── biometric.service.ts    # Main biometric service
├── device.driver.ts        # USB/COM communication
├── template.manager.ts     # Fingerprint template storage
└── attendance.recorder.ts  # Attendance logging
Fallback Handling:

Manual attendance entry with reason
PIN/password alternative
Supervisor approval required for manual entry
7. DEVELOPMENT ENVIRONMENT SETUP
Required Tools & Versions
System Requirements:

Windows 10 (Build 2004+) or Windows 11
8GB RAM minimum (16GB recommended for dev)
20GB free disk space
Stable internet connection (for npm packages)
Core Tools:

# Node.js & npm
node --version    # v18.17.0 or higher
npm --version     # v9.0 or higher

# Rust (required for Tauri)
rustc --version   # Latest stable
cargo --version   # Comes with Rust

# TypeScript
npm install -g typescript
tsc --version     # v5.0 or higher

# Tauri CLI
npm install -g @tauri-apps/cli
tauri --version   # Latest
Development IDEs
Recommended:

Visual Studio Code (free, popular, excellent extensions)
WebStorm (paid, but has great TypeScript/NestJS support)
Visual Studio Community (free, heavyweight but powerful)
Essential VS Code Extensions:

ES7+ React/Redux/React-Native snippets
Tauri
Rust-analyzer
SQLite
Thunder Client (API testing)
GitLens
Development Dependencies
Frontend (frontend/package.json):

{
  "dependencies": {
    "react": "^18.x",
    "react-router-dom": "^6.x",
    "@tauri-apps/api": "^1.x",
    "zustand": "^4.x"
  },
  "devDependencies": {
    "typescript": "^5.x",
    "vite": "^4.x",
    "@vitejs/plugin-react": "^4.x",
    "@types/react": "^18.x"
  }
}
Backend (backend/package.json):

{
  "dependencies": {
    "@nestjs/common": "^10.x",
    "@nestjs/core": "^10.x",
    "@nestjs/typeorm": "^9.x",
    "typeorm": "^0.3.x",
    "sqlite3": "^5.x",
    "sqlcipher": "^4.x",
    "uuid": "^9.x"
  },
  "devDependencies": {
    "typescript": "^5.x",
    "@types/node": "^20.x",
    "ts-node-dev": "^2.x"
  }
}
Environment Configuration
Frontend Environment Variables (.env):

VITE_API_URL=http://localhost:3000
VITE_APP_NAME=FlexPOS
VITE_VERSION=1.0.0
Backend Environment Variables (.env):

NODE_ENV=development
PORT=3000
DATABASE_PATH=%APPDATA%/FlexPOS/data/flexpos.db
DB_ENCRYPTION_KEY=xxx  # From Windows Credential Manager
LOG_LEVEL=debug
8. DEPLOYMENT & DISTRIBUTION
Windows Installer (.exe)
Generated By: Tauri automatically
Location: src-tauri/target/release/bundle/msi/ (after build)
File Size: ~150-200MB (includes Webview2 runtime)

Installer Features:

System-level installation
Auto-start shortcut on Desktop
Start menu entries
Add/Remove Programs integration
Custom installation path selection
Build Command:

npm run tauri build
Auto-Updater Configuration
Mechanism:

Tauri built-in updater
Release files hosted on GitHub (or custom server)
Semantic versioning (1.0.0, 1.0.1, etc.)
Process:

App checks for updates on startup (or manual check)
If new version available: display notification
User clicks "Update"
New installer downloaded in background
Install when download complete
App restarts automatically
Configuration (tauri.conf.json):

{
  "updater": {
    "active": true,
    "endpoints": ["https://releases.example.com/"],
    "dialog": true,
    "pubkey": "xxx"
  }
}
Offline Installer Support
For customers without reliable internet:

Include backend + database bundle in installer
Pre-configured with default settings
First-run wizard for business setup
No additional downloads needed
Code Signing (Windows)
Purpose: Prevent "Unknown Publisher" security warning

Requirements:

Code signing certificate (from DigiCert, Sectigo, etc., ~300-500 USD/year)
Private key stored securely
Certificate configured in CI/CD pipeline
Execution:

signtool sign /f cert.pfx /p password /t http://timestamp.server app.exe
Installation Prerequisites
Windows 10 Specific:

WebView2 Runtime must be installed (automatic installer bundle)
.NET Framework 4.5+ (usually already installed)
Windows 11:

WebView2 built-in, no additional install needed
9. QUALITY ASSURANCE & TESTING SETUP
Testing Frameworks
Unit Tests:

Jest (frontend and backend)
Vitest (lighter alternative)
Coverage target: >80%
Integration Tests:

Supertest (for API endpoints)
TypeORM test utilities
In-memory SQLite for database testing
End-to-End Tests:

Tauri E2E testing library
Playwright or Cypress (for UI automation)
Full workflow testing (sale from start to finish)
Test Structure:

tests/
├── unit/
│   ├── inventory.service.test.ts
│   ├── sales.service.test.ts
│   └── ...
├── integration/
│   ├── sales-to-ledger.test.ts
│   ├── backup-restore.test.ts
│   └── ...
└── e2e/
    ├── pos-complete-sale.test.ts
    ├── offline-sync.test.ts
    └── ...
Performance Testing Targets
POS Transaction:

Time from barcode scan to receipt print: <1 second
Database write: <100ms
Receipt formatting: <50ms
Printer output: <500ms
Data Sync:

Sync 1000 transactions: <5 seconds
Conflict resolution: <1 second per conflict
Verification: <500ms
Backup Operations:

Full database backup: <2 minutes
Encryption: <500ms
Compression: <500ms
File write: <1 minute
Report Generation:

Sales summary (10,000+ records): <5 seconds
Profit & margin calculation: <3 seconds
PDF export: <2 seconds
Offline Testing Methodology
Scenarios to Test:

No Internet on Startup: App should load with cached license
Internet Drops During Sale: Transaction queued, completed when online
Grace Period Exceeded: Sales locked, data readable
Sync Conflicts: Manual resolution or automatic based on timestamp
Backup Without Internet: Local backup succeeds, Google Drive sync waits
Tools:

Windows Network Throttling (Dev Tools)
Airplane mode simulation
Network adapter disable/enable scripts
VPN disconnection scenarios
Crash Recovery Testing
Scenarios:

Power Failure Mid-Transaction: WAL recovery + transaction rollback
App Crash Mid-Backup: Backup marked incomplete, retry on next backup
Database Corruption: SQLite integrity check + auto-repair if possible
Incomplete Sync: Queue persists, resumes on reconnect
Tools:

Force kill process while transaction in progress
Unplug power (with UPS or simulator)
Corrupt database file manually, verify recovery
Truncate backup file, test validation
Load Testing
Scenarios:

10,000 products in database
100+ concurrent POS terminals (not required for MVP, future-ready)
10,000 sales transactions per day
1000+ employees with attendance records
Tools:

Artillery or JMeter for backend load testing
Custom scripts to populate database with test data
Performance profiling (Chrome DevTools, Node.js profiler)
10. SYSTEM REQUIREMENTS & COMPATIBILITY MATRIX
End-User System Requirements
Minimum Specification:

OS:        Windows 10 (Build 2004) or Windows 11
Processor: Intel Core 2 Duo / AMD equivalent (2GHz+)
RAM:       4GB
Storage:   10GB free space
Display:   1024x768 resolution
Keyboard:  Standard USB or wireless
Mouse:     Optional (touchscreen alternative)
Internet:  Optional (for sync/backups only)
Recommended Specification:

OS:        Windows 11
Processor: Intel Core i5 / AMD Ryzen 5 (3GHz+)
RAM:       8GB
Storage:   20GB free SSD space
Display:   1366x768 or higher
Keyboard:  Mechanical USB keyboard
Network:   Gigabit Ethernet or WiFi 5+
Hardware Compatibility Matrix
| Device | Model Examples | Windows Driver | Status |
|--------|---|---|---|
| Printer | Epson TM-T82, Star SM-L300 | Generic USB | ✅ Fully Supported |
| Scanner | Honeywell 3310g, Symbol DS3508 | Generic HID | ✅ Fully Supported |
| Biometric | ZKTeco U100, Essl TimeTrak | Manufacturer SDK | ✅ Supported |

Network Requirements
For Cloud Sync/Backups:

Minimum: 1 Mbps download/upload
Recommended: 10 Mbps
Connection type: WiFi or Ethernet
No special firewall rules (uses standard HTTPS)
For Local Network (Future):

Multi-branch sync: 100 Mbps LAN
Latency: <50ms
Stability: Automatic retry on disconnect
11. TECHNOLOGY RATIONALE
Why Tauri (not Electron)?
| Factor | Tauri | Electron |
|--------|-------|----------|
| Bundle Size | 150MB | 300MB+ |
| Memory Usage | 50-100MB | 200-300MB |
| Windows Native | ✅ Yes | ❌ No |
| Auto-updater | ✅ Built-in | ⚠️ Requires lib |
| Code Signing | ✅ Easy | ⚠️ Complex |
| WebView2 | ✅ Fast | ❌ Chromium bundled |
| Startup Time | <2s | 3-5s |

Verdict: Tauri is lighter, faster, and better for resource-constrained businesses.

Why SQLite + SQLCipher (not PostgreSQL)?
| Factor | SQLite | PostgreSQL |
|--------|--------|-----------|
| Server Required | ❌ No | ✅ Yes |
| Offline Operation | ✅ Native | ⚠️ Requires sync |
| Setup Complexity | Simple | Complex |
| Zero Data Loss | ✅ WAL mode | ✅ Native |
| Encryption | ✅ SQLCipher | ⚠️ Extension |
| Typical DB Size | <1GB | Unlimited |

Verdict: SQLite is perfect for offline-first, single-business, resource-constrained deployment.

Why NestJS (not Express)?
| Factor | NestJS | Express |
|--------|--------|---------|
| Structure | ✅ Opinionated | ❌ Flexible |
| TypeScript | ✅ Native | ⚠️ Via babel |
| DI Container | ✅ Built-in | ❌ Manual |
| Modules | ✅ Built-in | ❌ Manual |
| ORM Integration | ✅ Easy | ⚠️ Manual |
| Enterprise | ✅ Designed for | ⚠️ Basic |

Verdict: NestJS enforces clean architecture from day one, critical for multi-module system.

Why Supabase (not Firebase)?
| Factor | Supabase | Firebase |
|--------|----------|----------|
| Open Source | ✅ Yes | ❌ No |
| SQL Database | ✅ PostgreSQL | ⚠️ Firestore NoSQL |
| Cost Control | ✅ Predictable | ⚠️ Pay-as-you-go |
| Self-hosted | ✅ Option | ❌ No |
| OAuth | ✅ Full | ✅ Full |
| Licensing Module | ✅ Custom SQL | ⚠️ Limited by NoSQL |

Verdict: Supabase gives us full control + SQL for complex licensing queries.

12. KNOWN LIMITATIONS & CONSTRAINTS
Platform Limitations
Webview2 Runtime Requirement (Windows 10)
Users must install WebView2 runtime (free, automated)
Windows 11
