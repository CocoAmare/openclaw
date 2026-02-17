# CLAUDE.md — OpenClaw AI Assistant Guide

## Project Overview

OpenClaw is a multi-channel personal AI assistant gateway. It bridges messaging
channels (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Google Chat,
Microsoft Teams, Matrix, and more) to LLM backends. The project is a monorepo
containing the core Node.js gateway, a Lit-based web UI, native iOS/macOS/Android
apps, a plugin/extension system, and a skill library.

## Quick Reference

| What | Command |
|---|---|
| Install deps | `pnpm install` |
| Build | `pnpm build` |
| Type check | `pnpm tsgo` |
| Lint | `pnpm lint` |
| Format | `pnpm format` |
| All checks | `pnpm check` (format:check + tsgo + lint) |
| Unit tests | `pnpm test` |
| Fast unit tests (no extensions) | `pnpm test:fast` |
| E2E tests | `pnpm test:e2e` |
| Single test file | `npx vitest run path/to/file.test.ts` |
| Dev server | `pnpm dev` |
| Gateway dev | `pnpm gateway:dev` |
| UI dev | `pnpm ui:dev` |
| Build UI | `pnpm ui:build` |

## Repository Structure

```
openclaw/
├── src/                    # Core TypeScript source
│   ├── agents/             # Agent execution, skills, subagents
│   ├── channels/           # Base channel infrastructure and plugin types
│   ├── cli/                # CLI commands and TUI
│   ├── config/             # Configuration loading, validation, env vars
│   ├── gateway/            # Gateway HTTP/WS server, server-methods/
│   ├── plugins/            # Plugin discovery, loader, registry, hooks
│   ├── infra/              # Infrastructure (state, migrations, updates)
│   ├── memory/             # Memory/knowledge management
│   ├── providers/          # Auth providers (Copilot, Google, Qwen, etc.)
│   ├── discord/            # Discord channel implementation
│   ├── slack/              # Slack channel implementation
│   ├── telegram/           # Telegram channel implementation
│   ├── whatsapp/           # WhatsApp channel implementation
│   ├── signal/             # Signal channel implementation
│   ├── imessage/           # iMessage channel implementation
│   ├── browser/            # Web browsing capabilities
│   ├── tts/                # Text-to-speech
│   ├── media/              # Media handling
│   ├── media-understanding/# Media analysis
│   ├── sessions/           # Session management
│   ├── routing/            # Message routing
│   ├── hooks/              # Bundled hooks system
│   ├── cron/               # Scheduled tasks
│   ├── security/           # Security utilities
│   ├── logging/            # Logging (tslog-based, subsystem loggers)
│   ├── plugin-sdk/         # Plugin SDK exports
│   ├── shared/             # Shared utilities (config eval, frontmatter)
│   ├── test-helpers/       # Test utilities (used from src/)
│   ├── test-utils/         # More test utilities
│   └── types/              # Shared TypeScript type definitions
├── extensions/             # 41 plugin extensions (channels, auth, services)
├── packages/               # Workspace packages (clawdbot, moltbot)
├── apps/
│   ├── macos/              # Swift macOS app
│   ├── ios/                # Swift iOS app
│   ├── android/            # Kotlin Android app
│   └── shared/             # Shared Swift code (OpenClawKit)
├── ui/                     # Lit web components UI (Vite build)
├── skills/                 # 54 skill definitions (SKILL.md files)
├── docs/                   # Mintlify documentation site
├── scripts/                # Build, test, CI, packaging scripts
├── test/                   # Test setup, helpers, fixtures, mocks
├── git-hooks/              # Pre-commit hook (oxlint + oxfmt)
└── vendor/                 # Vendored dependencies
```

## Tech Stack

- **Runtime:** Node.js >= 22.12.0
- **Language:** TypeScript (strict mode), ES modules
- **Package manager:** pnpm 10.x (monorepo workspaces)
- **Build:** tsdown (bundler), tsgo (type checker from @typescript/native-preview)
- **Test:** Vitest (unit, e2e, live, gateway, extensions configs)
- **Lint:** oxlint (with unicorn, typescript, oxc plugins)
- **Format:** oxfmt (with import sorting)
- **HTTP:** Express 5
- **UI:** Lit 3 web components + Vite
- **Schema validation:** Zod 4, TypeBox
- **Logging:** tslog (subsystem-based loggers)
- **Mobile:** Swift 6.2 (iOS/macOS), Kotlin (Android)

## Coding Conventions

### TypeScript

- **Strict mode** is enforced. No `any` types — `typescript/no-explicit-any` is
  an error in oxlint. Use `unknown` and narrow with type guards.
- **ESM imports with `.js` extensions.** All local imports must include the `.js`
  extension (e.g., `import { foo } from "./bar.js"`). This is required by the
  NodeNext module resolution.
- **Target:** ES2023. Use modern JS features (structuredClone, Array.at, etc.).
- **File size limit:** 500 lines max per `.ts` file (enforced by `check:loc`).
  Split large files into focused modules.

### Naming

- **Files:** kebab-case (`gateway-rpc.ts`, `parse-bytes.ts`)
- **Test files:** `filename.test.ts`, `filename.e2e.test.ts`, `filename.live.test.ts`
- **Functions:** camelCase, prefixed with action verbs (`runAgent`, `parseConfig`, `resolveDefaults`)
- **Types/Classes:** PascalCase (`GatewayRpcOpts`, `ChannelPlugin`)
- **Error classes:** PascalCase with `Error` suffix (`WizardCancelledError`)
- **Constants:** UPPER_SNAKE_CASE (`DEFAULT_DELAY_MS`, `ADMIN_SCOPE`)

### Imports

oxfmt auto-sorts imports. The conventional order is:
1. Node builtins (`import fs from "node:fs"`)
2. External packages
3. Local imports (with `.js` extension)

Type-only imports use the `type` keyword: `import type { Foo } from "./foo.js"`.

### Error Handling

- Custom error classes extending `Error` for domain-specific errors.
- Discriminated union return types: `{ ok: true; result } | { ok: false; error }`.
- Wrap caught errors with `String(err)` for safe stringification.

### Logging

Use subsystem loggers, not `console.log`:
```ts
import { createSubsystemLogger } from "../logging/subsystem.js";
const log = createSubsystemLogger("my-feature");
log.info("message");
log.warn(`something: ${details}`);
```

### Control UI (Lit)

The web UI uses Lit with **legacy decorators** (`experimentalDecorators: true`).
Use the legacy style for reactive fields:
```ts
@state() foo = "bar";
@property({ type: Number }) count = 0;
```
Do not use standard `accessor` decorators — the current build tooling does not support them.

## Testing

### Running Tests

```bash
pnpm test              # Parallel unit tests (all configs)
pnpm test:fast         # Unit tests only (no extensions/gateway)
pnpm test:e2e          # End-to-end tests
pnpm test:live         # Live API tests (requires credentials)
npx vitest run src/path/to/file.test.ts   # Single file
```

### Test Patterns

- **Framework:** Vitest with `describe`/`it` blocks.
- **Mocking:** Use `vi.hoisted()` for mock declarations before imports, then
  `vi.mock("./module.js", () => ({ ... }))`.
- **Setup:** Global test setup in `test/setup.ts` — provides isolated HOME dirs,
  plugin registry stubs, and process warning filters.
- **Cleanup:** `afterEach` to clear mocks. Fake timers are auto-restored.
- **Six vitest configs:** `vitest.config.ts` (base), `vitest.unit.config.ts`,
  `vitest.e2e.config.ts`, `vitest.gateway.config.ts`, `vitest.live.config.ts`,
  `vitest.extensions.config.ts`.

### Coverage

Coverage thresholds (for `src/` only): 70% lines, 70% functions, 55% branches.
Run with `pnpm test:coverage`.

## Linting & Formatting

### Commands

```bash
pnpm format            # Auto-format with oxfmt
pnpm format:check      # Check formatting (CI)
pnpm lint              # Run oxlint (type-aware)
pnpm lint:fix          # Auto-fix lint issues + format
pnpm check             # format:check + tsgo + lint (full CI check)
```

### Key Lint Rules

- `typescript/no-explicit-any`: **error** — avoid `any`, use `unknown`
- `curly`: **error** — always use braces for control flow
- Correctness, perf, suspicious categories: all **error**
- To suppress a rule inline: `// oxlint-disable-next-line rule-name`

### Pre-commit Hook

The `git-hooks/pre-commit` hook runs oxlint (with --fix) and oxfmt (with --write)
on staged files automatically.

### Swift (iOS/macOS)

- SwiftFormat + SwiftLint configured via `.swiftformat` and `.swiftlint.yml`
- Run: `pnpm format:swift`, `pnpm lint:swift`

## Build System

```bash
pnpm build             # Full build (tsdown + plugin-sdk DTS + post-build scripts)
pnpm build:plugin-sdk:dts  # Plugin SDK type declarations only
pnpm ui:build          # Build Lit web UI
```

The build uses tsdown to bundle multiple entry points (index, entry, daemon-cli,
plugin-sdk, hooks, etc.). Post-build scripts copy assets, generate build info,
and write CLI compatibility shims.

## Plugin / Extension System

Extensions live in `extensions/` and follow this structure:
```
extensions/my-extension/
├── index.ts               # Main entry, exports plugin object
├── package.json           # Workspace package
├── openclaw.plugin.json   # Plugin manifest (id, name, description)
└── *.test.ts              # Tests
```

Plugins register via the plugin API:
```ts
const plugin = {
  id: "my-channel",
  name: "My Channel",
  description: "...",
  configSchema: emptyPluginConfigSchema(),
  register(api: OpenClawPluginApi) {
    api.registerChannel({ plugin: myChannelPlugin });
  },
};
export default plugin;
```

## Environment & Configuration

- **Config format:** JSON5 (supports comments, trailing commas)
- **Env vars:** Prefixed with `OPENCLAW_` (e.g., `OPENCLAW_STATE_DIR`,
  `OPENCLAW_CONFIG_PATH`, `OPENCLAW_GATEWAY_PORT`, `OPENCLAW_PROFILE`)
- **Config hierarchy:** defaults -> config file -> env vars -> runtime overrides
- **Validation:** Zod schemas in `src/config/zod-schema.ts`

## Architecture Key Concepts

- **Gateway:** The central HTTP/WS server that manages channels, sessions,
  agents, and plugins. Routes messages between channels and LLM backends.
- **Channels:** Messaging platform integrations (WhatsApp, Telegram, etc.),
  implemented as plugins with standard interfaces.
- **Agents:** LLM-powered execution contexts with skill access, workspace
  isolation, and session management.
- **Skills:** Declarative tool definitions (in `skills/`) that agents can invoke.
  Each skill has a `SKILL.md` with frontmatter metadata.
- **Plugins:** Runtime-discovered extensions that register channels, providers,
  or services via the plugin API.
- **Sessions:** Conversation state management with file-based persistence.

## PR Checklist

Before submitting:
1. `pnpm check` passes (formatting, types, lint)
2. `pnpm test` passes
3. Files stay under 500 lines
4. No `any` types introduced
5. All imports use `.js` extensions
6. Tests added for new functionality
