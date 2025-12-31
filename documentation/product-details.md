# LC Platform - Product Details

**Version**: 1.0.0
**Last Updated**: 2025-12-31
**Audience**: Product Managers, Architects, Technical Leads

## Table of Contents

1. [Platform Components](#platform-components)
2. [Service Interface Definitions](#service-interface-definitions)
3. [Configuration Schemas](#configuration-schemas)
4. [API Reference](#api-reference)
5. [Deployment Guides](#deployment-guides)
6. [Integration Patterns](#integration-patterns)
7. [Migration Strategies](#migration-strategies)

---

## Platform Components

### 1. lc-platform-dev-accelerators

**Status**: ✅ MVP Complete
**Package**: `@stainedhead/lc-platform-dev-accelerators`
**Runtime**: Bun 1.0+

#### Control Plane Services (14)

Control Plane services manage infrastructure provisioning and lifecycle:

| Service | Purpose | AWS Implementation | Azure Implementation |
|---------|---------|-------------------|---------------------|
| `WebHostingService` | Deploy and manage containerized web applications | AWS App Runner | Azure Container Apps |
| `FunctionHostingService` | Deploy and manage serverless functions | AWS Lambda | Azure Function Apps |
| `BatchService` | Schedule and run batch processing jobs | AWS Batch | Azure Batch |
| `DataStoreService` | Provision and manage relational databases | RDS PostgreSQL | Database for PostgreSQL |
| `DocumentStoreService` | Provision and manage NoSQL databases | DocumentDB | Cosmos DB |
| `ObjectStoreService` | Create and manage object storage buckets | S3 | Blob Storage |
| `QueueService` | Create and manage message queues | SQS | Storage Queues |
| `EventBusService` | Create and manage event buses | EventBridge | Event Grid |
| `SecretsService` | Provision secrets storage | Secrets Manager | Key Vault |
| `ConfigurationService` | Provision configuration storage | AppConfig | App Configuration |
| `NotificationService` | Create notification topics | SNS | Notification Hubs |
| `AuthenticationService` | Provision authentication services | Cognito + Okta | Azure AD B2C |
| `CacheService` | Provision Redis cache instances | ElastiCache Redis | Cache for Redis |
| `ContainerRepoService` | Create container registries | ECR | Container Registry |

#### Data Plane Clients (11)

Data Plane clients perform runtime operations on provisioned infrastructure:

| Client | Purpose | Operations |
|--------|---------|-----------|
| `QueueClient` | Send and receive messages | send, receive, delete, purge |
| `ObjectClient` | Store and retrieve objects | put, get, delete, list |
| `SecretsClient` | Read secrets at runtime | get, list |
| `ConfigClient` | Read configuration at runtime | get, list, watch |
| `EventPublisher` | Publish events to event bus | publish, publishBatch |
| `NotificationClient` | Send notifications | publish, publishBatch |
| `DocumentClient` | Query and mutate NoSQL data | query, get, put, delete, update |
| `DataClient` | Execute SQL queries | query, execute, transaction |
| `AuthClient` | Authenticate users at runtime | login, logout, verify, refresh |
| `CacheClient` | Cache operations | get, set, delete, exists, ttl |
| `ContainerRepoClient` | Push/pull container images | push, pull, list, delete |

### 2. lc-platform-cli

**Status**: 📋 Planned
**Purpose**: Control Plane command-line interface

#### Capabilities

```bash
# Infrastructure Management
lc-cli provision web-app --config app.json
lc-cli provision database --type postgres --size small
lc-cli provision queue --name user-events

# Application Deployment
lc-cli deploy --app my-app --env production
lc-cli deploy --function my-func --runtime nodejs18

# Configuration Management
lc-cli config set DATABASE_URL --env production
lc-cli config list --env staging
lc-cli config export --env production > config.json

# Environment Management
lc-cli env create staging
lc-cli env list
lc-cli env delete staging

# Status and Monitoring
lc-cli status --app my-app
lc-cli logs --app my-app --follow
lc-cli metrics --app my-app --last 1h
```

### 3. lc-platform-config

**Status**: 📋 Planned
**Purpose**: Centralized configuration management

#### Features

- Environment-based configuration
- Service dependency declarations
- JSON Schema validation
- Configuration versioning
- Secrets integration
- Environment variable substitution

#### Configuration Structure

```typescript
interface PlatformConfig {
  version: string;
  environment: string;
  provider: 'aws' | 'azure' | 'gcp' | 'mock';
  services: ServiceConfig[];
  dependencies: DependencyConfig[];
}

interface ServiceConfig {
  name: string;
  type: ServiceType;
  config: Record<string, unknown>;
}

interface DependencyConfig {
  service: string;
  dependsOn: string[];
  optional: boolean;
}
```

### 4. lc-platform-rest-api

**Status**: 📋 Future
**Purpose**: REST API for platform operations

#### Planned Endpoints

```
POST   /api/v1/applications              # Deploy application
GET    /api/v1/applications              # List applications
GET    /api/v1/applications/:id          # Get application details
PUT    /api/v1/applications/:id          # Update application
DELETE /api/v1/applications/:id          # Delete application

POST   /api/v1/infrastructure            # Provision infrastructure
GET    /api/v1/infrastructure            # List infrastructure
DELETE /api/v1/infrastructure/:id        # Deprovision infrastructure

POST   /api/v1/configurations            # Create configuration
GET    /api/v1/configurations/:env       # Get configuration
PUT    /api/v1/configurations/:env       # Update configuration
```

---

## Service Interface Definitions

### Example: ObjectStoreService (Control Plane)

```typescript
interface ObjectStoreService {
  /**
   * Create a new object storage bucket
   */
  createBucket(config: BucketConfig): Promise<Bucket>;

  /**
   * Delete an existing bucket
   */
  deleteBucket(bucketName: string): Promise<void>;

  /**
   * List all buckets
   */
  listBuckets(): Promise<Bucket[]>;

  /**
   * Configure bucket settings (versioning, lifecycle, etc.)
   */
  configureBucket(bucketName: string, config: BucketSettings): Promise<void>;
}

interface BucketConfig {
  name: string;
  region: string;
  versioning?: boolean;
  encryption?: EncryptionConfig;
  lifecycleRules?: LifecycleRule[];
}

interface Bucket {
  name: string;
  region: string;
  createdAt: Date;
  versioning: boolean;
  encryption: EncryptionConfig;
}
```

### Example: ObjectClient (Data Plane)

```typescript
interface ObjectClient {
  /**
   * Upload an object to storage
   */
  putObject(bucket: string, key: string, data: Buffer, metadata?: Metadata): Promise<void>;

  /**
   * Download an object from storage
   */
  getObject(bucket: string, key: string): Promise<ObjectData>;

  /**
   * Delete an object
   */
  deleteObject(bucket: string, key: string): Promise<void>;

  /**
   * List objects in a bucket
   */
  listObjects(bucket: string, prefix?: string): Promise<ObjectInfo[]>;
}

interface ObjectData {
  data: Buffer;
  metadata: Metadata;
  contentType: string;
  size: number;
  lastModified: Date;
}
```

---

## Configuration Schemas

### Application Configuration Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "pattern": "^[a-z][a-z0-9-]*$"
    },
    "type": {
      "type": "string",
      "enum": ["web", "function", "batch"]
    },
    "runtime": {
      "type": "string",
      "enum": ["nodejs18", "nodejs20", "python3.11", "python3.12"]
    },
    "dependencies": {
      "type": "object",
      "properties": {
        "database": {
          "type": "object",
          "properties": {
            "type": { "enum": ["postgres", "mysql", "cosmosdb"] },
            "size": { "enum": ["small", "medium", "large"] }
          }
        },
        "storage": {
          "type": "object",
          "properties": {
            "buckets": {
              "type": "array",
              "items": { "type": "string" }
            }
          }
        },
        "queues": {
          "type": "array",
          "items": { "type": "string" }
        },
        "secrets": {
          "type": "array",
          "items": { "type": "string" }
        }
      }
    },
    "scaling": {
      "type": "object",
      "properties": {
        "min": { "type": "integer", "minimum": 1 },
        "max": { "type": "integer", "minimum": 1 },
        "targetCPU": { "type": "integer", "minimum": 1, "maximum": 100 }
      }
    }
  },
  "required": ["name", "type", "runtime"]
}
```

---

## API Reference

### LCPlatform Class (Control Plane)

```typescript
class LCPlatform {
  constructor(config: PlatformConfig);

  // Control Plane Services
  getWebHosting(): WebHostingService;
  getFunctionHosting(): FunctionHostingService;
  getBatchService(): BatchService;
  getDataStore(): DataStoreService;
  getDocumentStore(): DocumentStoreService;
  getObjectStore(): ObjectStoreService;
  getQueueService(): QueueService;
  getEventBus(): EventBusService;
  getSecretsService(): SecretsService;
  getConfigService(): ConfigurationService;
  getNotificationService(): NotificationService;
  getAuthService(): AuthenticationService;
  getCacheService(): CacheService;
  getContainerRepo(): ContainerRepoService;
}
```

### LCAppRuntime Class (Data Plane)

```typescript
class LCAppRuntime {
  constructor(config: RuntimeConfig);

  // Data Plane Clients
  getQueueClient(queueName: string): QueueClient;
  getObjectClient(bucketName: string): ObjectClient;
  getSecretsClient(): SecretsClient;
  getConfigClient(): ConfigClient;
  getEventPublisher(): EventPublisher;
  getNotificationClient(): NotificationClient;
  getDocumentClient(database: string): DocumentClient;
  getDataClient(connectionString: string): DataClient;
  getAuthClient(): AuthClient;
  getCacheClient(cacheName: string): CacheClient;
  getContainerRepoClient(repoName: string): ContainerRepoClient;
}
```

---

## Deployment Guides

### Deploying a Web Application

#### Step 1: Create Application Configuration

```json
{
  "name": "my-web-app",
  "type": "web",
  "runtime": "nodejs20",
  "dependencies": {
    "database": {
      "type": "postgres",
      "size": "small"
    },
    "storage": {
      "buckets": ["uploads", "assets"]
    },
    "queues": ["user-events"],
    "secrets": ["DATABASE_PASSWORD", "API_KEY"]
  },
  "scaling": {
    "min": 2,
    "max": 10,
    "targetCPU": 70
  }
}
```

#### Step 2: Deploy Using CLI (Planned)

```bash
# Deploy to AWS
lc-cli deploy --config app.json --provider aws --env production

# Or deploy to Azure
lc-cli deploy --config app.json --provider azure --env production
```

#### Step 3: Access from Application Code

```typescript
import { LCAppRuntime } from '@stainedhead/lc-platform-dev-accelerators';

const runtime = new LCAppRuntime({ provider: 'aws' });

// Access provisioned resources
const db = runtime.getDataClient(process.env.DATABASE_URL);
const storage = runtime.getObjectClient('uploads');
const queue = runtime.getQueueClient('user-events');
const secrets = runtime.getSecretsClient();

// Use cloud-agnostic APIs
const apiKey = await secrets.get('API_KEY');
await storage.putObject('file.txt', data);
await queue.send({ event: 'user-signup', userId: 123 });
```

---

## Integration Patterns

### Pattern 1: Event-Driven Architecture

```typescript
// Publisher Service
const eventBus = runtime.getEventPublisher();

await eventBus.publish('user.created', {
  userId: user.id,
  email: user.email,
  timestamp: new Date()
});

// Subscriber Service
const queue = runtime.getQueueClient('user-events');

while (true) {
  const messages = await queue.receive({ maxMessages: 10 });

  for (const message of messages) {
    const event = JSON.parse(message.body);
    await processUserEvent(event);
    await queue.delete(message.receiptHandle);
  }
}
```

### Pattern 2: Secrets Management

```typescript
// Application startup
const secrets = runtime.getSecretsClient();

const config = {
  database: await secrets.get('DATABASE_URL'),
  apiKey: await secrets.get('API_KEY'),
  jwtSecret: await secrets.get('JWT_SECRET')
};

// Use configuration throughout app
const db = connectDatabase(config.database);
const api = createAPIClient(config.apiKey);
```

### Pattern 3: Multi-Tier Storage

```typescript
// Cache-aside pattern
async function getUser(userId: string): Promise<User> {
  const cache = runtime.getCacheClient('users');

  // Try cache first
  const cached = await cache.get(`user:${userId}`);
  if (cached) {
    return JSON.parse(cached);
  }

  // Fall back to database
  const db = runtime.getDocumentClient('users');
  const user = await db.get({ id: userId });

  // Update cache
  await cache.set(`user:${userId}`, JSON.stringify(user), { ttl: 3600 });

  return user;
}
```

---

## Migration Strategies

### Migrating from AWS SDK to LC Platform

#### Before (AWS-Specific)

```typescript
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

const s3 = new S3Client({ region: 'us-east-1' });

await s3.send(new PutObjectCommand({
  Bucket: 'my-bucket',
  Key: 'file.txt',
  Body: data
}));
```

#### After (Cloud-Agnostic)

```typescript
import { LCAppRuntime } from '@stainedhead/lc-platform-dev-accelerators';

const runtime = new LCAppRuntime({ provider: 'aws' });
const storage = runtime.getObjectClient('my-bucket');

await storage.putObject('file.txt', data);
```

### Migration Checklist

- [ ] Identify cloud-specific SDK usage
- [ ] Map to LC Platform service interfaces
- [ ] Replace cloud SDK imports with LC Platform imports
- [ ] Update configuration to use LC Platform config
- [ ] Add tests using mock provider
- [ ] Deploy to test environment
- [ ] Verify functionality
- [ ] Deploy to production

---

**For more information**, see:
- [Product Summary](product-summary.md) - High-level overview
- [Technical Details](technical-details.md) - Architecture deep dive
- [AGENTS.md](../AGENTS.md) - Development rules and context
