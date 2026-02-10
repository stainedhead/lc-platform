# LC Platform PRD: POC to Feature-Complete Product

## Document Purpose
This PRD defines the changes required to transform LC Platform from a proof-of-concept into a feature-complete initial release (v1.0.0) suitable for sharing with development teams and gathering enterprise feedback.

---

## Context

LC Platform is a cloud-agnostic development platform built on hexagonal architecture. It enables teams to build cloud solutions without deep cloud expertise. The platform currently exists as four repositories:

1. **lc-platform** (parent monorepo) - Governance, docs, submodule coordination
2. **lc-platform-dev-accelerators** (MVP Complete) - 14 control plane services + 11 data plane clients; AWS + Mock providers fully implemented
3. **lc-platform-dev-cli** (Partial MVP) - CLI with core commands but NOT integrated with processing-lib
4. **lc-platform-processing-lib** (Domain complete, adapters stubbed) - Clean Architecture domain model with stub adapters

**The core problem**: These three packages operate in isolation. The CLI bypasses the processing library (using mock filesystem at `~/.lcp/mock-apps.json`). The processing library's adapters are stubs (in-memory Maps). And critically, **developers must still pass raw cloud resource names** (S3 bucket names, SQS queue URLs) to every API call -- the platform does not yet deliver on its promise of hiding cloud complexity behind business-domain names.

**The goal**: A developer should be able to write `runtime.store('backups').put('report.pdf', data)` without ever knowing the underlying S3 bucket name. An enterprise domain expert should be able to run `lcp dependency add backups --type object-store` without understanding S3. An AI agent should be able to read a manifest file and generate correct platform code.

---

## 1. Current State Assessment

### What Works Well
- Hexagonal architecture with clean ports/adapters separation
- 14 service interfaces + 11 client interfaces -- comprehensive cloud service coverage
- `BaseProviderFactory<T>` pattern enables clean provider selection
- AWS implementations production-ready; Mock implementations excellent for testing
- `LCPlatformApp` with dependency registration and `nameGenerator.ts` for deterministic resource naming
- Domain model in processing-lib (Application, Version, Deployment entities with Result<T,E> types)
- CLI configuration system with 3-tier precedence (CLI > project-local > global)
- Test coverage: 81-94% across packages

### Critical Gaps

**Gap 1: No Moniker-Based Resource Access**
```typescript
// TODAY: Developer must know the cloud resource name
const client = runtime.getObjectClient();
await client.put('lcp-123456-backend-order-svc-store-backups', 'file.txt', data);

// TARGET: Developer uses business-domain moniker
const backups = runtime.store('backups');
await backups.put('file.txt', data);
```
The `nameGenerator.ts` generates resource names from `(account, team, moniker, depType, depName)` but nothing reverses this at runtime. `LCAppRuntime` has zero awareness of application identity or dependencies.

**Gap 2: No Component Integration**
- CLI `init.ts` (line 56): `// TODO: Replace with actual core library integration when available`
- CLI uses `loadMockApps()` / `saveMockApps()` writing to `~/.lcp/mock-apps.json`
- Processing-lib `AcceleratorStorageAdapter`: `private readonly inMemoryStorage = new Map<string, unknown>()`
- Zero imports of processing-lib exist in CLI source code

**Gap 3: No Application Manifest**
No declarative file defines what resources an application needs. Dependencies are scattered across in-memory state, mock files, and processing-lib storage.

**Gap 4: No Environment-Aware Resolution**
`nameGenerator.ts` has no environment parameter. Dev and prod resources would collide.

**Gap 5: DependencyConfiguration Union Type Incomplete**
`dependency.ts:144-148` only covers 4 of 14 dependency types in the union:
```typescript
export type DependencyConfiguration =
  | ObjectStoreConfiguration | QueueConfiguration
  | SecretsConfiguration | DataStoreConfiguration;
// Missing: DocumentStore, EventBus, Notification, Cache, WebHosting, etc.
```

**Gap 6: No Active Application Context in CLI**
Every CLI command requires the developer to re-specify `--account`, `--team`, and `--moniker` (or rely on config files). There is no concept of a "currently active application" that persists across commands within a session. This creates friction:
```bash
# TODAY: repetitive and error-prone
lcp app init --account 123456 --team backend --moniker order-svc
lcp dependency add backups --type object-store --account 123456 --team backend --moniker order-svc
lcp dependency add order-queue --type queue --account 123456 --team backend --moniker order-svc
```
Developers expect to "select" an app once and then issue commands against it, similar to `kubectl config use-context` or `git checkout`.

---

## 2. Vision: Target Developer Experience

### Application Developer
```typescript
import { LCAppRuntime } from '@stainedhead/lc-platform-dev-accelerators';

// Runtime configured from manifest + environment
const runtime = await LCAppRuntime.fromManifest('./lcp-manifest.yaml');

// Access resources by business name -- no cloud identifiers needed
const backups = runtime.store('backups');
await backups.put('reports/2026-Q1.pdf', reportData);

const orders = runtime.queue('order-processing');
await orders.send({ orderId: '12345', action: 'process' });

const apiKey = await runtime.secrets('stripe-api-key').get();
const cache = runtime.cache('session-store');
await cache.set('user:42', sessionData, { ttl: 3600 });
```

### CLI User (Developer or Enterprise Staff)
```bash
# Initialize app -- automatically becomes the active app context
lcp app init --team backend --moniker order-svc
# Active app: backend/order-svc

# All subsequent commands target the active app -- no flags needed
lcp dependency add backups --type object-store --encryption aes256
lcp dependency add order-processing --type queue --fifo
lcp dependency add stripe-api-key --type secrets

# Deploy and inspect -- cloud details are hidden
lcp deploy dependencies --env dev
lcp dependency list
# NAME               TYPE           STATUS     RESOURCE
# backups            object-store   deployed   lcp-123456-backend-order-svc-store-backups
# order-processing   queue          deployed   lcp-123456-backend-order-svc-queue-order-processing
```

```bash
# Working with multiple apps -- switch context explicitly
lcp app list
# TEAM       MONIKER      STATUS
# backend    order-svc    active
# backend    user-svc     active
# frontend   web-app      active

lcp app use order-svc              # switch active app (resolves by moniker)
lcp app use backend/user-svc       # or use team/moniker for disambiguation
lcp dependency list                # now shows user-svc dependencies

# One-off command against a different app (does not change active context)
lcp -a web-app dependency list
```

### AI Agent
```bash
lcp context export --json  # Machine-readable topology with access patterns
```
```json
{
  "dependencies": {
    "backups": {
      "type": "object-store",
      "accessPattern": "runtime.store('backups')"
    }
  }
}
```

---

## 3. User Personas

| Persona | Description | Key Need |
|---------|-------------|----------|
| **Application Developer** | TypeScript developer, limited cloud expertise | Write cloud code without learning S3/SQS/etc. APIs |
| **Enterprise Domain Expert** | Business analyst learning to code | Use business terminology, guided workflows |
| **AI Coding Agent** | LLM generating platform code | Deterministic interfaces, manifest-driven context |
| **Platform Engineer** | DevOps managing multi-cloud | Standardized naming, policy enforcement, auditing |

---

## 4. Requirements

### Theme A: Developer Abstraction Layer

#### A.1 Application Manifest File (Must-have)

**Problem**: No single declarative file defines an application's infrastructure needs.

**Solution**: Introduce `lcp-manifest.yaml` at project root.

```yaml
# lcp-manifest.yaml
apiVersion: lcp/v1
kind: Application
metadata:
  team: backend
  moniker: order-svc
  environment: ${LCP_ENVIRONMENT:-dev}
  owner: sarah@company.com
provider:
  type: ${LCP_PROVIDER:-mock}
  region: ${LCP_REGION:-us-east-1}
dependencies:
  backups:
    type: object-store
    config:
      versioning: true
      encryption: aes256
  order-processing:
    type: queue
    config:
      fifo: true
      visibilityTimeout: 30
      encryption: true
  stripe-api-key:
    type: secrets
    config:
      rotationEnabled: true
```

**Acceptance Criteria**:
- `lcp app init` generates manifest skeleton
- Supports `${ENV_VAR:-default}` interpolation
- `lcp validate` validates against schema with clear errors
- TypeScript types generated from manifest schema
- Dependency name (e.g., `backups`) is the moniker key

**Dependencies**: None (foundation for all other features)

**Critical files**:
- New: Schema definition in lc-platform-config (see C.1)
- `lc-platform-dev-accelerators/src/core/types/dependency.ts` -- `DependencyType` enum anchors the type field

---

#### A.2 Runtime Resource Resolver (Must-have)

**Problem**: No mechanism maps moniker -> cloud resource name at runtime.

**Solution**: `ResourceResolver` class that loads manifest, combines with runtime context, and resolves monikers using existing `generateDependencyResourceName()`.

```typescript
// New: lc-platform-dev-accelerators/src/resolution/ResourceResolver.ts
export class ResourceResolver {
  constructor(private readonly contextLoader: IContextLoader) {}
  async initialize(options: ApplicationContextOptions): Promise<void>;
  resolve(moniker: string): ResolvedDependency;
  resolveTyped(moniker: string, expectedType: DependencyType): ResolvedDependency;
  listMonikers(): string[];
}

// Port interface for loading app context
export interface IContextLoader {
  load(options: ApplicationContextOptions): Promise<ApplicationContext>;
}

// Implementations:
// - ManifestContextLoader (reads lcp-manifest.yaml + uses nameGenerator)
// - StorageContextLoader (reads from platform config bucket)
// - InMemoryContextLoader (for testing)
```

**Reuses**: `generateDependencyResourceName()` from `lc-platform-dev-accelerators/src/utils/nameGenerator.ts` -- this function already implements the naming convention `lcp-{account}-{team}-{moniker}-{serviceType}-{name}`.

**Acceptance Criteria**:
- Resolves all 14 `DependencyType` values
- Throws `ResourceNotFoundError` for undeclared monikers (with list of available ones)
- Injectable context loader for testing
- Works with mock provider

**Dependencies**: A.1

---

#### A.3 Scoped Client Accessors on LCAppRuntime (Must-have)

**Problem**: `LCAppRuntime` returns raw clients requiring bucket/queue names as first parameter on every call.

**Solution**: Add moniker-based accessor methods returning "scoped clients" -- thin wrappers that pre-bind the resolved resource name.

```typescript
// New scoped client example:
// lc-platform-dev-accelerators/src/core/clients/scoped/ScopedObjectClient.ts
export class ScopedObjectClient {
  constructor(private readonly client: ObjectClient, private readonly bucket: string) {}
  async get(key: string): Promise<ObjectData> { return this.client.get(this.bucket, key); }
  async put(key: string, data: Buffer | ReadableStream, metadata?: ObjectMetadata): Promise<void> {
    return this.client.put(this.bucket, key, data, metadata);
  }
  async delete(key: string): Promise<void> { return this.client.delete(this.bucket, key); }
  async list(prefix?: string, options?: ListOptions): Promise<ObjectInfo[]> {
    return this.client.list(this.bucket, prefix, options);
  }
  async exists(key: string): Promise<boolean> { return this.client.exists(this.bucket, key); }
  async getMetadata(key: string): Promise<ObjectMetadata> { return this.client.getMetadata(this.bucket, key); }
  async getSignedUrl(key: string, op: 'get'|'put', expiresIn?: number): Promise<string> {
    return this.client.getSignedUrl(this.bucket, key, op, expiresIn);
  }
  getResourceName(): string { return this.bucket; }
}
```

New methods on `LCAppRuntime` (all existing `get*Client()` methods preserved for backward compat):

```typescript
// Modified: lc-platform-dev-accelerators/src/LCAppRuntime.ts
class LCAppRuntime {
  // Existing methods unchanged
  getObjectClient(): ObjectClient { ... }
  getQueueClient(): QueueClient { ... }
  // ...

  // NEW: Moniker-based accessors (require initialize() first)
  store(moniker: string): ScopedObjectClient;
  queue(moniker: string): ScopedQueueClient;
  secrets(moniker: string): ScopedSecretsClient;
  config(moniker: string): ScopedConfigClient;
  events(moniker: string): ScopedEventPublisher;
  notifications(moniker: string): ScopedNotificationClient;
  documents(moniker: string): ScopedDocumentClient;
  data(moniker: string): ScopedDataClient;
  auth(moniker: string): ScopedAuthClient;
  cache(moniker: string): ScopedCacheClient;
  containerRepo(moniker: string): ScopedContainerRepoClient;

  // NEW: Factory methods
  static fromManifest(path: string, overrides?: Partial<RuntimeConfig>): Promise<LCAppRuntime>;
  static fromEnvironment(): Promise<LCAppRuntime>;
  async initialize(): Promise<void>;
}
```

**Acceptance Criteria**:
- All 11 data plane client types have corresponding `Scoped*Client` wrappers
- Scoped clients have identical signatures minus the resource-name parameter
- `runtime.store('nonexistent')` throws `ResourceNotFoundError` with helpful message listing available monikers
- Scoped clients are cached (same moniker returns same instance)
- Full TypeScript type inference (autocomplete works)
- Existing `get*Client()` methods remain unchanged

**Dependencies**: A.2

**Critical files to modify**:
- `lc-platform-dev-accelerators/src/LCAppRuntime.ts` (250 lines currently)
- New: `lc-platform-dev-accelerators/src/core/clients/scoped/` (11 files)

---

### Theme B: Component Integration

#### B.1 Wire CLI Through Processing-Lib (Must-have)

**Problem**: CLI commands bypass processing-lib, using mock filesystem storage.

**Solution**: Replace mock implementations with calls to processing-lib's `ApplicationConfigurator` and `VersionConfigurator`.

```typescript
// BEFORE (lc-platform-dev-cli/src/cli/commands/app/init.ts lines 57-84):
async function initializeApp(context, _config) {
  const mockApps = loadMockApps();          // reads ~/.lcp/mock-apps.json
  mockApps.add(appKey);
  saveMockApps(mockApps);                   // writes ~/.lcp/mock-apps.json
  return { id: `app-${Date.now()}`, status: 'active' };
}

// AFTER:
async function initializeApp(context, _config) {
  const storageAdapter = createStorageAdapter(context);
  const configurator = new ApplicationConfigurator(storageAdapter);
  const result = await configurator.init({
    account: context.account!, team: context.team!, moniker: context.moniker!,
  });
  if (!result.success) throw mapConfigError(result.error);
  return { id: result.value.id.toString(), status: 'active' };
}
```

**Commands to rewire**:
- `app init` -> `ApplicationConfigurator.init()`
- `app read` -> `ApplicationConfigurator.read()`
- `app update` -> `ApplicationConfigurator.update()`
- `app validate` -> `ApplicationConfigurator.validate()`
- `version add` -> `VersionConfigurator.init()`
- `version read` -> `VersionConfigurator.read()`
- `version deploy` -> `DeployDependencies` + `DeployApplication`

**Acceptance Criteria**:
- `~/.lcp/mock-apps.json` approach removed entirely
- Error messages from processing-lib mapped to human-readable CLI messages
- `--dry-run` still works (intercepts before calling configurator)

**Dependencies**: B.2

---

#### B.2 Implement Processing-Lib Adapters (Must-have)

**Problem**: All three processing-lib adapters are stubs.

**Solution**: Replace with real implementations using dev-accelerators.

```typescript
// BEFORE (lc-platform-processing-lib/src/adapters/storage/AcceleratorStorageAdapter.ts):
export class AcceleratorStorageAdapter implements IStorageProvider {
  private readonly inMemoryStorage = new Map<string, unknown>();  // stub!
}

// AFTER:
export class AcceleratorStorageAdapter implements IStorageProvider {
  constructor(
    private readonly objectClient: ObjectClient,
    private readonly configBucket: string
  ) {}
  async read<T>(path: string): Promise<Result<T, StorageError>> {
    const data = await this.objectClient.get(this.configBucket, path);
    return { success: true, value: JSON.parse(data.data.toString()) as T };
  }
  // ... write, delete, uploadArtifact using objectClient
}
```

**Files to modify**:
- `lc-platform-processing-lib/src/adapters/storage/AcceleratorStorageAdapter.ts`
- `lc-platform-processing-lib/src/adapters/policy/AcceleratorPolicyAdapter.ts` (use policySerializer from dev-accelerators)
- `lc-platform-processing-lib/src/adapters/deployment/AcceleratorDeploymentAdapter.ts` (use LCPlatform services)
- New: `AcceleratorStorageAdapterFactory.ts` -- factory creating adapters from context

**Acceptance Criteria**:
- All adapters use dev-accelerators (ObjectClient, LCPlatform services)
- All work with mock provider (no cloud credentials needed for testing)
- Integration tests verify CLI -> processing-lib -> dev-accelerators chain

**Dependencies**: None (dev-accelerators already exists)

---

#### B.3 Dependency Management CLI Commands (Must-have)

**Problem**: No CLI commands for managing individual resource dependencies by moniker.

**Solution**: New `lcp dependency` command group that edits `lcp-manifest.yaml`.

```bash
lcp dependency add backups --type object-store --encryption aes256 --versioning
lcp dependency add order-processing --type queue --fifo --visibility-timeout 30
lcp dependency list
lcp dependency show backups --json
lcp dependency remove backups
lcp dependency validate
```

**Acceptance Criteria**:
- All commands support `--json` and `--dry-run`
- Type-specific flags (e.g., `--fifo` for queues, `--encryption` for storage)
- Commands update `lcp-manifest.yaml`
- Validation delegates to processing-lib

**Dependencies**: A.1, B.1

**New files**: `lc-platform-dev-cli/src/cli/commands/dependency/` (add, remove, list, show, validate)

---

### Theme C: Configuration

#### C.1 Shared Configuration Package -- lc-platform-config (Must-have)

**Problem**: Configuration scattered across three repos with different schemas. The README references this package but it doesn't exist.

**Solution**: Create `lc-platform-config` as a new submodule providing:
- Manifest schema definition and YAML parsing with env var interpolation
- Configuration resolution with documented precedence
- Shared types used by all packages

```typescript
// Shared types
export interface LCPlatformConfig {
  account: string; team: string; moniker: string;
  provider: ProviderType; region: string; environment: string;
}
export interface ManifestData {
  apiVersion: string; kind: 'Application';
  metadata: ManifestMetadata; provider: ProviderDeclaration;
  dependencies: Record<string, DependencyDeclaration>;
}

// Resolution: CLI args > env vars > project config > global config > manifest defaults
export function resolveConfig(sources: ConfigSources): LCPlatformConfig;
```

**Acceptance Criteria**:
- Package exists as git submodule
- Shared types used by CLI, processing-lib, and dev-accelerators
- 80%+ test coverage

**Dependencies**: A.1

---

#### C.2 Environment-Aware Resource Resolution (Must-have)

**Problem**: `nameGenerator.ts` has no environment parameter. Dev/prod resources collide.

**Solution**: Add optional `environment` parameter to `generateDependencyResourceName()`.

```typescript
// Modified: lc-platform-dev-accelerators/src/utils/nameGenerator.ts
export function generateDependencyResourceName(
  account: string, team: string, moniker: string,
  dependencyType: DependencyType, dependencyName: string,
  environment?: string  // NEW
): string {
  // lcp-{account}-{team}-{moniker}-{serviceType}-{name}[-{environment}]
}
```

**Acceptance Criteria**:
- Backward compatible (omitting environment = same behavior as today)
- `lcp deploy dependencies --env staging` creates resources with environment qualifier
- `ResourceResolver` passes current environment to name generation

**Dependencies**: A.2

**File to modify**: `lc-platform-dev-accelerators/src/utils/nameGenerator.ts`

---

#### C.3 Complete DependencyConfiguration Union Type (Must-have)

**Problem**: `DependencyConfiguration` at `dependency.ts:144-148` only covers 4 of 14 types.

**Solution**: Add configuration interfaces for all remaining dependency types.

**File to modify**: `lc-platform-dev-accelerators/src/core/types/dependency.ts`

---

### Theme D: Developer Experience

#### D.1 Error Message Quality (Must-have)

**Problem**: Error messages are technical and unhelpful. CLI uses bare `process.exit(1)` with `console.error()`.

**Solution**: Error enrichment layer mapping processing-lib errors to actionable CLI messages with suggestions.

```
Error: Application 'order-svc' already exists for team 'backend'.

Suggestions:
  - To update it, run: lcp app update
  - To see its current state, run: lcp app read

Use --debug for full error details.
```

**Acceptance Criteria**:
- All CLI errors include message, suggestion, and error code
- `--debug` shows stack trace; `--json` outputs machine-parseable errors
- Every processing-lib error enum value mapped to CLI error message

**Dependencies**: B.1

---

#### D.2 SDK Code Generation (Should-have)

**Problem**: No automated way to generate typed boilerplate from manifest.

**Solution**: `lcp codegen --output src/platform.ts` generates typed SDK module.

```typescript
// Auto-generated from lcp-manifest.yaml
import { LCAppRuntime } from '@stainedhead/lc-platform-dev-accelerators';
const runtime = LCAppRuntime.fromManifest('./lcp-manifest.yaml');
/** Object storage for backup files */
export const backups = runtime.store('backups');
/** FIFO queue for order processing */
export const orderProcessing = runtime.queue('order-processing');
```

**Dependencies**: A.1, A.3

---

#### D.3 Interactive Dependency Setup (Should-have)

**Problem**: Adding dependencies requires memorizing flags.

**Solution**: `lcp dependency add order-queue --type queue --interactive` with guided prompts.

**Dependencies**: B.3

---

#### D.4 Active Application Context (Must-have)

**Problem**: Developers must repeat `--account`, `--team`, and `--moniker` on every command, or manually maintain config files. There is no concept of a "currently active application" that persists across commands. This is tedious for the common case of working on a single app for an extended period.

**Solution**: Introduce an active application context that is automatically set and can be explicitly switched.

**Behaviors**:
1. **Auto-select on init**: `lcp app init --team backend --moniker order-svc` automatically sets `backend/order-svc` as the active app in the project-local config (`.lcp/config.json`).
2. **Explicit switch**: `lcp app use <moniker>` or `lcp app use <team>/<moniker>` sets the active app. Moniker-only form works when unambiguous; team-qualified form resolves ambiguity.
3. **List and show current**: `lcp app list` shows all known apps with the active one highlighted. `lcp app current` (or indicator in `lcp context read`) shows which app is active.
4. **One-off override**: `lcp -a <moniker> <command>` runs a single command against a different app without changing the active context. The `-a` / `--app` global flag overrides the active context for that invocation only.
5. **All commands inherit**: Every command that needs account/team/moniker reads from the active app context when those flags are not explicitly provided.

```bash
# Init sets active context automatically
lcp app init --team backend --moniker order-svc
# Active app: backend/order-svc (stored in .lcp/config.json)

# Subsequent commands just work -- no flags needed
lcp dependency add backups --type object-store
lcp dependency list
lcp version add --ver v1.0.0 --config version.json

# Switch to a different app
lcp app use user-svc

# One-off command without switching
lcp -a order-svc dependency list
```

**Acceptance Criteria**:
- `lcp app init` automatically sets the new app as active context
- `lcp app use <moniker>` switches active app; fails with helpful message if moniker is ambiguous
- Active app stored in project-local `.lcp/config.json` (not global, so different projects can have different active apps)
- `-a <moniker>` global flag overrides active context for a single command without persisting
- `lcp context read` shows the active app
- All existing commands work without modification when active context is set
- Clear error message when no active app is set and no flags provided: `"No active application. Run 'lcp app use <moniker>' or provide --account, --team, --moniker."`

**Dependencies**: B.1

**Files to modify**:
- `lc-platform-dev-cli/src/config/types.ts` -- Add active app fields to `CliContext`
- `lc-platform-dev-cli/src/cli/options.ts` -- Add `-a` global flag, resolve active app in context
- `lc-platform-dev-cli/src/cli/commands/app/index.ts` -- Add `use`, `list`, `current` subcommands
- `lc-platform-dev-cli/src/cli/commands/app/init.ts` -- Auto-set active app on init

---

### Theme E: Agentic Development Support

#### E.1 Manifest-Driven Agent Context Export (Must-have)

**Problem**: AI agents have no structured way to understand application infrastructure.

**Solution**: `lcp context export --json` produces machine-readable topology with access patterns.

**Acceptance Criteria**:
- Export includes TypeScript access pattern for each dependency
- Export includes code examples per dependency type
- Dependency names are business-meaningful (enforced by naming validation)

**Dependencies**: A.1, A.2

---

#### E.2 SpecKit Resource Awareness (Should-have)

**Problem**: SpecKit commands don't know about declared dependencies.

**Solution**: SpecKit commands detect and load `lcp-manifest.yaml`, making dependency info available during implementation.

**Dependencies**: A.1, E.1

---

### Theme F: Enterprise Accessibility

#### F.1 Domain-Driven Naming Validation (Should-have)

**Problem**: Nothing guides users toward business-domain dependency names.

**Solution**: Warn when dependency names contain cloud terms (`s3`, `sqs`, `bucket`, `blob`, etc.).

```
Warning: Dependency name 's3-documents' contains cloud-specific term 's3'.
Consider using a business-domain name like 'customer-documents'.
```

---

#### F.2 Getting-Started Documentation (Must-have)

**Problem**: Current docs assume familiarity with hexagonal architecture and TypeScript.

**Solution**: Progressive tutorials in `documentation/getting-started/`:
1. `01-your-first-app.md` -- Zero to working app in 15 minutes
2. `02-adding-dependencies.md` -- Declaring storage, queues, databases
3. `03-writing-application-code.md` -- Using moniker-based APIs
4. `04-deploying-to-cloud.md` -- From mock to real cloud
5. `05-working-with-ai-agents.md` -- Claude/Copilot with LC Platform

**Acceptance Criteria**:
- Tutorial 01 completable in under 15 minutes
- All use moniker-based APIs (not raw bucket names)
- No cloud credentials needed for tutorials 01-03 (mock provider)

**Dependencies**: A.3, B.3

---

## 5. Requirements Summary

| ID | Feature | Priority | Theme | Depends On |
|----|---------|----------|-------|------------|
| A.1 | Application Manifest File | Must-have | Abstraction | -- |
| A.2 | Runtime Resource Resolver | Must-have | Abstraction | A.1 |
| A.3 | Scoped Client Accessors | Must-have | Abstraction | A.2 |
| B.1 | Wire CLI Through Processing-Lib | Must-have | Integration | B.2 |
| B.2 | Implement Processing-Lib Adapters | Must-have | Integration | -- |
| B.3 | Dependency Management CLI Commands | Must-have | Integration | A.1, B.1 |
| C.1 | lc-platform-config Package | Must-have | Configuration | A.1 |
| C.2 | Environment-Aware Resolution | Must-have | Configuration | A.2 |
| C.3 | Complete DependencyConfiguration Types | Must-have | Configuration | -- |
| D.1 | Error Message Quality | Must-have | DX | B.1 |
| D.2 | SDK Code Generation | Should-have | DX | A.1, A.3 |
| D.3 | Interactive Dependency Setup | Should-have | DX | B.3 |
| D.4 | Active Application Context | Must-have | DX | B.1 |
| E.1 | Agent Context Export | Must-have | Agentic | A.1, A.2 |
| E.2 | SpecKit Resource Awareness | Should-have | Agentic | A.1, E.1 |
| F.1 | Domain-Driven Naming Validation | Should-have | Enterprise | B.3 |
| F.2 | Getting-Started Documentation | Must-have | Enterprise | A.3, B.3 |

**Must-have count**: 12 | **Should-have count**: 5

---

## 6. Architecture Changes

### New Package: lc-platform-config
```
src/
  manifest/schema.ts, parser.ts, interpolation.ts
  resolution/resolver.ts, types.ts
  validation/naming.ts, dependency.ts
  index.ts
```

### Changes to lc-platform-dev-accelerators

**New files (19)**:
- `src/resolution/ResourceResolver.ts` -- Core resolver
- `src/resolution/IContextLoader.ts` -- Port interface
- `src/resolution/ManifestContextLoader.ts` -- Loads from YAML manifest
- `src/resolution/StorageContextLoader.ts` -- Loads from platform config bucket
- `src/resolution/InMemoryContextLoader.ts` -- For testing
- `src/core/types/resolution.ts` -- `ApplicationContext`, `ResolvedDependency`
- `src/core/clients/scoped/ScopedObjectClient.ts` -- 1 of 11 scoped clients
- `src/core/clients/scoped/ScopedQueueClient.ts`
- `src/core/clients/scoped/ScopedSecretsClient.ts`
- `src/core/clients/scoped/ScopedConfigClient.ts`
- `src/core/clients/scoped/ScopedEventPublisher.ts`
- `src/core/clients/scoped/ScopedNotificationClient.ts`
- `src/core/clients/scoped/ScopedDocumentClient.ts`
- `src/core/clients/scoped/ScopedDataClient.ts`
- `src/core/clients/scoped/ScopedAuthClient.ts`
- `src/core/clients/scoped/ScopedCacheClient.ts`
- `src/core/clients/scoped/ScopedContainerRepoClient.ts`
- `src/core/clients/scoped/index.ts` -- Barrel export

**Modified files (4)**:
- `src/LCAppRuntime.ts` -- Add moniker-based accessors, `fromManifest()`, `fromEnvironment()`, `initialize()`
- `src/utils/nameGenerator.ts` -- Add optional `environment` parameter
- `src/core/types/dependency.ts` -- Complete the `DependencyConfiguration` union
- `src/index.ts` -- Export new types and classes

### Changes to lc-platform-processing-lib

**Modified files (3)**:
- `src/adapters/storage/AcceleratorStorageAdapter.ts` -- Real implementation using ObjectClient
- `src/adapters/policy/AcceleratorPolicyAdapter.ts` -- Wire to policySerializer
- `src/adapters/deployment/AcceleratorDeploymentAdapter.ts` -- Wire to LCPlatform services

**New files (1)**:
- `src/adapters/storage/AcceleratorStorageAdapterFactory.ts` -- Factory from context

### Changes to lc-platform-dev-cli

**New command groups (2)**:
- `src/cli/commands/dependency/` -- add, remove, list, show, validate
- `src/cli/commands/codegen/` -- Code generation command

**Modified files (10)**:
- `src/cli/commands/app/init.ts` -- Replace mock with processing-lib; auto-set active app
- `src/cli/commands/app/read.ts` -- Replace mock with processing-lib
- `src/cli/commands/app/update.ts` -- Replace mock with processing-lib
- `src/cli/commands/app/validate.ts` -- Replace mock with processing-lib
- `src/cli/commands/app/index.ts` -- Add `use`, `list`, `current` subcommands
- `src/cli/commands/version/add.ts` -- Replace mock with processing-lib
- `src/cli/commands/version/read.ts` -- Replace mock with processing-lib
- `src/cli/commands/version/deploy.ts` -- Replace mock with processing-lib
- `src/cli/commands/context/read.ts` -- Add `--export` flag for agent context; show active app
- `src/cli/options.ts` -- Add `-a` / `--app` global flag for one-off app override

**New files (1)**:
- `src/cli/shared/adapter-factory.ts` -- Creates processing-lib adapters from CLI context

### Updated Dependency Graph
```
lc-platform-config (NEW)
  Used by: dev-accelerators, processing-lib, dev-cli

lc-platform-dev-accelerators (ENHANCED: scoped clients, resolver)
  Used by: processing-lib adapters

lc-platform-processing-lib (ENHANCED: real adapters)
  Used by: dev-cli commands

lc-platform-dev-cli (ENHANCED: integrated commands, dependency mgmt)
```

---

## 7. Implementation Phases

### Phase 1: Foundation (Weeks 1-3)
1. Create `lc-platform-config` package with manifest schema, YAML parsing, shared types
2. Define `lcp-manifest.yaml` schema and validation
3. Complete `DependencyConfiguration` union type (C.3)
4. Add environment parameter to `nameGenerator.ts` (C.2)
5. Create `ResourceResolver` + `IContextLoader` + `InMemoryContextLoader` (A.2)

### Phase 2: Scoped Clients & Resolution (Weeks 4-6)
6. Create all 11 `Scoped*Client` wrappers (A.3)
7. Add moniker-based accessors to `LCAppRuntime` (A.3)
8. Add `fromManifest()` and `fromEnvironment()` static factories (A.3)
9. Create `ManifestContextLoader` (reads YAML, uses nameGenerator)
10. Full test coverage for resolution layer and scoped clients

### Phase 3: Adapter Integration (Weeks 7-8)
11. Implement real `AcceleratorStorageAdapter` using dev-accelerators ObjectClient (B.2)
12. Implement real `AcceleratorPolicyAdapter` (B.2)
13. Implement real `AcceleratorDeploymentAdapter` (B.2)
14. Create `AcceleratorStorageAdapterFactory` (B.2)
15. Integration tests: processing-lib -> dev-accelerators with mock provider

### Phase 4: CLI Integration (Weeks 9-11)
16. Create `adapter-factory.ts` in CLI shared utilities (B.1)
17. Rewire all app/ and version/ commands through processing-lib (B.1)
18. Remove mock filesystem storage (B.1)
19. Implement `lcp dependency` command group (B.3)
20. Implement error enrichment layer (D.1)
21. Add `lcp context export --json` (E.1)
22. Implement active application context: `lcp app use`, `-a` flag, auto-select on init (D.4)

### Phase 5: Polish & Documentation (Weeks 12-14)
23. Getting-started tutorials (F.2)
24. `lcp codegen` command (D.2, if time)
25. Interactive dependency setup (D.3, if time)
26. Domain-driven naming validation (F.1, if time)
27. SpecKit resource awareness (E.2, if time)
28. Update parent README to reflect actual package names and structure

---

## 8. Release Criteria (v1.0.0)

### Must pass (all required):
- [ ] `lcp-manifest.yaml` is the declarative source of truth for application topology
- [ ] `runtime.store('moniker')` resolves and works end-to-end (mock + AWS)
- [ ] CLI commands delegate through processing-lib (zero mock filesystem usage)
- [ ] `lcp dependency add/list/remove/show/validate` fully functional
- [ ] `lc-platform-config` package provides shared types and manifest handling
- [ ] Environment-aware resource names work
- [ ] Error messages include suggestions and error codes
- [ ] `lcp context export --json` provides machine-readable topology
- [ ] Getting-started docs enable new user app in 15 minutes
- [ ] 80%+ test coverage across all packages
- [ ] Mock provider works for complete local dev (no cloud credentials)
- [ ] All existing tests pass (backward compatibility)

### Out of scope for v1.0.0:
- Azure/GCP provider implementations
- REST API (`lc-platform-rest-api`)
- Multi-tenant support
- GUI / web dashboard

---

## 9. Risks and Mitigations

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Breaking changes to dev-accelerators API | High | Low | All existing methods preserved; new methods are additive only |
| Manifest file too complex for enterprise users | Medium | Medium | Sensible defaults; `lcp dependency add` edits manifest programmatically |
| Interface mismatches between processing-lib ports and dev-accelerators | High | Medium | Implement storage adapter first as proof; ports can be adjusted (pre-1.0) |
| Circular dependencies between packages | High | Low | lc-platform-config is types/validation only, zero runtime deps on other packages |
| Env var interpolation leaking secrets in errors | Medium | Low | Log variable names never values; secrets use `secrets` dependency type |

---

## 10. Verification Plan

### Unit Testing
- ResourceResolver with InMemoryContextLoader
- All 11 ScopedClient wrappers
- Manifest YAML parsing + validation
- nameGenerator with environment parameter
- Error enrichment mappings

### Integration Testing
- CLI -> processing-lib -> dev-accelerators (full chain with mock provider)
- `lcp app init` -> `ApplicationConfigurator.init()` -> `AcceleratorStorageAdapter` -> MockObjectClient
- `runtime.store('backups').put(...)` -> ScopedObjectClient -> MockObjectClient

### E2E Testing
- Full workflow: `lcp app init` -> add dependencies -> deploy -> write app code -> run with mock
- Manifest-driven: create manifest -> `LCAppRuntime.fromManifest()` -> use scoped clients
- Agent workflow: `lcp context export --json` -> verify all access patterns correct

### Manual Verification
- New developer follows getting-started tutorial 01 in under 15 minutes
- AI agent (Claude) generates correct code from `lcp context export` output
