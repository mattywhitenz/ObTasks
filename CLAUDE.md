# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Obsidian Tasks is a community plugin for [Obsidian](https://obsidian.md/) that provides task management across a vault. Users write tasks as Markdown checklist items with emoji-based metadata (due dates, recurrence, priorities) and query them via `tasks` code blocks. The plugin parses these queries, filters/sorts/groups the vault's cached tasks, and renders results as interactive HTML.

## Commands

```bash
yarn                    # Install dependencies
yarn dev                # Build + watch for development
yarn build              # Production build (minified, no sourcemap)
yarn build:dev          # Development build (no watch)
yarn test               # Run all tests (jest --ci)
yarn test:dev           # Run tests in watch mode
yarn test -- --testPathPattern="Query/Filter" # Run tests matching a path pattern
yarn test -- -t "PriorityField"              # Run tests matching a name pattern
yarn lint               # ESLint (src + tests) + tsc --noEmit + svelte-check
yarn lint:markdown      # Markdown linting
```

Lefthook runs lint + related tests on pre-commit and the full suite (build + lint + test) on pre-push.

## Architecture

**Entry point:** `src/main.ts` — `TasksPlugin` extends Obsidian's `Plugin`. On load it initializes i18n, settings, the `Cache` (which indexes all tasks in the vault), the `InlineRenderer`, and the `QueryRenderer`.

**Core domain model:**

- `src/Task/Task.ts` — Immutable Task class. All mutations produce new Task instances. Properties include description, status, dates (due/scheduled/start/created/done/cancelled), recurrence, priority, dependencies, and location metadata.
- `src/Task/Recurrence.ts` — Handles recurring task logic, backed by the `rrule` library.
- `src/Statuses/StatusRegistry.ts` — Singleton registry of task statuses (e.g., TODO, IN_PROGRESS, DONE). Configurable via settings.

**Query engine (`src/Query/`):**

- `Query.ts` — Parses a `tasks` code block into filters, sorters, groupers, and layout options. Implements `IQuery` interface.
- `Filter/Field.ts` — Abstract base class. Each filterable/sortable/groupable property has a concrete `Field` subclass (e.g., `PriorityField`, `DueDateField`, `TagsField`). To add a new filter: create a `Field` subclass and register it in `FilterParser.ts`.
- `Group/TaskGroups.ts` — Entry point for grouping tasks by field values.
- `Sort/Sort.ts` — Entry point for multi-key sorting.
- `Matchers/` — Substring and regex matching abstractions used by text filters.
- `Presets/` — Reusable named instruction blocks for queries.

**Serialization (`src/TaskSerializer/`):**

- `DefaultTaskSerializer.ts` — Reads/writes tasks in the default emoji format.
- `DataviewTaskSerializer.ts` — Reads/writes tasks in Dataview inline field format.

**Rendering (`src/Renderer/`):** Converts `QueryResult` objects into interactive HTML. `QueryRenderer` registers the `tasks` code block processor with Obsidian.

**UI (`src/ui/`):** Svelte components for the "Create or edit Task" modal. `EditInstructions/` provides an abstraction for atomic task edits.

**Obsidian integration (`src/Obsidian/`):** Code tightly coupled to the Obsidian API (`Cache`, `File`, `InlineRenderer`, `LivePreviewExtension`). Minimal test coverage due to API limitations outside a running vault.

## Testing Patterns

Tests mirror `src/` structure under `tests/`. Key patterns to know:

- **All tests run in UTC** (`tests/global-setup.js` sets `process.env.TZ = 'UTC'`).
- **TaskBuilder** (`tests/TestingTools/TaskBuilder.ts`) — Fluent builder for creating Task instances in tests. Use this instead of constructing Tasks directly.
- **Custom Jest matchers** (`tests/CustomMatchers/`) — Domain-specific matchers like `toMatchTask`, `toMatchTaskWithDescription`, `toHaveExplanation`, `groupHeadingsToBe`, `toToggleTo`, etc. Registered globally via `jest.custom_matchers.setup.ts`.
- **Approval testing** — Many tests use `.approved.*` files (text, HTML, Markdown, Mermaid) as golden outputs. The `approvals` library compares actual output against these files. When output intentionally changes, update the `.approved.*` file.
- **`it.failing()` for bug fixes** — Project convention: first write a test with `it.failing()` that reproduces the bug, then fix the code, then change to `it()`.
- **FilterTestHelpers** (`tests/TestingTools/FilterTestHelpers.ts`) — Helpers for testing Field/Filter implementations.

## Code Style

- Prettier: 120 char line width, 4-space indent, single quotes, trailing commas
- ESLint enforces import ordering (`import/order` + `sort-imports`)
- Unix line endings enforced
- Svelte components use `svelte-preprocess` for TypeScript

## Key Dependencies

- `chrono-node` — Natural language date parsing
- `rrule` — Recurrence rule computation (RFC 5545)
- `moment` — Date manipulation (Obsidian API convention)
- `boon-js` — Boolean expression parsing for complex filter combinations
- `mustache` / `mustache-validator` — Template rendering
- `i18next` — Internationalization
- `flatpickr` — Date picker UI
- `@floating-ui/dom` — Tooltip/popover positioning
