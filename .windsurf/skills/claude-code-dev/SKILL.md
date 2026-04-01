---
name: claude-code-dev
description: How to develop, modify, and extend the Claude Code codebase following its established programming style, patterns, and conventions. Use this skill whenever working on the claude-code source — adding tools, commands, utils, components, types, hooks, services, or fixing bugs. Also use when the user asks about claude-code architecture, coding standards, file organization, or wants to understand how the codebase works. This skill ensures all contributions match the existing codebase style exactly.
---

# Claude Code Development Style Guide

This skill captures the programming style, patterns, and conventions used in the Claude Code codebase (`@anthropic-ai/claude-code`). Follow these guidelines when writing or modifying any code in this project.

## Project Overview

Claude Code is an ESM TypeScript Node.js CLI application that uses Ink (React for terminals) for its UI. It is built with esbuild (transpile-only, no bundling) and uses Bun internally for feature flags and dead code elimination.

**Key technologies:**
- TypeScript with strict types
- ESM modules (`"type": "module"`)
- Ink / React for terminal UI
- Zod v4 for schema validation
- lodash-es (cherry-picked imports)
- esbuild for build, Bun for internal builds

## Directory Structure

The source lives in `src/` with this layout:

```
src/
├── entrypoints/     # CLI entry points
├── commands/        # Slash commands (/help, /config, etc.)
├── tools/           # Model-invocable tools (BashTool, GrepTool, etc.)
├── components/      # Ink React UI components
├── hooks/           # React hooks
├── services/        # Backend services (analytics, MCP, etc.)
├── utils/           # Shared utilities (largest directory)
├── types/           # Pure type definitions (no runtime deps)
├── state/           # App state definitions
├── constants/       # Shared constants
├── context/         # Context providers
├── skills/          # Skill loading and management
├── tasks/           # Background task management
├── bridge/          # Remote bridge connection
├── cli/             # CLI transport and I/O
├── ink/             # Ink framework extensions
├── keybindings/     # Keyboard shortcut handling
└── migrations/      # Config migrations
```

Important top-level source files:
- `Tool.ts` — Core `Tool` type definition and `buildTool()` factory
- `Task.ts` — Task types and helpers
- `commands.ts` — Command registry and loading
- `tools.ts` — Tool registry and assembly
- `context.ts` — System/user context for conversations
- `query.ts` — Main query execution loop
- `main.tsx` — Main REPL application component

## Import Conventions

### Always use `.js` extensions on relative imports

TypeScript ESM convention — the compiled output is `.js`, so imports must reference `.js`:

```typescript
import { getCwd } from '../../utils/cwd.js'
import type { AppState } from './state/AppState.js'
```

### Separate `import type` from value imports

Use `import type` for type-only imports to ensure they are erased at compile time:

```typescript
import type { ToolResultBlockParam } from '@anthropic-ai/sdk/resources/index.mjs'
import type { z } from 'zod/v4'
import type { UUID } from 'crypto'
```

When a module provides both types and values, use separate import statements:

```typescript
import { buildTool } from '../../Tool.js'
import type { ToolDef, ValidationResult } from '../../Tool.js'
```

### Cherry-pick lodash imports

Never import all of lodash. Import individual functions from `lodash-es`:

```typescript
import memoize from 'lodash-es/memoize.js'
import uniqBy from 'lodash-es/uniqBy.js'
```

### Zod imports from v4

```typescript
import { z } from 'zod/v4'
```

### Feature flag conditional imports

Use `feature()` from `bun:bundle` with `require()` for dead code elimination. This pattern allows the build to tree-shake entire modules:

```typescript
import { feature } from 'bun:bundle'

// Dead code elimination: conditional import
/* eslint-disable @typescript-eslint/no-require-imports */
const SleepTool = feature('PROACTIVE')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
/* eslint-enable @typescript-eslint/no-require-imports */
```

### Import order

When import order matters (e.g., ANT-ONLY markers), add:
```typescript
// biome-ignore-all assist/source/organizeImports: ANT-ONLY import markers must not be reordered
```

### Breaking import cycles

Extract pure types to `src/types/` files with no runtime dependencies:

```typescript
// src/types/permissions.ts
/**
 * Pure permission type definitions extracted to break import cycles.
 *
 * This file contains only type definitions and constants with no runtime dependencies.
 */
```

Use re-exports for backwards compatibility when moving types:

```typescript
// Re-export types from centralized location
export type { ToolPermissionRulesBySource }
```

## Naming Conventions

| Category | Convention | Examples |
|---|---|---|
| Functions, variables | `camelCase` | `getCommands`, `isDebugMode`, `logError` |
| Types, interfaces, classes | `PascalCase` | `ToolUseContext`, `TaskStatus`, `AppState` |
| Constants | `UPPER_SNAKE_CASE` | `MAX_HISTORY_ITEMS`, `DEFAULT_HEAD_LIMIT` |
| Tool names | `PascalCase` | `GrepTool`, `BashTool`, `FileEditTool` |
| Tool directories | `PascalCase` | `GrepTool/`, `BashTool/`, `AgentTool/` |
| Command directories | `kebab-case` | `add-dir/`, `install-github-app/` |
| Util files | `camelCase.ts` | `errors.ts`, `debug.ts`, `sleep.ts` |
| Component files | `PascalCase.tsx` | `Spinner.tsx`, `Markdown.tsx` |
| Unused parameters | `_` prefix | `_input`, `_ctx`, `_exhaustive` |

### Intentionally long names for safety

When a name carries a safety obligation, make it long and explicit:

```typescript
export class TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS extends Error { }

type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = string
```

## TypeScript Patterns

### Branded types for ID safety

Use branded types to prevent accidentally mixing up string IDs:

```typescript
export type SessionId = string & { readonly __brand: 'SessionId' }
export type AgentId = string & { readonly __brand: 'AgentId' }

export function asSessionId(id: string): SessionId {
  return id as SessionId
}
```

### Discriminated unions

Use string literal unions for status/mode/behavior enums:

```typescript
export type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'
export type PermissionBehavior = 'allow' | 'deny' | 'ask'
```

### `as const` + derived types

Define literal arrays and derive types from them:

```typescript
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits', 'bypassPermissions', 'default', 'dontAsk', 'plan',
] as const

export type ExternalPermissionMode = (typeof EXTERNAL_PERMISSION_MODES)[number]
```

### `satisfies` for type checking without widening

```typescript
export const INTERNAL_PERMISSION_MODES = [
  ...EXTERNAL_PERMISSION_MODES,
  ...(feature('TRANSCRIPT_CLASSIFIER') ? (['auto'] as const) : ([] as const)),
] as const satisfies readonly PermissionMode[]
```

### Utility types

Use `DeepImmutable<>` for deeply frozen context objects:

```typescript
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
}>
```

### Zod schemas with `lazySchema`

Use `lazySchema()` for tool input schemas to defer evaluation:

```typescript
const inputSchema = lazySchema(() =>
  z.strictObject({
    pattern: z.string().describe('The regex pattern to search for'),
    path: z.string().optional().describe('File or directory to search in'),
  }),
)
type InputSchema = ReturnType<typeof inputSchema>
```

## Commenting Style

### JSDoc for public APIs

Use `/** ... */` with `@param`, `@returns`, `@example` for exported functions:

```typescript
/**
 * Normalize an unknown value into an Error.
 * Use at catch-site boundaries when you need an Error instance.
 */
export function toError(e: unknown): Error {
  return e instanceof Error ? e : new Error(String(e))
}
```

### Explain *why*, not just *what*

Comments should explain the reasoning behind decisions. The codebase favors understanding over mechanical description:

```typescript
// Case-insensitive-safe alphabet (digits + lowercase) for task IDs.
// 36^8 ≈ 2.8 trillion combinations, sufficient to resist brute-force symlink attacks.
const TASK_ID_ALPHABET = '0123456789abcdefghijklmnopqrstuvwxyz'
```

### Reference PRs and issues in comments

```typescript
// PR #19134 blanket-blocked all slash commands from bridge inbound because
// `/model` from iOS was popping the local Ink picker.
```

### Inline implementation notes with `//`

```typescript
// Skip history when running in a tmux session spawned by Claude Code's Tungsten tool.
// This prevents verification/test sessions from polluting the user's real command history.
if (isEnvTruthy(process.env.CLAUDE_CODE_SKIP_PROMPT_HISTORY)) {
  return
}
```

### Section headers in scripts

Use decorated section headers in build/entry scripts:

```typescript
// ── Step 1: Discover all TS/TSX source files ────────────────────────────
// ── Step 2: Build with esbuild (transpile-only, no bundling) ────────────
```

### Explain lint suppression

Always explain why a lint rule is suppressed:

```typescript
// eslint-disable-next-line custom-rules/no-process-exit
process.exit(1)

// biome-ignore lint/suspicious/noConsole:: intentional crash output
console.error('[HARD FAIL]', err.message)
```

### Property/field documentation

Use `/** ... */` inline for type properties:

```typescript
export type Tool = {
  /** Optional aliases for backwards compatibility when a tool is renamed. */
  aliases?: string[]
  /**
   * One-line capability phrase used by ToolSearch for keyword matching.
   * 3–10 words, no trailing period.
   */
  searchHint?: string
  /** Defaults to false. Only set when the tool performs irreversible operations. */
  isDestructive?(input: z.infer<Input>): boolean
}
```

## Error Handling

### Custom error classes

Extend `Error` and set `this.name`:

```typescript
export class AbortError extends Error {
  constructor(message?: string) {
    super(message)
    this.name = 'AbortError'
  }
}
```

Use constructor parameter properties for data-carrying errors:

```typescript
export class ShellError extends Error {
  constructor(
    public readonly stdout: string,
    public readonly stderr: string,
    public readonly code: number,
    public readonly interrupted: boolean,
  ) {
    super('Shell command failed')
    this.name = 'ShellError'
  }
}
```

### Utility functions over raw casts

Never cast `e as NodeJS.ErrnoException`. Use utility functions:

```typescript
// GOOD
const code = getErrnoCode(e)
if (isENOENT(e)) { ... }
if (isFsInaccessible(e)) { ... }

// BAD
if ((e as NodeJS.ErrnoException).code === 'ENOENT') { ... }
```

### Normalize unknown errors at catch boundaries

```typescript
import { toError, errorMessage } from './utils/errors.js'

try { ... } catch (err) {
  logError(toError(err))          // when you need an Error instance
  const msg = errorMessage(err)   // when you only need the string
}
```

### Graceful degradation

Never crash on non-critical failures. Use fallback returns:

```typescript
try {
  const [skillDirCommands, pluginSkills] = await Promise.all([
    getSkillDirCommands(cwd).catch(err => {
      logError(toError(err))
      return []
    }),
    getPluginSkills().catch(err => {
      logError(toError(err))
      return []
    }),
  ])
} catch (err) {
  // This should never happen since we catch at the Promise level, but defensive
  logError(toError(err))
  return { skillDirCommands: [], pluginSkills: [], bundledSkills: [], builtinPluginSkills: [] }
}
```

### Silent catch for expected failures

Use comments to explain silent catches:

```typescript
} catch {
  // Silently fail if symlink creation fails
}

} catch {
  // pass
}

} catch {
  // Directory already exists
}
```

## Function Patterns

### `buildTool()` factory for tools

All tools must use `buildTool()` which provides safe defaults:

```typescript
export const GrepTool = buildTool({
  name: GREP_TOOL_NAME,
  inputSchema,
  // ... tool methods
})
```

### `memoize()` for expensive computations

Use lodash `memoize` for costly one-time computations:

```typescript
export const isDebugMode = memoize((): boolean => {
  return runtimeDebugEnabled || isEnvTruthy(process.env.DEBUG)
})
```

Clear memoize caches when underlying state changes:

```typescript
export function enableDebugLogging(): boolean {
  runtimeDebugEnabled = true
  isDebugMode.cache.clear?.()
  return wasActive
}
```

### Arrow functions vs `function` declarations

- **Arrow functions**: short callbacks, inline expressions, module-level constants
- **`function` declarations**: named exported functions, hoisted helpers, methods needing `this`

```typescript
// Arrow for constants / short callbacks
const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
const isEnabled = tools.map(tool => tool.isEnabled())

// function declaration for named helpers
function applyHeadLimit<T>(items: T[], limit: number | undefined): { ... }
export function findCommand(name: string, commands: Command[]): Command | undefined { ... }
```

### `void` for fire-and-forget promises

Explicitly mark intentionally unhandled promises:

```typescript
void addToPromptHistory(command)
void flushPromptHistory(retries + 1)
void updateLatestDebugLogSymlink()
```

### Guard clause / early return

Return early to avoid deep nesting:

```typescript
export function logError(error: unknown): void {
  const err = toError(error)
  if (isEnvTruthy(process.env.DISABLE_ERROR_REPORTING)) {
    return
  }
  // ... main logic
}
```

### Async generators for streaming data

```typescript
export async function* getHistory(): AsyncGenerator<HistoryEntry> {
  for await (const entry of makeLogEntryReader()) {
    if (!entry || typeof entry.project !== 'string') continue
    yield await logEntryToHistoryEntry(entry)
  }
}
```

### Defensive defaults with `??` and `|| []`

```typescript
const modelUsage = getUsageForModel(model) ?? {
  inputTokens: 0, outputTokens: 0, cacheReadInputTokens: 0, ...
}
return TASK_ID_PREFIXES[type] ?? 'x'
```

## Tool Architecture

Each tool lives in its own directory under `src/tools/`:

```
ToolName/
├── ToolName.ts     # Core logic, buildTool() call
├── prompt.ts       # Tool name constant + description function
└── UI.tsx          # React rendering (renderToolUseMessage, renderToolResultMessage)
```

### Tool definition pattern

```typescript
// prompt.ts
export const GREP_TOOL_NAME = 'Grep'
export function getDescription(): string {
  return `A powerful search tool built on ripgrep ...`
}

// ToolName.ts
import { buildTool } from '../../Tool.js'
import type { ToolDef } from '../../Tool.js'

const inputSchema = lazySchema(() => z.strictObject({ ... }))
type InputSchema = ReturnType<typeof inputSchema>

export const GrepTool = buildTool({
  name: GREP_TOOL_NAME,
  inputSchema,
  async description(input, options) { return getDescription() },
  async prompt(options) { return getDescription() },
  async call(args, context, canUseTool, parentMessage, onProgress) { ... },
  isReadOnly: () => true,
  isConcurrencySafe: () => true,
  maxResultSizeChars: 20_000,
  renderToolUseMessage(input, options) { ... },
  renderToolResultMessage(content, progress, options) { ... },
  // ...
})
```

## Command Architecture

Commands live in `src/commands/` in their own directories:

```
command-name/
├── command-name.ts   # or command-name.tsx for JSX commands
└── index.ts          # Default export of the Command object
```

Command types: `'prompt'` (model-invocable), `'local'` (runs locally), `'local-jsx'` (renders Ink UI).

## State Management

- Module-level `let` for mutable state (with getter/setter pairs)
- `registerCleanup()` for process exit cleanup
- `AppState` threaded through `ToolUseContext`
- `setAppState(f: (prev: AppState) => AppState)` — functional updates

```typescript
let systemPromptInjection: string | null = null

export function getSystemPromptInjection(): string | null {
  return systemPromptInjection
}

export function setSystemPromptInjection(value: string | null): void {
  systemPromptInjection = value
  getUserContext.cache.clear?.()
  getSystemContext.cache.clear?.()
}
```

## Testing Conventions

- `_resetForTesting()` / `_resetErrorLogForTesting()` internal helpers with `@internal` tag
- `process.env.NODE_ENV === 'test'` guards for test-only behavior
- Test-only tool exports (e.g., `TestingPermissionTool`)

```typescript
/**
 * Reset error log state for testing purposes only.
 * @internal
 */
export function _resetErrorLogForTesting(): void {
  errorLogSink = null
  errorQueue.length = 0
  inMemoryErrorLog = []
}
```

## Internal vs External Code

- `process.env.USER_TYPE === 'ant'` gates internal-only features
- `feature('FLAG_NAME')` for feature-flagged code
- `INTERNAL_ONLY_COMMANDS` array for ant-only commands
- Conditional `require()` with `/* eslint-disable/enable */` wrapping

## Formatting Rules

- No semicolons (the codebase omits them)
- Single quotes for strings
- Trailing commas in multiline constructs
- 2-space indentation
- No explicit `return undefined` — just `return`
- Template literals for string interpolation: `` `Found ${count} files` ``
- Ternary expressions for simple conditionals (multiline for complex)
- Spread for combining arrays: `[...builtInTools, ...mcpTools]`
- `Promise.all()` for parallel async work
- `as const` on literal arrays/objects

## Quick Reference: Adding a New Tool

1. Create `src/tools/MyTool/` directory
2. Create `prompt.ts` with `MY_TOOL_NAME` constant and `getDescription()` function
3. Create `MyTool.ts` with `buildTool({ ... })` export
4. Create `UI.tsx` with render functions
5. Register in `src/tools.ts` (add to `getAllBaseTools()`)
6. Use `lazySchema()` + `z.strictObject()` for input schema
7. Use `import type` for type-only imports, `.js` extensions on all relative imports

## Quick Reference: Adding a New Command

1. Create `src/commands/my-command/` directory
2. Create `my-command.ts` (or `.tsx`) with command logic
3. Create `index.ts` with default export of the `Command` object
4. Register in `src/commands.ts` (import and add to `COMMANDS()` array)
5. Set `type`, `name`, `description`, `source: 'builtin'`
