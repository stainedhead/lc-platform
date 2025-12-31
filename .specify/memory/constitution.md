<!--
Sync Impact Report:
Version: NEW → 1.0.0 (Initial constitution)
Modified Principles: N/A (new document)
Added Sections: All core principles and governance rules
Removed Sections: N/A
Templates Requiring Updates:
  ✅ plan-template.md - Constitution Check section aligned with AGENTS.md reference
  ✅ spec-template.md - No updates required (requirements-driven, constitution-agnostic)
  ✅ tasks-template.md - No updates required (follows plan/spec structure)
Follow-up TODOs: None
-->

# LC Platform Constitution

## Core Principles

### I. AGENTS.md as Single Source of Truth

**AGENTS.md is the authoritative reference for all development rules and product context.**

This constitution establishes governance and coordination across the LC Platform ecosystem.
For detailed development rules, architectural patterns, quality standards, and implementation
guidelines, refer to:

- **Parent Level**: `/AGENTS.md` - Platform-wide principles and cross-package coordination
- **Package Level**: `{package}/AGENTS.md` - Package-specific implementation rules

**Rationale**: Maintaining a single, living document (AGENTS.md) ensures consistency across
all packages while enabling AI agents to access complete context. This constitution provides
governance structure, while AGENTS.md provides operational guidance.

### II. Hexagonal Architecture (NON-NEGOTIABLE)

**All packages MUST follow Hexagonal Architecture (Ports and Adapters) pattern.**

Requirements:
- Core domain logic MUST be cloud-agnostic and provider-independent
- Ports (interfaces) MUST define contracts without implementation details
- Adapters MUST contain all cloud-provider-specific code (AWS, Azure, GCP, Mock)
- Application code MUST depend on abstractions, never on concrete implementations
- No cloud SDK types MUST leak into core interfaces or return values

**Rationale**: Hexagonal architecture enables true cloud portability, vendor independence,
and testability through mock providers. This is the foundation of the platform's value
proposition.

### III. Test-Driven Development (NON-NEGOTIABLE)

**Tests MUST be written before implementation begins.**

Requirements:
- All tests MUST be written first and MUST fail before implementation
- Minimum 80% code coverage for all public interfaces
- Unit tests use mock provider (no cloud credentials required)
- Integration tests use LocalStack (AWS) or Azurite (Azure)
- Contract tests verify provider parity (AWS ↔ Mock ↔ Azure)

**Rationale**: TDD ensures interface stability, prevents regression, enables rapid
development without cloud costs, and validates provider abstraction integrity.

### IV. Provider Independence

**No cloud-specific concepts or types in core domain.**

Requirements:
- Core interfaces MUST use generic, cloud-agnostic terminology
- Configuration-driven provider selection (environment variables)
- Mock provider MUST support all interface methods for local development
- Provider switching MUST require only configuration changes, never code changes

**Rationale**: Provider independence is the core value proposition. Without it, the
platform offers no advantage over using cloud SDKs directly.

### V. Documentation as Code

**AGENTS.md files MUST be maintained as development rules evolve.**

Requirements:
- Update parent `AGENTS.md` when platform-wide patterns change
- Update package `AGENTS.md` when package-specific rules change
- Documentation updates are part of Definition of Done
- Changes to architecture MUST be reflected in AGENTS.md before merging

**Rationale**: AGENTS.md provides AI agents and developers with current context. Stale
documentation leads to inconsistencies and technical debt.

### VI. Quality Gates

**Code quality standards MUST be enforced before commit.**

Requirements (for TypeScript packages):
- Auto-format code first (`bun run format`) before all commits
- Zero Critical/High severity linting errors allowed
- All tests MUST pass before commit
- Type checking MUST pass without errors (`bun run typecheck`)

**Rationale**: Automated quality gates prevent technical debt and maintain consistent
code quality across all packages and contributors.

## Cross-Package Coordination

### Package Dependency Management

**Changes affecting multiple packages MUST follow dependency order.**

Process:
1. Update foundation packages first (`lc-platform-dev-accelerators`)
2. Update shared packages second (`lc-platform-config`)
3. Update consumer packages last (`lc-platform-cli`, `lc-platform-rest-api`)
4. Coordinate version bumps across dependent packages
5. Integration testing MUST verify cross-package compatibility

**Rationale**: Following dependency order prevents breaking changes and ensures stable
releases across the platform ecosystem.

### SpecKit Workflow Integration

**All packages MUST use SpecKit for structured development.**

Required commands:
- `/speckit.specify` - Create/update feature specifications
- `/speckit.plan` - Generate implementation plans with design artifacts
- `/speckit.tasks` - Create dependency-ordered tasks
- `/speckit.implement` - Execute implementations
- `/speckit.clarify` - Refine underspecified requirements
- `/speckit.analyze` - Verify cross-artifact consistency

**Rationale**: SpecKit provides consistent, AI-assisted workflow across all packages,
reduces ambiguity, and ensures thorough planning before implementation.

## Governance

### Amendment Process

**This constitution MUST be updated when governance structure changes.**

To amend this constitution:
1. Propose changes with clear rationale in pull request
2. Document impact on AGENTS.md files (parent and packages)
3. Update version number following semantic versioning rules
4. Update LAST_AMENDED_DATE to current date
5. Cascade changes to affected AGENTS.md files
6. Notify all package maintainers of governance changes

### Versioning Policy

**Constitution versions follow semantic versioning (MAJOR.MINOR.PATCH):**

- **MAJOR**: Backward-incompatible changes (removing/redefining core principles)
- **MINOR**: New principles added or material expansions to guidance
- **PATCH**: Clarifications, wording improvements, typo fixes

### Compliance Review

**All development work MUST comply with this constitution and AGENTS.md.**

Verification requirements:
- PRs MUST reference relevant sections of AGENTS.md for new patterns
- Architecture reviews MUST verify Hexagonal Architecture compliance
- Test coverage reports MUST meet 80% threshold
- Code reviews MUST verify no cloud SDK types in core interfaces
- Constitution violations MUST be justified and documented in plan.md

### AGENTS.md as Runtime Guidance

**For operational development guidance, always consult AGENTS.md files:**

- **Platform-wide rules**: See parent `/AGENTS.md`
- **Package-specific rules**: See `{package}/AGENTS.md`
- **When in conflict**: Package-specific rules take precedence within that package

This constitution defines what MUST be true; AGENTS.md defines how to achieve it.

**Version**: 1.0.0 | **Ratified**: 2025-12-31 | **Last Amended**: 2025-12-31
