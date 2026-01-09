# FlexPOS - Offline-First POS SaaS for Pakistani SMBs

> **Status:** Phase 1 - Architecture & Design  
> **Platform:** Windows 10/11  
> **Architecture:** Offline-First, Clean Architecture  

## 🎯 Project Overview

FlexPOS is an enterprise-grade Point of Sale (POS) system designed specifically for Pakistani Small and Medium Businesses (SMBs). Built with an **offline-first** architecture, FlexPOS ensures your business never stops, regardless of internet connectivity.

### Key Features

- ✅ **100% Offline Operation** - Core POS functions work without internet
- ✅ **Zero Data Loss** - Transaction-safe operations with automatic backups
- ✅ **Sub-Second Performance** - <1 second transaction completion time
- ✅ **Enterprise Security** - AES-256 database encryption (SQLCipher)
- ✅ **Cloud Sync** - Automatic synchronization when online
- ✅ **Hardware Support** - Thermal printers, barcode scanners, biometric devices
- ✅ **Multi-User** - Role-based access control with permissions
- ✅ **7-Day Grace Period** - Continue working offline for up to 7 days

## 📚 Documentation

| Document | Description | Status |
|----------|-------------|--------|
| [architecture.md](./architecture.md) | Complete system architecture with diagrams | ✅ Complete |
| [api-contracts.md](./api-contracts.md) | Module boundaries and API contracts | ✅ Complete |
| data-flows.md | Critical data flow diagrams | ⏳ Next Task |
| tech-setup.md | Technology stack setup guide | ⏳ Pending |
| security-review.md | Security review and checklist | ⏳ Pending |

## 🏗️ Architecture Layers

```
┌─────────────────────────────────────────────────────────┐
│  UI Layer (Tauri + React + TypeScript)                 │
├─────────────────────────────────────────────────────────┤
│  Application Layer (Tauri Rust Backend)                │
├─────────────────────────────────────────────────────────┤
│  Business Logic Layer (NestJS + TypeScript)            │
├─────────────────────────────────────────────────────────┤
│  Domain Layer (Core Business Entities & Rules)         │
├─────────────────────────────────────────────────────────┤
│  Infrastructure Layer (SQLite + SQLCipher + Hardware)  │
├─────────────────────────────────────────────────────────┤
│  SaaS Integration Layer (Supabase Cloud)              │
└─────────────────────────────────────────────────────────┘
```

## 🧩 Core Modules

### 1. Sales Module
- Cart management
- Checkout processing
- Payment handling (cash, card, mobile, credit)
- Receipt printing
- Sale void and returns

### 2. Inventory Module
- Product catalog management
- Stock tracking and adjustments
- Categories and suppliers
- Purchase orders
- Low stock alerts

### 3. Customer Module
- Customer management
- Loyalty points program
- Customer accounts (credit)
- Purchase history

### 4. Auth Module
- User authentication (password + biometric)
- Role-based access control (RBAC)
- Permission management
- Session handling

### 5. Sync Module
- Async cloud synchronization
- Conflict detection and resolution
- Sync queue management
- Online/offline status monitoring

### 6. Hardware Module
- Thermal printer integration (ESC/POS)
- Barcode scanner support
- Biometric device integration
- Cash drawer control

### 7. Backup Module
- Automated daily backups
- Manual backup/restore
- Cloud backup upload
- Data export (JSON, CSV)

### 8. License Module
- License validation
- Feature flag management
- 7-day offline grace period
- Subscription management

### 9. Reports Module
- Daily/weekly/monthly sales reports
- Inventory reports
- Customer analytics
- Dashboard metrics

### 10. Config Module
- Store configuration
- System settings
- Tax rate management
- Receipt templates

## 🛠️ Technology Stack

### Frontend
- **Framework:** Tauri 1.5+ (Rust)
- **UI Library:** React 18+
- **Language:** TypeScript 5+
- **Styling:** TailwindCSS 3+
- **Components:** shadcn/ui
- **State Management:** Zustand + React Query
- **Forms:** React Hook Form
- **Tables:** TanStack Table

### Backend
- **Framework:** NestJS 10+
- **Runtime:** Node.js 20 LTS
- **Language:** TypeScript 5+
- **Database:** SQLite 3.42+ with SQLCipher 4.5+
- **ORM:** Prisma 5+
- **Validation:** class-validator
- **Logging:** Winston

### Cloud Services
- **Platform:** Supabase
- **Database:** PostgreSQL 15+
- **Auth:** Supabase Auth (OAuth, JWT)
- **Storage:** Supabase Storage
- **Realtime:** Supabase Realtime

### Hardware Integration
- **Printers:** node-escpos (ESC/POS protocol)
- **Scanners:** node-serialport
- **Biometric:** Vendor SDKs via FFI

## 📋 Phase 1 Progress

### ✅ Completed Tasks

- [x] **Task 1:** System Architecture Document
  - High-level architecture diagram
  - Layered architecture details
  - Component overview
  - Offline-first pattern design
  - Data isolation boundaries
  - Technology stack justification
  - Performance targets
  - Security architecture
  
- [x] **Task 2:** Module Boundaries & API Contracts
  - 10 core modules with clear boundaries
  - Tauri IPC command contracts (Frontend ↔ Rust)
  - NestJS REST API contracts (Rust/Frontend ↔ Backend)
  - Database repository contracts
  - Hardware driver interfaces
  - Supabase integration contracts
  - Shared TypeScript type definitions
  - Standardized error handling
  - State management contracts

### ⏳ Remaining Tasks

- [ ] **Task 3:** Data Flow Diagrams (Critical Flows)
  - Sale transaction flow (end-to-end)
  - Product lookup and stock check
  - Sync operation flow
  - Conflict resolution flow
  - License validation flow
  - Hardware communication flow
  
- [ ] **Task 4:** Technology Stack Confirmation & Setup
  - Development environment setup guide
  - Dependency installation instructions
  - Database initialization
  - Configuration templates
  - Hardware driver setup
  
- [ ] **Task 5:** Design Approval Checklist & Security Review
  - Architecture validation checklist
  - Security audit checklist
  - Performance benchmarks
  - Scalability considerations
  - Compliance review (if applicable)

## 🚀 Quick Start (Coming Soon)

Setup instructions will be available after Phase 1 Task 4 completion.

## 📖 API Documentation

### Tauri IPC Commands

All commands follow this pattern:

```typescript
import { invoke } from '@tauri-apps/api/tauri';

// Example: Create a sale
const sale = await invoke<Sale>('checkout', {
  cartId: 'cart-123',
  customerId: 'customer-456',
  payments: [
    { method: 'cash', amount: 1000 }
  ]
});
```

See [api-contracts.md](./api-contracts.md) for complete command reference.

### REST API

Base URL: `http://localhost:3000/api/v1`

All requests require JWT authentication:

```
Authorization: Bearer <jwt_token>
```

Standard response format:

```typescript
// Success
{
  "success": true,
  "data": { ... }
}

// Error
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message",
    "details": { ... }
  }
}
```

See [api-contracts.md](./api-contracts.md) for complete API reference.

## 🔒 Security Features

- **Database Encryption:** AES-256 encryption at rest (SQLCipher)
- **Network Encryption:** HTTPS/TLS 1.3 for all cloud communication
- **Authentication:** JWT tokens with refresh mechanism
- **Authorization:** Role-based access control (RBAC)
- **Audit Logging:** Comprehensive audit trail for all critical operations
- **Biometric Security:** Local-only fingerprint storage (never synced)
- **Secure Key Storage:** License keys encrypted in Windows Registry
- **Code Signing:** Windows Authenticode for trusted installation

## 📊 Performance Targets

| Operation | Target | Maximum |
|-----------|--------|---------|
| POS Transaction | <500ms | <1000ms |
| Product Search | <100ms | <200ms |
| Database Query | <50ms | <100ms |
| UI Interaction | <16ms | <32ms |
| Application Startup | <3s | <5s |
| Receipt Printing | <2s | <3s |

## 🛣️ Roadmap

### Phase 1: Architecture & Design (Current)
- System architecture
- API contracts
- Data flow diagrams
- Technology setup
- Security review

### Phase 2: Database Design
- Schema design
- Relationship mapping
- Migration strategy
- Data seeding

### Phase 3-4: Backend Implementation
- NestJS modules
- Database repositories
- Business logic
- Hardware drivers
- Sync service

### Phase 5: Frontend Implementation
- Tauri application shell
- React UI components
- State management
- Hardware integration

### Phase 6: Testing & Deployment
- Unit tests
- Integration tests
- E2E tests
- Performance testing
- Windows installer
- Deployment guide

## 🤝 Development Guidelines

### Code Style
- **TypeScript:** Strict mode enabled
- **ESLint:** Airbnb + custom rules
- **Prettier:** Enforced formatting
- **Naming:** camelCase (variables/functions), PascalCase (classes/types)

### Git Workflow
- **Branching:** `phase{N}-task{M}-{description}`
- **Commits:** Conventional commits format
- **PRs:** Required for all changes
- **Reviews:** Mandatory code review

### Testing Requirements
- **Unit Tests:** Minimum 80% coverage
- **Integration Tests:** All critical flows
- **E2E Tests:** Main user journeys
- **Performance Tests:** Meet performance targets

## 📝 License

Proprietary - All rights reserved

## 📧 Contact

For questions or support, please contact the FlexPOS development team.

---

**Last Updated:** 2024  
**Version:** 1.0.0 (Phase 1 - Design)  
**Status:** 🟡 In Development
