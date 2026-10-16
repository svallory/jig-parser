# Implicit Indentation Control for Edge.js

## Context

Edge.js is being adapted for generating indentation-sensitive output (e.g., Python, YAML). Currently, any indentation added inside block tags for template readability is preserved verbatim in the output. This forces template authors to choose between readable templates and correct output — you can't have both.

**The fix**: Automatically dedent block tag children based on a simple rule, and auto-indent `@include` output based on the tag's position. No new syntax needed.

**Rule**: `dedent = firstContentLineIndent - tagIndent`

```
    @if(show)           ← tag indent = 4
        def hello():    ← first content indent = 8, dedent = 8-4 = 4
            pass        ← 12 - 4 = 8 spaces in output
    @endif

Output:
    def hello():        ← 4 spaces (8 - 4)
        pass            ← 8 spaces (12 - 4)
```

## Repos & Branches

| Repo | Path | Remote | Branch |
|------|------|--------|--------|
| edge-lexer | `/work/edge/agent/repos/lexer` | `svallory/jig-lexer` | `feat/indent-control` |
| edge-parser | `/work/edge/agent/repos/parser` | `svallory/jig-parser` | `feat/indent-control` |
| edge.js | `/work/edge` | (current) | `feat/indent-control` |

During development, edge.js will use `file:` references to the local lexer/parser builds. No npm publishing in this plan.

---

## Step 1: edge-lexer — Expose tag indentation

The lexer already computes `whitespaceLeft` from the TAG_REGEX but discards it. We just need to store it.

### 1.1 Add `indent` to types

**File**: `src/types.ts`

```typescript
// Add to RuntimeTag (after hasBrace):
indent: number

// Add to TagToken (after filename):
indent: number
```

### 1.2 Return `indent` from `getTag()`

**File**: `src/detector.ts` — line ~82, add to the return object:

```typescript
return {
  // ... existing fields ...
  indent: whitespaceLeft,   // ← ADD (already computed at line 63)
}
```

### 1.3 Propagate to TagToken

**File**: `src/tokenizer.ts` — `#getTagNode()` method (~line 133), add to the return object:

```typescript
return {
  // ... existing fields ...
  indent: tag.indent,   // ← ADD
}
```

### 1.4 Tests

**File**: `tests/tokenizer_tags.spec.ts`

- Add tests for `indent` field on tag tokens:
  - Tag at column 0 → `indent: 0`
  - Tag with 4 spaces leading → `indent: 4`
  - Nested block tag (child of another tag) → correct indent
- Existing tests that deep-compare token objects will need `indent: 0` added (since test fixtures typically have tags at column 0)

**File**: `tests/detector.spec.ts`

- Add test that `getTag()` returns correct `indent` value

### 1.5 Build & validate

```bash
cd /work/edge/agent/repos/lexer
bun run quick:test   # all tests pass
bun run build        # builds successfully
```

---

## Step 2: edge-parser — Bump lexer dependency

The parser doesn't need functional changes. It passes `TagToken` through to tag compile methods untouched. But it needs to consume the updated lexer.

### 2.1 Update dependency

**File**: `package.json` — point `edge-lexer` to local build:

```json
"edge-lexer": "file:../lexer"
```

### 2.2 Validate

```bash
cd /work/edge/agent/repos/parser
bun install
bun run quick:test   # all existing tests pass
bun run build
```

---

## Step 3: edge.js — Implement dedent logic

### 3.1 Update dependencies

**File**: `package.json` — point both to local builds:

```json
"edge-lexer": "file:./agent/repos/lexer",
"edge-parser": "file:./agent/repos/parser"
```

Then `bun install`.

### 3.2 Add utility functions

**File**: `src/utils.ts` — add three functions:

#### `computeDedent(children: Token[], tagIndent: number): number`

Scans children to find the first raw token that follows a newline (i.e., the first line of actual content). Measures its leading whitespace and returns `max(0, firstContentIndent - tagIndent)`.

- Starts with `afterNewline = true` (content right after the opening tag line counts)
- Skips newline tokens (but sets `afterNewline = true`)
- When encountering a raw token with `afterNewline = true`: measure leading spaces, return `max(0, leadingSpaces - tagIndent)`
- When encountering a tag token with `afterNewline = true`: return `max(0, (child as TagToken).indent - tagIndent)` — nested tags also need dedenting, and their indent is available
- If no qualifying token found: return `0`

#### `dedentRaw(text: string, amount: number): string`

Strips up to `amount` leading space characters from `text`. If text has fewer leading spaces, strips all of them (never negative). Returns the modified string.

#### `processChildrenDedented(parser, buffer, children, dedent): void`

Iterates children with `afterNewline` tracking (starts `true`):

- **newline token**: `parser.processToken(child, buffer)`, set `afterNewline = true`
- **raw token when `afterNewline && dedent > 0`**: `buffer.outputRaw(dedentRaw(child.value, dedent))`, set `afterNewline = false`
- **tag token (block tag child like `@else`, `@elseif`)**: `parser.processToken(child, buffer)`, set `afterNewline = false` (these tags close/open braces, they don't output raw text)
- **everything else**: `parser.processToken(child, buffer)`, set `afterNewline = false`

### 3.3 Update block tags

Each block tag that iterates `token.children` gets the same pattern:

```typescript
const dedent = computeDedent(token.children, token.indent)
processChildrenDedented(parser, buffer, token.children, dedent)
```

**Tags to update**:

| File | Current code | Notes |
|------|-------------|-------|
| `src/tags/if.ts:52` | `token.children.forEach((child) => parser.processToken(child, buffer))` | Direct replacement |
| `src/tags/unless.ts` | Same pattern as if | Direct replacement |
| `src/tags/each.ts:175` | `token.children.forEach((child) => parser.processToken(child, buffer))` | Replace main children loop |
| `src/tags/each.ts:192` | `elseChildren.forEach((elseChild) => parser.processToken(elseChild, buffer))` | `@else` children use same dedent from the `@each` tag |
| `src/tags/push_to.ts:61-63` | `for (let child of token.children) { parser.processToken(child, stackBuffer) }` | Replace with processChildrenDedented into stackBuffer |
| `src/tags/push_once_to.ts` | Same pattern as push_to | Same replacement |
| `src/tags/component.ts` | Complex — see 3.4 | |

### 3.4 Component tag (most complex)

**File**: `src/tags/component.ts`

The component tag routes children to separate slot buffers. The dedent logic integrates as follows:

**Main slot**: After collecting main slot children (non-`@slot` children), compute `mainDedent = computeDedent(mainSlotChildren, token.indent)`. The existing loop at line ~232 that iterates `token.children` already separates slots from main content. For main slot children, use `processChildrenDedented` into `mainSlot.buffer` (preserving the existing first-newline skip).

**Named slots**: Each `@slot` child tag has its own `indent` field. Compute per-slot dedent: `slotDedent = computeDedent(child.children, (child as TagToken).indent)`. Use `processChildrenDedented` for slot children into their respective buffers.

### 3.5 Include indentation

When `@include` is indented, its output should be indented to match.

**File**: `src/template.ts` — add method:

```typescript
/**
 * Indents every line of output by the given prefix string.
 * First line is NOT indented (it follows the @include tag's position
 * which is already on the correct line).
 */
indentOutput(output: string, indent: string): string {
  if (!indent || !output) return output
  return output.replace(/\n/g, '\n' + indent)
}
```

Note: Only subsequent lines get indented. The first line is emitted inline where the `@include` expression is, so it's already at the right position.

**File**: `src/tags/include.ts` — wrap the render expression:

```typescript
compile(parser, buffer, token) {
  const awaitKeyword = parser.asyncMode ? 'await ' : ''
  const parsed = parseJsArg(parser, token)
  // ... validation ...

  const renderExpr = `${awaitKeyword}${getRenderExpression(parser, parsed)}`

  if (token.indent > 0) {
    const indentStr = JSON.stringify(' '.repeat(token.indent))
    buffer.outputExpression(
      `template.indentOutput(${renderExpr}, ${indentStr})`,
      token.filename,
      token.loc.start.line,
      false
    )
  } else {
    buffer.outputExpression(renderExpr, token.filename, token.loc.start.line, false)
  }
}
```

**File**: `src/tags/include_if.ts` — same wrapping pattern inside the `if` block.

### 3.6 Update existing fixtures

Many `compiled.js` files will change because raw text strings lose leading whitespace. The `index.txt` (rendered output) files may also change for fixtures where block tag content was indented.

**Strategy**: Run the test suite, identify failing fixtures, regenerate `compiled.js` by inspecting actual compiler output, and update `index.txt` by inspecting actual rendered output.

Fixtures likely affected:
- All `if-tag`, `else-tag`, `else-if-tag`, `unless-tag`, `nested-if-tag` — content strings lose indent
- All `each-tag*` — loop body content loses indent
- All `components-*` — slot content loses indent
- All `inject-*` — may lose indent if inside block context
- `newline_fixtures/yaml*` — YAML output fixtures (critical to validate)

### 3.7 New test fixtures

Create fixture directories to specifically test dedent:

| Fixture | Tests |
|---------|-------|
| `fixtures/dedent-if-basic/` | `@if` with indented body, verify dedent |
| `fixtures/dedent-nested-blocks/` | Nested `@if` > `@each`, verify each level dedents independently |
| `fixtures/dedent-no-indent/` | Block tag at col 0 with content at col 0 — no change (backward compat) |
| `fixtures/dedent-include-indent/` | `@include` with leading whitespace, verify output indented |
| `fixtures/dedent-component-slots/` | Component with indented named slots |
| `fixtures/dedent-empty-lines/` | Block with empty lines interspersed — empty lines stay empty |
| `fixtures/dedent-each-else/` | `@each` with `@else`, both branches dedented |
| `fixtures/dedent-mixed-content/` | Raw text + mustache + nested tags in one block |

### 3.8 Unit tests

**File**: `tests/dedent.spec.ts` (new)

- Test `computeDedent()` with mock token arrays
- Test `dedentRaw()` edge cases: more indent than content, zero dedent, empty string
- Test `template.indentOutput()` with single-line, multi-line, empty string

---

## Step 4: Verify end-to-end

```bash
# 1. Build lexer
cd /work/edge/agent/repos/lexer
bun run build

# 2. Build parser
cd /work/edge/agent/repos/parser
bun install && bun run build

# 3. Install & test edge.js
cd /work/edge
bun install
bun run quick:test

# 4. Run specific test files
node --import=@poppinss/ts-exec --enable-source-maps bin/test.ts -- --files="tests/fixtures.spec.ts"
node --import=@poppinss/ts-exec --enable-source-maps bin/test.ts -- --files="tests/async_fixtures.spec.ts"
node --import=@poppinss/ts-exec --enable-source-maps bin/test.ts -- --files="tests/newline_fixtures.spec.ts"
node --import=@poppinss/ts-exec --enable-source-maps bin/test.ts -- --files="tests/dedent.spec.ts"

# 5. Validate YAML fixture (critical for indentation-sensitive output)
# The newline_fixtures/yaml/ fixture should produce identical output since
# its content isn't "over-indented" relative to tags

# 6. Full build
bun run build
```

---

## Critical Files Summary

**Lexer** (3 files):
- `src/types.ts` — add `indent: number` to `RuntimeTag` and `TagToken`
- `src/detector.ts` — return `indent: whitespaceLeft` from `getTag()`
- `src/tokenizer.ts` — pass `indent` through in `#getTagNode()`

**Parser** (1 file):
- `package.json` — point `edge-lexer` to local build

**Edge.js** (10+ files):
- `package.json` — point deps to local builds
- `src/utils.ts` — add `computeDedent`, `dedentRaw`, `processChildrenDedented`
- `src/template.ts` — add `indentOutput` method
- `src/tags/if.ts` — use `processChildrenDedented`
- `src/tags/unless.ts` — use `processChildrenDedented`
- `src/tags/each.ts` — use `processChildrenDedented` for main + else children
- `src/tags/component.ts` — use `processChildrenDedented` for main + named slots
- `src/tags/push_to.ts` — use `processChildrenDedented`
- `src/tags/push_once_to.ts` — use `processChildrenDedented`
- `src/tags/include.ts` — wrap with `template.indentOutput()`
- `src/tags/include_if.ts` — wrap with `template.indentOutput()`
- `tests/dedent.spec.ts` — new test file
- `fixtures/dedent-*/` — new fixture directories
- Many existing `compiled.js` and some `index.txt` fixture files — update expected output

## Execution Order

1. Lexer changes + lexer tests → build
2. Parser dependency update → build
3. Edge.js dependency update → install
4. Add utility functions to `src/utils.ts`
5. Add `indentOutput` to `src/template.ts`
6. Update all block tags (if → unless → each → component → push_to → push_once_to)
7. Update include/include_if tags
8. Create new dedent test fixtures
9. Create unit tests
10. Update existing broken fixtures
11. Run full test suite
12. Full build
