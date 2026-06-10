
# ADR-001: Storage Abstraction Layer

## Context
The Idempotency Framework must support diverse storage engines depending on the target infrastructure of the deploying application. For instance, high-throughput microservice fabrics require ultra-low-latency in-memory databases like Redis. Conversely, small-scale deployments or strict financial applications might prefer leveraging their existing relational systems (e.g., PostgreSQL or MySQL) to maintain structural consistency without introducing operational overhead from new infrastructure dependencies. 

Hardcoding data-access operations or relying on specific Object-Relational Mapping frameworks (ORMs) creates tight coupling, prevents pluggable growth, and limits adoption across different technology stacks.

## Options Considered

### 1. Hardcode Redis Connection Drivers
* **Description:** Optimize the internal engine exclusively for Redis protocols, utilizing raw connection strings, standard Lua scripts for atomic status modifications, and native TTL expiration windows.
* **Pros:** Exceptional operational throughput ($P99 < 2\text{ms}$ updates), simple code footprint, and native support for distributed lock patterns via single-instance keys or cluster setups.
* **Cons:** Forces an extra database dependency onto every engineering group, even if their application only requires occasional background job deduping.

### 2. Hardcode PostgreSQL / Enterprise Relational Targets
* **Description:** Build the idempotency transaction log using standard relational databases, relying on ACID guarantees, raw SQL tables, and row-level locking mechanisms (`SELECT FOR UPDATE`).
* **Pros:** Highly consistent, easy to monitor, and fits naturally into standard database backups without needing separate infrastructure setup.
* **Cons:** Slower transaction lookups, higher connection pool overhead, and less efficient data purging compared to dedicated in-memory key-value caches.

### 3. Abstract Storage & Locking Provider Interface Layers
* **Description:** Isolate all persistence, verification, and mutual exclusion actions behind pure TypeScript interface signatures (`IIdempotencyStore` and `ILockProvider`). The core engine interacts only with these abstract interface boundaries.
* **Pros:** Complete database flexibility, highly testable through simple in-memory mock adapters, and decoupled from framework-specific dependencies.
* **Cons:** Adds abstract structural layers to the core codebase, requiring clear interface maintenance and documentation for team developers building custom database adapters.

## Decision
We will use **Option 3: Abstract Storage & Locking Provider Interface Layers**.

By establishing clean interface definitions, the core framework remains pure, framework-agnostic, and completely testable in isolation. Storage implementations will be decoupled into independent, pluggable packages (e.g., `@framework/store-redis`, `@framework/store-postgres`).

---

## Technical Interface Blueprint

The framework's core runtime engine must interact exclusively with the following abstract interface contracts:

```typescript
export interface IdempotencyRecord {
  keyHash: string;
  clientKey: string;
  status: 'STARTED' | 'PROCESSING' | 'COMPLETED' | 'FAILED';
  payloadHash?: string;
  responseCode?: number;
  responseHeaders?: Record<string, string>;
  responseBody?: string | Buffer;
  lockedAt: Date;
  expiresAt: Date;
}

export interface IIdempotencyStore {
  /** Retrieves an active or completed record matching a SHA-256 key hash. */
  get(keyHash: string): Promise<IdempotencyRecord | null>;

  /** Creates a raw idempotency baseline record during initial validation phases. */
  create(record: IdempotencyRecord): Promise<void>;

  /** Atomic transition of an existing record state to guard against concurrent writes. */
  updateStatus(keyHash: string, status: IdempotencyRecord['status'], optionalData?: Partial<IdempotencyRecord>): Promise<void>;

  /** Erases or forcefully evicts an invalid or expired key record. */
  evict(keyHash: string): Promise<void>;
}

export interface ILockProvider {
  /** Acquires an exclusive distributed lease for a specific timeframe. */
  acquireLock(lockKey: string, ttlMs: number): Promise<boolean>;

  /** Forcefully terminates an active lock lease, opening the execution pathway for retries. */
  releaseLock(lockKey: string): Promise<void>;
}
```

## Consequences

### Positive Impacts

- **Pluggable Architecture:** Engineering groups can choose whichever data store fits their active runtime infrastructure best.
    
- **High Testability:** Developers can run comprehensive unit tests using clean, in-memory arrays to mock the database layer, eliminating the need to spin up external Docker containers during CI pipelines.
    
- **Future-Proofing:** Simplifies porting the core logic to other platforms or runtime variants (such as AWS Lambda or Hono edge nodes) without rewriting the core engine.
    

### Negative Impacts

- **Slight Implementation Overhead:** Third-party storage adapters must implement these interfaces exactly, requiring clear conformance testing suites to ensure consistency across different database backends.
    
- **Abstraction Complexity:** Abstracting away specific database optimization features (like Redis Lua scripts or Postgres advisory locks) means we must carefully manage atomic operations through the interface layer itself.