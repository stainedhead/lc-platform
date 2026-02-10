# LC Platform v2 - Feature-Complete Release Specification

**Created:** 2026-02-09
**Version:** 1.0
**Status:** Draft
**Source PRD:** `lc-platform-v2-prd.md`

---

## Executive Summary

Transform LC Platform from proof-of-concept into a feature-complete v1.0.0 release that delivers on its core promise: enabling developers to build cloud solutions using business-domain names without needing cloud expertise. This release integrates the three existing packages (dev-accelerators, processing-lib, and CLI), introduces a declarative manifest system, and provides moniker-based resource access.

**Key Deliverables:**
- Application manifest file (`lcp-manifest.yaml`) as declarative source of truth
- Moniker-based runtime API (`runtime.store('backups')` instead of raw S3 bucket names)
- Fully integrated CLI that delegates through processing-lib (eliminating mock filesystem)
- Shared configuration package (`lc-platform-config`) for manifest handling and type definitions
- Active application context for streamlined CLI workflows
- Comprehensive getting-started documentation for 15-minute onboarding

**Timeline:** 12-14 weeks (Phases 1-5)

---

## Problem Statement

### Current State

LC Platform exists as four separate repositories:
1. **lc-platform** (parent monorepo) - Governance and documentation
2. **lc-platform-dev-accelerators** (MVP Complete) - 14 control plane services + 11 data plane clients with AWS and Mock providers
3. **lc-platform-dev-cli** (Partial MVP) - CLI with core commands but using mock filesystem (`~/.lcp/mock-apps.json`)
4. **lc-platform-processing-lib** (Domain complete) - Clean Architecture domain model with stub adapters (in-memory Maps)

### Pain Points

- **No moniker-based resource access**: Developers must still pass raw cloud resource names (S3 bucket names, SQS queue URLs) to every API call
- **Components operate in isolation**: CLI bypasses processing-lib; processing-lib adapters are stubs; no integration between packages
- **No application manifest**: No declarative file defines what resources an application needs
- **No environment-aware resolution**: Dev and prod resources would collide (no environment parameter in name generation)
- **Incomplete type definitions**: `DependencyConfiguration` union only covers 4 of 14 dependency types
- **Repetitive CLI usage**: Developers must re-specify `--account`, `--team`, `--moniker` on every command
- **Missing shared configuration package**: Configuration scattered across repos with inconsistent schemas

### Desired State

Developers should be able to:
```typescript
// Runtime configured from manifest + environment
const runtime = await LCAppRuntime.fromManifest('./lcp-manifest.yaml');

// Access resources by business name -- no cloud identifiers
const backups = runtime.store('backups');
await backups.put('reports/2026-Q1.pdf', reportData);

const orders = runtime.queue('order-processing');
await orders.send({ orderId: '12345', action: 'process' });
```

CLI users should be able to:
```bash
# Initialize app -- automatically becomes active context
lcp app init --team backend --moniker order-svc

# All subsequent commands use active context -- no flags needed
lcp dependency add backups --type object-store --encryption aes256
lcp dependency list
lcp deploy dependencies --env dev
```

---

## Goals and Non-Goals

### Goals

1. **Developer Abstraction**: Enable `runtime.store('moniker')` access pattern without cloud resource knowledge
2. **Component Integration**: Wire CLI → processing-lib → dev-accelerators in complete chain
3. **Declarative Configuration**: `lcp-manifest.yaml` as single source of truth for application topology
4. **Streamlined Workflows**: Active application context eliminates repetitive flag usage
5. **Enterprise Accessibility**: Getting-started tutorials enable new users in 15 minutes
6. **Agentic Support**: Machine-readable context export for AI agent integration
7. **Production Readiness**: 80%+ test coverage, quality error messages, comprehensive documentation

### Non-Goals

- Azure/GCP provider implementations (v2.0+)
- REST API service (`lc-platform-rest-api`) (v2.0+)
- Multi-tenant support (future)
- GUI/web dashboard (future)
- Breaking changes to existing dev-accelerators API (maintain backward compatibility)

---

## User Requirements

### Functional Requirements

#### FR-A01: Application Manifest File
**Priority:** P0 (Critical)

**Description:**
Introduce `lcp-manifest.yaml` as declarative configuration file defining application infrastructure needs.

**Acceptance Criteria:**
- [ ] `lcp app init` generates manifest skeleton with metadata and provider configuration
- [ ] Supports `${ENV_VAR:-default}` environment variable interpolation
- [ ] `lcp validate` validates manifest against schema with actionable error messages
- [ ] TypeScript types generated from manifest schema
- [ ] Dependency name (e.g., `backups`) serves as moniker key
- [ ] Schema supports all 14 dependency types from `DependencyType` enum

**Example:**
```yaml
apiVersion: lcp/v1
kind: Application
metadata:
  team: backend
  moniker: order-svc
  environment: ${LCP_ENVIRONMENT:-dev}
provider:
  type: ${LCP_PROVIDER:-mock}
  region: ${LCP_REGION:-us-east-1}
dependencies:
  backups:
    type: object-store
    config:
      versioning: true
      encryption: aes256
```

---

#### FR-A02: Runtime Resource Resolver
**Priority:** P0 (Critical)

**Description:**
`ResourceResolver` class maps moniker → cloud resource name at runtime using manifest and name generation.

**Acceptance Criteria:**
- [ ] Resolves all 14 `DependencyType` values
- [ ] Throws `ResourceNotFoundError` for undeclared monikers with list of available monikers
- [ ] Injectable context loader (ManifestContextLoader, StorageContextLoader, InMemoryContextLoader)
- [ ] Works with mock provider (no cloud credentials needed)
- [ ] Uses existing `generateDependencyResourceName()` from nameGenerator.ts
- [ ] Passes current environment to name generation

**Example:**
```typescript
const resolver = new ResourceResolver(new ManifestContextLoader());
await resolver.initialize({ manifestPath: './lcp-manifest.yaml' });
const resolved = resolver.resolve('backups'); // Returns full resource name
```

---

#### FR-A03: Scoped Client Accessors on LCAppRuntime
**Priority:** P0 (Critical)

**Description:**
Add moniker-based accessor methods to `LCAppRuntime` returning "scoped clients" that pre-bind resource names.

**Acceptance Criteria:**
- [ ] All 11 data plane client types have corresponding `Scoped*Client` wrappers
- [ ] Scoped clients have identical signatures minus resource-name parameter
- [ ] `runtime.store('nonexistent')` throws `ResourceNotFoundError` with helpful message
- [ ] Scoped clients are cached (same moniker returns same instance)
- [ ] Full TypeScript type inference and autocomplete
- [ ] Existing `get*Client()` methods remain unchanged (backward compatibility)
- [ ] `LCAppRuntime.fromManifest()` and `fromEnvironment()` factory methods
- [ ] `initialize()` method loads manifest and sets up resolver

**Example:**
```typescript
const runtime = await LCAppRuntime.fromManifest('./lcp-manifest.yaml');
const backups = runtime.store('backups'); // ScopedObjectClient
await backups.put('file.txt', data); // No bucket name needed
```

---

#### FR-B01: Wire CLI Through Processing-Lib
**Priority:** P0 (Critical)

**Description:**
Replace CLI mock implementations with calls to processing-lib's `ApplicationConfigurator` and `VersionConfigurator`.

**Acceptance Criteria:**
- [ ] `~/.lcp/mock-apps.json` approach removed entirely
- [ ] `app init` → `ApplicationConfigurator.init()`
- [ ] `app read` → `ApplicationConfigurator.read()`
- [ ] `app update` → `ApplicationConfigurator.update()`
- [ ] `app validate` → `ApplicationConfigurator.validate()`
- [ ] `version add` → `VersionConfigurator.init()`
- [ ] `version read` → `VersionConfigurator.read()`
- [ ] Error messages from processing-lib mapped to human-readable CLI messages
- [ ] `--dry-run` still works (intercepts before calling configurator)

---

#### FR-B02: Implement Processing-Lib Adapters
**Priority:** P0 (Critical)

**Description:**
Replace stub adapters in processing-lib with real implementations using dev-accelerators.

**Acceptance Criteria:**
- [ ] `AcceleratorStorageAdapter` uses `ObjectClient` from dev-accelerators
- [ ] `AcceleratorPolicyAdapter` uses `policySerializer` from dev-accelerators
- [ ] `AcceleratorDeploymentAdapter` uses `LCPlatform` services from dev-accelerators
- [ ] All adapters work with mock provider (no cloud credentials for testing)
- [ ] `AcceleratorStorageAdapterFactory` creates adapters from context
- [ ] Integration tests verify CLI → processing-lib → dev-accelerators chain

---

#### FR-B03: Dependency Management CLI Commands
**Priority:** P0 (Critical)

**Description:**
New `lcp dependency` command group for managing resource dependencies.

**Acceptance Criteria:**
- [ ] `lcp dependency add <moniker> --type <type> [options]` adds dependency to manifest
- [ ] `lcp dependency list` shows all dependencies with status
- [ ] `lcp dependency show <moniker> --json` displays dependency details
- [ ] `lcp dependency remove <moniker>` removes dependency from manifest
- [ ] `lcp dependency validate` validates all dependencies
- [ ] All commands support `--json` and `--dry-run` flags
- [ ] Type-specific flags (e.g., `--fifo` for queues, `--encryption` for storage)
- [ ] Commands update `lcp-manifest.yaml` programmatically

**Example:**
```bash
lcp dependency add backups --type object-store --encryption aes256 --versioning
lcp dependency add order-queue --type queue --fifo --visibility-timeout 30
lcp dependency list
```

---

#### FR-C01: Shared Configuration Package (lc-platform-config)
**Priority:** P0 (Critical)

**Description:**
Create `lc-platform-config` package providing manifest schema, YAML parsing, and shared types.

**Acceptance Criteria:**
- [ ] Package exists as git submodule in parent repo
- [ ] Manifest schema definition with validation
- [ ] YAML parsing with `${ENV_VAR:-default}` interpolation
- [ ] Configuration resolution with documented precedence (CLI args > env vars > project config > global config)
- [ ] Shared types used by dev-accelerators, processing-lib, and CLI
- [ ] 80%+ test coverage
- [ ] Zero runtime dependencies on other LC Platform packages

---

#### FR-C02: Environment-Aware Resource Resolution
**Priority:** P0 (Critical)

**Description:**
Add optional `environment` parameter to `generateDependencyResourceName()` to prevent dev/prod collisions.

**Acceptance Criteria:**
- [ ] `environment` parameter added to `generateDependencyResourceName()`
- [ ] Backward compatible (omitting environment preserves existing behavior)
- [ ] `lcp deploy dependencies --env staging` creates resources with environment qualifier
- [ ] `ResourceResolver` passes current environment from manifest to name generation
- [ ] Resource naming pattern: `lcp-{account}-{team}-{moniker}-{serviceType}-{name}[-{environment}]`

---

#### FR-C03: Complete DependencyConfiguration Union Type
**Priority:** P0 (Critical)

**Description:**
Add configuration interfaces for all 14 dependency types in `DependencyConfiguration` union.

**Acceptance Criteria:**
- [ ] All 14 dependency types have corresponding configuration interfaces
- [ ] Each interface includes type-specific configuration options
- [ ] Type definitions exported from dev-accelerators
- [ ] Used by manifest schema in lc-platform-config

**File:** `lc-platform-dev-accelerators/src/core/types/dependency.ts`

---

#### FR-D01: Error Message Quality
**Priority:** P0 (Critical)

**Description:**
Error enrichment layer mapping processing-lib errors to actionable CLI messages.

**Acceptance Criteria:**
- [ ] All CLI errors include message, suggestion, and error code
- [ ] `--debug` shows stack trace
- [ ] `--json` outputs machine-parseable errors
- [ ] Every processing-lib error enum value mapped to CLI error message
- [ ] Suggestions provided for common errors (e.g., "To update, run: lcp app update")

**Example:**
```
Error: Application 'order-svc' already exists for team 'backend'.

Suggestions:
  - To update it, run: lcp app update
  - To see its current state, run: lcp app read

Use --debug for full error details.
```

---

#### FR-D04: Active Application Context
**Priority:** P0 (Critical)

**Description:**
Introduce active application context that persists across commands to eliminate repetitive flag usage.

**Acceptance Criteria:**
- [ ] `lcp app init` automatically sets new app as active context in `.lcp/config.json`
- [ ] `lcp app use <moniker>` switches active app (resolves by moniker)
- [ ] `lcp app use <team>/<moniker>` for disambiguation when multiple teams have same moniker
- [ ] Active app stored in project-local `.lcp/config.json` (not global)
- [ ] `-a <moniker>` global flag overrides active context for single command
- [ ] `lcp context read` shows active app
- [ ] All commands inherit active context when flags not provided
- [ ] Clear error message when no active app and no flags: "No active application. Run 'lcp app use <moniker>' or provide --account, --team, --moniker."
- [ ] `lcp app list` shows all known apps with active one highlighted
- [ ] `lcp app current` shows current active app

**Example:**
```bash
# Init sets active context
lcp app init --team backend --moniker order-svc
# Active app: backend/order-svc

# Subsequent commands just work
lcp dependency add backups --type object-store

# Switch context
lcp app use user-svc

# One-off without switching
lcp -a order-svc dependency list
```

---

#### FR-E01: Manifest-Driven Agent Context Export
**Priority:** P0 (Critical)

**Description:**
`lcp context export --json` produces machine-readable topology with TypeScript access patterns.

**Acceptance Criteria:**
- [ ] Export includes TypeScript access pattern for each dependency
- [ ] Export includes code examples per dependency type
- [ ] Dependency names are business-meaningful (validated)
- [ ] JSON format suitable for AI agent consumption
- [ ] Export includes manifest metadata (team, moniker, environment)

**Example:**
```json
{
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

#### FR-F02: Getting-Started Documentation
**Priority:** P0 (Critical)

**Description:**
Progressive tutorials in `documentation/getting-started/` enabling new users in 15 minutes.

**Acceptance Criteria:**
- [ ] `01-your-first-app.md` - Zero to working app in 15 minutes
- [ ] `02-adding-dependencies.md` - Declaring storage, queues, databases
- [ ] `03-writing-application-code.md` - Using moniker-based APIs
- [ ] `04-deploying-to-cloud.md` - From mock to real cloud
- [ ] `05-working-with-ai-agents.md` - Claude/Copilot integration
- [ ] Tutorial 01 completable in under 15 minutes
- [ ] All use moniker-based APIs (not raw bucket names)
- [ ] No cloud credentials needed for tutorials 01-03 (mock provider)

---

### Non-Functional Requirements

#### NFR-001: Test Coverage
**Category:** Reliability

**Description:**
Maintain high test coverage across all packages to ensure reliability and prevent regressions.

**Metrics:**
- Overall coverage: ≥80%
- Critical path coverage: ≥90%
- All public interfaces tested

---

#### NFR-002: Backward Compatibility
**Category:** Reliability

**Description:**
Preserve existing dev-accelerators API to avoid breaking existing users.

**Metrics:**
- All existing `get*Client()` methods unchanged
- All existing tests pass
- No breaking changes to public interfaces

---

#### NFR-003: Mock Provider Support
**Category:** Usability

**Description:**
Complete local development workflow with mock provider (no cloud credentials).

**Metrics:**
- All CLI commands work with mock provider
- All runtime operations work with mock provider
- Tutorials 01-03 use only mock provider

---

## System Architecture

### Affected Layers

- [x] Domain Layer (processing-lib entities, interfaces)
- [x] Use Case Layer (processing-lib configurators)
- [x] Infrastructure Layer (processing-lib adapters, dev-accelerators clients)
- [x] Adapter Layer (CLI commands, scoped clients)

### New Components

- **lc-platform-config** package: Manifest schema, YAML parsing, shared types
- **ResourceResolver**: Maps monikers to cloud resource names
- **IContextLoader** + implementations: Loads application context (manifest, storage, in-memory)
- **Scoped*Client** (11 types): Pre-bound client wrappers
- **AcceleratorStorageAdapterFactory**: Creates processing-lib adapters from context
- **CLI dependency commands**: `add`, `remove`, `list`, `show`, `validate`
- **CLI app context commands**: `use`, `list`, `current`
- **Error enrichment layer**: Maps processing-lib errors to CLI messages

### Modified Components

- **LCAppRuntime**: Add moniker-based accessors, `fromManifest()`, `initialize()`
- **nameGenerator.ts**: Add optional `environment` parameter
- **dependency.ts**: Complete `DependencyConfiguration` union type (14 types)
- **CLI commands**: Wire through processing-lib (app init/read/update, version add/read)
- **Processing-lib adapters**: Replace stubs with real implementations
- **CLI options**: Add `-a`/`--app` global flag for active context override

---

## Scope of Changes

### Files to Create

**lc-platform-config (new package):**
- `src/manifest/schema.ts` - Manifest schema definition
- `src/manifest/parser.ts` - YAML parsing with interpolation
- `src/resolution/resolver.ts` - Config resolution logic
- `src/validation/naming.ts` - Domain-driven naming validation
- `src/validation/dependency.ts` - Dependency validation

**lc-platform-dev-accelerators:**
- `src/resolution/ResourceResolver.ts` - Core resolver
- `src/resolution/IContextLoader.ts` - Port interface
- `src/resolution/ManifestContextLoader.ts` - Loads from YAML
- `src/resolution/StorageContextLoader.ts` - Loads from config bucket
- `src/resolution/InMemoryContextLoader.ts` - For testing
- `src/core/types/resolution.ts` - ApplicationContext, ResolvedDependency
- `src/core/clients/scoped/Scoped*Client.ts` (11 files) - One per data plane client

**lc-platform-processing-lib:**
- `src/adapters/storage/AcceleratorStorageAdapterFactory.ts` - Factory from context

**lc-platform-dev-cli:**
- `src/cli/commands/dependency/` - add, remove, list, show, validate
- `src/cli/commands/app/use.ts`, `list.ts`, `current.ts` - Active context commands
- `src/cli/shared/adapter-factory.ts` - Creates processing-lib adapters
- `src/cli/shared/error-enrichment.ts` - Error message mapping

**Documentation:**
- `documentation/getting-started/01-your-first-app.md`
- `documentation/getting-started/02-adding-dependencies.md`
- `documentation/getting-started/03-writing-application-code.md`
- `documentation/getting-started/04-deploying-to-cloud.md`
- `documentation/getting-started/05-working-with-ai-agents.md`

### Files to Modify

**lc-platform-dev-accelerators:**
- `src/LCAppRuntime.ts` - Add moniker-based accessors
- `src/utils/nameGenerator.ts` - Add environment parameter
- `src/core/types/dependency.ts` - Complete DependencyConfiguration union
- `src/index.ts` - Export new types and classes

**lc-platform-processing-lib:**
- `src/adapters/storage/AcceleratorStorageAdapter.ts` - Real implementation
- `src/adapters/policy/AcceleratorPolicyAdapter.ts` - Real implementation
- `src/adapters/deployment/AcceleratorDeploymentAdapter.ts` - Real implementation

**lc-platform-dev-cli:**
- `src/cli/commands/app/init.ts` - Wire to processing-lib, auto-set active app
- `src/cli/commands/app/read.ts` - Wire to processing-lib
- `src/cli/commands/app/update.ts` - Wire to processing-lib
- `src/cli/commands/app/validate.ts` - Wire to processing-lib
- `src/cli/commands/version/add.ts` - Wire to processing-lib
- `src/cli/commands/version/read.ts` - Wire to processing-lib
- `src/cli/commands/version/deploy.ts` - Wire to processing-lib
- `src/cli/commands/context/read.ts` - Add `--export`, show active app
- `src/cli/options.ts` - Add `-a`/`--app` global flag
- `src/config/types.ts` - Add active app fields to CliContext

### Dependencies

**New External Dependencies:**
- None (all use existing dependencies)

**Internal Package Dependencies:**
```
lc-platform-config (NEW)
  ↑ used by
  ├── lc-platform-dev-accelerators
  ├── lc-platform-processing-lib
  └── lc-platform-dev-cli

lc-platform-dev-accelerators (ENHANCED)
  ↑ used by
  └── lc-platform-processing-lib (adapters)

lc-platform-processing-lib (ENHANCED)
  ↑ used by
  └── lc-platform-dev-cli (commands)
```

---

## Breaking Changes

### API Changes

**None - All changes are additive:**
- Existing `get*Client()` methods on `LCAppRuntime` remain unchanged
- New moniker-based accessors added (non-breaking)
- `generateDependencyResourceName()` environment parameter is optional (backward compatible)

### Configuration Changes

- **New file**: `lcp-manifest.yaml` (required for new features but not breaking)
- **New file**: `.lcp/config.json` (project-local, stores active app context)
- **Removed**: `~/.lcp/mock-apps.json` (internal mock implementation)

**Migration Path:**
- Existing users without manifest continue to use explicit flags
- New users start with manifest-based workflow
- Documentation guides migration to manifest

### Database Schema Changes

None - Processing-lib adapters use cloud storage, not local database

---

## Success Criteria

### Acceptance Criteria

- [ ] `lcp-manifest.yaml` is the declarative source of truth for application topology
- [ ] `runtime.store('moniker')` resolves and works end-to-end (mock + AWS)
- [ ] CLI commands delegate through processing-lib (zero mock filesystem usage)
- [ ] `lcp dependency add/list/remove/show/validate` fully functional
- [ ] `lc-platform-config` package provides shared types and manifest handling
- [ ] Environment-aware resource names work (`--env` flag)
- [ ] Active application context works (`lcp app use`, `-a` flag)
- [ ] Error messages include suggestions and error codes
- [ ] `lcp context export --json` provides machine-readable topology
- [ ] Getting-started docs enable new user app in 15 minutes
- [ ] Mock provider works for complete local dev (no cloud credentials)

### Quality Gates

- [ ] All tests pass (unit, integration, E2E)
- [ ] Code coverage ≥80% across all packages
- [ ] All existing tests pass (backward compatibility)
- [ ] Documentation complete (API docs + tutorials)
- [ ] Linter passes (zero critical/high violations)
- [ ] Type checking passes (TypeScript strict mode)
- [ ] Performance benchmarks met (E2E workflow < 30 seconds)

### User Validation

- [ ] New developer completes tutorial 01 in under 15 minutes
- [ ] AI agent (Claude) generates correct code from `lcp context export` output
- [ ] CLI user successfully deploys app with dependencies using mock provider
- [ ] Application developer writes code using moniker-based APIs without cloud knowledge

---

## Risks and Mitigation

### Risk 1: Breaking Changes to dev-accelerators API
**Likelihood:** Low
**Impact:** High

**Mitigation:**
- All existing methods preserved
- New methods are additive only
- Comprehensive regression testing
- Clear deprecation policy for any future changes

---

### Risk 2: Manifest File Too Complex for Enterprise Users
**Likelihood:** Medium
**Impact:** Medium

**Mitigation:**
- Sensible defaults reduce required configuration
- `lcp dependency add` edits manifest programmatically (users don't hand-edit)
- Interactive prompts guide configuration
- Clear validation messages

---

### Risk 3: Interface Mismatches Between Processing-Lib and Dev-Accelerators
**Likelihood:** Medium
**Impact:** High

**Mitigation:**
- Implement storage adapter first as proof
- Ports can be adjusted pre-1.0 if needed
- Integration tests verify compatibility early
- Phase 3 focuses on adapter integration before CLI changes

---

### Risk 4: Circular Dependencies Between Packages
**Likelihood:** Low
**Impact:** High

**Mitigation:**
- lc-platform-config is types/validation only (zero runtime deps)
- Clear dependency hierarchy: config → accelerators → processing-lib → CLI
- Strict import linting

---

### Risk 5: Environment Variable Interpolation Leaking Secrets
**Likelihood:** Low
**Impact:** Medium

**Mitigation:**
- Log variable names, never values
- Secrets use `secrets` dependency type (never inline in manifest)
- Validation warns against sensitive data in manifest

---

## Timeline and Milestones

### Phase 1: Foundation (Weeks 1-3)
**Deliverables:**
- lc-platform-config package created
- lcp-manifest.yaml schema defined
- DependencyConfiguration union type completed
- Environment parameter added to nameGenerator
- ResourceResolver + context loaders

**Estimated Duration:** 60-80 hours

---

### Phase 2: Scoped Clients & Resolution (Weeks 4-6)
**Deliverables:**
- All 11 Scoped*Client wrappers
- Moniker-based accessors on LCAppRuntime
- fromManifest() and fromEnvironment() factories
- ManifestContextLoader implementation
- Full test coverage for resolution layer

**Estimated Duration:** 80-100 hours

---

### Phase 3: Adapter Integration (Weeks 7-8)
**Deliverables:**
- Real AcceleratorStorageAdapter
- Real AcceleratorPolicyAdapter
- Real AcceleratorDeploymentAdapter
- AcceleratorStorageAdapterFactory
- Integration tests for processing-lib → dev-accelerators

**Estimated Duration:** 40-60 hours

---

### Phase 4: CLI Integration (Weeks 9-11)
**Deliverables:**
- CLI commands wired through processing-lib
- Mock filesystem storage removed
- lcp dependency command group
- Error enrichment layer
- lcp context export --json
- Active application context (app use, -a flag)

**Estimated Duration:** 80-100 hours

---

### Phase 5: Polish & Documentation (Weeks 12-14)
**Deliverables:**
- Getting-started tutorials (5 documents)
- lcp codegen command (if time permits)
- Interactive dependency setup (if time permits)
- Domain-driven naming validation (if time permits)
- Updated parent README

**Estimated Duration:** 60-80 hours

---

**Total Estimated Duration:** 320-420 hours (12-14 weeks full-time)

---

## References

- **Source PRD:** `lc-platform-v2-prd.md`
- **Related Specs:** None (this is the first comprehensive spec)
- **External Documentation:**
  - [Clean Architecture Principles](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
  - [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)
  - [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
