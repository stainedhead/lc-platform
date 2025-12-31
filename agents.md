# AGENTS.md - LC Platform Parent Repository

This file provides guidance to Claude Code (claude.ai/code) when working across all projects in the LC Platform ecosystem.

## Platform Overview

**LC Platform** is a cloud-agnostic platform that enables development teams to create and manage cloud solutions without requiring deep cloud expertise. The platform provides:

- **Cloud Portability**: Switch between AWS, Azure, and GCP without application code changes
- **Clean Architecture**: Hexagonal Engineering patterns with provider abstraction
- **Agentic Tooling Support**: First-class support for AI-powered development workflows
- **Unified Configuration**: Centralized platform configuration management

## Product Architecture

The LC Platform consists of independent packages that work together as a cohesive system:

### Package Structure

```
lc-platform/                          # Parent directory (this repo)
├── AGENTS.md                        # Cross-repo agentic configuration (this file)
├── CLAUDE.md                        # Pointer to AGENTS.md
│
├── lc-platform-dev-accelerators/    # Cloud-agnostic service wrappers
│   └── AGENTS.md                    # Package-specific agent rules
│
├── lc-platform-cli/                 # Control Plane CLI (future)
│   └── AGENTS.md                    # Package-specific agent rules
│
├── lc-platform-config/              # Platform configuration package (future)
│   └── AGENTS.md                    # Package-specific agent rules
│
└── lc-platform-rest-api/            # REST API service (future)
    └── AGENTS.md                    # Package-specific agent rules
```

### Platform Components

#### 1. **lc-platform-dev-accelerators** (MVP Complete)
- **Purpose**: Provider-agnostic service wrappers using Hexagonal Architecture
- **Status**: MVP Complete with AWS and Mock providers
- **Components**:
  - 14 Control Plane services (infrastructure management)
  - 11 Data Plane clients (application runtime)
- **Runtime**: Bun 1.0+
- **Package**: `@stainedhead/lc-platform-dev-accelerators`

#### 2. **lc-platform-cli** (Planned)
- **Purpose**: Control Plane command-line interface
- **Capabilities**:
  - Infrastructure provisioning and management
  - Application deployment and lifecycle
  - Configuration management
  - Multi-environment support
- **Integration**: Uses dev-accelerators package internally

#### 3. **lc-platform-config** (Planned)
- **Purpose**: Centralized platform configuration management
- **Features**:
  - Environment configuration
  - Service dependency declarations
  - Validation and schema management
  - Configuration versioning
- **Integration**: Shared by CLI and accelerators packages

#### 4. **lc-platform-rest-api** (Future)
- **Purpose**: REST API for platform operations
- **Capabilities**:
  - Programmatic access to Control Plane
  - Deployment management APIs
  - Configuration management APIs
  - Multi-tenant support
- **Integration**: Alternative interface to CLI

## Architectural Principles

### 1. Hexagonal Architecture (Ports and Adapters)

All packages follow Clean Architecture principles:
- **Core Domain**: Cloud-agnostic business logic and interfaces
- **Ports**: Abstract interfaces defining contracts
- **Adapters**: Cloud-provider-specific implementations (AWS, Azure, GCP, Mock)
- **Dependency Inversion**: Applications depend on abstractions, not implementations

### 2. Provider Independence

- No cloud-specific types in core interfaces
- Configuration-driven provider selection
- Seamless switching between cloud providers
- Mock provider for testing and local development

### 3. Dual-Plane Separation

- **Control Plane**: Infrastructure provisioning and management (CLI, REST API)
- **Data Plane**: Application runtime operations (accelerators clients)
- Clear separation of concerns between planes

### 4. Agentic Tooling First

- SpecKit workflow integration for feature development
- AI-assisted code generation and refactoring
- Cross-repo context awareness through shared AGENTS.md
- Automated testing and validation

## Cross-Repository Guidelines

### Multi-Package Development

When working across multiple packages:

1. **Context Awareness**: Each package has its own `AGENTS.md` for package-specific rules
2. **Shared Principles**: This parent `AGENTS.md` provides overarching guidelines
3. **Dependency Management**: Changes to accelerators may impact CLI and config packages
4. **Version Coordination**: Maintain compatible versions across dependent packages

### Package Dependencies

```
lc-platform-cli         → depends on → lc-platform-dev-accelerators
                        → depends on → lc-platform-config

lc-platform-rest-api    → depends on → lc-platform-dev-accelerators
                        → depends on → lc-platform-config

lc-platform-config      → independent (shared by all)

lc-platform-dev-accelerators → independent (foundation)
```

### Making Cross-Repo Changes

When changes affect multiple packages:

1. **Start with Foundation**: Update `lc-platform-dev-accelerators` first
2. **Update Configuration**: Modify `lc-platform-config` if schemas change
3. **Update Consumers**: Then update CLI and REST API packages
4. **Coordinate Versions**: Bump version numbers consistently
5. **Integration Testing**: Test cross-package integration

## Development Workflow

### SpecKit Integration

All packages use **SpecKit** for structured development:

- `/speckit.specify` - Create/update feature specifications
- `/speckit.plan` - Generate implementation plans
- `/speckit.tasks` - Create dependency-ordered tasks
- `/speckit.implement` - Execute implementations
- `/speckit.clarify` - Refine underspecified areas
- `/speckit.analyze` - Verify consistency

### Quality Standards (Platform-Wide)

All packages must maintain:

- **Test-Driven Development**: Write tests before implementation
- **80%+ Code Coverage**: Minimum coverage for public interfaces
- **Clean Architecture**: No cloud SDK types in core interfaces
- **Auto-formatting**: Prettier before linting (for TypeScript packages)
- **Linting**: ESLint with TypeScript rules (zero critical/high violations)

### Runtime Standards

- **TypeScript Packages**: Bun 1.0+ runtime (not Node.js)
- **Other Languages**: To be determined per package
- **Testing**: Bun test runner for TypeScript packages

## Documentation Structure

### Parent Level (This Directory)
- **`README.md`**: Platform overview and getting started
- **`AGENTS.md`**: Cross-repo development guidelines (this file)
- **`CLAUDE.md`**: Pointer to AGENTS.md

### Package Level
Each package maintains:
- **`README.md`**: Package-specific documentation
- **`AGENTS.md`**: Package-specific development rules
- **`documentation/`**: Detailed technical documentation

### Documentation Hierarchy

```
Parent AGENTS.md          → Platform-wide guidelines and principles
    ↓
Package AGENTS.md         → Package-specific implementation rules
    ↓
Package documentation/    → Detailed technical specifications
```

## Working with This Repository

### Adding a New Package

1. Create package directory: `lc-platform-{name}/`
2. Initialize with appropriate runtime (Bun for TypeScript)
3. Create package-specific `AGENTS.md` following template
4. Document dependencies in parent `AGENTS.md`
5. Update platform architecture diagrams
6. Add integration tests across packages

### Cross-Package Changes

1. **Read Parent AGENTS.md**: Understand platform-wide context
2. **Read Package AGENTS.md**: Understand package-specific rules
3. **Plan Impact**: Identify all affected packages
4. **Coordinate Changes**: Update packages in dependency order
5. **Update Documentation**: Both parent and package-level docs
6. **Integration Testing**: Verify cross-package functionality

### Package-Specific Rules

When working in a specific package directory:

1. **Primary Reference**: Use that package's `AGENTS.md`
2. **Platform Context**: Reference parent `AGENTS.md` for overall architecture
3. **Consistency**: Follow platform-wide standards from parent `AGENTS.md`
4. **Specificity**: Follow package-specific patterns from package `AGENTS.md`

## Key Design Decisions

### Why Separate Packages?

- **Independent Versioning**: Each package evolves at its own pace
- **Clear Boundaries**: Well-defined responsibilities and interfaces
- **Optional Adoption**: Users can choose CLI or REST API
- **Maintainability**: Smaller, focused codebases

### Why Parent AGENTS.md?

- **Shared Context**: AI agents understand cross-package relationships
- **Faster Development**: Changes propagate with full context
- **Consistency**: Platform-wide standards in one place
- **Coordination**: Cross-package changes are better coordinated

### Why Hexagonal Architecture?

- **Cloud Portability**: Switch providers without code changes
- **Testability**: Mock providers for fast, local testing
- **Vendor Independence**: Avoid cloud provider lock-in
- **Future-Proof**: Easy to add new cloud providers

## Current Status

### Completed
- ✅ **lc-platform-dev-accelerators**: MVP with AWS and Mock providers
- ✅ **Architecture Design**: Hexagonal pattern established
- ✅ **Testing Strategy**: Bun test runner, 85%+ coverage achieved

### In Progress
- 🚧 **Parent Repository Setup**: Documentation and structure

### Planned
- 📋 **lc-platform-cli**: Control Plane CLI
- 📋 **lc-platform-config**: Configuration management package
- 📋 **lc-platform-rest-api**: REST API service (future)
- 📋 **Azure Provider**: Azure implementations for accelerators
- 📋 **GCP Provider**: Google Cloud implementations for accelerators

## Getting Started

### For New Developers

1. **Read this file**: Understand platform architecture
2. **Choose a package**: Identify which package you'll work on
3. **Read package AGENTS.md**: Understand package-specific rules
4. **Review documentation**: Read package README and docs/
5. **Set up environment**: Follow package-specific setup instructions

### For AI Agents

1. **Load parent context**: Read this `AGENTS.md` for platform overview
2. **Load package context**: Read package-specific `AGENTS.md`
3. **Understand dependencies**: Review package dependency graph
4. **Coordinate changes**: Consider impact across packages
5. **Maintain consistency**: Follow platform-wide standards

## Support and Resources

### Documentation
- Parent `README.md`: Platform overview
- Package `README.md` files: Package-specific guides
- `documentation/` directories: Detailed technical docs

### Development Tools
- **SpecKit**: Feature development workflow
- **Bun**: TypeScript runtime and testing (for TS packages)
- **ESLint + Prettier**: Code quality and formatting
- **TypeDoc**: API documentation generation (for TS packages)

## Important Notes

### Making Changes to This File

**This is the parent AGENTS.md** - changes here affect platform-wide understanding:

1. **Coordinate Updates**: Discuss architectural changes with team
2. **Update Package Docs**: Cascade changes to package-level AGENTS.md files
3. **Version Control**: Document significant changes
4. **Communicate**: Notify all package maintainers of updates

### Package-Specific vs Platform-Wide

- **Platform-Wide** (this file): Architecture, principles, cross-package coordination
- **Package-Specific** (package AGENTS.md): Implementation details, package rules, testing strategies

When in doubt, package-specific rules take precedence within that package's directory.
