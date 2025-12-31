# LC Platform - Technical Details

**Version**: 1.0.0
**Last Updated**: 2025-12-31
**Audience**: Software Engineers, Technical Architects, DevOps Engineers

## Table of Contents

1. [Hexagonal Architecture Implementation](#hexagonal-architecture-implementation)
2. [Provider Abstraction Layer](#provider-abstraction-layer)
3. [Code Organization](#code-organization)
4. [Extension Points](#extension-points)
5. [Testing Architecture](#testing-architecture)
6. [Performance Considerations](#performance-considerations)
7. [Security Model](#security-model)
8. [Error Handling](#error-handling)

---

## Hexagonal Architecture Implementation

### Core Concepts

LC Platform implements pure Hexagonal Architecture (Ports and Adapters pattern):

```
┌────────────────────────────────────────────────────────────┐
│                    Application Layer                       │
│                 (Cloud-Agnostic Logic)                     │
└────────────────────┬───────────────────────────────────────┘
                     │
                     │ depends on
                     ▼
┌────────────────────────────────────────────────────────────┐
│                    Core Ports (Interfaces)                 │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ ObjectStore  │  │ QueueService │  │ AuthService  │   │
│  │  Service     │  │              │  │              │   │
│  └──────────────┘  └──────────────┘  └──────────────┘   │
│                                                            │
└────────────────────┬───────────────────────────────────────┘
                     │
                     │ implemented by
                     ▼
┌────────────────────────────────────────────────────────────┐
│                 Adapters (Implementations)                 │
│                                                            │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐         │
│  │   AWS    │     │  Azure   │     │   Mock   │         │
│  │ Adapter  │     │ Adapter  │     │ Adapter  │         │
│  └──────────┘     └──────────┘     └──────────┘         │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

#### 1. Application Layer (Consumer Code)

**Responsibilities**:
- Business logic implementation
- Workflow orchestration
- Domain model management

**Rules**:
- MUST depend only on Core Ports (interfaces)
- MUST NOT import adapter implementations
- MUST NOT use cloud SDK types
- MUST be testable with Mock adapter

**Example**:
```typescript
// ✅ CORRECT - Depends on interface
import { ObjectStoreService } from '@lc-platform/core';

class FileService {
  constructor(private storage: ObjectStoreService) {}

  async uploadFile(file: File): Promise<void> {
    await this.storage.putObject('uploads', file.name, file.data);
  }
}

// ❌ WRONG - Direct dependency on adapter
import { AWSS3Service } from '@lc-platform/aws';
```

#### 2. Core Ports (Interfaces)

**Responsibilities**:
- Define service contracts
- Specify cloud-agnostic types
- Document behavior expectations

**Rules**:
- MUST be cloud-agnostic (no AWS/Azure/GCP terminology)
- MUST use generic types only
- MUST NOT contain implementation logic
- MUST be independently versioned

**Example**:
```typescript
// src/core/services/ObjectStoreService.ts

/**
 * Cloud-agnostic object storage service
 */
export interface ObjectStoreService {
  /**
   * Upload an object to storage
   * @param bucket - Bucket/container name
   * @param key - Object identifier
   * @param data - Object data as Buffer
   */
  putObject(bucket: string, key: string, data: Buffer): Promise<void>;

  /**
   * Download an object from storage
   * @param bucket - Bucket/container name
   * @param key - Object identifier
   * @returns Object data and metadata
   */
  getObject(bucket: string, key: string): Promise<ObjectData>;
}

// ✅ CORRECT - Generic types
export interface ObjectData {
  data: Buffer;
  contentType: string;
  size: number;
  lastModified: Date;
}

// ❌ WRONG - Cloud-specific types
export interface ObjectData {
  data: Buffer;
  s3Metadata: S3Metadata; // AWS-specific!
}
```

#### 3. Adapters (Implementations)

**Responsibilities**:
- Implement Core Port interfaces
- Translate between cloud SDKs and generic interfaces
- Handle cloud-specific authentication
- Implement retry logic and error handling

**Rules**:
- MUST implement Core Port interfaces exactly
- MAY use cloud SDK types internally
- MUST NOT expose cloud SDK types in return values
- MUST provide equivalent functionality across providers

**Example**:
```typescript
// src/providers/aws/services/AWSS3ObjectStore.ts

import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { ObjectStoreService, ObjectData } from '@lc-platform/core';

export class AWSS3ObjectStore implements ObjectStoreService {
  private s3: S3Client;

  constructor(config: AWSConfig) {
    this.s3 = new S3Client({ region: config.region });
  }

  async putObject(bucket: string, key: string, data: Buffer): Promise<void> {
    // Translate generic request to AWS-specific command
    const command = new PutObjectCommand({
      Bucket: bucket,
      Key: key,
      Body: data
    });

    await this.s3.send(command);
  }

  async getObject(bucket: string, key: string): Promise<ObjectData> {
    // Implementation using AWS SDK
    // Returns generic ObjectData, not AWS-specific types
  }
}
```

---

## Provider Abstraction Layer

### Factory Pattern

LC Platform uses the Factory pattern for provider selection:

```typescript
// src/factory/LCPlatformFactory.ts

export class LCPlatformFactory {
  static create(config: PlatformConfig): LCPlatform {
    const provider = config.provider || process.env.LC_PLATFORM_PROVIDER || 'mock';

    switch (provider) {
      case 'aws':
        return new LCPlatform(new AWSServiceProvider(config));
      case 'azure':
        return new LCPlatform(new AzureServiceProvider(config));
      case 'mock':
        return new LCPlatform(new MockServiceProvider(config));
      default:
        throw new Error(`Unknown provider: ${provider}`);
    }
  }
}
```

### Service Provider Interface

```typescript
interface ServiceProvider {
  getObjectStore(): ObjectStoreService;
  getQueueService(): QueueService;
  getSecretsService(): SecretsService;
  // ... all other services
}

class AWSServiceProvider implements ServiceProvider {
  constructor(private config: AWSConfig) {}

  getObjectStore(): ObjectStoreService {
    return new AWSS3ObjectStore(this.config);
  }

  getQueueService(): QueueService {
    return new AWSSQSService(this.config);
  }

  // ... other services
}
```

### Configuration System

```typescript
interface PlatformConfig {
  provider: 'aws' | 'azure' | 'gcp' | 'mock';
  region?: string;
  credentials?: ProviderCredentials;
  retry?: RetryConfig;
  timeout?: number;
}

interface RetryConfig {
  maxRetries: number;
  retryDelay: number;
  retryMode: 'exponential' | 'fixed';
}
```

---

## Code Organization

### Monorepo Structure

```
lc-platform/
├── lc-platform-dev-accelerators/
│   ├── src/
│   │   ├── core/                   # Core interfaces (Ports)
│   │   │   ├── services/           # Control Plane interfaces
│   │   │   │   ├── ObjectStoreService.ts
│   │   │   │   ├── QueueService.ts
│   │   │   │   └── ...
│   │   │   ├── clients/            # Data Plane interfaces
│   │   │   │   ├── ObjectClient.ts
│   │   │   │   ├── QueueClient.ts
│   │   │   │   └── ...
│   │   │   └── types/              # Shared types
│   │   │       ├── common.ts
│   │   │       └── errors.ts
│   │   │
│   │   ├── providers/              # Adapters
│   │   │   ├── aws/
│   │   │   │   ├── services/       # AWS Control Plane
│   │   │   │   │   ├── AWSS3ObjectStore.ts
│   │   │   │   │   └── ...
│   │   │   │   └── clients/        # AWS Data Plane
│   │   │   │       ├── AWSS3Client.ts
│   │   │   │       └── ...
│   │   │   ├── azure/
│   │   │   │   ├── services/
│   │   │   │   └── clients/
│   │   │   └── mock/
│   │   │       ├── services/
│   │   │       └── clients/
│   │   │
│   │   ├── factory/                # Dependency injection
│   │   │   ├── LCPlatformFactory.ts
│   │   │   └── LCAppRuntimeFactory.ts
│   │   │
│   │   ├── LCPlatform.ts          # Control Plane entry point
│   │   ├── LCAppRuntime.ts        # Data Plane entry point
│   │   └── index.ts               # Public API exports
│   │
│   ├── tests/
│   │   ├── unit/                  # Unit tests (mock provider)
│   │   ├── integration/           # Integration tests (LocalStack)
│   │   └── contract/              # Contract tests (AWS ↔ Mock)
│   │
│   └── AGENTS.md                  # Package-specific rules
│
├── lc-platform-cli/               # CLI package (planned)
├── lc-platform-config/            # Config package (planned)
├── lc-platform-rest-api/          # REST API package (future)
│
├── AGENTS.md                      # Parent-level rules
├── CLAUDE.md                      # Pointer to AGENTS.md
├── .github/
│   └── copilot-instructions.md    # GitHub Copilot pointer
└── .specify/
    └── memory/
        └── constitution.md         # Governance
```

### Module Exports

```typescript
// src/index.ts - Public API

// Core entry points
export { LCPlatform } from './LCPlatform';
export { LCAppRuntime } from './LCAppRuntime';

// Core interfaces (for typing)
export * from './core/services';
export * from './core/clients';
export * from './core/types';

// Factory (optional, for advanced use)
export { LCPlatformFactory } from './factory/LCPlatformFactory';

// NOTE: Adapters are NOT exported - internal implementation detail
```

---

## Extension Points

### Adding a New Cloud Provider

#### Step 1: Implement Service Provider

```typescript
// src/providers/gcp/GCPServiceProvider.ts

import { ServiceProvider } from '../../factory/ServiceProvider';
import { ObjectStoreService, QueueService /* ... */ } from '../../core';

export class GCPServiceProvider implements ServiceProvider {
  constructor(private config: GCPConfig) {}

  getObjectStore(): ObjectStoreService {
    return new GCPCloudStorageService(this.config);
  }

  getQueueService(): QueueService {
    return new GCPPubSubService(this.config);
  }

  // Implement all other services...
}
```

#### Step 2: Implement Each Service

```typescript
// src/providers/gcp/services/GCPCloudStorageService.ts

import { Storage } from '@google-cloud/storage';
import { ObjectStoreService, ObjectData } from '../../../core';

export class GCPCloudStorageService implements ObjectStoreService {
  private storage: Storage;

  constructor(config: GCPConfig) {
    this.storage = new Storage({ projectId: config.projectId });
  }

  async putObject(bucket: string, key: string, data: Buffer): Promise<void> {
    const file = this.storage.bucket(bucket).file(key);
    await file.save(data);
  }

  async getObject(bucket: string, key: string): Promise<ObjectData> {
    const file = this.storage.bucket(bucket).file(key);
    const [data] = await file.download();
    const [metadata] = await file.getMetadata();

    return {
      data,
      contentType: metadata.contentType,
      size: metadata.size,
      lastModified: new Date(metadata.updated)
    };
  }
}
```

#### Step 3: Register Provider

```typescript
// src/factory/LCPlatformFactory.ts

case 'gcp':
  return new LCPlatform(new GCPServiceProvider(config));
```

### Adding a New Service

#### Step 1: Define Core Interface

```typescript
// src/core/services/MLService.ts

export interface MLService {
  /**
   * Deploy a machine learning model
   */
  deployModel(config: ModelConfig): Promise<ModelEndpoint>;

  /**
   * Invoke a deployed model
   */
  invokeModel(endpointId: string, input: unknown): Promise<unknown>;

  /**
   * Delete a deployed model
   */
  deleteModel(endpointId: string): Promise<void>;
}

export interface ModelConfig {
  name: string;
  framework: 'tensorflow' | 'pytorch' | 'scikit-learn';
  runtime: string;
  modelArtifactPath: string;
}

export interface ModelEndpoint {
  id: string;
  url: string;
  status: 'deploying' | 'ready' | 'failed';
}
```

#### Step 2: Implement for Each Provider

```typescript
// src/providers/aws/services/AWSMLService.ts

import { SageMakerClient, CreateEndpointCommand } from '@aws-sdk/client-sagemaker';
import { MLService, ModelConfig, ModelEndpoint } from '../../../core';

export class AWSMLService implements MLService {
  private sagemaker: SageMakerClient;

  constructor(config: AWSConfig) {
    this.sagemaker = new SageMakerClient({ region: config.region });
  }

  async deployModel(config: ModelConfig): Promise<ModelEndpoint> {
    // AWS SageMaker implementation
  }

  async invokeModel(endpointId: string, input: unknown): Promise<unknown> {
    // AWS SageMaker Runtime implementation
  }

  async deleteModel(endpointId: string): Promise<void> {
    // AWS SageMaker deletion implementation
  }
}
```

#### Step 3: Add to Platform

```typescript
// src/LCPlatform.ts

export class LCPlatform {
  // ... existing services

  getMLService(): MLService {
    return this.provider.getMLService();
  }
}
```

---

## Testing Architecture

### Test Pyramid

```
        ┌─────────────┐
        │   Contract  │  (Verify AWS ↔ Mock parity)
        │    Tests    │
        └─────────────┘
       ┌───────────────┐
       │  Integration  │  (LocalStack, Azurite)
       │     Tests     │
       └───────────────┘
      ┌─────────────────┐
      │   Unit Tests    │  (Mock provider, fast)
      │   (80%+ cov)    │
      └─────────────────┘
```

### Unit Tests (Mock Provider)

```typescript
// tests/unit/ObjectStoreService.test.ts

import { describe, it, expect, beforeEach } from 'bun:test';
import { LCPlatform } from '../../src';

describe('ObjectStoreService', () => {
  let platform: LCPlatform;
  let storage: ObjectStoreService;

  beforeEach(() => {
    platform = new LCPlatform({ provider: 'mock' });
    storage = platform.getObjectStore();
  });

  it('should store and retrieve objects', async () => {
    const data = Buffer.from('Hello World');

    await storage.putObject('test-bucket', 'file.txt', data);
    const result = await storage.getObject('test-bucket', 'file.txt');

    expect(result.data).toEqual(data);
    expect(result.size).toBe(11);
  });

  it('should list objects in bucket', async () => {
    await storage.putObject('test-bucket', 'file1.txt', Buffer.from('A'));
    await storage.putObject('test-bucket', 'file2.txt', Buffer.from('B'));

    const objects = await storage.listObjects('test-bucket');

    expect(objects).toHaveLength(2);
    expect(objects.map(o => o.key)).toContain('file1.txt');
    expect(objects.map(o => o.key)).toContain('file2.txt');
  });
});
```

### Integration Tests (LocalStack)

```typescript
// tests/integration/AWSS3.test.ts

import { describe, it, expect, beforeAll, afterAll } from 'bun:test';
import { LCPlatform } from '../../src';
import { startLocalStack, stopLocalStack } from '../helpers/localstack';

describe('AWS S3 Integration', () => {
  let platform: LCPlatform;

  beforeAll(async () => {
    await startLocalStack();
    platform = new LCPlatform({
      provider: 'aws',
      region: 'us-east-1',
      endpoint: 'http://localhost:4566' // LocalStack
    });
  });

  afterAll(async () => {
    await stopLocalStack();
  });

  it('should create bucket and store object', async () => {
    const storage = platform.getObjectStore();

    await storage.createBucket({ name: 'test-bucket', region: 'us-east-1' });
    await storage.putObject('test-bucket', 'test.txt', Buffer.from('data'));

    const result = await storage.getObject('test-bucket', 'test.txt');
    expect(result.data.toString()).toBe('data');
  });
});
```

### Contract Tests (Provider Parity)

```typescript
// tests/contract/ObjectStore.contract.ts

import { describe, it, expect } from 'bun:test';
import { LCPlatform } from '../../src';

const providers = ['aws', 'mock'] as const;

providers.forEach(provider => {
  describe(`ObjectStoreService Contract - ${provider}`, () => {
    let platform: LCPlatform;

    beforeEach(() => {
      platform = new LCPlatform({ provider });
    });

    it('putObject should accept Buffer data', async () => {
      const storage = platform.getObjectStore();
      const data = Buffer.from('test data');

      await expect(
        storage.putObject('bucket', 'key', data)
      ).resolves.toBeUndefined();
    });

    it('getObject should return ObjectData structure', async () => {
      const storage = platform.getObjectStore();

      await storage.putObject('bucket', 'key', Buffer.from('data'));
      const result = await storage.getObject('bucket', 'key');

      expect(result).toHaveProperty('data');
      expect(result).toHaveProperty('contentType');
      expect(result).toHaveProperty('size');
      expect(result).toHaveProperty('lastModified');
      expect(result.data).toBeInstanceOf(Buffer);
    });
  });
});
```

---

## Performance Considerations

### Connection Pooling

```typescript
class AWSServiceProvider {
  private s3Client: S3Client;
  private sqsClient: SQSClient;

  constructor(config: AWSConfig) {
    // Reuse clients for connection pooling
    this.s3Client = new S3Client({
      region: config.region,
      maxAttempts: 3
    });

    this.sqsClient = new SQSClient({
      region: config.region,
      maxAttempts: 3
    });
  }

  getObjectStore(): ObjectStoreService {
    // Return new service instance, but reuse client
    return new AWSS3ObjectStore(this.s3Client);
  }
}
```

### Batch Operations

```typescript
interface ObjectClient {
  /**
   * Upload multiple objects in a batch
   */
  putObjectsBatch(
    bucket: string,
    objects: Array<{ key: string; data: Buffer }>
  ): Promise<BatchResult>;
}

class AWSS3Client implements ObjectClient {
  async putObjectsBatch(
    bucket: string,
    objects: Array<{ key: string; data: Buffer }>
  ): Promise<BatchResult> {
    // Use Promise.all for parallel uploads
    const results = await Promise.allSettled(
      objects.map(obj =>
        this.s3.send(
          new PutObjectCommand({
            Bucket: bucket,
            Key: obj.key,
            Body: obj.data
          })
        )
      )
    );

    return {
      succeeded: results.filter(r => r.status === 'fulfilled').length,
      failed: results.filter(r => r.status === 'rejected').length
    };
  }
}
```

### Caching Strategy

```typescript
class CachedObjectStore implements ObjectStoreService {
  private cache = new LRUCache<string, ObjectData>({ max: 1000 });

  constructor(private underlying: ObjectStoreService) {}

  async getObject(bucket: string, key: string): Promise<ObjectData> {
    const cacheKey = `${bucket}:${key}`;
    const cached = this.cache.get(cacheKey);

    if (cached) {
      return cached;
    }

    const data = await this.underlying.getObject(bucket, key);
    this.cache.set(cacheKey, data);

    return data;
  }
}
```

---

## Security Model

### Workload Identity

Prefer IAM roles and managed identities over access keys:

```typescript
// ✅ PREFERRED - Uses workload identity
const platform = new LCPlatform({
  provider: 'aws',
  region: 'us-east-1'
  // No credentials - uses IAM role from EC2/ECS/Lambda
});

// ❌ AVOID - Access keys in code
const platform = new LCPlatform({
  provider: 'aws',
  credentials: {
    accessKeyId: 'AKIA...',
    secretAccessKey: 'secret'
  }
});
```

### Secrets Management

```typescript
// Application code
const secrets = runtime.getSecretsClient();

// Secrets are never logged or exposed
const apiKey = await secrets.get('API_KEY');
const dbPassword = await secrets.get('DB_PASSWORD');

// Use secrets without storing in memory longer than needed
await authenticateAPI(apiKey);
await connectDatabase(dbPassword);
```

### Least Privilege

```typescript
// Each service gets minimal required permissions
interface ServicePermissions {
  actions: string[];
  resources: string[];
}

const objectStorePermissions: ServicePermissions = {
  actions: ['s3:GetObject', 's3:PutObject', 's3:DeleteObject'],
  resources: ['arn:aws:s3:::my-bucket/*']
};
```

---

## Error Handling

### Error Hierarchy

```typescript
// Base error
export class LCPlatformError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly retryable: boolean
  ) {
    super(message);
    this.name = 'LCPlatformError';
  }
}

// Service-specific errors
export class ObjectStoreError extends LCPlatformError {
  constructor(message: string, code: string, retryable = false) {
    super(message, code, retryable);
    this.name = 'ObjectStoreError';
  }
}

// Common error types
export class NotFoundError extends LCPlatformError {
  constructor(resource: string) {
    super(`Resource not found: ${resource}`, 'NOT_FOUND', false);
    this.name = 'NotFoundError';
  }
}

export class ServiceUnavailableError extends LCPlatformError {
  constructor(service: string) {
    super(`Service unavailable: ${service}`, 'SERVICE_UNAVAILABLE', true);
    this.name = 'ServiceUnavailableError';
  }
}
```

### Retry Logic

```typescript
async function withRetry<T>(
  operation: () => Promise<T>,
  config: RetryConfig
): Promise<T> {
  let lastError: Error;

  for (let attempt = 0; attempt <= config.maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error as Error;

      if (!isRetryable(error) || attempt === config.maxRetries) {
        throw error;
      }

      const delay = config.retryMode === 'exponential'
        ? config.retryDelay * Math.pow(2, attempt)
        : config.retryDelay;

      await sleep(delay);
    }
  }

  throw lastError!;
}
```

---

**For more information**, see:
- [Product Summary](product-summary.md) - High-level overview
- [Product Details](product-details.md) - Detailed specifications
- [AGENTS.md](../AGENTS.md) - Development rules and context
