# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**edge-parser** converts Edge template strings into executable JavaScript function bodies. It takes lexer tokens (from `edge-lexer`), parses JavaScript expressions via **Acorn**, transforms the AST to prefix unknown variables with `state.`, and generates JavaScript code via **astring**. Published as `edge-parser` on npm.

Dependency chain: lexer → **parser** → jig (this is the middle package).

## Commands

```bash
# Run full test suite (lint + tests with coverage)
bun test

# Run tests without lint/coverage
bun run quick:test

# Run a specific test file
node --import=@poppinss/ts-exec --enable-source-maps bin/test.ts -- --files="tests/parser.spec.ts"

# Lint / Format / Type check
bun run lint
bun run format
bun run typecheck

# Build (tsdown + tsc declarations → build/)
bun run build
```

Testing uses **Japa** (`@japa/runner`) with `@japa/assert`. Config is in `bin/test.ts`.

## Architecture

### Core Classes

- **Parser** (`src/parser/main.ts`) — Orchestrates parsing. Holds a `Stack` for variable scope, exposes `utils` (generateAST, transformAst, stringify, makeCallable, makeMemberAccessor). Key methods: `tokenize(template, options)` to get lexer tokens, `processToken(token, buffer)` to compile each token.

- **EdgeBuffer** (`src/edge_buffer/index.ts`) — Accumulates compiled template code. Methods: `outputRaw()`, `outputExpression()`, `writeStatement()`, `writeExpression()`, `wrap()`, `flush()`. Injects `$lineNumber`/`$filename` tracking and try/catch error handling.

- **Stack** (`src/stack/index.ts`) — Variable scope tracker. `defineScope()`/`clearScope()` for nested scopes (arrow functions, etc.), `defineVariable()` to register locals, `has()` to check if a name is local.

### Expression Transformers (`src/expressions/`)

27 expression types, each in its own file with a default export containing `toStatement(statement, filename, parser)`. Recursively transforms child nodes via `transformAst()`. Key behavior: **Identifier** nodes are prefixed with `state.` unless they're local variables, function params, or JS globals.

### Variable Resolution Order

1. `parser.options.localVariables` — Directly accessible (e.g., `$filename`, `state`, `$context`)
2. `parser.stack.has(name)` — Locally defined (function params, destructured vars)
3. JavaScript globals (`Math`, `console`, etc.)
4. **Default**: Wrap as `state.variableName`

### Key Parser Options

- `statePropertyName` — Property for variable access (e.g., `state`)
- `escapeCallPath` — Path to escape function for mustache output
- `async` — Enable async/await support
- `onMustache` / `onTag` — Token transformers
- `claimTag` — Dynamic tag registration callback
- `localVariables` — Variables accessible without `state.` prefix

### Exports

```
edge-parser       → Parser, Stack, EdgeBuffer, expressions (name map)
edge-parser/types → ParserTagDefinitionContract, ParserOptions, AcornLoc
```

### Test Fixtures

Fixtures in `fixtures/` contain `index.edge` (template) and `index.js` (expected compiled output). Tests verify that compilation matches expected JavaScript.

Test helpers in `tests_helpers/`: `normalizeNewLines()` and `normalizeFilename()` for cross-platform test consistency.

## Conventions

- Each expression transformer is isolated in its own file under `src/expressions/`
- Acorn AST nodes are patched in-place with corrected line/column from lexer positions
- Private class fields (`#field`) throughout
- All imports use `.js` extensions (ESM)
- AdonisJS ecosystem configs: `@adonisjs/eslint-config`, `@adonisjs/prettier-config`, `@adonisjs/tsconfig`
- Conventional commits (Angular preset) enforced by `release-it`
