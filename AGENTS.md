# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Quick Start

**ALWAYS start by reading [INDEX.md](INDEX.md)** - it provides:
- Complete navigation to all documentation
- Task-based guide lookup
- Keyword search capabilities
- Document structure conventions

## Repository Purpose

This repository contains notes, documentation, and context for AI agent interactions, particularly focused on WildFly development workflows and best practices.

## Current Structure

```
agent-notes/
├── INDEX.md                          # START HERE - navigation index
├── README.md                         # Repository overview
├── AGENTS.md                         # This file - minimal agent guidance
├── general-development/              # Requirements for all code changes
│   └── logging-and-message-guide.md
├── feature-development/              # WildFly development guides
│   ├── management-model-version-bump-guide.md
│   ├── schema-version-bump-guide.md
│   └── subsystem-schema-test-requirements.md
└── reproducer-development/           # Reproducer creation guides
    └── wildfly-deployment-reproducer-guide.md
```

## Key Principles for Agents

1. **Use INDEX.md for navigation** - Don't load all files at once
2. **Follow document structure** - All guides use consistent heading conventions (see INDEX.md)
3. **Load only what you need** - Use INDEX.md to find relevant guides
4. **Update as you work** - These are living documents
5. **Double-check legacy edits** - Any work under `legacy/` is a red flag. Before changing it, verify that it belongs in the target branch's supported distribution and task. Follow [Verify Target Distributions Before Changing Packaging](feature-development/feature-implementation-guide.md#verify-target-distributions-before-changing-packaging); do not infer support or backport applicability from directory names alone.

## Document Conventions

All documentation follows conventions detailed in INDEX.md:
- Standard section markers (Overview, Process, Checklists, etc.)
- Consistent heading levels for navigation
- Visual markers (✅, ❌, ⚠️) for quick scanning
- Cross-references between related documents

See INDEX.md for complete details on navigation strategies and document structure.

---

**Last Updated**: 2026-11-18
**Version**: 2.1
