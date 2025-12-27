# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the documentation website for [Neatoo](https://github.com/NeatooDotNet/Neatoo), a DDD (Domain-Driven Design) framework for .NET Blazor and WPF applications. The site is built with Jekyll using the Minimal Mistakes theme and hosted on GitHub Pages.

## Build Commands

```bash
# Install dependencies
bundle install

# Run local development server
bundle exec jekyll serve

# Build the site
bundle exec jekyll build
```

Note: Changes to `_config.yml` require restarting the server.

## Site Architecture

- **Theme**: Minimal Mistakes (remote theme via `mmistakes/minimal-mistakes`)
- **Content location**: `_pages/` directory contains all documentation pages
- **Navigation**: Defined in `_data/navigation.yml` with `central` sidebar nav
- **Home page**: `index.md` at root

### Page Structure

Pages use front matter with:
- `layout: single` (default for pages)
- `permalink: /path/to/page/`
- `toc: true` and `toc_sticky: true` enabled by default for `_pages`

### Documentation Sections

- `/gettingstarted/` - Tutorial content
- `/setup/` - Client and server setup guides
- `/overview/` - Core concepts (Entity, Rule, Factory, Authorization)
- `/example/` - Code examples (Person entity)

## Content Guidelines

When editing documentation:
- Reference the main Neatoo repo at `https://github.com/NeatooDotNet/Neatoo`
- Link to source files in the main repo for code examples
- Follow existing front matter patterns in `_pages/`

### Neatoo vs RemoteFactory Terminology

**Important**: For this documentation site, treat "Neatoo + RemoteFactory" as simply "Neatoo". While RemoteFactory is technically a separable component, this site assumes users are using the full Neatoo framework.

- Refer to features (Value Objects, Authorization, etc.) as "Neatoo" features
- The separability of RemoteFactory should only be documented on its own dedicated page
- Do not distinguish between Neatoo and RemoteFactory elsewhere in the documentation
- The individual `/docs` folders in the Neatoo and RemoteFactory repositories handle their respective specific documentation
