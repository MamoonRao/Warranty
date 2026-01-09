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

---

## 3. LOCAL BACKEND STACK: NODE.JS + NESTJS

### Node.js Requirements

**Version:**
- Node.js 18.x LTS (minimum)
- Node.js 20.x LTS (recommended)
- Must be x64 architecture for Windows (x86 deprecated)

**Installation:**
- Download from nodejs.org
- Add to system PATH
- Verify: node --version and npm --version

**Windows Service Setup:**
- Backend runs as Windows service (via NSSM or similar)
- Auto-starts with Windows
- Auto-restart on crash
- Logs to Windows Event Viewer
NestJS Configuration
**Version:** 10.x or higher

**Project Structure:**

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

**Core Configuration:**
- Dependency Injection via NestJS (decorators)
- Module-based architecture (each feature = module)
- Global Exception Filters for error handling
- Middleware for logging, validation
- Guards for authorization checks

**API Port:**
- Default: 3000 (can be changed)
- Only accessible locally (127.0.0.1)
- No external network exposure (Tauri IPC only)

**Database Connection:**
- SQLite via TypeORM or Prisma
- Connection pooling (default: 5-10 connections)
- Transaction support for atomic operations
- Migrations via TypeORM/Prisma

**Logging:**
- Winston or Pino recommended
- Log files in %APPDATA%/FlexPOS/logs/
- Daily rotation, 30-day retention
- Separate logs for: general, database, errors, audit

**Background Jobs:**
- Bull for job queue
- Redis not required (in-memory option available)
- Backup jobs scheduled via node-cron
- Sync queue manager for offline operations

**Development Dependencies**
```bash
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
```
---

## 4. LOCAL DATABASE: SQLITE + SQLCIPHER

### SQLite Configuration

**Version:** 3.44.0 or higher
- Lightweight, serverless architecture
- File-based (single .db file)
- No separate database server needed
- ACID compliance for data safety

**Location:**
%APPDATA%\FlexPOS\data\flexpos.db

**Backup Location:**
%APPDATA%\FlexPOS\backups\
├── daily/           # Daily backups (7 kept)
├── weekly/          # Weekly backups (4 kept)
├── manual/          # User-initiated backups
└── cloud/           # Google Drive synced backups

### SQLCipher Encryption

**Purpose:** Encrypt the entire SQLite database at rest

**Version:** SQLCipher v4 (compatible with SQLite 3.x)

**Encryption Standard:**
- AES-256 encryption
- 64,000 PBKDF2 iterations (default)
- Each database has unique encryption key

**Key Management:**
- Master key stored in Windows Credential Manager (secure)
- Alternative: Environment variable (less secure, fallback only)
- Key never written to disk in plaintext
- On app startup: retrieve key from Credential Manager, unlock database

**Backup Encryption:**
- Backup files inherit database encryption
- Encrypted backups are portable between machines
- Restore requires correct master key

**Performance Impact:**
- ~5-10% slower than unencrypted
- WAL mode mitigates impact
- Acceptable for POS operations

### Write-Ahead Logging (WAL) Mode

**Purpose:** Prevent data corruption during crashes/power failures

**Configuration:**
```bash
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;  -- Balance between safety and speed
PRAGMA cache_size = -64000;   -- 64MB cache
PRAGMA temp_store = MEMORY;
```

**Benefits:**
- Readers don't block writers
- Better concurrency
- Faster transactions
- Automatic recovery from crashes

**Automatic Cleanup:**
- WAL checkpoint every 1000 pages
- Checkpoints trigger on connection close
- Manual checkpoint on app exit

### Connection Pooling

**Strategy:**
- Pool size: 5-10 connections (configurable)
- Min idle: 2 connections
- Connection timeout: 30 seconds
- Idle timeout: 5 minutes

**TypeORM/Prisma Setup:**
```bash
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
```

### Transaction Handling

**Isolation Levels:**
- Default: READ_COMMITTED (SQLite default)
- SERIALIZABLE for critical operations (sales, payroll)
- Transaction timeout: 30 seconds

**Atomicity Guarantees:**
- All-or-nothing: either all operations succeed or all rollback
- No partial transactions in database
- Automatic rollback on error

**Example Critical Operations:**
Sales Transaction = atomic
├── Create Sale record
├── Update Inventory
├── Create Ledger entries
├── Print Receipt
└── Trigger Backup
(If any step fails, entire transaction rolls back)

### Database Size & Performance

**Expected Growth:**
- 10,000 products: ~5-10MB
- 100,000 transactions: ~20-30MB
- Full year of operations: ~200-500MB

**Indexing Strategy:**
- Primary indexes on: id, created_at, product_id, customer_id
- Composite indexes for: (user_id, created_at), (product_id, warehouse_id)
- Query analyzer to identify missing indexes
- Avoid over-indexing (slows writes)

**Performance Targets:**
- Insert: <5ms
- Query (with index): <10ms
- Batch insert (1000 records): <500ms
- Complex report query: <2 seconds

---

## 5. CLOUD SERVICES: SUPABASE

### Supabase Project Setup

**Purpose:**
- Licensing management (SaaS control)
- Admin dashboard (multi-tenant)
- Optional: Cloud backup storage
- Optional: Cloud sync (future phase)

**Setup Steps:**
1. Create Supabase project at supabase.com
2. Configure authentication method
3. Create tables for: licenses, businesses, admin users
4. Generate API keys (public + secret)
5. Store keys in environment variables

### Authentication (JWT-based)

**Local Authentication:**
- Username/password for business users
- Stored locally in SQLite
- PBKDF2 password hashing

**SaaS Admin Authentication:**
- OAuth via Supabase Auth or Google
- JWT token for admin dashboard
- Multi-user support (different admins for different clients)

**License Validation:**
- Verify license status with Supabase on startup
- Cache validation result locally (grace period: 7 days)
- Re-verify when app goes online

### Real-Time Capabilities

**Not required for MVP,** but available for:
- Admin notifications (license expiry, unusual activity)
- Multi-user sync notifications
- Future phase: collaborative features

### Google Drive Backup Integration

**OAuth Flow:**
1. User clicks "Backup to Google Drive"
2. Browser opens Google OAuth consent screen
3. User grants permission
4. OAuth token stored securely in Credential Manager
5. Backup file encrypted and uploaded to user's Google Drive
6. Sync status shown to user

**File Structure on Google Drive:**

My Drive/
└── FlexPOS Backups/
    ├── Business Name/
    │   ├── daily/
    │   ├── weekly/
    │   └── manual/

**Permissions Required:**
- https://www.googleapis.com/auth/drive.file (create, update, delete only)
- NOT full Drive access

### API Key Management

**Storage:**
- Public API key: In code (safe, anonymous)
- Secret API key: Environment variable or Credential Manager
- License keys: Supabase database (encrypted column)

**Environment Variables:**
```bash
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_PUBLIC_KEY=eyJxxx...
SUPABASE_SECRET_KEY=eyJzzz...  # Server-side only
GOOGLE_OAUTH_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=xxx
```

---

## 6. HARDWARE INTEGRATION

### ESC/POS Thermal Printers

**Compatible Printers:**
- Epson TM series (TM-U220, TM-T20, TM-T82, etc.)
- Star Micronics series
- Zebra ZD series
- Any ESC/POS compatible printer

**Driver Installation:**
- Windows uses generic USB or LPT drivers
- ESC/POS is a standard protocol (not vendor-specific)
- No special driver needed (direct USB communication)

**Protocol Specifications:**
- ESC/POS command set (standard)
- 80mm or 58mm paper width support
- 203 DPI resolution typical
- Baud rate: 9600 (serial) or USB native

**Windows Integration:**
- Direct USB communication via WinUSB driver
- Alternative: Windows Print Spooler (slower but reliable)
- Device detection via USB VID:PID matching

**Integration Code Location:**
backend/src/hardware/printer/
├── printer.service.ts      # Main printer service
├── espos.driver.ts         # ESC/POS protocol handler
├── device.detector.ts      # USB printer detection
└── receipt.formatter.ts    # Receipt template formatting

**Fallback Mechanism:**
- If printer unavailable: Show receipt on screen (PDF option)
- Buffer receipt data for later printing
- Alert user to connect printer

### USB Barcode Scanners

**Compatible Devices:**
- Any USB HID keyboard-emulation scanner
- Common in Pakistan: Honeywell, Symbol, Datalogic, etc.
- Cost: 500-2000 PKR

**HID Protocol:**
- Scanners emulate keyboard input
- No special driver needed (Windows HID driver handles it)
- Scanned code appears as keyboard text + Enter

**Scanning Protocols Supported:**
- Code128, Code39, EAN13, EAN8, UPC-A, QR codes
- Configurable via scanner buttons (manufacturer-specific)

**Integration:**
- Capture keyboard input in POS screen
- Focus on barcode input field automatically
- Scan → field filled → Enter key triggers product lookup

**Fallback:**
- Manual product entry if scanner unavailable
- Keyboard navigation still works

### HID Biometric Attendance Devices

**Common Devices in Pakistan Market:**
- U-Trust Biometric readers (USB, fingerprint, ~2000-5000 PKR)
- Essl Time Track (fingerprint + RFID, ~3000-7000 PKR)
- ZKTeco USB (fingerprint, ~1500-3000 PKR)

**HID Compatibility:**
- Most devices present as USB HID or serial COM port
- Proprietary SDKs available from manufacturers
- USB data format varies by manufacturer

**Fingerprint Data Storage:**
- Raw fingerprint templates stored locally (encrypted)
- NOT raw fingerprint images
- Employee ID linked to template
- Templates encrypted in SQLite

**Attendance Workflow:**
1. Employee places finger on device
2. Device captures fingerprint template
3. Device sends employee ID + timestamp to USB interface
4. App receives via COM port or HID
5. Record written to attendance table with timestamp
6. Manual override option available (supervisor PIN required)

**Device Integration:**
backend/src/hardware/biometric/
├── biometric.service.ts    # Main biometric service
├── device.driver.ts        # USB/COM communication
├── template.manager.ts     # Fingerprint template storage
└── attendance.recorder.ts  # Attendance logging

Fallback Handling:
- Manual attendance entry with reason
- PIN/password alternative
- Supervisor approval required for manual entry

---

## 7. DEVELOPMENT ENVIRONMENT SETUP

### Required Tools & Versions

**System Requirements:**
- Windows 10 (Build 2004+) or Windows 11
- 8GB RAM minimum (16GB recommended for dev)
- 20GB free disk space
- Stable internet connection (for npm packages)

**Core Tools:**
```bash
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
```

### Development IDEs

**Recommended:**
- Visual Studio Code (free, popular, excellent extensions)
- WebStorm (paid, but has great TypeScript/NestJS support)
- Visual Studio Community (free, heavyweight but powerful)

**Essential VS Code Extensions:**
- ES7+ React/Redux/React-Native snippets
- Tauri
- Rust-analyzer
- SQLite
- Thunder Client (API testing)
- GitLens

### Development Dependencies

**Frontend (frontend/package.json):**
```json
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
```
**Backend (backend/package.json):**
```json
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
```

### Environment Configuration

**Frontend Environment Variables (.env):**
```typescript
VITE_API_URL=http://localhost:3000
VITE_APP_NAME=FlexPOS
VITE_VERSION=1.0.0
```
**Backend Environment Variables (.env):**
```react
NODE_ENV=development
PORT=3000
DATABASE_PATH=%APPDATA%/FlexPOS/data/flexpos.db
DB_ENCRYPTION_KEY=xxx  # From Windows Credential Manager
LOG_LEVEL=debug
```

---

## 8. DEPLOYMENT & DISTRIBUTION

### Windows Installer (.exe)

**Generated By:** Tauri automatically
**Location:** src-tauri/target/release/bundle/msi/ (after build)
**File Size:** ~150-200MB (includes Webview2 runtime)

**Installer Features:**
- System-level installation
- Auto-start shortcut on Desktop
- Start menu entries
- Add/Remove Programs integration
- Custom installation path selection

**Build Command:**
```bash
npm run tauri build
```

### Auto-Updater Configuration

**Mechanism:**
- Tauri built-in updater
- Release files hosted on GitHub (or custom server)
- Semantic versioning (1.0.0, 1.0.1, etc.)

**Process:**
1. App checks for updates on startup (or manual check)
2. If new version available: display notification
3. User clicks "Update"
4. New installer downloaded in background
5. Install when download complete
6. App restarts automatically

**Configuration (tauri.conf.json):**
```json
{
  "updater": {
    "active": true,
    "endpoints": ["https://releases.example.com/"],
    "dialog": true,
    "pubkey": "xxx"
  }
}
```

### Offline Installer Support

**For customers without reliable internet:**
- Include backend + database bundle in installer
- Pre-configured with default settings
- First-run wizard for business setup
- No additional downloads needed

### Code Signing (Windows)

**Purpose:** Prevent "Unknown Publisher" security warning

**Requirements:**
- Code signing certificate (from DigiCert, Sectigo, etc., ~300-500 USD/year)
- Private key stored securely
- Certificate configured in CI/CD pipeline

**Execution:**
```bash
signtool sign /f cert.pfx /p password /t http://timestamp.server app.exe
```

### Installation Prerequisites

**Windows 10 Specific:**
- WebView2 Runtime must be installed (automatic installer bundle)
- .NET Framework 4.5+ (usually already installed)

**Windows 11:**
- WebView2 built-in, no additional install needed

---

## 9. QUALITY ASSURANCE & TESTING SETUP

### Testing Frameworks

**Unit Tests:**
- Jest (frontend and backend)
- Vitest (lighter alternative)
- Coverage target: >80%

**Integration Tests:**
- Supertest (for API endpoints)
- TypeORM test utilities
- In-memory SQLite for database testing

**End-to-End Tests:**
- Tauri E2E testing library
- Playwright or Cypress (for UI automation)
- Full workflow testing (sale from start to finish)

**Test Structure:**

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

### Performance Testing Targets

**POS Transaction:**
- Time from barcode scan to receipt print: **<1 second**
- Database write: <100ms
- Receipt formatting: <50ms
- Printer output: <500ms

**Data Sync:**
- Sync 1000 transactions: **<5 seconds**
- Conflict resolution: <1 second per conflict
- Verification: <500ms

**Backup Operations:**
- Full database backup: **<2 minutes**
- Encryption: <500ms
- Compression: <500ms
- File write: <1 minute

**Report Generation:**

- Sales summary (10,000+ records): **<5 seconds**
- Profit & margin calculation: <3 seconds
- PDF export: <2 seconds

### Offline Testing Methodology

**Scenarios to Test:**
1. No Internet on Startup: App should load with cached license
2. Internet Drops During Sale: Transaction queued, completed when online
3. Grace Period Exceeded: Sales locked, data readable
4. Sync Conflicts: Manual resolution or automatic based on timestamp
5. Backup Without Internet: Local backup succeeds, Google Drive sync waits

**Tools:**
- Windows Network Throttling (Dev Tools)
- Airplane mode simulation
- Network adapter disable/enable scripts
- VPN disconnection scenarios

### Crash Recovery Testing

**Scenarios:**
1. **Power Failure Mid-Transaction:** WAL recovery + transaction rollback
2. **App Crash Mid-Backup:** Backup marked incomplete, retry on next backup
3. **Database Corruption:** SQLite integrity check + auto-repair if possible
4. **Incomplete Sync:** Queue persists, resumes on reconnect

**Tools:**
- Force kill process while transaction in progress
- Unplug power (with UPS or simulator)
- Corrupt database file manually, verify recovery
- Truncate backup file, test validation

### Load Testing

**Scenarios:**
- 10,000 products in database
- 100+ concurrent POS terminals (not required for MVP, future-ready)
- 10,000 sales transactions per day
- 1000+ employees with attendance records

**Tools:**
- Artillery or JMeter for backend load testing
- Custom scripts to populate database with test data
- Performance profiling (Chrome DevTools, Node.js profiler)

---

## 10. SYSTEM REQUIREMENTS & COMPATIBILITY MATRIX

### End-User System Requirements

**Minimum Specification:**

OS:        Windows 10 (Build 2004) or Windows 11
Processor: Intel Core 2 Duo / AMD equivalent (2GHz+)
RAM:       4GB
Storage:   10GB free space
Display:   1024x768 resolution
Keyboard:  Standard USB or wireless
Mouse:     Optional (touchscreen alternative)
Internet:  Optional (for sync/backups only)

**Recommended Specification:**

OS:        Windows 11
Processor: Intel Core i5 / AMD Ryzen 5 (3GHz+)
RAM:       8GB
Storage:   20GB free SSD space
Display:   1366x768 or higher
Keyboard:  Mechanical USB keyboard
Network:   Gigabit Ethernet or WiFi 5+

### Hardware Compatibility Matrix

| Device | Model Examples | Windows Driver | Status |
|--------|---|---|---|
| Printer | Epson TM-T82, Star SM-L300 | Generic USB | ✅ Fully Supported |
| Scanner | Honeywell 3310g, Symbol DS3508 | Generic HID | ✅ Fully Supported |
| Biometric | ZKTeco U100, Essl TimeTrak | Manufacturer SDK | ✅ Supported |

### Network Requirements

**For Cloud Sync/Backups:**
- Minimum: 1 Mbps download/upload
- Recommended: 10 Mbps
- Connection type: WiFi or Ethernet
- No special firewall rules (uses standard HTTPS)

**For Local Network (Future):**
- Multi-branch sync: 100 Mbps LAN
- Latency: <50ms
- Stability: Automatic retry on disconnect

---

## 11. TECHNOLOGY RATIONALE

### Why Tauri (not Electron)?

| Factor | Tauri | Electron |
|--------|-------|----------|
| Bundle Size | 150MB | 300MB+ |
| Memory Usage | 50-100MB | 200-300MB |
| Windows Native | ✅ Yes | ❌ No |
| Auto-updater | ✅ Built-in | ⚠️ Requires lib |
| Code Signing | ✅ Easy | ⚠️ Complex |
| WebView2 | ✅ Fast | ❌ Chromium bundled |
| Startup Time | <2s | 3-5s |

**Verdict:** Tauri is lighter, faster, and better for resource-constrained businesses.

### Why SQLite + SQLCipher (not PostgreSQL)?

| Factor | SQLite | PostgreSQL |
|--------|--------|-----------|
| Server Required | ❌ No | ✅ Yes |
| Offline Operation | ✅ Native | ⚠️ Requires sync |
| Setup Complexity | Simple | Complex |
| Zero Data Loss | ✅ WAL mode | ✅ Native |
| Encryption | ✅ SQLCipher | ⚠️ Extension |
| Typical DB Size | <1GB | Unlimited |

**Verdict:** SQLite is perfect for offline-first, single-business, resource-constrained deployment.

### Why NestJS (not Express)?

| Factor | NestJS | Express |
|--------|--------|---------|
| Structure | ✅ Opinionated | ❌ Flexible |
| TypeScript | ✅ Native | ⚠️ Via babel |
| DI Container | ✅ Built-in | ❌ Manual |
| Modules | ✅ Built-in | ❌ Manual |
| ORM Integration | ✅ Easy | ⚠️ Manual |
| Enterprise | ✅ Designed for | ⚠️ Basic |

**Verdict:** NestJS enforces clean architecture from day one, critical for multi-module system.

### Why Supabase (not Firebase)?

| Factor | Supabase | Firebase |
|--------|----------|----------|
| Open Source | ✅ Yes | ❌ No |
| SQL Database | ✅ PostgreSQL | ⚠️ Firestore NoSQL |
| Cost Control | ✅ Predictable | ⚠️ Pay-as-you-go |
| Self-hosted | ✅ Option | ❌ No |
| OAuth | ✅ Full | ✅ Full |
| Licensing Module | ✅ Custom SQL | ⚠️ Limited by NoSQL |

**Verdict:** Supabase gives us full control + SQL for complex licensing queries.

---

## 12. KNOWN LIMITATIONS & CONSTRAINTS

### Platform Limitations

1. **WebView2 Runtime Requirement (Windows 10)**
   - Users must install WebView2 runtime (free, automated)
   - Windows 11 has it built-in
   - One-time installation, <150MB

2. **Single-Machine License Model**
   - License tied to specific device
   - Cannot move license between machines without admin unlock
   - Justification: Security, anti-piracy, device tracking

3. **Offline Grace Period (7 days)**
   - After 7 days without internet, sales are locked
   - Data remains accessible (read-only)
   - Justification: License enforcement, prevents indefinite offline use

4. **SQLite Single-Writer Limitation**
   - Only one process can write at a time (enforced by Tauri/NestJS single backend)
   - Multiple POS terminals on same machine require queuing
   - Justification: Prevents database corruption, keeps design simple

5. **Backup File Size Limits**
   - Practical limit: ~500MB per backup
   - Google Drive upload: Encrypted, subject to bandwidth
   - Justification: Cloud storage costs, bandwidth constraints in Pakistan

### Device-Specific Limitations

1. **Printer Availability**
   - System degrades gracefully if printer unavailable
   - Receipt shown on screen as PDF alternative
   - No forced printing (user choice)

2. **Biometric Device Compatibility**
   - Limited to HID or standard USB serial devices
   - Custom manufacturer SDKs may require integration
   - Fingerprint template storage format manufacturer-specific

3. **Barcode Scanner**
   - Keyboard emulation required (HID standard)
   - Proprietary scanners may need special drivers

---

## 13. MIGRATION & UPGRADE STRATEGY

### Database Migrations

**Approach:** TypeORM or Prisma migrations
- Each schema change = new migration file
- Migrations are version-numbered and ordered
- Backward compatibility maintained (old data not deleted)
- Automatic rollback on error

**Upgrade Process:**
1. App detects schema version mismatch
2. Runs pending migrations in order
3. Validates migration success
4. Creates backup before migration (fail-safe)
5. User notified of upgrade completion

**Example Migration:**
```typescript
// migration/1704067200000-AddProductCategory.ts
export class AddProductCategory implements MigrationInterface {
  async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.addColumn('product', 
      new TableColumn({ name: 'category_id', type: 'varchar' })
    );
  }
  
  async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropColumn('product', 'category_id');
  }
}
```

### Backup Compatibility

**Restore Requirements:**
- Backup includes schema version
- Restore process validates compatibility
- If incompatible: user prompted to upgrade first, then restore
- Prevents data loss from version mismatches

### Code Updates

**Update Delivery:**
- Auto-updater downloads new app version
- Staging directory, validate before install
- Automatic restart and relaunch after update
- No manual intervention needed

**Rollback Safety:**
- Previous version kept as backup
- If new version fails to start, rollback automatic
- User notified of rollback

---

## 14. MAINTENANCE & OPERATIONS

### Logging Strategy

**Log Locations:**
%APPDATA%\FlexPOS\logs\
├── app-{YYYY-MM-DD}.log          # General application logs
├── database-{YYYY-MM-DD}.log     # Database operations
├── errors-{YYYY-MM-DD}.log       # Error and exceptions
├── audit-{YYYY-MM-DD}.log        # Compliance and audit trail
└── sync-{YYYY-MM-DD}.log         # Offline sync operations

**Log Retention:**
- Daily rotation (new file per day)
- 30-day retention (older logs auto-deleted)
- Compressed (gzip) after 7 days to save space
- Accessible via UI for troubleshooting

**Log Levels:**
- ERROR: Critical failures
- WARN: Unusual conditions
- INFO: Major operations (sale, sync, backup)
- DEBUG: Development only (disabled in production)

### Monitoring & Diagnostics

**Health Checks:**
- Database integrity check on startup
- Disk space monitoring
- Backup status verification
- License status display

**Diagnostic Tools:**
- Export diagnostic bundle (logs + database info, no sensitive data)
- Send to support for analysis
- No PII included (business names hashed, customer counts only)

### Crash Reporting (Optional)

**SaaS Admin Dashboard Shows:**
- Crash frequency by version
- Common error patterns
- Affected customers
- Helps prioritize bug fixes

**User Privacy:**
- No automatic crash reporting (opt-in)
- No PII transmitted
- Errors hashed and anonymized

---

## 15. SECURITY ARCHITECTURE

### Data Encryption

**At Rest:**
- SQLite + SQLCipher: AES-256
- Backup files: Encrypted same as source
- Local configuration: Windows Credential Manager
- API keys: Never logged, stored in Credential Manager

**In Transit:**
- Cloud sync: HTTPS TLS 1.2+
- Google Drive: OAuth token over HTTPS
- Supabase: All connections HTTPS

**Biometric Data:**
- Fingerprint templates only (not raw images)
- Stored encrypted in SQLite
- Never transmitted to cloud
- Deleted on employee termination

### Access Control

**Role-Based Access Control (RBAC):**
- Super Admin (SaaS owner): Full control
- Business Owner: Business configuration, user management
- Manager: Sales, inventory, reports, staff management
- Cashier: POS only
- Accountant: Ledger, reports, no sales modification

**Permission Enforcement:**
- Every action validated against user role
- Denied actions logged
- Override actions require manager/owner approval
- Audit trail recorded

### Audit Trail

**Captured Events:**
- User login/logout
- Sale creation/modification/deletion
- Inventory adjustments
- Payment processing
- Payroll processing
- System settings changes
- License expiry/renewal
- Backup creation/restore

**Audit Log Fields:**
- User ID
- Action (what happened)
- Timestamp
- Affected records (IDs)
- Before/after values (for modifications)
- IP address (local only, 127.0.0.1)
- Status (success/failure)

---

## 16. PERFORMANCE TUNING

### Database Optimization

**Indexes (Critical):**
```bash
CREATE INDEX idx_sales_date ON sales(created_at);
CREATE INDEX idx_inventory_product ON inventory(product_id);
CREATE INDEX idx_customer_phone ON customers(phone);
CREATE INDEX idx_payroll_employee ON payroll(employee_id, month);
```

**Query Optimization:**
- N+1 query prevention (batch loading)
- Pagination for large result sets
- Caching frequently accessed data
- Materialized views for complex reports

**Cache Strategy:**
- L1: In-memory (Redux/Zustand) - frontend state
- L2: Backend memory cache (1 hour TTL)
- L3: Database with indexes

### Memory Management

**Frontend:**
- Code splitting (lazy load modules)
- Image optimization
- Virtual scrolling for large lists
- Cleanup on component unmount

**Backend:**
- Connection pooling (prevents resource exhaustion)
- Request timeout (30 seconds)
- Memory leak monitoring
- Worker threads for heavy computation

### Disk Space Management

**Monitoring:**
- Check available space on startup
- Warn if <1GB free
- Prevent operation if <500MB free
- Auto-cleanup old backups (keep only 30 days)

**Optimization:**
- WAL checkpoint on app exit (compacts database)
- Backup compression (gzip, typically 70% reduction)
- Log rotation (auto-delete >30 days)

---

## 17. DISASTER RECOVERY

### Backup Verification

**Backup Integrity Checks:**
1. File size validation (non-zero)
2. SQLite pragma check (file is valid SQLite)
3. Record count validation (expected rows present)
4. Encryption verification (encrypted flag set)
5. CRC checksum validation

**Automatic Verification:**
- After every backup completes
- Weekly verification of all backups
- Alerts if corruption detected
- Suggestions to create new backups

### Restore Process

**Pre-Restore Checks:**
1. Backup file validation
2. Version compatibility check
3. Free disk space verification
4. Current database backup created (extra safety)

**Restore Steps:**
1. Stop NestJS backend
2. Decrypt backup file
3. Validate data integrity
4. Swap database file
5. Run post-restore migrations (if needed)
6. Restart backend
7. Verify database accessibility
8. Display completion notification

**Automatic Rollback:**
- If restore fails at any step
- Original database restored automatically
- User notified of failure and reason
- Support contact information provided

---

###  18-Month Retention Policy

**Backup Retention:**
Local Backups:
├── Last 7 daily backups (kept)
├── Last 4 weekly backups (kept)
└── Monthly snapshots (kept for 12 months)

Google Drive Backups:
└── All uploaded (user's responsibility to manage)

Total Local Storage: ~2-5GB for full year

---

## 18. COMPLIANCE & STANDARDS

### Data Protection

**Local Data:**
- GDPR-compliant storage (no data transmitted without consent)
- Encryption at rest (AES-256)
- No auto-backup to cloud without user opt-in

**Cloud Data (Supabase):**
- GDPR compliance depends on Supabase data location
- Customer data isolated per business
- Automatic deletion on account closure (30-day grace)

### Audit Compliance

**Financial Compliance:**
- Immutable transaction records (no editing, only reversals)
- Complete audit trail (who did what, when)
- Ledger balanced checks
- Monthly reconciliation recommended

**Pakistan Regulatory:**
- NTN/STR number storage (optional)
- GST compliance (tax calculations logged)
- Invoice format flexible (customizable)

---

## 19. SUPPORT & DOCUMENTATION

### Developer Documentation

**Location:** /docs/ folder in repository
- Architecture decisions (ADRs)
- API contracts
- Database schema
- Module responsibilities
- Setup guides
- Troubleshooting guide

### User Documentation

**Location:** In-app Help + PDF manual
- Getting started guide
- POS workflow walkthrough
- Backup/Restore procedures
- Troubleshooting
- FAQ

### SaaS Admin Documentation

**Location:** Supabase admin dashboard
- Client management
- License issuance/revocation
- Feature toggles
- Payment tracking
- Support access

---

## 20. COST ESTIMATION

### Infrastructure Costs (Annual)

| Component | Cost | Notes |
|-----------|------|-------|
| Supabase (Pro) | $25/month | 500GB storage, auth, realtime |
| Google Cloud (OAuth) | Free | Within free tier |
| Code Signing Cert | $400/year | DigiCert or Sectigo |
| Development | In-house | 6-8 months dev time |
| **Total First Year** | **~$700+dev** | Recurring: ~$300/year |

### Per-Installation Costs

| Component | Cost | Notes |
|-----------|------|-------|
| License per business | 500-1,000 PKR/month | Recurring SaaS |
| Installation | Free | Installer download |
| Support | Included | Basic support in license |
| Hardware (optional) | 10,000-100,000 PKR | Printer, scanner, biometric |
| **Total Setup** | **10,000-100,000 PKR** | One-time |

---

## 21. NEXT STEPS (PHASE 2)

### Database Schema Design (Phase 2 Task 1)
- Define all tables (Products, Sales, Inventory, etc.)
- Relationships and foreign keys
- Indexes and constraints
- Migration files

### Core Engines Implementation (Phase 2 Tasks 2-6)
- Inventory Engine
- Sales Engine
- Ledger Engine
- Payroll Engine
- Licensing Engine
- Backup Engine

---

## SUMMARY

This technology stack is **production-grade, offline-first, and designed for Pakistani SMBs** with:
- **Lightweight:** Minimal resource requirements
- **Secure:** Encryption at rest and in transit
- **Reliable:** Zero-data-loss guarantees via WAL + backups
- **Scalable:** Supports 10,000+ products, thousands of transactions
- **Maintainable:** Clean architecture, well-documented, enterprise patterns

### Ready for Phase 2 database schema design and core engine implementation.
