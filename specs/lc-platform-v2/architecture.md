# LC Platform v2 - System Architecture

**Created:** 2026-02-09
**Version:** 1.0
**Status:** Draft
**Last Updated:** 2026-02-09

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [System Context](#system-context)
3. [Package Architecture](#package-architecture)
4. [Component Architecture](#component-architecture)
5. [Data Flow](#data-flow)
6. [Integration Points](#integration-points)
7. [Architectural Decisions](#architectural-decisions)

---

## Architecture Overview

**High-Level Summary:**

LC Platform v2 transforms a proof-of-concept into a feature-complete release by integrating four independent packages (parent repo, dev-accelerators, processing-lib, CLI) into a cohesive system. The architecture follows Clean Architecture (hexagonal) principles with clear separation between domain logic, business use cases, infrastructure implementations, and adapter interfaces.

**Architectural Style:** Clean Architecture (Hexagonal) + Multi-Package Monorepo

**Key Principles:**
- **Dependency Inversion**: Inner layers define interfaces; outer layers implement them
- **Single Source of Truth**: lcp-manifest.yaml defines application topology
- **Provider Independence**: Cloud-agnostic core with provider-specific adapters
- **Separation of Concerns**: Control Plane (infrastructure) vs Data Plane (application runtime)
- **Testability**: Mock provider enables complete local development

**Package Dependency Graph:**
```
lc-platform-config (types, schema, validation)
  ↓ imports
lc-platform-dev-accelerators (clients, services, runtime)
  ↓ imports
lc-platform-processing-lib (configurators, domain)
  ↓ imports
lc-platform-dev-cli (commands, UI)
```

---

## System Context

**Package Overview:**

```
┌─────────────────────────────────────────────────────────────┐
│                    lc-platform (parent)                      │
│         Governance, docs, submodule coordination             │
└─────────────────────────────────────────────────────────────┘
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │ lc-platform- │  │ lc-platform- │  │ lc-platform- │
    │    config    │  │     dev-     │  │  processing- │
    │              │  │ accelerators │  │     lib      │
    └──────────────┘  └──────────────┘  └──────────────┘
            ▲                 ▲                 ▲
            │                 │                 │
            └─────────────────┴─────────────────┘
                              │
                    ┌─────────────────┐
                    │  lc-platform-   │
                    │   dev-cli       │
                    └─────────────────┘
```

**External Systems:**
```
┌──────────────┐
│  Developer   │
│  (TypeScript)│
└──────┬───────┘
       │ imports @stainedhead/lc-platform-dev-accelerators
       ▼
┌────────────────────────────────────────┐
│        LCAppRuntime                    │
│  ┌──────────────────────────────────┐ │
│  │  runtime.store('backups')        │ │
│  │  runtime.queue('orders')         │ │
│  └──────────────────────────────────┘ │
└──────┬─────────────────────┬──────────┘
       │                     │
       ▼                     ▼
┌──────────────┐      ┌──────────────┐
│  AWS (S3,    │      │  Mock        │
│  SQS, etc.)  │      │  Provider    │
└──────────────┘      └──────────────┘


┌──────────────┐
│  Enterprise  │
│  Domain User │
└──────┬───────┘
       │ uses lcp CLI
       ▼
┌────────────────────────────────────────┐
│           lcp CLI                      │
│  ┌──────────────────────────────────┐ │
│  │ lcp dependency add backups       │ │
│  │ lcp deploy dependencies --env dev│ │
│  └──────────────────────────────────┘ │
└──────┬─────────────────────┬──────────┘
       │                     │
       ▼                     ▼
┌──────────────────┐  ┌──────────────────┐
│ processing-lib   │  │ lcp-manifest.yaml│
│ (configurators)  │  │ (declarative)    │
└──────────────────┘  └──────────────────┘
```

---

## Package Architecture

### lc-platform-config (NEW)

**Purpose:** Shared configuration, manifest schema, type definitions

**Exports:**
- Manifest schema (ManifestData, DependencyDeclaration, etc.)
- YAML parser with environment variable interpolation
- Configuration resolution logic
- Validation functions

**Dependencies:** None (zero runtime deps on other LC packages)

**Key Files:**
```
src/
  manifest/
    schema.ts          # ManifestData, metadata, provider types
    parser.ts          # YAML parsing with ${ENV:-default}
  resolution/
    resolver.ts        # Config precedence resolution
  validation/
    naming.ts          # Domain-driven naming validation
    dependency.ts      # Dependency validation
```

---

### lc-platform-dev-accelerators (ENHANCED)

**Purpose:** Provider-agnostic service wrappers and runtime

**NEW in v2:**
- ResourceResolver (maps monikers to resource names)
- 11 Scoped*Client wrappers
- Moniker-based LCAppRuntime accessors
- fromManifest() and fromEnvironment() factories

**Layers:**
```
Domain Layer (Core)
  ↓
Ports (Interfaces)
  ↓
Adapters (AWS, Mock, Azure, GCP)
```

**Key Components:**
- **LCAppRuntime**: Runtime entry point with moniker-based methods
- **ResourceResolver**: Resolves moniker → cloud resource name
- **IContextLoader**: Port for loading application context
- **Scoped Clients**: Pre-bound client wrappers (ScopedObjectClient, etc.)
- **Provider Factories**: BaseProviderFactory<T> pattern

---

### lc-platform-processing-lib (ENHANCED)

**Purpose:** Clean Architecture domain model and use cases

**NEW in v2:**
- Real adapter implementations (AcceleratorStorageAdapter, etc.)
- AcceleratorStorageAdapterFactory

**Layers:**
```
Domain (entities, interfaces)
  ↓
Use Cases (configurators)
  ↓
Adapters (implementations)
```

**Key Components:**
- **ApplicationConfigurator**: Application lifecycle use cases
- **VersionConfigurator**: Version management use cases
- **IStorageProvider**: Port for persistence
- **AcceleratorStorageAdapter**: Implementation using dev-accelerators ObjectClient

---

### lc-platform-dev-cli (ENHANCED)

**Purpose:** Command-line interface for platform operations

**NEW in v2:**
- lcp dependency commands (add, remove, list, show, validate)
- Active application context (app use, -a flag)
- Error enrichment layer
- Context export (--json for agents)

**Architecture:**
```
CLI Commands (Adapter Layer)
  ↓
processing-lib (Use Case Layer)
  ↓
dev-accelerators (Infrastructure Layer)
  ↓
Cloud Providers (AWS, Mock, etc.)
```

---

## Component Architecture

### Component: ResourceResolver

**Responsibility:**
Maps business-domain monikers (e.g., "backups") to cloud resource names (e.g., "lcp-123456-backend-order-svc-store-backups-dev")

**Dependencies:**
- IContextLoader (interface)
- generateDependencyResourceName() (from nameGenerator.ts)

**Provides:**
- `resolve(moniker): ResolvedDependency`
- `resolveTyped(moniker, type): ResolvedDependency`
- `listMonikers(): string[]`

**Lifecycle:**
- Created per-application (not singleton)
- Initialized with ApplicationContext from loader
- Cached by LCAppRuntime

**Concurrency:**
- Thread-safe (read-only after initialization)

---

### Component: LCAppRuntime (Enhanced)

**Responsibility:**
Provide unified runtime API for accessing cloud resources by moniker

**NEW Methods:**
```typescript
// Factory methods
static fromManifest(path: string): Promise<LCAppRuntime>
static fromEnvironment(): Promise<LCAppRuntime>

// Initialization
async initialize(): Promise<void>

// Moniker-based accessors (11 total)
store(moniker: string): ScopedObjectClient
queue(moniker: string): ScopedQueueClient
secrets(moniker: string): ScopedSecretsClient
config(moniker: string): ScopedConfigClient
events(moniker: string): ScopedEventPublisher
notifications(moniker: string): ScopedNotificationClient
documents(moniker: string): ScopedDocumentClient
data(moniker: string): ScopedDataClient
auth(moniker: string): ScopedAuthClient
cache(moniker: string): ScopedCacheClient
containerRepo(moniker: string): ScopedContainerRepoClient
```

**PRESERVED Methods (backward compatibility):**
```typescript
getObjectClient(): ObjectClient
getQueueClient(): QueueClient
// ... all existing get*Client() methods
```

---

### Component: CLI Dependency Commands (NEW)

**Commands:**
- `lcp dependency add <moniker> --type <type> [options]`
- `lcp dependency list`
- `lcp dependency show <moniker>`
- `lcp dependency remove <moniker>`
- `lcp dependency validate`

**Responsibility:**
Manage dependencies in lcp-manifest.yaml programmatically

**Integration:**
- Reads/writes lcp-manifest.yaml
- Validates against schema (lc-platform-config)
- Calls processing-lib for complex operations

---

## Data Flow

### Flow 1: Application Developer Using Runtime

**Scenario:** Developer writes code using moniker-based API

```
Developer Code
  |
  | const runtime = await LCAppRuntime.fromManifest('./lcp-manifest.yaml');
  ▼
LCAppRuntime.fromManifest()
  |
  | 1. Reads lcp-manifest.yaml
  | 2. Creates ManifestContextLoader
  | 3. Creates ResourceResolver
  | 4. Calls initialize()
  ▼
ResourceResolver.initialize()
  |
  | Loads ApplicationContext from manifest
  ▼
Developer Code
  |
  | const backups = runtime.store('backups');
  ▼
LCAppRuntime.store(moniker)
  |
  | 1. Calls resolver.resolveTyped('backups', 'object-store')
  | 2. Gets ResolvedDependency with resourceName
  | 3. Creates ScopedObjectClient(rawClient, resourceName)
  | 4. Caches and returns scoped client
  ▼
ScopedObjectClient
  |
  | Developer calls: await backups.put('file.txt', data)
  ▼
ScopedObjectClient.put(key, data)
  |
  | Calls: this.client.put(this.bucket, key, data)
  ▼
ObjectClient.put(bucket, key, data)
  |
  | Provider-specific implementation (AWS or Mock)
  ▼
Cloud Storage (S3 or Mock)
```

---

### Flow 2: CLI User Managing Dependencies

**Scenario:** User adds dependency via CLI

```
User Command
  |
  | lcp dependency add backups --type object-store --encryption aes256
  ▼
CLI Parser (options.ts)
  |
  | 1. Parse command and flags
  | 2. Resolve active app context (if -a not specified)
  | 3. Build CliContext
  ▼
dependency/add.ts Command Handler
  |
  | 1. Read lcp-manifest.yaml
  | 2. Validate dependency doesn't exist
  | 3. Create DependencyDeclaration
  | 4. Add to manifest.dependencies
  | 5. Validate complete manifest
  | 6. Write lcp-manifest.yaml
  ▼
lcp-manifest.yaml (updated)
  |
  | dependencies:
  |   backups:
  |     type: object-store
  |     config:
  |       encryption: aes256
  ▼
User sees success message
```

---

### Flow 3: CLI User Deploying Dependencies

**Scenario:** User deploys infrastructure

```
User Command
  |
  | lcp deploy dependencies --env dev
  ▼
CLI deploy Command
  |
  | 1. Read lcp-manifest.yaml
  | 2. Resolve configuration (env, provider, region)
  | 3. Create processing-lib adapters (via factory)
  ▼
processing-lib DeployDependencies Use Case
  |
  | For each dependency in manifest:
  |   1. Resolve resource name (with environment)
  |   2. Call LCPlatform service to create resource
  ▼
dev-accelerators LCPlatform Services
  |
  | e.g., LCObjectStoreService.create(bucket, config)
  ▼
Provider Adapter (AWS or Mock)
  |
  | AWS: Creates S3 bucket
  | Mock: Stores in-memory
  ▼
Cloud Infrastructure Created
```

---

## Integration Points

### Integration 1: Manifest Loading

**Type:** File System
**Purpose:** Read lcp-manifest.yaml for configuration
**Protocol:** YAML file read

**Implementation:**
```typescript
// lc-platform-config/src/manifest/parser.ts
export async function parseManifest(path: string): Promise<ManifestData> {
  const content = await fs.readFile(path, 'utf-8');
  const interpolated = interpolateEnvVars(content);
  const parsed = yaml.parse(interpolated);
  validateManifest(parsed);
  return parsed as ManifestData;
}
```

**Error Handling:**
- File not found: Clear error message with suggestion to run `lcp app init`
- Invalid YAML: Syntax error with line number
- Schema violation: Validation error with field path

---

### Integration 2: CLI → Processing-Lib

**Type:** TypeScript module imports
**Purpose:** CLI commands delegate to processing-lib use cases
**Protocol:** Function calls

**Implementation:**
```typescript
// lc-platform-dev-cli/src/cli/commands/app/init.ts
import { ApplicationConfigurator } from '@stainedhead/lc-platform-processing-lib';
import { createStorageAdapter } from '../../shared/adapter-factory';

async function initializeApp(context: CliContext) {
  const storageAdapter = createStorageAdapter(context);
  const configurator = new ApplicationConfigurator(storageAdapter);
  const result = await configurator.init({
    account: context.account!,
    team: context.team!,
    moniker: context.moniker!,
  });

  if (!result.success) {
    throw enrichError(result.error);
  }

  return result.value;
}
```

**Error Handling:**
- Result<T, E> pattern from processing-lib
- CLI enriches errors with suggestions
- `--debug` shows full error details

---

### Integration 3: Processing-Lib → Dev-Accelerators

**Type:** TypeScript module imports
**Purpose:** Processing-lib adapters use dev-accelerators clients
**Protocol:** Function calls

**Implementation:**
```typescript
// lc-platform-processing-lib/src/adapters/storage/AcceleratorStorageAdapter.ts
import { ObjectClient } from '@stainedhead/lc-platform-dev-accelerators';

export class AcceleratorStorageAdapter implements IStorageProvider {
  constructor(
    private readonly objectClient: ObjectClient,
    private readonly configBucket: string
  ) {}

  async read<T>(path: string): Promise<Result<T, StorageError>> {
    try {
      const data = await this.objectClient.get(this.configBucket, path);
      return { success: true, value: JSON.parse(data.data.toString()) };
    } catch (error) {
      return { success: false, error: mapToStorageError(error) };
    }
  }
}
```

---

## Architectural Decisions

### ADR-001: Use lcp-manifest.yaml as Single Source of Truth

**Date:** 2026-02-09
**Status:** Accepted

**Context:**
Current system has configuration scattered across CLI args, config files, mock filesystem, and in-memory state. No declarative view of application infrastructure.

**Decision:**
Introduce `lcp-manifest.yaml` as canonical declarative configuration for application topology. All dependency declarations, metadata, and provider settings live in this file.

**Rationale:**
- **Infrastructure as Code**: Manifest can be version-controlled, reviewed, deployed
- **AI Agent Friendly**: Machine-readable structure for code generation
- **Enterprise Accessible**: Non-developers can understand and modify YAML
- **Validation**: Schema validation catches errors before deployment
- **Consistency**: Single source eliminates drift between systems

**Consequences:**

**Positive:**
- Clear contract for application infrastructure needs
- Enables manifest-driven workflows
- Simplifies CLI (fewer flags needed)
- Better developer experience (autocomplete from manifest)

**Negative:**
- Learning curve for YAML syntax
- File management overhead (keep in sync with code)
- Migration path needed for existing users

**Mitigation:**
- `lcp dependency add` edits manifest programmatically (users don't hand-edit)
- Sensible defaults minimize required configuration
- Clear validation messages guide corrections

**Alternatives Considered:**

1. **JSON Configuration**
   - Pros: Native TypeScript support, no YAML parser needed
   - Cons: No comments, less human-readable, less familiar to enterprise users
   - Why rejected: YAML is industry standard for config (Kubernetes, Docker Compose)

2. **Code-Based Configuration (TypeScript)**
   - Pros: Type safety, IDE support, programmatic
   - Cons: Requires compilation, not accessible to non-developers, harder for AI agents
   - Why rejected: Excludes enterprise domain experts

---

### ADR-002: Preserve Backward Compatibility for dev-accelerators API

**Date:** 2026-02-09
**Status:** Accepted

**Context:**
dev-accelerators is MVP complete with existing users. Need to add moniker-based API without breaking existing code.

**Decision:**
All existing `get*Client()` methods on `LCAppRuntime` remain unchanged. New moniker-based accessors (`store()`, `queue()`, etc.) are additive.

**Rationale:**
- **No Breaking Changes**: Existing users continue to work
- **Gradual Migration**: Users can migrate at their own pace
- **Dual API**: Support both raw clients and scoped clients

**Consequences:**

**Positive:**
- Zero disruption to existing users
- Confidence in upgrade path
- Both APIs coexist peacefully

**Negative:**
- Slight API surface expansion
- Need to maintain both patterns
- Documentation must explain both approaches

**Mitigation:**
- Mark raw methods as "advanced usage" in docs
- Guide new users to moniker-based API
- Consider deprecation in v2.0 (not v1.0)

---

### ADR-003: Bottom-Up Clean Architecture Implementation Order

**Date:** 2026-02-09
**Status:** Accepted

**Context:**
Four packages need integration. Must choose implementation order.

**Decision:**
Implement bottom-up following dependency graph:
1. lc-platform-config (types, schema)
2. dev-accelerators (resolver, scoped clients)
3. processing-lib (adapters)
4. CLI (commands)

**Rationale:**
- **Foundation First**: Build stable base before higher layers
- **TDD-Friendly**: Can test each layer independently as built
- **Early Integration**: Catch interface mismatches early (Phase 3)
- **Incremental Value**: Each phase delivers working, tested code

**Consequences:**

**Positive:**
- Predictable build order
- Strong foundation reduces rework
- Each phase has clear deliverables
- Integration issues caught in Phase 3 (before CLI work)

**Negative:**
- No visible user-facing features until Phase 4
- Delayed feedback from end-users
- Risk of over-engineering lower layers

**Mitigation:**
- Regular demos of intermediate layers
- Integration tests in Phase 3 validate approach
- Spec provides clear target to avoid over-engineering

---

## Trade-offs

### Trade-off 1: Manifest Complexity vs. Flexibility

**Choice:** YAML manifest with type-specific configuration options

**Benefits:**
- Flexible enough for all 14 dependency types
- Extensible for future dependency types
- Clear structure (metadata, provider, dependencies)

**Costs:**
- Users must learn YAML syntax
- Type-specific options add complexity
- Validation errors can be cryptic

**Mitigation:**
- `lcp dependency add` generates correct YAML
- Interactive mode guides through options
- Clear validation messages with examples
- Sensible defaults reduce required config

---

### Trade-off 2: Provider Abstraction vs. Cloud-Specific Features

**Choice:** Cloud-agnostic core interfaces; provider-specific options in config

**Benefits:**
- Switch providers without code changes
- Consistent API across clouds
- Mock provider for local dev

**Costs:**
- May not expose all cloud-specific features
- Lowest-common-denominator for some services
- Provider-specific options in manifest reduce portability

**Mitigation:**
- Focus on common 80% use cases
- Document cloud-specific options clearly
- Escape hatch: use raw clients for advanced needs

---

## Performance Considerations

**Bottlenecks:**
- **Manifest parsing**: Could be slow for large manifests. Mitigation: Cache parsed manifest in memory.
- **Resource resolution**: Repeated moniker resolution. Mitigation: Cache ResolvedDependency objects.
- **Scoped client creation**: Could create many instances. Mitigation: Cache scoped clients by moniker.

**Optimization Strategies:**
- In-memory caching of parsed manifests
- Lazy initialization of scoped clients
- Reuse ObjectClient/QueueClient instances across scoped clients

**Performance Targets:**
- Manifest parsing + resolution: <500ms
- CLI command response: <1 second
- E2E workflow (init → add deps → deploy): <30 seconds

---

## References

- [spec.md](spec.md) - Feature specification
- [data-dictionary.md](data-dictionary.md) - Data structures
- [plan.md](plan.md) - Implementation plan
- Source PRD: `lc-platform-v2-prd.md`
