# LC Platform v2 - Implementation Plan

**Created:** 2026-02-09
**Version:** 1.0
**Status:** Planning
**Estimated Duration:** 320-420 hours (12-14 weeks)

---

## Table of Contents

1. [Development Approach](#development-approach)
2. [Phase Breakdown](#phase-breakdown)
3. [Critical Path](#critical-path)
4. [Dependency Graph](#dependency-graph)
5. [Testing Strategy](#testing-strategy)
6. [Success Metrics](#success-metrics)

---

## Development Approach

### Methodology

**TDD + Bottom-Up Clean Architecture**

```
Red → Green → Refactor (for each component)
  ↓
Domain → Infrastructure → Use Case → Adapter
  ↓
Integration → Examples → Documentation
```

**Why This Approach?**
- **TDD ensures quality**: Write tests before implementation catches bugs early
- **Bottom-up builds foundation**: Start with types and interfaces, build up to user-facing features
- **Clean Architecture**: Clear separation of concerns, testable layers
- **Incremental delivery**: Each phase produces working, tested code

**Key Principles:**
1. **Test-first development**: Red-green-refactor for all components
2. **Interface-driven**: Define ports before implementations
3. **Mock provider for testing**: No cloud credentials needed during development
4. **Backward compatibility**: Preserve all existing APIs

**Incremental Milestones:**
- Phase 1: Foundation (types, schema, resolver) - testable, no UI
- Phase 2: Scoped clients - usable by developers, no CLI
- Phase 3: Adapter integration - processing-lib works end-to-end
- Phase 4: CLI integration - full user experience
- Phase 5: Documentation and polish - production-ready

---

## Phase Breakdown

### Phase 1: Foundation (60-80 hours)

**Goal:** Create shared configuration package and resource resolution layer

**Deliverables:**
- lc-platform-config package (manifest schema, YAML parsing, validation)
- lcp-manifest.yaml schema definition with JSON Schema validation
- Complete DependencyConfiguration union type (all 14 types)
- Environment parameter added to nameGenerator.ts
- ResourceResolver + IContextLoader + InMemoryContextLoader
- Full test coverage (≥90%)

**Tasks:**
1. Create lc-platform-config package structure (4-6h)
   - Initialize package with TypeScript, tests
   - Set up exports and documentation

2. Define manifest schema types (8-10h)
   - ManifestData, ManifestMetadata, ProviderDeclaration
   - DependencyDeclaration and all 14 config interfaces
   - Validation rules and constraints

3. Implement YAML parser with interpolation (8-10h)
   - Parse YAML to ManifestData
   - ${ENV_VAR:-default} interpolation
   - Error handling with clear messages

4. Complete DependencyConfiguration union (6-8h)
   - Add 10 missing configuration interfaces
   - Document type-specific options
   - Validation for each type

5. Add environment to nameGenerator (4-6h)
   - Update generateDependencyResourceName() signature
   - Implement naming pattern with optional environment
   - Update all tests

6. Create ResourceResolver (12-16h)
   - Implement ResourceResolver class
   - Define IContextLoader interface
   - Implement ManifestContextLoader
   - Implement InMemoryContextLoader (for testing)
   - ApplicationContext and ResolvedDependency types

7. Test coverage (8-12h)
   - Unit tests for all components
   - Integration tests for manifest parsing + resolution
   - Edge case coverage (invalid manifests, missing env vars, etc.)

**Dependencies:** None (foundation phase)

**Acceptance Criteria:**
- [ ] lc-platform-config package builds and exports types
- [ ] Manifest parsing handles all valid YAML formats
- [ ] Environment variable interpolation works with defaults
- [ ] ResourceResolver resolves all 14 dependency types
- [ ] InMemoryContextLoader works for testing
- [ ] All tests pass with ≥90% coverage
- [ ] Documentation complete (README, API docs)

**Quality Gates:**
- [ ] TypeScript strict mode passes
- [ ] All tests passing
- [ ] Code coverage ≥90%
- [ ] Linter passes (zero critical violations)

---

### Phase 2: Scoped Clients & Resolution (80-100 hours)

**Goal:** Enable moniker-based runtime API for developers

**Deliverables:**
- 11 Scoped*Client wrapper classes
- Moniker-based accessor methods on LCAppRuntime
- fromManifest() and fromEnvironment() static factories
- ManifestContextLoader implementation
- Full test coverage for resolution layer and scoped clients

**Tasks:**
1. Create scoped client wrappers (40-50h)
   - ScopedObjectClient (6-8h)
   - ScopedQueueClient (6-8h)
   - ScopedSecretsClient (4-6h)
   - ScopedConfigClient (4-6h)
   - ScopedEventPublisher (4-6h)
   - ScopedNotificationClient (4-6h)
   - ScopedDocumentClient (4-6h)
   - ScopedDataClient (4-6h)
   - ScopedAuthClient (4-6h)
   - ScopedCacheClient (4-6h)
   - ScopedContainerRepoClient (4-6h)

2. Add moniker-based accessors to LCAppRuntime (12-16h)
   - store(), queue(), secrets(), config(), etc. (11 methods)
   - Scoped client caching by moniker
   - Error handling (ResourceNotFoundError)
   - Type-safe resolution (TypeScript generics)

3. Add factory methods to LCAppRuntime (8-12h)
   - fromManifest(path, overrides?) static method
   - fromEnvironment() static method
   - initialize() method for resolver setup

4. Implement ManifestContextLoader (8-10h)
   - Loads lcp-manifest.yaml
   - Resolves configuration with precedence
   - Returns ApplicationContext

5. Test coverage (12-16h)
   - Unit tests for all scoped clients
   - Integration tests for runtime + resolver
   - Test manifest loading and resolution
   - Test error cases (missing monikers, type mismatches)

**Dependencies:** Phase 1 complete

**Acceptance Criteria:**
- [ ] All 11 scoped clients implemented with consistent API
- [ ] LCAppRuntime.fromManifest() loads manifest and initializes resolver
- [ ] runtime.store('moniker') returns ScopedObjectClient
- [ ] Scoped clients work with mock provider (no cloud credentials)
- [ ] Helpful error messages for missing/mismatched monikers
- [ ] Existing get*Client() methods unchanged (backward compatibility)
- [ ] All tests pass with ≥90% coverage

**Quality Gates:**
- [ ] All tests passing
- [ ] Code coverage ≥90%
- [ ] TypeScript strict mode passes
- [ ] Backward compatibility verified (existing tests pass)

---

### Phase 3: Adapter Integration (40-60 hours)

**Goal:** Wire processing-lib adapters to dev-accelerators

**Deliverables:**
- Real AcceleratorStorageAdapter using ObjectClient
- Real AcceleratorPolicyAdapter using policySerializer
- Real AcceleratorDeploymentAdapter using LCPlatform services
- AcceleratorStorageAdapterFactory
- Integration tests: processing-lib → dev-accelerators

**Tasks:**
1. Implement AcceleratorStorageAdapter (12-16h)
   - read() using ObjectClient.get()
   - write() using ObjectClient.put()
   - delete() using ObjectClient.delete()
   - uploadArtifact() for application bundles
   - Error mapping: ObjectClient errors → StorageError
   - JSON serialization/deserialization

2. Implement AcceleratorPolicyAdapter (8-10h)
   - Wire to policySerializer from dev-accelerators
   - IAM policy generation
   - Error handling

3. Implement AcceleratorDeploymentAdapter (8-12h)
   - Wire to LCPlatform services
   - Call create methods for each dependency type
   - Handle deployment status
   - Error handling

4. Create AcceleratorStorageAdapterFactory (4-6h)
   - Factory method from CLI context
   - Create ObjectClient with correct provider
   - Determine config bucket name

5. Integration testing (8-12h)
   - Test complete chain: processing-lib → dev-accelerators → mock provider
   - Test all configurator operations (init, read, update)
   - Test deployment operations
   - Verify no cloud credentials needed with mock provider

**Dependencies:** Phase 1 complete (Phase 2 can run in parallel)

**Acceptance Criteria:**
- [ ] AcceleratorStorageAdapter reads/writes JSON to ObjectClient
- [ ] All processing-lib adapters use dev-accelerators (no stubs)
- [ ] Integration tests pass with mock provider
- [ ] Processing-lib configurators work end-to-end
- [ ] All tests pass with ≥90% coverage

**Quality Gates:**
- [ ] Integration tests passing
- [ ] All tests passing
- [ ] Code coverage ≥90%
- [ ] Works with mock provider (no cloud credentials)

---

### Phase 4: CLI Integration (80-100 hours)

**Goal:** Complete user-facing CLI with full integration

**Deliverables:**
- CLI commands wired through processing-lib (no mock filesystem)
- lcp dependency command group (add, remove, list, show, validate)
- Error enrichment layer
- lcp context export --json
- Active application context (app use, list, current)
- -a/--app global flag

**Tasks:**
1. Create adapter factory for CLI (6-8h)
   - Create processing-lib adapters from CliContext
   - Handle provider selection
   - Config bucket resolution

2. Rewire app commands (16-20h)
   - app init → ApplicationConfigurator.init()
   - app read → ApplicationConfigurator.read()
   - app update → ApplicationConfigurator.update()
   - app validate → ApplicationConfigurator.validate()
   - Remove loadMockApps()/saveMockApps() functions

3. Rewire version commands (12-16h)
   - version add → VersionConfigurator.init()
   - version read → VersionConfigurator.read()
   - version deploy → DeployDependencies use case

4. Implement lcp dependency commands (24-30h)
   - dependency add (8-10h)
   - dependency list (4-6h)
   - dependency show (4-6h)
   - dependency remove (4-6h)
   - dependency validate (4-6h)

5. Implement error enrichment (8-10h)
   - Map processing-lib errors to CLI messages
   - Add suggestions for common errors
   - --debug flag for full details
   - --json flag for machine-readable output

6. Implement lcp context export (6-8h)
   - Read manifest
   - Generate access patterns for each dependency
   - Output JSON with examples

7. Implement active application context (12-16h)
   - app use <moniker> command
   - app list command with active indicator
   - app current command
   - -a/--app global flag
   - Auto-set active app on init
   - Store in .lcp/config.json

8. Testing (12-16h)
   - Unit tests for all new commands
   - Integration tests for CLI → processing-lib → dev-accelerators
   - E2E workflow tests (init → add deps → deploy)

**Dependencies:** Phase 3 complete

**Acceptance Criteria:**
- [ ] CLI commands delegate through processing-lib (zero mock filesystem)
- [ ] lcp dependency add/list/remove/show/validate work
- [ ] Error messages include suggestions and error codes
- [ ] lcp context export --json produces valid output
- [ ] Active app context works (app use, -a flag)
- [ ] All existing CLI tests pass (backward compatibility)
- [ ] All tests pass with ≥80% coverage

**Quality Gates:**
- [ ] All tests passing
- [ ] E2E tests passing
- [ ] Code coverage ≥80%
- [ ] Backward compatibility verified

---

### Phase 5: Polish & Documentation (60-80 hours)

**Goal:** Production-ready release with comprehensive documentation

**Deliverables:**
- Getting-started tutorials (5 documents)
- Updated parent README
- lcp codegen command (if time permits - should-have)
- Interactive dependency setup (if time permits - should-have)
- Domain-driven naming validation (if time permits - should-have)

**Tasks:**
1. Getting-started tutorials (32-40h)
   - 01-your-first-app.md (8-10h)
     - Zero to working app in 15 minutes
     - Use mock provider
     - Verify tutorial under 15 minutes
   - 02-adding-dependencies.md (6-8h)
     - lcp dependency add examples
     - All 14 dependency types covered
   - 03-writing-application-code.md (6-8h)
     - Moniker-based runtime API
     - Code examples for all client types
   - 04-deploying-to-cloud.md (6-8h)
     - Switch from mock to AWS
     - Deploy dependencies
     - Deploy application
   - 05-working-with-ai-agents.md (6-8h)
     - lcp context export usage
     - AI agent code generation
     - SpecKit integration

2. Update parent README (4-6h)
   - Reflect actual package names and structure
   - Update architecture diagram
   - Document v2 features

3. lcp codegen command (12-16h - should-have)
   - Generate typed SDK from manifest
   - Export const for each dependency
   - TypeScript types

4. Interactive dependency setup (8-12h - should-have)
   - Prompts for type-specific options
   - Guided configuration
   - Validate as you go

5. Domain-driven naming validation (4-6h - should-have)
   - Warn on cloud-specific terms (s3, sqs, bucket)
   - Suggest business-domain alternatives

6. Final testing and validation (8-12h)
   - Run all tutorials
   - Verify 15-minute target for tutorial 01
   - Full regression testing
   - Performance benchmarks

**Dependencies:** Phase 4 complete

**Acceptance Criteria:**
- [ ] Tutorial 01 completable in under 15 minutes
- [ ] All tutorials use moniker-based APIs
- [ ] No cloud credentials needed for tutorials 01-03
- [ ] Parent README updated
- [ ] All must-have features complete
- [ ] Should-have features complete (if time permits)

**Quality Gates:**
- [ ] All tests passing
- [ ] All tutorials tested by fresh user
- [ ] Documentation reviewed
- [ ] Performance benchmarks met

---

## Critical Path

**Critical Path Tasks (Sequential):**

```
Phase 1: lc-platform-config + ResourceResolver
  ↓ (60-80h)
Phase 2: LCAppRuntime.fromManifest() + Scoped Clients
  ↓ (80-100h)
Phase 3: AcceleratorStorageAdapter (processing-lib integration)
  ↓ (40-60h)
Phase 4: CLI integration (dependency commands, active context)
  ↓ (80-100h)
Phase 5: Documentation (getting-started tutorials)
  ↓ (60-80h)
RELEASE v1.0.0

Total Critical Path: 320-420 hours
```

**Parallel Work Opportunities:**
- Phase 2 and Phase 3 can overlap partially (scoped clients don't depend on adapters)
- Documentation drafts can start during Phase 4
- Code generation and interactive setup can be developed in parallel during Phase 5

**Blocking Dependencies:**
- Phase 2 BLOCKS Phase 4 (CLI needs runtime API)
- Phase 3 BLOCKS Phase 4 (CLI needs processing-lib adapters)
- Phase 4 BLOCKS Phase 5 (tutorials need working CLI)

---

## Dependency Graph

**Visual Dependency Map:**
```
Phase 1: Foundation (No dependencies)
  ├─ lc-platform-config package
  ├─ Manifest schema
  ├─ DependencyConfiguration types
  ├─ nameGenerator environment param
  └─ ResourceResolver
       ↓
Phase 2: Scoped Clients (Depends on Phase 1)
  ├─ 11 Scoped*Client wrappers
  ├─ LCAppRuntime moniker methods
  └─ fromManifest() factory
       ↓
Phase 3: Adapter Integration (Depends on Phase 1, parallel with Phase 2)
  ├─ AcceleratorStorageAdapter
  ├─ AcceleratorPolicyAdapter
  ├─ AcceleratorDeploymentAdapter
  └─ AcceleratorStorageAdapterFactory
       ↓
Phase 4: CLI Integration (Depends on Phase 2 AND Phase 3)
  ├─ Rewire app/version commands
  ├─ lcp dependency commands
  ├─ Error enrichment
  ├─ Context export
  └─ Active app context
       ↓
Phase 5: Documentation (Depends on Phase 4)
  ├─ Getting-started tutorials
  ├─ Updated README
  └─ Should-have features (if time)
```

**External Dependencies:**
- None (all use existing TypeScript/Bun tooling)

---

## Testing Strategy

### Unit Testing

**Approach:**
- Test-first (TDD): Write test before implementation
- One test file per implementation file
- Aim for ≥90% code coverage for critical path

**Test Organization:**
```
lc-platform-config/src/
  manifest/__tests__/parser.test.ts
  resolution/__tests__/resolver.test.ts

lc-platform-dev-accelerators/src/
  resolution/__tests__/ResourceResolver.test.ts
  core/clients/scoped/__tests__/ScopedObjectClient.test.ts

lc-platform-processing-lib/src/
  adapters/storage/__tests__/AcceleratorStorageAdapter.test.ts

lc-platform-dev-cli/src/
  cli/commands/dependency/__tests__/add.test.ts
```

**Key Test Scenarios:**
- Manifest parsing (valid, invalid, missing fields)
- Environment variable interpolation (with/without defaults)
- Resource resolution (valid monikers, missing monikers, type mismatches)
- Scoped client operations (all methods, error cases)
- CLI commands (success, validation failures, processing-lib errors)

### Integration Testing

**Approach:**
- Test component interactions
- Use real implementations with mock provider
- Test complete chains (CLI → processing-lib → dev-accelerators)

**Integration Test Suites:**
1. **Manifest → Runtime**: Load manifest, create runtime, access resources
2. **CLI → Processing-Lib**: Execute CLI commands, verify processing-lib calls
3. **Processing-Lib → Dev-Accelerators**: Call configurators, verify cloud operations (mock)

### End-to-End Testing

**Approach:**
- Full application lifecycle
- Real user workflows
- Mock provider (no cloud credentials)

**E2E Test Scenarios:**
1. **Developer Workflow**: Init app → add dependencies → write code → run with mock
2. **Manifest-Driven**: Create manifest → fromManifest() → use scoped clients
3. **Agent Workflow**: lcp context export → verify JSON structure

**Location:** `e2e/lc-platform-v2.test.ts`

### Performance Testing

**Benchmarks:**
- Manifest parsing + resolution: <500ms (target)
- CLI command response time: <1 second (target)
- E2E workflow (init → add deps → deploy): <30 seconds (target)

**Load Testing:**
- Manifest with 50+ dependencies: parse in <1 second
- 100 concurrent CLI commands: no failures

---

## Success Metrics

### Development Metrics

**Code Quality:**
- Test coverage: ≥80% overall, ≥90% for critical path
- Linter warnings: 0 critical/high violations
- TypeScript strict mode: passing

**Velocity:**
- Phase completion within ±20% of estimate
- 0 rework due to interface mismatches

### Release Metrics

**Functionality:**
- All 12 must-have requirements implemented
- 3+ should-have requirements implemented (stretch goal)

**Performance:**
- Manifest parsing + resolution: <500ms
- CLI command response: <1 second
- E2E workflow: <30 seconds

**Reliability:**
- All existing tests passing (backward compatibility)
- 0 critical bugs at release
- Mock provider works for 100% of features

### User Metrics

**Adoption:**
- Tutorial 01 completable in <15 minutes
- New user can deploy first app without docs (after tutorial)

**Satisfaction:**
- AI agents generate correct code from context export
- Error messages reduce support requests

---

## Timeline Summary

**Total Estimated Duration:** 320-420 hours (12-14 weeks at 24-30 hours/week)

**Phase Breakdown:**
- Phase 1: 60-80 hours - Weeks 1-3
- Phase 2: 80-100 hours - Weeks 4-6
- Phase 3: 40-60 hours - Weeks 7-8
- Phase 4: 80-100 hours - Weeks 9-11
- Phase 5: 60-80 hours - Weeks 12-14

**Key Milestones:**
- Week 3: Foundation complete (manifest, resolver, types)
- Week 6: Runtime API complete (moniker-based accessors work)
- Week 8: Integration complete (CLI → processing-lib → dev-accelerators)
- Week 11: CLI complete (dependency commands, active context)
- Week 14: **Release v1.0.0** (docs, tutorials, polish)

**Contingency:**
- Buffer: 20% (64-84 hours) for unknown unknowns
- Should-have features can be deferred to v1.1 if needed

---

## Next Steps

1. ✅ Review and approve this plan
2. Create detailed task breakdown (tasks.md) - IN PROGRESS
3. Set up feature branch: `feature/lc-platform-v2`
4. Begin Phase 1 implementation
   - Create lc-platform-config package
   - Define manifest schema types
5. Daily/weekly progress updates in status.md
6. Update status.md as phases complete (MANDATORY)

---

## Notes

**Assumptions:**
- Bun 1.0+ runtime for all TypeScript packages
- Mock provider sufficient for all testing (no cloud credentials)
- Existing dev-accelerators tests provide regression coverage
- Processing-lib ports align with dev-accelerators APIs (verified in Phase 3)

**Open Questions:**
- Config bucket naming convention for AcceleratorStorageAdapter
- Manifest schema validation library choice (JSON Schema vs Zod)
- Migration documentation approach for existing CLI users

**Constraints:**
- Must maintain backward compatibility (no breaking changes)
- Must work with mock provider (no cloud credentials required)
- Must complete must-have features before release (12 requirements)
