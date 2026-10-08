# Requirement: UI + Full Backend Transformation (Docker → Database)

## Overview

This document defines the plan to transform the Latch project from a Docker-backed architecture to a database-backed architecture, while maintaining a common template that works across different backends (Midnight, zk, etc.). The goal is to decouple the UI from specific backend implementations and replace Docker infrastructure with database operations.

---

## Current Architecture (Docker-Backed)

### Stack
- **Docker Compose** (`docker-compose.yml`): Runs node, indexer, and proof-server
- **API** (`api/`): Uses `@midnight-ntwrk/midnight-js-*` providers that connect to:
  - Indexer (GraphQL/REST for contract state)
  - Proof server (HTTP for ZK proof generation)
  - Wallet providers (DApp connector API)
- **Contract** (`contract/`): Compact moat contract code
- **Web** (`web/`): React UI with two modes:
  - **Demo mode**: Deterministic SHA-256 fixtures (`MockMoatClient`)
  - **Real mode**: Connects to Midnight network via `RealMoatClient`

### Data Flow
```
UI → MoatClient (interface) → MockMoatClient / RealMoatClient → API providers
       ↑                                      ↑
   Demo fixtures                        Indexer/Proof providers
```

### Key Docker Services
| Service | Image | Purpose |
|---------|-------|---------|
| `node` | `midnightntwrk/midnight-node:0.22.5` | Midnight blockchain node |
| `indexer` | `midnightntwrk/indexer-standalone:4.0.2` | GraphQL API for contract state |
| `proof-server` | `midnightntwrk/proof-server:8.0.3` | ZK proof generation |

---

## Target Architecture (Database-Backed)

### Core Principle
Replace Docker infrastructure with a **persistent database** that stores:
- Capability state (policy, spend-state, commitments)
- Authorization receipts and nullifiers
- Authorization requests and proofs
- Capability metadata (alias, agent, category, budget, uses)

### Database Requirements
- **Schema** for capabilities, receipts, requests, nullifiers
- **Operations**: create, read, update, revoke, verify
- **Must support** the same logical operations as the current Docker stack
- **Should be backend-agnostic** (work with PostgreSQL, SQLite, MySQL, etc.)

### New Data Flow
```
UI → MoatClient (interface) → API → Database → API → UI
                                    ↑
                              Template pattern
```

---

## Template Pattern for Backend Agnosticism

The key requirement is that the **template remains the same** for backend, midnight, zk, etc. This means:

### 1. Abstract the MoatClient Interface
The `MoatClient` interface in `web/src/services/moat-client.ts` should remain unchanged. Implementations can swap:

```typescript
// Demo implementation (no backend needed)
import { MockMoatClient } from './services/mock-moat-client';

// Midnight backend
import { RealMoatClient } from './services/real-moat-client';
import { createMoatProviders } from './api/providers';

// zk/backend-XXX implementation
import { ZkMoatClient } from './services/zk-moat-client';
```

### 2. API Layer Abstraction
The API layer should depend on an abstract **DatabaseProvider** interface, not concrete Midnight.js providers:

```typescript
// Before (Docker/Midnight-specific)
import { createMoatProviders } from '@bond/api/src/providers';

// After (database-backed)
import { DatabaseProvider } from '@bond/shared/src/database';
```

### 3. Unified API Endpoints
Replace Docker-dependent API routes with database operations:

| Current (Docker) | Target (Database) |
|------------------|-------------------|
| `GET /capability/:id` → query indexer | `GET /capability/:id` → query DB |
| `POST /capability` → deploy contract | `POST /capability` → insert into DB |
| `POST /authorize` → proof submission | `POST /authorize` → update DB receipts |
| `GET /receipt/:commitment` → query contract | `GET /receipt/:commitment` → query DB |

### 4. Backend-Specific Implementations
Each backend (Midnight, zk, etc.) provides its own:

- **Database adapter** (PostgreSQL, MongoDB, etc.)
- **Proof provider** (or skip if not needed)
- **Wallet connector** (may remain unchanged)

The UI and API remain the same; only the database implementation changes.

---

## Migration Steps

### Phase 1: Foundation (Weeks 1-2)
1. **Design database schema** for capability state, receipts, nullifiers
2. **Create database adapter layer** in `api/` with abstract interface
3. **Extract MockMoatClient logic** into database operations
4. **Keep web UI unchanged** - it already uses the `MoatClient` interface

### Phase 2: API Transformation (Weeks 3-4)
1. **Replace docker-compose API calls** with database queries
2. **Remove Midnight.js provider dependencies** from `api/src/providers.ts`
3. **Implement database operations**:
   - `createCapability()` → DB insert
   - `authorizeSpend()` → DB update + receipt generation
   - `revokeCapability()` → DB update
   - `getCapability()` → DB query
   - `verifyReceipt()` → DB check
4. **Update API scripts** (`deploy.ts`, etc.) to use DB instead of contract deployment

### Phase 3: Backend Template (Weeks 5-6)
1. **Create template implementations** for different backends:
   - `database/midnight/` - Midnight-specific adapter
   - `database/zk/` - zk-specific adapter
   - `database/sqlite/` - SQLite for testing/local
2. **Ensure same API surface** across all backends
3. **Update documentation** for each backend setup

### Phase 4: UI Verification (Weeks 7-8)
1. **Run existing tests** with database backend
2. **Verify demo mode still works** (no backend needed)
3. **Test real mode** with each backend template
4. **Update lint/typecheck** configurations

---

## Backend Template Structure

Each backend template should expose the same `MoatProviders` shape:

```typescript
// Common interface all backends must implement
export interface DatabaseProvider {
  // Capability operations
  createCapability(input: CreateCapabilityInput): Promise<CreateCapabilityResult>;
  getCapability(capabilityId: string): Promise<CapabilityOwnerState | null>;
  revokeCapability(capabilityId: string): Promise<TxResult>;
  
  // Authorization operations
  authorizeSpend(input: { capabilityId: string; request: SpendRequest }): Promise<AuthorizationResult>;
  verifyReceipt(receiptCommitment: string): Promise<boolean>;
  generateOneTimeDestination(merchant: MerchantMetaAddress, requestNonce: string): Promise<OneTimeDestination>;
  
  // State queries
  getContractState?: () => Promise<any>;
}
```

### Example: SQLite Template (`database/sqlite/`)

```typescript
// database/sqlite/index.ts
import { DatabaseProvider } from '../interfaces';

export class SqliteProvider implements DatabaseProvider {
  // Implement all methods using better-sqlite3
  async createCapability(input: CreateCapabilityInput): Promise<CreateCapabilityResult> {
    // DB insert logic
  }
  // ... other methods
}
```

### Example: Midnight Template (`database/midnight/`)

```typescript
// database/midnight/index.ts  
import { DatabaseProvider } from '../interfaces';
import { createMoatProviders } from '@bond/api/src/providers';

// Wrap Midnight.js providers with DB operations
export class MidnightProvider implements DatabaseProvider {
  // Uses existing Midnight.js stack but persists to DB
}
```

---

## Key Changes Required

### 1. `api/` Directory
- [ ] Replace `providers.ts` with database-agnostic implementation
- [ ] Create `database/` directory with template adapters
- [ ] Update `deploy.ts` to use DB instead of `deployContract`
- [ ] Update `mock-client.ts`/`real-client.ts` to use DB
- [ ] Remove/dockerize-free `local:*` scripts

### 2. `contract/` Directory
- [ ] Compact contract may still be needed for Midnight backend
- [ ] Or replace with database-equivalent operations
- [ ] Keep `moat.compact` if Midnight-specific proofs required

### 3. `web/` Directory
- [ ] **Minor changes** - the UI already uses abstract `MoatClient` interface
- [ ] May need to swap `MockMoatClient` ↔ `RealMoatClient` based on config
- [ ] No fundamental UI redesign needed if interface stays the same

### 4. `docker-compose.yml`
- [ ] **Remove or repurpose** - no longer needed as primary infrastructure
- [ ] Keep for backward compatibility / local development if desired
- [ ] Replace `local:*` npm scripts with database equivalents

### 5. `.env.example`
- [ ] Replace Midnight network variables with database connection vars
- [ ] Add `DATABASE_URL` or equivalent
- [ ] Keep `MIDNIGHT_NETWORK=demo` as fallback

---

## Success Criteria

- [ ] UI functions identically in demo mode (no backend needed)
- [ ] API endpoints work with database backend (create, read, update, revoke capabilities)
- [ ] Same `MoatClient` interface works across all backend templates
- [ ] Template pattern allows swapping backend without UI changes
- [ ] Docker compose no longer required for basic functionality
- [ ] All existing tests pass with database backend
- [ ] New backends can be added by implementing `DatabaseProvider` interface

---

## Starting Point

**Begin with Phase 1**: Design the database schema and extract the `MoatClient` operations into database methods. The web UI is already prepared for this change since it depends on the abstract interface, not concrete implementations.

**Key file to modify first**: `api/src/providers.ts` - replace Midnight.js provider creation with database-agnostic logic.