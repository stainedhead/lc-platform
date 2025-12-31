# GitHub Copilot Instructions for LC Platform

> **⚠️ IMPORTANT: This file is a reference pointer only**

## Source of Truth

**All development rules, guidelines, and agent instructions are maintained in [`AGENTS.md`](../AGENTS.md)**.

This file exists solely to redirect GitHub Copilot to the canonical source of development process rules.

## Primary Reference

For complete information about:
- LC Platform architecture and design patterns
- Hexagonal Architecture implementation
- Development workflow and SpecKit commands
- Test-Driven Development practices
- Quality standards and code coverage requirements
- Cross-package coordination
- Documentation maintenance
- CI/CD processes and pre-commit workflows

**👉 Please refer to [`AGENTS.md`](../AGENTS.md) at the repository root**

## Package-Specific Guidance

When working in a specific package directory, consult **both**:

1. **Parent [`AGENTS.md`](../AGENTS.md)** - Platform-wide principles and architecture
2. **Package `AGENTS.md`** - Package-specific implementation rules
   - `lc-platform-dev-accelerators/AGENTS.md` - Accelerators package rules
   - `lc-platform-cli/AGENTS.md` - CLI package rules (when created)
   - `lc-platform-config/AGENTS.md` - Config package rules (when created)
   - `lc-platform-rest-api/AGENTS.md` - REST API package rules (when created)

## Making Changes

**❌ DO NOT suggest or make changes to this file (copilot-instructions.md).**

If you need to update development rules, guidelines, or agent instructions:

1. ✅ Make all changes to **[`AGENTS.md`](../AGENTS.md)** - the central source of truth
2. ✅ Update package-specific `AGENTS.md` files for package-level rules
3. ❌ Do NOT modify `.github/copilot-instructions.md` (this file) - it's just a pointer

## Why This Structure?

- **Single Source of Truth**: All rules maintained in [`AGENTS.md`](../AGENTS.md)
- **Multi-Agent Support**: Shared context across GitHub Copilot, Claude Code, and other AI tools
- **Consistency**: Prevents divergence between multiple instruction files
- **Maintainability**: Updates only need to be made in one location
- **Clarity**: Clear hierarchy prevents confusion about which file to update

## Quick Reference

### Core Principles (from AGENTS.md)

1. **Hexagonal Architecture** - All packages follow Clean Architecture with provider abstraction
2. **Test-Driven Development** - Write tests first, minimum 80% coverage
3. **Provider Independence** - No cloud SDK types in core interfaces
4. **SpecKit Workflow** - Use `/speckit.*` commands for feature development
5. **Quality Gates** - Auto-format, lint, test before every commit

### For TypeScript Packages (Bun Runtime)

```bash
bun run format         # Format code with Prettier (run this FIRST)
bun run lint           # Run ESLint
bun test               # Run tests with coverage
bun run typecheck      # Type-check without building
bun run build          # Compile TypeScript
```

### Pre-Commit Checklist

1. `bun run format` - Auto-format code
2. `git add -A` - Stage changes
3. `bun run lint` - Verify linting passes
4. `bun test` - Verify tests pass
5. `git commit -m "message"` - Commit
6. `git push` - Push to remote

---

**🔗 Go to [`AGENTS.md`](../AGENTS.md) now for complete development guidance.**

---

## Constitution

This project follows the **LC Platform Constitution** (version 1.0.0) which establishes:
- Governance structure and amendment process
- Non-negotiable principles (Hexagonal Architecture, TDD)
- Cross-package coordination rules
- Quality gates and compliance requirements

See [`.specify/memory/constitution.md`](../.specify/memory/constitution.md) for governance details.

The constitution defines **what MUST be true**; [`AGENTS.md`](../AGENTS.md) defines **how to achieve it**.
