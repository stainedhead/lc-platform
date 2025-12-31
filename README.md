# LC Platform

> A cloud-agnostic platform that enables development teams to create and manage cloud solutions without requiring deep cloud expertise.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.9.3-blue.svg)](https://www.typescriptlang.org/)
[![Bun](https://img.shields.io/badge/Bun-1.0%2B-orange.svg)](https://bun.sh/)

## Overview

LC Platform is a comprehensive cloud-agnostic platform built on **Hexagonal Architecture** principles. It provides development teams with the ability to build, deploy, and manage cloud applications across AWS, Azure, and GCP without vendor lock-in.

### Key Features

- **☁️ Cloud Portability** - Switch between AWS, Azure, and GCP without changing application code
- **🏗️ Clean Architecture** - Hexagonal Engineering patterns with complete provider abstraction
- **🤖 Agentic Tooling** - First-class support for AI-powered development workflows
- **⚙️ Unified Configuration** - Centralized platform configuration management
- **🧪 Mock Provider** - Local development and testing without cloud credentials
- **📦 Independent Packages** - Modular design with clear boundaries and versioning

## Platform Architecture

LC Platform consists of four independent packages that work together as a cohesive ecosystem:

```
lc-platform/
├── lc-platform-dev-accelerators/    # Cloud-agnostic service wrappers (MVP Complete)
├── lc-platform-cli/                 # Control Plane CLI (Planned)
├── lc-platform-config/              # Platform configuration (Planned)
└── lc-platform-rest-api/            # REST API service (Future)
```

### Package Overview

| Package | Status | Purpose | Runtime |
|---------|--------|---------|---------|
| **dev-accelerators** | ✅ MVP Complete | Provider-agnostic service wrappers with 14 Control Plane services & 11 Data Plane clients | Bun 1.0+ |
| **cli** | 📋 Planned | Command-line interface for infrastructure provisioning and application deployment | TBD |
| **config** | 📋 Planned | Centralized configuration management with validation and versioning | TBD |
| **rest-api** | 📋 Future | REST API for programmatic platform access | TBD |

## Architectural Principles

### 1. Hexagonal Architecture (Ports and Adapters)

```
┌─────────────────────────────────────────┐
│        Application Layer                │
│   (Cloud-Agnostic Business Logic)      │
└──────────────┬──────────────────────────┘
               │
       ┌───────▼────────┐
       │  Core Ports    │ (Interfaces)
       │  (Contracts)   │
       └───────┬────────┘
               │
    ┌──────────┴──────────────┐
    │                         │
┌───▼─────┐            ┌──────▼───┐
│   AWS   │            │  Azure   │
│ Adapter │            │  Adapter │
└─────────┘            └──────────┘
                    ┌──────▼───┐
                    │   Mock   │
                    │  Adapter │
                    └──────────┘
```

**Key Concepts:**
- **Core Domain**: Cloud-agnostic business logic and interfaces
- **Ports**: Abstract interfaces defining contracts (e.g., `ObjectStoreService`, `QueueClient`)
- **Adapters**: Cloud-provider implementations (AWS, Azure, GCP, Mock)
- **Dependency Inversion**: Applications depend on abstractions, not concrete implementations

### 2. Dual-Plane Separation

- **Control Plane**: Infrastructure provisioning and management (CLI, REST API)
- **Data Plane**: Application runtime operations (accelerators clients)

### 3. Provider Independence

No cloud-specific types in core interfaces. Switch providers via configuration:

```typescript
// Application code remains unchanged
const platform = new LCPlatform({ provider: 'aws' });  // or 'azure', 'mock'
const storage = platform.getObjectStore();
await storage.putObject('bucket', 'key', data);
```

## Quick Start

### Prerequisites

- **Bun 1.0+** (for TypeScript packages)
- **Git** for source control
- Cloud credentials (AWS/Azure) or use mock provider for local development

### Installation

```bash
# Clone the parent repository
git clone https://github.com/stainedhead/lc-platform.git
cd lc-platform

# Navigate to a specific package
cd lc-platform-dev-accelerators

# Install dependencies (using Bun)
bun install

# Run tests
bun test

# Build package
bun run build
```

### Using Dev Accelerators Package

```bash
# Install from GitHub Packages
bun add @stainedhead/lc-platform-dev-accelerators

# Or install locally
cd lc-platform-dev-accelerators
bun link
cd your-project
bun link @stainedhead/lc-platform-dev-accelerators
```

## Development Workflow

### SpecKit Integration

All packages use **SpecKit** for structured, AI-assisted development:

```bash
/speckit.specify    # Create/update feature specification
/speckit.clarify    # Refine underspecified areas
/speckit.plan       # Generate implementation plan
/speckit.tasks      # Create dependency-ordered tasks
/speckit.implement  # Execute implementation
/speckit.analyze    # Verify consistency
```

### Quality Standards

All packages maintain:

- ✅ **Test-Driven Development** - Write tests before implementation
- ✅ **80%+ Code Coverage** - Minimum coverage for public interfaces
- ✅ **Clean Architecture** - No cloud SDK types in core interfaces
- ✅ **Auto-formatting** - Prettier before linting
- ✅ **Zero Critical/High Linting Errors** - ESLint enforcement

### Pre-Commit Workflow (TypeScript Packages)

```bash
bun run format      # 1. Format code first
git add -A          # 2. Stage changes
bun run lint        # 3. Verify linting
bun test            # 4. Run tests
git commit -m "msg" # 5. Commit
git push            # 6. Push
```

## Package Dependencies

```
lc-platform-cli ──────┬──> lc-platform-dev-accelerators
                      │
                      └──> lc-platform-config

lc-platform-rest-api ─┬──> lc-platform-dev-accelerators
                      │
                      └──> lc-platform-config

lc-platform-config ───────> (independent, shared by all)

lc-platform-dev-accelerators ──> (foundation package)
```

## Service Mappings (Dev Accelerators)

| Service | AWS | Azure | Interface |
|---------|-----|-------|-----------|
| Web Hosting | App Runner | Container Apps | `WebHostingService` |
| Function Hosting | Lambda | Function Apps | `FunctionHostingService` |
| Batch Jobs | AWS Batch | Batch Service | `BatchService` |
| Secrets | Secrets Manager | Key Vault | `SecretsService` |
| Configuration | AppConfig | App Configuration | `ConfigurationService` |
| Object Storage | S3 | Blob Storage | `ObjectStoreService` |
| Queues | SQS | Storage Queues | `QueueService` |
| Events | EventBridge | Event Grid | `EventBusService` |
| Notifications | SNS | Notification Hubs | `NotificationService` |
| NoSQL DB | DocumentDB | Cosmos DB | `DocumentStoreService` |
| SQL DB | RDS PostgreSQL | Database for PostgreSQL | `DataStoreService` |
| Authentication | Cognito + Okta | Azure AD B2C | `AuthenticationService` |
| Cache | ElastiCache Redis | Cache for Redis | `CacheService` |
| Container Repo | ECR | Container Registry | `ContainerRepoService` |

## Documentation

### Parent Level (Platform-Wide)
- **[README.md](README.md)** - This file, platform overview
- **[AGENTS.md](AGENTS.md)** - Development rules and product context (single source of truth)
- **[CLAUDE.md](CLAUDE.md)** - Pointer to AGENTS.md for Claude Code
- **[.github/copilot-instructions.md](.github/copilot-instructions.md)** - Pointer to AGENTS.md for GitHub Copilot
- **[documentation/product-summary.md](documentation/product-summary.md)** - High-level product overview
- **[documentation/product-details.md](documentation/product-details.md)** - Detailed specifications
- **[documentation/technical-details.md](documentation/technical-details.md)** - Technical architecture

### Package Level
Each package maintains its own documentation:
- `{package}/README.md` - Package-specific guide
- `{package}/AGENTS.md` - Package-specific development rules
- `{package}/documentation/` - Detailed technical docs

## Current Status

### Completed ✅
- **lc-platform-dev-accelerators**: MVP with AWS and Mock providers
- **Architecture Design**: Hexagonal pattern established
- **Testing Strategy**: Bun test runner, 85%+ coverage achieved
- **Constitution**: Governance structure and principles defined

### In Progress 🚧
- **Parent Repository Setup**: Documentation and structure
- **Package Integration**: Cross-package testing and coordination

### Planned 📋
- **lc-platform-cli**: Control Plane CLI
- **lc-platform-config**: Configuration management package
- **lc-platform-rest-api**: REST API service
- **Azure Provider**: Azure implementations for accelerators
- **GCP Provider**: Google Cloud implementations for accelerators

## Contributing

We welcome contributions! Please follow these guidelines:

1. **Read Documentation**:
   - [AGENTS.md](AGENTS.md) - Development rules and context
   - [Constitution](.specify/memory/constitution.md) - Governance and principles
   - Package-specific `AGENTS.md` files

2. **Development Workflow**:
   - Use SpecKit workflow (`/speckit.*` commands)
   - Follow Test-Driven Development (TDD)
   - Maintain 80%+ code coverage
   - Follow Hexagonal Architecture patterns

3. **Code Quality**:
   - Auto-format before commit (`bun run format`)
   - Zero critical/high linting errors
   - All tests must pass
   - Type checking must pass

4. **Pull Requests**:
   - Reference relevant AGENTS.md sections
   - Include tests for new features
   - Update documentation as needed
   - Follow commit message conventions

## Support and Resources

### Documentation
- [Product Summary](documentation/product-summary.md) - High-level overview
- [Product Details](documentation/product-details.md) - Detailed specifications
- [Technical Details](documentation/technical-details.md) - Architecture deep dive

### Development Tools
- **SpecKit**: Feature development workflow
- **Bun**: TypeScript runtime and testing
- **ESLint + Prettier**: Code quality and formatting
- **TypeDoc**: API documentation generation

### Getting Help
- **Issues**: [GitHub Issues](https://github.com/stainedhead/lc-platform/issues)
- **Discussions**: [GitHub Discussions](https://github.com/stainedhead/lc-platform/discussions)
- **Documentation**: See `documentation/` directory

## License

MIT License - See [LICENSE](LICENSE) file for details.

## Acknowledgments

Built with:
- **Bun** - Fast JavaScript runtime with native TypeScript support
- **SpecKit** - AI-assisted feature development workflow
- **Hexagonal Architecture** - Clean architecture patterns
- **Test-Driven Development** - Quality-first approach

---

**LC Platform** - Cloud-agnostic platform for development teams
