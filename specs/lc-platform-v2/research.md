# LC Platform v2 - Research

**Created:** 2026-02-09
**Source:** `lc-platform-v2-prd.md`
**Status:** In Progress

---

## Overview

This document captures research findings for transforming LC Platform from proof-of-concept to feature-complete v1.0.0 release. The primary focus is understanding existing implementations, API patterns, and integration points between the three packages (dev-accelerators, processing-lib, CLI).

**Research Questions:**
1. How does `nameGenerator.ts` currently generate resource names, and what changes are needed for environment support?
2. What is the current structure of `DependencyConfiguration` union type and what are the missing types?
3. How do existing CLI commands use mock filesystem storage, and what needs to be replaced?
4. What is the interface contract of processing-lib adapters (IStorageProvider, IPolicyProvider, IDeploymentProvider)?
5. How do dev-accelerators clients (ObjectClient, QueueClient, etc.) expect resource names as parameters?
6. What is the current CLI configuration precedence system (global, project-local, flags)?
7. How does `LCAppRuntime` currently initialize and what factory methods exist?
8. What YAML parsing libraries are used in existing packages (if any)?

---

## Existing Implementation Analysis

### nameGenerator.ts (dev-accelerators)

**Current Signature:**
```typescript
export function generateDependencyResourceName(
  account: string,
  team: string,
  moniker: string,
  dependencyType: DependencyType,
  dependencyName: string
): string
```

**Current Naming Pattern:**
```
lcp-{account}-{team}-{moniker}-{serviceType}-{name}
```

**Findings:**
- No environment parameter currently
- Returns deterministic names from 5 inputs
- Used by `LCPlatformApp` for dependency registration
- Need to add optional `environment` parameter: `[-{environment}]` suffix

**Action Items:**
- [ ] Add optional `environment?: string` parameter (backward compatible)
- [ ] Update naming pattern to: `lcp-{account}-{team}-{moniker}-{serviceType}-{name}[-{environment}]`
- [ ] Update tests to cover with and without environment

---

### DependencyConfiguration Union Type (dev-accelerators)

**File:** `lc-platform-dev-accelerators/src/core/types/dependency.ts`

**Current Implementation (Lines 144-148):**
```typescript
export type DependencyConfiguration =
  | ObjectStoreConfiguration
  | QueueConfiguration
  | SecretsConfiguration
  | DataStoreConfiguration;
```

**Missing Types (10 of 14):**
- DocumentStoreConfiguration
- EventBusConfiguration
- NotificationConfiguration
- CacheConfiguration
- WebHostingConfiguration
- AuthConfiguration
- FileTransferConfiguration
- WorkflowConfiguration
- SearchConfiguration
- LoggingConfiguration

**DependencyType Enum:**
Need to verify all 14 types in the DependencyType enum and create corresponding configuration interfaces.

**Action Items:**
- [ ] Review DependencyType enum for complete list
- [ ] Create configuration interface for each missing type
- [ ] Define type-specific options for each (e.g., `fifo` for queues, `versioning` for storage)
- [ ] Document configuration fields in data-dictionary.md

---

### CLI Mock Filesystem Usage

**File:** `lc-platform-dev-cli/src/cli/commands/app/init.ts`

**Current Implementation (Lines 57-84):**
```typescript
async function initializeApp(context, _config) {
  const mockApps = loadMockApps();          // reads ~/.lcp/mock-apps.json
  mockApps.add(appKey);
  saveMockApps(mockApps);                   // writes ~/.lcp/mock-apps.json
  return { id: `app-${Date.now()}`, status: 'active' };
}
```

**Mock Storage Functions:**
- `loadMockApps()` - Reads from `~/.lcp/mock-apps.json`
- `saveMockApps()` - Writes to `~/.lcp/mock-apps.json`

**Commands Using Mock Storage:**
- `app init` → `loadMockApps()` / `saveMockApps()`
- `app read` → `loadMockApps()`
- `app update` → `loadMockApps()` / `saveMockApps()`
- `version add` → `loadMockApps()` / `saveMockApps()`
- `version read` → `loadMockApps()`

**Action Items:**
- [ ] Map each CLI command to corresponding processing-lib configurator method
- [ ] Create adapter factory for processing-lib adapters from CLI context
- [ ] Remove `loadMockApps()` and `saveMockApps()` functions
- [ ] Delete `~/.lcp/mock-apps.json` references

---

### Processing-Lib Adapter Interfaces

**IStorageProvider Interface:**
```typescript
// lc-platform-processing-lib/src/ports/storage/IStorageProvider.ts
interface IStorageProvider {
  read<T>(path: string): Promise<Result<T, StorageError>>;
  write<T>(path: string, data: T): Promise<Result<void, StorageError>>;
  delete(path: string): Promise<Result<void, StorageError>>;
  uploadArtifact(localPath: string, remotePath: string): Promise<Result<void, StorageError>>;
}
```

**Current AcceleratorStorageAdapter (Stub):**
```typescript
export class AcceleratorStorageAdapter implements IStorageProvider {
  private readonly inMemoryStorage = new Map<string, unknown>();  // stub!
}
```

**Required Implementation:**
- Use `ObjectClient` from dev-accelerators
- Read/write JSON to platform config bucket
- `uploadArtifact()` for application bundles

**Action Items:**
- [ ] Identify platform config bucket naming convention
- [ ] Implement read/write using `ObjectClient.get()` / `ObjectClient.put()`
- [ ] Handle JSON serialization/deserialization
- [ ] Error mapping: ObjectClient errors → StorageError

---

### Dev-Accelerators Client Interfaces

**ObjectClient Interface:**
```typescript
interface ObjectClient {
  get(bucket: string, key: string): Promise<ObjectData>;
  put(bucket: string, key: string, data: Buffer | ReadableStream, metadata?: ObjectMetadata): Promise<void>;
  delete(bucket: string, key: string): Promise<void>;
  list(bucket: string, prefix?: string, options?: ListOptions): Promise<ObjectInfo[]>;
  exists(bucket: string, key: string): Promise<boolean>;
  getMetadata(bucket: string, key: string): Promise<ObjectMetadata>;
  getSignedUrl(bucket: string, key: string, op: 'get'|'put', expiresIn?: number): Promise<string>;
}
```

**Pattern:** First parameter is always resource name (bucket, queue URL, etc.)

**Scoped Client Wrapper Pattern:**
```typescript
class ScopedObjectClient {
  constructor(private client: ObjectClient, private bucket: string) {}
  async get(key: string) { return this.client.get(this.bucket, key); }
  async put(key: string, data: Buffer, metadata?) { return this.client.put(this.bucket, key, data, metadata); }
  // ... etc, removing bucket parameter
}
```

**Action Items:**
- [ ] Create scoped wrapper for all 11 data plane clients
- [ ] Ensure method signatures match (minus resource name parameter)
- [ ] Add `getResourceName(): string` helper to each scoped client

---

### CLI Configuration System

**Config Precedence (Current):**
1. CLI flags (highest)
2. Project-local config (`.lcp/config.json` in project root)
3. Global config (`~/.lcp/config.json`)
4. Defaults (lowest)

**CliContext Structure:**
```typescript
interface CliContext {
  account?: string;
  team?: string;
  moniker?: string;
  provider?: ProviderType;
  region?: string;
  // ... other fields
}
```

**Action Items:**
- [ ] Add `activeApp?: { team: string; moniker: string }` to CliContext
- [ ] Add `-a` / `--app` global flag for one-off overrides
- [ ] Implement `lcp app use <moniker>` to set active app in `.lcp/config.json`
- [ ] Update context resolution to read active app from project-local config

---

### LCAppRuntime Current Implementation

**Current Factory/Initialization:**
```typescript
// Current: Manual construction with provider factory
const runtime = new LCAppRuntime(providerFactory);
```

**Needed Factory Methods:**
```typescript
// NEW: Load from manifest
static async fromManifest(path: string, overrides?: Partial<RuntimeConfig>): Promise<LCAppRuntime>

// NEW: Load from environment variables
static async fromEnvironment(): Promise<LCAppRuntime>

// NEW: Initialize with resolver
async initialize(): Promise<void>
```

**Action Items:**
- [ ] Add ResourceResolver as optional dependency
- [ ] Implement fromManifest() factory (loads YAML, creates resolver)
- [ ] Implement fromEnvironment() factory (reads env vars)
- [ ] Add initialize() method to set up resolver
- [ ] Cache scoped clients by moniker

---

## Industry Standards

### YAML Manifest Patterns

**Kubernetes-style Manifests:**
```yaml
apiVersion: v1
kind: Application
metadata:
  name: example
  namespace: default
spec:
  # ... specification
```

**Our Pattern (Simplified):**
```yaml
apiVersion: lcp/v1
kind: Application
metadata:
  team: backend
  moniker: order-svc
dependencies:
  # ... dependencies
```

**YAML Libraries (TypeScript):**
- `js-yaml` - Popular, well-maintained
- `yaml` (npm package) - Modern, full spec support

**Action Items:**
- [ ] Choose YAML library for lc-platform-config
- [ ] Define schema validation approach (JSON Schema? Zod? Custom?)
- [ ] Implement environment variable interpolation (`${VAR:-default}`)

---

## Existing Implementations

### Similar Runtime Patterns

**AWS Amplify:**
```typescript
// Configure once
Amplify.configure(config);

// Use without re-specifying config
const result = await Storage.put('key', data);
```

**Firebase:**
```typescript
const app = initializeApp(config);
const storage = getStorage(app);
const ref = ref(storage, 'path/to/file');
```

**Our Pattern:**
```typescript
const runtime = await LCAppRuntime.fromManifest('./lcp-manifest.yaml');
const backups = runtime.store('backups');
await backups.put('file.txt', data);
```

**Similarities:**
- One-time initialization
- Context passed implicitly
- Resource accessors return scoped clients

---

## API Documentation

### Manifest Schema (To Be Created)

**Top-Level Fields:**
```yaml
apiVersion: string        # "lcp/v1"
kind: string             # "Application"
metadata: object         # Application metadata
provider: object         # Provider configuration
dependencies: object     # Map of moniker -> dependency declaration
```

**Metadata Fields:**
```yaml
metadata:
  team: string          # Required
  moniker: string       # Required
  environment: string   # Optional, supports ${ENV:-default}
  owner: string         # Optional
  description: string   # Optional
```

**Provider Fields:**
```yaml
provider:
  type: string          # "aws" | "azure" | "gcp" | "mock"
  region: string        # Cloud region
  account: string       # Optional, can use ${ACCOUNT} interpolation
```

**Dependency Declaration:**
```yaml
dependencies:
  <moniker>:
    type: DependencyType
    config: <type-specific configuration>
```

---

## Code Examples

### Example: Scoped Client Wrapper

```typescript
// lc-platform-dev-accelerators/src/core/clients/scoped/ScopedObjectClient.ts
export class ScopedObjectClient {
  constructor(
    private readonly client: ObjectClient,
    private readonly bucket: string
  ) {}

  async get(key: string): Promise<ObjectData> {
    return this.client.get(this.bucket, key);
  }

  async put(key: string, data: Buffer | ReadableStream, metadata?: ObjectMetadata): Promise<void> {
    return this.client.put(this.bucket, key, data, metadata);
  }

  async delete(key: string): Promise<void> {
    return this.client.delete(this.bucket, key);
  }

  async list(prefix?: string, options?: ListOptions): Promise<ObjectInfo[]> {
    return this.client.list(this.bucket, prefix, options);
  }

  async exists(key: string): Promise<boolean> {
    return this.client.exists(this.bucket, key);
  }

  async getMetadata(key: string): Promise<ObjectMetadata> {
    return this.client.getMetadata(this.bucket, key);
  }

  async getSignedUrl(key: string, op: 'get'|'put', expiresIn?: number): Promise<string> {
    return this.client.getSignedUrl(this.bucket, key, op, expiresIn);
  }

  getResourceName(): string {
    return this.bucket;
  }
}
```

---

### Example: ResourceResolver Usage

```typescript
// lc-platform-dev-accelerators/src/resolution/ResourceResolver.ts
export class ResourceResolver {
  constructor(private readonly contextLoader: IContextLoader) {}

  async initialize(options: ApplicationContextOptions): Promise<void> {
    this.context = await this.contextLoader.load(options);
    // context includes: account, team, moniker, environment, dependencies
  }

  resolve(moniker: string): ResolvedDependency {
    const dep = this.context.dependencies[moniker];
    if (!dep) {
      throw new ResourceNotFoundError(
        `Dependency '${moniker}' not found. Available: ${Object.keys(this.context.dependencies).join(', ')}`
      );
    }

    const resourceName = generateDependencyResourceName(
      this.context.account,
      this.context.team,
      this.context.moniker,
      dep.type,
      moniker,
      this.context.environment
    );

    return {
      moniker,
      type: dep.type,
      resourceName,
      config: dep.config
    };
  }

  resolveTyped(moniker: string, expectedType: DependencyType): ResolvedDependency {
    const resolved = this.resolve(moniker);
    if (resolved.type !== expectedType) {
      throw new TypeError(
        `Dependency '${moniker}' is type '${resolved.type}', expected '${expectedType}'`
      );
    }
    return resolved;
  }

  listMonikers(): string[] {
    return Object.keys(this.context.dependencies);
  }
}
```

---

### Example: AcceleratorStorageAdapter (Real Implementation)

```typescript
// lc-platform-processing-lib/src/adapters/storage/AcceleratorStorageAdapter.ts
export class AcceleratorStorageAdapter implements IStorageProvider {
  constructor(
    private readonly objectClient: ObjectClient,
    private readonly configBucket: string
  ) {}

  async read<T>(path: string): Promise<Result<T, StorageError>> {
    try {
      const data = await this.objectClient.get(this.configBucket, path);
      const parsed = JSON.parse(data.data.toString()) as T;
      return { success: true, value: parsed };
    } catch (error) {
      return {
        success: false,
        error: {
          code: 'STORAGE_READ_FAILED',
          message: `Failed to read ${path}: ${error.message}`
        }
      };
    }
  }

  async write<T>(path: string, data: T): Promise<Result<void, StorageError>> {
    try {
      const json = JSON.stringify(data, null, 2);
      await this.objectClient.put(this.configBucket, path, Buffer.from(json));
      return { success: true, value: undefined };
    } catch (error) {
      return {
        success: false,
        error: {
          code: 'STORAGE_WRITE_FAILED',
          message: `Failed to write ${path}: ${error.message}`
        }
      };
    }
  }

  async delete(path: string): Promise<Result<void, StorageError>> {
    try {
      await this.objectClient.delete(this.configBucket, path);
      return { success: true, value: undefined };
    } catch (error) {
      return {
        success: false,
        error: {
          code: 'STORAGE_DELETE_FAILED',
          message: `Failed to delete ${path}: ${error.message}`
        }
      };
    }
  }

  async uploadArtifact(localPath: string, remotePath: string): Promise<Result<void, StorageError>> {
    try {
      const fs = await import('fs');
      const data = await fs.promises.readFile(localPath);
      await this.objectClient.put(this.configBucket, remotePath, data);
      return { success: true, value: undefined };
    } catch (error) {
      return {
        success: false,
        error: {
          code: 'ARTIFACT_UPLOAD_FAILED',
          message: `Failed to upload ${localPath} to ${remotePath}: ${error.message}`
        }
      };
    }
  }
}
```

---

## Best Practices

### Manifest Design Principles

1. **Declarative Over Imperative**: Describe desired state, not operations
2. **Environment Agnostic**: Use interpolation for environment-specific values
3. **Sensible Defaults**: Minimize required configuration
4. **Validation Early**: Fail fast with clear error messages
5. **Version Compatibility**: Support schema evolution with `apiVersion`

### TypeScript API Design

1. **Type Safety**: Full TypeScript types for all APIs
2. **Error Handling**: Result<T, E> pattern for expected failures
3. **Immutability**: Prefer readonly and const
4. **Dependency Injection**: Use interfaces for testability
5. **Factory Methods**: Provide convenient initialization patterns

### CLI UX Best Practices

1. **Progressive Disclosure**: Simple commands for common tasks, flags for advanced
2. **Helpful Errors**: Include suggestions and error codes
3. **Consistent Flags**: Global flags (`-a`, `--json`, `--dry-run`) work everywhere
4. **Context Awareness**: Reduce repetition with active application context
5. **Machine-Readable**: Support `--json` for scripting/agents

---

## Open Questions

### Question 1: Platform Config Bucket Naming

**Context:** AcceleratorStorageAdapter needs to know which bucket to use for platform configuration storage.

**Research Needed:**
- How is the config bucket name determined?
- Is it per-account? Per-team? Per-application?
- Does it use the same naming convention as generateDependencyResourceName()?

**Decision:** TBD

---

### Question 2: Manifest Schema Validation Approach

**Context:** Need to validate lcp-manifest.yaml against schema with helpful errors.

**Research Needed:**
- Use JSON Schema with YAML?
- Use TypeScript schema validation library (Zod, Yup, io-ts)?
- Custom validation?

**Decision:** TBD (likely JSON Schema for standardization)

---

### Question 3: Environment Variable Interpolation Security

**Context:** Manifest supports `${ENV_VAR:-default}` but we don't want to leak sensitive values.

**Research Needed:**
- Should we restrict which env vars can be interpolated?
- How to prevent secrets in manifest (force use of `secrets` dependency type)?
- Should we validate variable names (e.g., warn if `PASSWORD` appears)?

**Decision:** TBD

---

### Question 4: Migration Path for Existing CLI Users

**Context:** Current CLI users may have projects/workflows without manifest files.

**Research Needed:**
- Should `lcp app init` auto-detect existing project structure?
- Can we generate manifest from existing `.lcp/config.json`?
- How to communicate migration path?

**Decision:** TBD (document in getting-started)

---

## References

- Source PRD: `lc-platform-v2-prd.md`
- Dev-accelerators source: `lc-platform-dev-accelerators/src/`
- Processing-lib source: `lc-platform-processing-lib/src/`
- CLI source: `lc-platform-dev-cli/src/`
