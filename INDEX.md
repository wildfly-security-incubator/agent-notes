# Agent Notes Index

This index helps AI agents quickly locate relevant documentation without loading all files.

## Repository Purpose

This repository contains documentation, guides, and context to assist developers working on WildFly, with particular emphasis on WildFly Elytron development. Content covers development workflows, PR review guidelines, issue triage, and best practices for day-to-day engineering work.

## Quick Navigation

### Core Documentation
- **[README.md](README.md)** - Repository overview and purpose
- **[AGENTS.md](AGENTS.md)** - Agent-specific instructions and guidance

### Reproducer Development Guides

Located in `reproducer-development/`:

#### Deployment Reproducers
- **[wildfly-deployment-reproducer-guide.md](reproducer-development/wildfly-deployment-reproducer-guide.md)**
  - **Purpose**: How to create a deployment-based reproducer project for diagnosing WildFly issues
  - **Use When**: Building a minimal project to reproduce a bug or verify behaviour in WildFly
  - **Key Topics**: Archetype bootstrapping, multi-module structure, Galleon layers, CLI scripts, server lifecycle, verification with curl
  - **Related To**: WildFly Galleon Guide (https://docs.wildfly.org/41/Galleon_Guide.html)

#### Development Workflows
- **[wildfly-development-workflow.md](reproducer-development/wildfly-development-workflow.md)**
  - **Purpose**: Development workflows for working with WildFly and its components in a reproducer
  - **Use When**: Debugging WildFly boot process, testing SNAPSHOT builds, overriding component versions
  - **Key Topics**: Remote debugging (JDWP, suspend=y), WildFly SNAPSHOT builds, Galleon channels/manifests for version overrides
  - **Related To**: wildfly-deployment-reproducer-guide.md (complementary workflow guide)

#### Jakarta EE Security
- **[jakarta-ee-security-patterns.md](reproducer-development/jakarta-ee-security-patterns.md)**
  - **Purpose**: Patterns specific to Jakarta EE Security (Soteria) reproducers in WildFly
  - **Use When**: Creating reproducers for Jakarta EE Security issues, debugging Soteria authentication mechanisms
  - **Key Topics**: Authentication mechanism definitions, integrated-jaspi configuration, IdentityStore setup, multi-WAR EAR bugs, Soteria version override
  - **Related To**: wildfly-deployment-reproducer-guide.md, wildfly-development-workflow.md

### Feature Development Guides

Located in `feature-development/`:

#### Feature Implementation
- **[feature-implementation-guide.md](feature-development/feature-implementation-guide.md)**
  - **Purpose**: Guide for implementing new features in WildFly subsystems (after version bumps)
  - **Use When**: Adding new management attributes, XML configuration options, or subsystem capabilities
  - **Key Topics**: OBJECT attributes, string handling, parser patterns, runtime integration, backward compatibility
  - **Related To**: Requires version bumps first; see version management guides below
  - **Prerequisites**: Coordinate version bumps in Zulip #wildfly-elytron before starting

#### Version Management
- **[management-model-version-bump-guide.md](feature-development/management-model-version-bump-guide.md)**
  - **Purpose**: Guide for bumping WildFly subsystem management model versions
  - **Use When**: Making changes to subsystem management API (attributes, operations, capabilities)
  - **Key Topics**: Version bump process, transformer chains, PR review guidelines, stability levels
  - **Related To**: Schema version bumps (often done together); prerequisite for feature implementation

- **[schema-version-bump-guide.md](feature-development/schema-version-bump-guide.md)**
  - **Purpose**: Guide for bumping WildFly subsystem XML schema versions
  - **Use When**: Changing XML configuration format, adding/removing elements, promoting stability levels
  - **Key Topics**: Schema versioning, XSD files, parser registration, stability level annotations
  - **Related To**: Management model version bumps (often coordinated); prerequisite for feature implementation

#### Testing Requirements
- **[subsystem-schema-test-requirements.md](feature-development/subsystem-schema-test-requirements.md)**
  - **Purpose**: Requirements for WildFly subsystem schema test files
  - **Use When**: Creating or updating test XML files for schema versions
  - **Key Topics**: Test coverage requirements, element ordering, expression coverage, stability-level testing
  - **Related To**: Schema version bumps (tests required for each schema version)

## Finding Specific Information

### By Task Type

| Task | Primary Guide | Supporting Guides |
|------|--------------|-------------------|
| Create a deployment reproducer | wildfly-deployment-reproducer-guide.md | wildfly-development-workflow.md |
| Create Jakarta EE Security reproducer | jakarta-ee-security-patterns.md | wildfly-deployment-reproducer-guide.md |
| Debug WildFly boot process | wildfly-development-workflow.md | wildfly-deployment-reproducer-guide.md |
| Debug Jakarta EE Security | jakarta-ee-security-patterns.md | wildfly-development-workflow.md |
| Test WildFly SNAPSHOT builds | wildfly-development-workflow.md | wildfly-deployment-reproducer-guide.md |
| Override component versions | wildfly-development-workflow.md | wildfly-deployment-reproducer-guide.md |
| Override Soteria version | jakarta-ee-security-patterns.md | wildfly-development-workflow.md |
| Implement new subsystem feature | feature-implementation-guide.md | Both version bump guides + test requirements |
| Add management attribute | feature-implementation-guide.md | management-model-version-bump-guide.md |
| Add XML configuration option | feature-implementation-guide.md | schema-version-bump-guide.md |
| Bump management model version | management-model-version-bump-guide.md | schema-version-bump-guide.md |
| Bump schema version | schema-version-bump-guide.md | management-model-version-bump-guide.md |
| Create/update test files | subsystem-schema-test-requirements.md | schema-version-bump-guide.md |
| Promote feature stability | Both version bump guides | subsystem-schema-test-requirements.md |
| Review version bump PR | Both version bump guides | - |

### By Keyword

- **Reproducer**: wildfly-deployment-reproducer-guide.md
- **Galleon layers**: wildfly-deployment-reproducer-guide.md
- **Galleon channels**: wildfly-development-workflow.md
- **Archetype**: wildfly-deployment-reproducer-guide.md
- **Server lifecycle**: wildfly-deployment-reproducer-guide.md
- **EAR/WAR deployment**: wildfly-deployment-reproducer-guide.md
- **Jakarta EE Security**: jakarta-ee-security-patterns.md
- **Soteria**: jakarta-ee-security-patterns.md
- **JASPIC**: jakarta-ee-security-patterns.md
- **integrated-jaspi**: jakarta-ee-security-patterns.md
- **IdentityStore**: jakarta-ee-security-patterns.md
- **HttpAuthenticationMechanism**: jakarta-ee-security-patterns.md
- **BASIC authentication**: jakarta-ee-security-patterns.md
- **FORM authentication**: jakarta-ee-security-patterns.md
- **Remote debugging**: wildfly-development-workflow.md, jakarta-ee-security-patterns.md
- **JDWP**: wildfly-development-workflow.md
- **SNAPSHOT builds**: wildfly-development-workflow.md
- **Version override**: wildfly-development-workflow.md
- **Manifest**: wildfly-development-workflow.md
- **Component override**: wildfly-development-workflow.md
- **Version bump**: management-model-version-bump-guide.md, schema-version-bump-guide.md
- **Feature implementation**: feature-implementation-guide.md
- **OBJECT attribute**: feature-implementation-guide.md
- **Attribute definition**: feature-implementation-guide.md, management-model-version-bump-guide.md
- **String constants**: feature-implementation-guide.md
- **Resource bundles**: feature-implementation-guide.md
- **Measurement units**: feature-implementation-guide.md
- **Transformer**: management-model-version-bump-guide.md
- **Schema/XSD**: schema-version-bump-guide.md, feature-implementation-guide.md
- **Parser**: schema-version-bump-guide.md, feature-implementation-guide.md
- **Runtime integration**: feature-implementation-guide.md
- **Backward compatibility**: feature-implementation-guide.md, management-model-version-bump-guide.md
- **System properties**: feature-implementation-guide.md
- **Test coverage**: subsystem-schema-test-requirements.md
- **Stability levels**: All guides (PREVIEW, COMMUNITY, DEFAULT, EXPERIMENTAL)
- **PR review**: management-model-version-bump-guide.md, schema-version-bump-guide.md
- **WildFly release**: All guides (version targeting)
- **Zulip coordination**: feature-implementation-guide.md

## Document Structure Conventions

All guides follow these conventions to aid agent navigation:

### Standard Sections
- **Overview** - Purpose and scope
- **Pre-[Action] Checklist** - Prerequisites before starting
- **[Action] Process** - Step-by-step instructions
- **Verification Steps** - How to confirm success
- **Common Patterns** - Best practices and workflows
- **PR Review Guidelines** - What to check when reviewing
- **Troubleshooting** - Common issues and solutions
- **Summary Checklist** - Quick reference for completion

### Heading Markers
- Use `##` for major sections
- Use `###` for subsections
- Use `####` for detailed steps
- Use `**Bold**` for critical requirements
- Use `✅` for correct patterns
- Use `❌` for incorrect patterns

### Code Examples
- Inline code: `ClassName` or `method()`
- Code blocks: Include language identifier (java, xml, bash)
- File paths: Use absolute paths in examples

## Usage Tips for Agents

1. **Start with the index** - Don't load all files immediately
2. **Use keyword search** - Find relevant guide by task or keyword
3. **Check related guides** - Version bumps often require multiple guides
4. **Follow section structure** - Guides use consistent organization
5. **Look for checklists** - Each guide has verification checklists
6. **Check for examples** - Guides include practical examples and patterns

## Maintenance

This index should be updated when:
- New guides are added to the repository
- Existing guides are significantly restructured
- New subdirectories are created
- Guide purposes or scopes change

---

**Last Updated**: 2026-09-12
**Index Version**: 1.3