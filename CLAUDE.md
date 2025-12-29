# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **documentation website for Neatoo**, a DDD (Domain-Driven Design) framework for .NET Blazor and WPF applications. "Neatoo" on this site encompasses the combination of two repositories:

- **Neatoo Core**: [github.com/NeatooDotNet/Neatoo](https://github.com/NeatooDotNet/Neatoo) - The DDD framework with entities, rules, validation, and source generators
- **Neatoo.RemoteFactory**: [github.com/NeatooDotNet/RemoteFactory](https://github.com/NeatooDotNet/RemoteFactory) - The client-server communication layer enabling shared domain models

The site is built with **Jekyll** using the **Minimal Mistakes** theme and hosted on **GitHub Pages**.

## Repository Links

| Repository | Purpose |
|------------|---------|
| [NeatooDotNet/Neatoo](https://github.com/NeatooDotNet/Neatoo) | Core DDD framework |
| [NeatooDotNet/RemoteFactory](https://github.com/NeatooDotNet/RemoteFactory) | Client-server communication |
| [NeatooDotNet/neatoodotnet.github.io](https://github.com/NeatooDotNet/neatoodotnet.github.io) | This documentation site |

### Technical Documentation (Source of Truth)

More detailed technical documentation lives in the main repositories:
- **Neatoo**: `https://github.com/NeatooDotNet/Neatoo/docs`
- **RemoteFactory**: `https://github.com/NeatooDotNet/RemoteFactory/docs`

These repositories are the **source of truth**. This documentation site aims to stay current but may lag behind.

### Pending Changes to Incorporate

Check these locations for changes that need to be incorporated into this site:
- `https://github.com/NeatooDotNet/Neatoo/docs/todos`
- `https://github.com/NeatooDotNet/RemoteFactory/docs/todos`

## Commit Tracking

Track which commits from each repository this documentation site is synchronized with:

| Repository | Last Synced Commit | Date |
|------------|-------------------|------|
| Neatoo | `a16fb5b` | 2025-12-29 |
| RemoteFactory | `cb0db17` | 2025-12-29 |

When updating documentation, review commits since the last sync and update this table.

## Neatoo Skill Maintenance

A shared Neatoo skill file exists at the user level for Claude Code:

**Location**: `~/.claude/skills/neatoo.md`

This skill provides Claude with Neatoo framework knowledge across all projects. When updating documentation from the Neatoo or RemoteFactory repositories, **also update the Neatoo skill** to keep it current.

### Skill Commit Tracking

Track which commits have been incorporated into the Neatoo skill:

| Repository | Last Synced Commit | Date |
|------------|-------------------|------|
| Neatoo | `a16fb5b` | 2025-12-29 |
| RemoteFactory | `cb0db17` | 2025-12-29 |

### Update Checklist

When syncing from the source repositories:

1. Update the documentation site (`_pages/`)
2. Update the Neatoo skill (`~/.claude/skills/neatoo.md`)
3. Update **both** commit tracking tables (this site and the skill)

The skill should contain the same core technical content as the reference documentation but formatted as a single comprehensive reference for Claude.

## Build Commands

```bash
# Install dependencies
bundle install

# Run local development server
bundle exec jekyll serve

# Build the site
bundle exec jekyll build
```

**Note**: Changes to `_config.yml` require restarting the server.

## Site Architecture

- **Theme**: Minimal Mistakes (remote theme via `mmistakes/minimal-mistakes`)
- **Content location**: `_pages/` directory contains all documentation pages
- **Navigation**: Defined in `_data/navigation.yml` with `central` sidebar nav
- **Home page**: `index.md` at root
- **Assets**: Logo and images in `assets/`

### Page Structure

Pages use front matter with:
- `layout: single` (default for pages)
- `permalink: /path/to/page/`
- `toc: true` and `toc_sticky: true` enabled by default
- `sidebar: nav: "central"` for navigation

### Documentation Sections

| Section | Path | Content |
|---------|------|---------|
| Getting Started | `/gettingstarted/` | Introduction, Quick Start, Client/Server Setup |
| Concepts | `/concepts/` | DDD Overview, Aggregates, Rules, Factories (with Commands & Queries), Client-Server, Properties, Authorization |
| Reference | `/reference/` | EntityBase, ValidateBase, Base & Value Objects, EntityListBase, Rules Engine, Factory Operations, Data Mapping, Authorization System, Dependency Injection, Exception Handling |
| Guides | `/guides/` | Blazor Integration, Database-Dependent Validation, Troubleshooting |
| Examples | `/example/` | Person entity, Order Aggregate |

## Audience and Scope

### Target Audience
Developers who want to use Neatoo for building .NET applications with proper DDD architecture.

### Documentation Principles

This site should **fully explain** not just how to use Neatoo, but the underlying principles:

1. **Domain-Driven Design (DDD)**
   - Entities, Value Objects, Aggregates
   - Aggregate Roots and entity graphs
   - Business rules as first-class citizens
   - Ubiquitous language

2. **Blazor**
   - WebAssembly architecture
   - Component binding patterns
   - INotifyPropertyChanged integration
   - Client-server communication

3. **Roslyn Source Generators**
   - Compile-time code generation
   - Why source generators enable Neatoo's patterns
   - How generated code reduces boilerplate

### Writing Approach

The existing documentation uses:
- Problem-oriented introductions (start with the pain point)
- Real-world analogies (driver's license for Entity, $20 bill for Value Object, spreadsheet for Rules)
- Traditional vs. Neatoo comparisons
- Production-quality code examples
- Mermaid diagrams and ASCII art for visualization
- "Best Practices" and "Common Pitfalls" sections
- "Related Topics" cross-references

## Current Content Summary

### Well-Covered Topics
- Introduction and value proposition
- DDD concepts mapped to Neatoo implementation
- Aggregates and entity graphs with propagation patterns
- Rules philosophy (validation + transformation) with comprehensive attribute validation docs
- Factory pattern with 5 lifecycle operations + Commands & Queries pattern
- Client-server architecture with single `/api/neatoo` endpoint
- Properties and meta-properties (11 entity-level, 4 per-property)
- Authorization model with `[AuthorizeFactory]`
- Blazor integration with MudNeatoo components
- Complete Order Aggregate example (1100+ lines)
- Exception handling and error patterns
- Database-dependent validation guide
- Troubleshooting guide

### Lighter Coverage (Opportunities)
- WPF-specific guidance
- Migration from traditional architectures
- Performance considerations
- Advanced source generator topics

## Terminology Guidelines

**Important**: For this documentation site, "Neatoo" encompasses both Neatoo and Neatoo.RemoteFactory. Users of this site are assumed to be using the full framework.

- Refer to all features as "Neatoo" features
- Do not distinguish between Neatoo and RemoteFactory in general documentation
- The separability of RemoteFactory should only be mentioned on its dedicated page
- Each repository's `/docs` folder handles its own specific technical documentation

## Content Guidelines

When editing documentation:
- Follow the existing problem-first writing style
- Include code examples that are production-quality
- Use analogies to explain abstract concepts
- Cross-reference related topics
- Keep examples consistent with the Person and Order patterns already established
- Reference source files in the main repos when appropriate

## Key Files

| File | Purpose |
|------|---------|
| `_config.yml` | Site configuration, theme, plugins |
| `_data/navigation.yml` | Sidebar navigation structure |
| `index.md` | Home page with feature highlights |
| `_pages/concepts/*.md` | Deep-dive concept explanations |
| `_pages/gettingstarted/*.md` | Onboarding and setup |
| `_pages/guides/*.md` | Practical how-to guides |
| `_pages/reference/*.md` | API reference |
| `_pages/examples/*.md` | Complete working examples |
