# LC Platform v2 - Data Dictionary

**Created:** 2026-02-09
**Version:** 1.0
**Status:** Draft

---

## Overview

This document defines all data structures, types, interfaces, and constants for the LC Platform v2 implementation. Organized by package and architectural layer.

**Purpose:**
- Single source of truth for all data types across four packages
- Ensure consistency across lc-platform-config, dev-accelerators, processing-lib, and CLI
- Document validation rules and constraints
- Specify manifest schema and configuration types

---

## Table of Contents

1. [lc-platform-config Package](#lc-platform-config-package)
2. [lc-platform-dev-accelerators Package](#lc-platform-dev-accelerators-package)
3. [lc-platform-processing-lib Package](#lc-platform-processing-lib-package)
4. [lc-platform-dev-cli Package](#lc-platform-dev-cli-package)
5. [Manifest Schema](#manifest-schema)
6. [Type Aliases & Enums](#type-aliases--enums)
7. [API Types](#api-types)

---

## lc-platform-config Package

Location: `lc-platform-config/src/`

### 1. ManifestData (Manifest Schema)

**File:** `manifest/schema.ts`

```typescript
// ManifestData represents the complete lcp-manifest.yaml structure
export interface ManifestData {
  apiVersion: string;         // "lcp/v1"
  kind: 'Application';
  metadata: ManifestMetadata;
  provider: ProviderDeclaration;
  dependencies: Record<string, DependencyDeclaration>;
}
```

**Validation Rules:**
- `apiVersion` must be "lcp/v1" (only supported version)
- `kind` must be "Application"
- `metadata` is required
- `provider` is required
- `dependencies` can be empty object but not undefined

**Example:**
```yaml
apiVersion: lcp/v1
kind: Application
metadata:
  team: backend
  moniker: order-svc
provider:
  type: mock
dependencies:
  backups:
    type: object-store
```

---

### 2. ManifestMetadata

**File:** `manifest/schema.ts`

```typescript
export interface ManifestMetadata {
  team: string;               // Required: team name
  moniker: string;            // Required: application moniker
  environment?: string;       // Optional: dev, staging, prod, etc.
  owner?: string;             // Optional: email or identifier
  description?: string;       // Optional: human-readable description
}
```

**Validation Rules:**
- `team` required, kebab-case, 1-64 chars
- `moniker` required, kebab-case, 1-64 chars
- `environment` optional, alphanumeric + hyphen, 1-32 chars
- `owner` optional, email format or identifier
- `description` optional, max 512 chars

---

### 3. ProviderDeclaration

**File:** `manifest/schema.ts`

```typescript
export interface ProviderDeclaration {
  type: ProviderType;         // "aws" | "azure" | "gcp" | "mock"
  region?: string;            // Cloud region (optional for mock)
  account?: string;           // Account ID (optional, can use env vars)
}

export type ProviderType = 'aws' | 'azure' | 'gcp' | 'mock';
```

**Validation Rules:**
- `type` required, must be valid ProviderType
- `region` optional, format depends on provider
- `account` optional, supports `${ACCOUNT}` interpolation

---

### 4. DependencyDeclaration

**File:** `manifest/schema.ts`

```typescript
export interface DependencyDeclaration {
  type: DependencyType;
  config?: DependencyConfigMap;
}

// Maps to one of 14 configuration types
export type DependencyConfigMap =
  | ObjectStoreConfig
  | QueueConfig
  | SecretsConfig
  | DataStoreConfig
  | DocumentStoreConfig
  | EventBusConfig
  | NotificationConfig
  | CacheConfig
  | WebHostingConfig
  | AuthConfig
  | FileTransferConfig
  | WorkflowConfig
  | SearchConfig
  | LoggingConfig;
```

---

### 5. LCPlatformConfig (Resolved Configuration)

**File:** `resolution/types.ts`

```typescript
export interface LCPlatformConfig {
  account: string;
  team: string;
  moniker: string;
  provider: ProviderType;
  region: string;
  environment: string;
}
```

**Resolution Precedence:**
1. CLI args (highest)
2. Environment variables
3. Project config (.lcp/config.json)
4. Global config (~/.lcp/config.json)
5. Manifest defaults
6. System defaults (lowest)

---

## lc-platform-dev-accelerators Package

Location: `lc-platform-dev-accelerators/src/`

### 1. ResourceResolver

**File:** `resolution/ResourceResolver.ts`

```typescript
export class ResourceResolver {
  constructor(private readonly contextLoader: IContextLoader) {}
  async initialize(options: ApplicationContextOptions): Promise<void>;
  resolve(moniker: string): ResolvedDependency;
  resolveTyped(moniker: string, expectedType: DependencyType): ResolvedDependency;
  listMonikers(): string[];
}
```

---

### 2. IContextLoader (Port Interface)

**File:** `resolution/IContextLoader.ts`

```typescript
export interface IContextLoader {
  load(options: ApplicationContextOptions): Promise<ApplicationContext>;
}
```

**Implementations:**
- `ManifestContextLoader` - Reads lcp-manifest.yaml
- `StorageContextLoader` - Reads from platform config bucket
- `InMemoryContextLoader` - For testing

---

### 3. ApplicationContext

**File:** `core/types/resolution.ts`

```typescript
export interface ApplicationContext {
  account: string;
  team: string;
  moniker: string;
  environment: string;
  provider: ProviderType;
  region: string;
  dependencies: Record<string, DependencyDeclaration>;
}
```

---

### 4. ResolvedDependency

**File:** `core/types/resolution.ts`

```typescript
export interface ResolvedDependency {
  moniker: string;
  type: DependencyType;
  resourceName: string;      // Full cloud resource name
  config: DependencyConfigMap;
}
```

**Example:**
```typescript
{
  moniker: 'backups',
  type: 'object-store',
  resourceName: 'lcp-123456-backend-order-svc-store-backups-dev',
  config: { versioning: true, encryption: 'aes256' }
}
```

---

### 5. Scoped*Client Classes (11 Types)

**File:** `core/clients/scoped/ScopedObjectClient.ts` (example)

```typescript
export class ScopedObjectClient {
  constructor(
    private readonly client: ObjectClient,
    private readonly bucket: string
  ) {}

  async get(key: string): Promise<ObjectData>;
  async put(key: string, data: Buffer | ReadableStream, metadata?: ObjectMetadata): Promise<void>;
  async delete(key: string): Promise<void>;
  async list(prefix?: string, options?: ListOptions): Promise<ObjectInfo[]>;
  async exists(key: string): Promise<boolean>;
  async getMetadata(key: string): Promise<ObjectMetadata>;
  async getSignedUrl(key: string, op: 'get'|'put', expiresIn?: number): Promise<string>;
  getResourceName(): string;  // Returns bucket name
}
```

**All 11 Scoped Clients:**
- ScopedObjectClient
- ScopedQueueClient
- ScopedSecretsClient
- ScopedConfigClient
- ScopedEventPublisher
- ScopedNotificationClient
- ScopedDocumentClient
- ScopedDataClient
- ScopedAuthClient
- ScopedCacheClient
- ScopedContainerRepoClient

---

### 6. DependencyConfiguration (Complete Union)

**File:** `core/types/dependency.ts`

```typescript
export type DependencyConfiguration =
  | ObjectStoreConfiguration
  | QueueConfiguration
  | SecretsConfiguration
  | DataStoreConfiguration
  | DocumentStoreConfiguration      // NEW
  | EventBusConfiguration          // NEW
  | NotificationConfiguration      // NEW
  | CacheConfiguration             // NEW
  | WebHostingConfiguration        // NEW
  | AuthConfiguration              // NEW
  | FileTransferConfiguration      // NEW
  | WorkflowConfiguration          // NEW
  | SearchConfiguration            // NEW
  | LoggingConfiguration;          // NEW
```

**Each configuration interface includes type-specific options.**

---

## lc-platform-processing-lib Package

Location: `lc-platform-processing-lib/src/`

### 1. IStorageProvider (Port)

**File:** `ports/storage/IStorageProvider.ts`

```typescript
export interface IStorageProvider {
  read<T>(path: string): Promise<Result<T, StorageError>>;
  write<T>(path: string, data: T): Promise<Result<void, StorageError>>;
  delete(path: string): Promise<Result<void, StorageError>>;
  uploadArtifact(localPath: string, remotePath: string): Promise<Result<void, StorageError>>;
}
```

---

### 2. AcceleratorStorageAdapter (Implementation)

**File:** `adapters/storage/AcceleratorStorageAdapter.ts`

```typescript
export class AcceleratorStorageAdapter implements IStorageProvider {
  constructor(
    private readonly objectClient: ObjectClient,
    private readonly configBucket: string
  ) {}

  async read<T>(path: string): Promise<Result<T, StorageError>>;
  async write<T>(path: string, data: T): Promise<Result<void, StorageError>>;
  async delete(path: string): Promise<Result<void, StorageError>>;
  async uploadArtifact(localPath: string, remotePath: string): Promise<Result<void, StorageError>>;
}
```

---

## lc-platform-dev-cli Package

Location: `lc-platform-dev-cli/src/`

### 1. CliContext (Updated)

**File:** `config/types.ts`

```typescript
export interface CliContext {
  // Existing fields
  account?: string;
  team?: string;
  moniker?: string;
  provider?: ProviderType;
  region?: string;

  // NEW: Active application context
  activeApp?: {
    team: string;
    moniker: string;
  };

  // Other fields...
}
```

---

### 2. CLI Error Types

**File:** `cli/shared/error-types.ts`

```typescript
export interface CliError {
  code: string;
  message: string;
  suggestions: string[];
  originalError?: Error;
}

export enum CliErrorCode {
  APP_ALREADY_EXISTS = 'APP_ALREADY_EXISTS',
  APP_NOT_FOUND = 'APP_NOT_FOUND',
  DEPENDENCY_NOT_FOUND = 'DEPENDENCY_NOT_FOUND',
  INVALID_MANIFEST = 'INVALID_MANIFEST',
  NO_ACTIVE_APP = 'NO_ACTIVE_APP',
  // ... other codes
}
```

---

## Manifest Schema

### Complete lcp-manifest.yaml Schema

```yaml
# Schema version and type
apiVersion: lcp/v1
kind: Application

# Application metadata
metadata:
  team: string              # Required: team name (kebab-case)
  moniker: string           # Required: application moniker (kebab-case)
  environment: string       # Optional: dev, staging, prod (supports ${ENV:-dev})
  owner: string             # Optional: email or identifier
  description: string       # Optional: human-readable description

# Provider configuration
provider:
  type: string              # Required: "aws" | "azure" | "gcp" | "mock"
  region: string            # Optional: cloud region (supports ${REGION:-us-east-1})
  account: string           # Optional: account ID (supports ${ACCOUNT})

# Dependency declarations (map of moniker -> declaration)
dependencies:
  <moniker>:
    type: string            # Required: DependencyType
    config: object          # Optional: type-specific configuration
```

---

## Type Aliases & Enums

### DependencyType Enum

```typescript
export enum DependencyType {
  OBJECT_STORE = 'object-store',
  QUEUE = 'queue',
  SECRETS = 'secrets',
  DATA_STORE = 'data-store',
  DOCUMENT_STORE = 'document-store',
  EVENT_BUS = 'event-bus',
  NOTIFICATION = 'notification',
  CACHE = 'cache',
  WEB_HOSTING = 'web-hosting',
  AUTH = 'auth',
  FILE_TRANSFER = 'file-transfer',
  WORKFLOW = 'workflow',
  SEARCH = 'search',
  LOGGING = 'logging'
}
```

---

### ProviderType Enum

```typescript
export enum ProviderType {
  AWS = 'aws',
  AZURE = 'azure',
  GCP = 'gcp',
  MOCK = 'mock'
}
```

---

## API Types

### Context Export (Agent API)

**File:** `lc-platform-dev-cli/src/cli/commands/context/types.ts`

```typescript
export interface ContextExport {
  metadata: {
    team: string;
    moniker: string;
    environment: string;
  };
  dependencies: Record<string, DependencyExport>;
}

export interface DependencyExport {
  type: DependencyType;
  accessPattern: string;      // e.g., "runtime.store('backups')"
  example: string;            // Code example
  resourceName?: string;      // Optional: resolved name
}
```

**Example JSON:**
```json
{
  "metadata": {
    "team": "backend",
    "moniker": "order-svc",
    "environment": "dev"
  },
  "dependencies": {
    "backups": {
      "type": "object-store",
      "accessPattern": "runtime.store('backups')",
      "example": "await runtime.store('backups').put('file.txt', data)"
    }
  }
}
```

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-09 | Initial version - LC Platform v2 data dictionary |
