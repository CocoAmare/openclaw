# CLAUDE.md — OpenClaw Comprehensive Architecture Guide

## Project Overview

OpenClaw is a multi-channel personal AI assistant gateway. It bridges messaging
channels (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Google Chat,
Microsoft Teams, Matrix, and more) to LLM backends. The project is a monorepo
containing the core Node.js gateway, a Lit-based web UI, native iOS/macOS/Android
apps, a plugin/extension system, and a skill library.

The product is a **single-user personal assistant** — the gateway is the control
plane, and the assistant is the product. Users interact via whichever channels
they already use, and the gateway orchestrates agent execution, skill invocation,
memory retrieval, and message delivery across all of them.

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
│   ├── agents/             # Agent execution, skills, subagents, tools
│   ├── channels/           # Base channel infrastructure and plugin types
│   ├── cli/                # CLI commands and TUI
│   ├── config/             # Configuration loading, validation, env vars
│   ├── gateway/            # Gateway HTTP/WS server, server-methods/
│   ├── plugins/            # Plugin discovery, loader, registry, hooks
│   ├── infra/              # Infrastructure (state, migrations, updates)
│   ├── memory/             # Memory/knowledge management (embeddings, hybrid search)
│   ├── providers/          # Auth providers (Copilot, Google, Qwen, etc.)
│   ├── discord/            # Discord channel implementation
│   ├── slack/              # Slack channel implementation
│   ├── telegram/           # Telegram channel implementation
│   ├── whatsapp/           # WhatsApp channel implementation
│   ├── signal/             # Signal channel implementation
│   ├── imessage/           # iMessage channel implementation
│   ├── browser/            # Web browsing capabilities
│   ├── tts/                # Text-to-speech (OpenAI, ElevenLabs, Edge)
│   ├── media/              # Media handling (fetch, store, parse)
│   ├── media-understanding/# Media analysis (transcription, captioning)
│   ├── sessions/           # Session management and persistence
│   ├── routing/            # Message routing (bindings, session keys)
│   ├── hooks/              # Bundled hooks system (lifecycle events)
│   ├── cron/               # Scheduled tasks (cron expressions, intervals)
│   ├── security/           # Security audit, fixes, tool policies
│   ├── logging/            # Logging (tslog-based, subsystem loggers)
│   ├── plugin-sdk/         # Plugin SDK exports for extension authors
│   ├── shared/             # Shared utilities (config eval, frontmatter)
│   ├── test-helpers/       # Test utilities (used from src/)
│   ├── test-utils/         # More test utilities
│   └── types/              # Shared TypeScript type definitions
├── extensions/             # 41+ plugin extensions (channels, auth, services)
├── packages/               # Workspace packages (clawdbot, moltbot)
├── apps/
│   ├── macos/              # Swift macOS app
│   ├── ios/                # Swift iOS app
│   ├── android/            # Kotlin Android app
│   └── shared/             # Shared Swift code (OpenClawKit)
├── ui/                     # Lit web components UI (Vite build)
├── skills/                 # 54+ skill definitions (SKILL.md files)
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

---

## Architecture Deep Dive

### System Overview — How Everything Connects

```
┌───────────────────────────────────────────────────────────────────┐
│ EXTERNAL CLIENTS                                                  │
│ (CLI, WebChat, iOS app, Android app, macOS app, SDK)             │
└──────────────┬────────────────────────────────────────────────────┘
               │ WebSocket (JSON frames: req/res/evt)
               ▼
┌───────────────────────────────────────────────────────────────────┐
│ GATEWAY SERVER (central orchestrator)                             │
│                                                                   │
│  ┌───────────┐  ┌────────────┐  ┌─────────────────────────────┐ │
│  │ HTTP Layer │  │ WebSocket  │  │ RPC Handler Registry        │ │
│  │ (hooks,    │  │ (handshake,│  │ (100+ methods: agent,       │ │
│  │  webhooks, │  │  auth,     │  │  chat, send, config,        │ │
│  │  Slack,    │  │  frames,   │  │  sessions, cron, wizard,    │ │
│  │  OpenAI    │  │  heartbeat)│  │  skills, models, health)    │ │
│  │  API,      │  │            │  │                             │ │
│  │  ControlUI)│  │            │  │                             │ │
│  └─────┬──────┘  └─────┬──────┘  └──────────┬──────────────────┘ │
│        │               │                     │                    │
│        ▼               ▼                     ▼                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                  CORE SUBSYSTEMS                            │  │
│  │                                                            │  │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌─────────────────┐    │  │
│  │  │ Agent  │ │Channel │ │Plugin  │ │ Config           │    │  │
│  │  │Executor│ │Manager │ │Registry│ │ (JSON5+Zod)      │    │  │
│  │  └───┬────┘ └───┬────┘ └───┬────┘ └─────────────────┘    │  │
│  │      │          │          │                               │  │
│  │  ┌───┴────┐ ┌───┴────┐ ┌──┴─────┐ ┌─────────────────┐   │  │
│  │  │Session │ │Message │ │ Hooks  │ │ Memory           │   │  │
│  │  │Store   │ │Router  │ │ System │ │ (embeddings+FTS) │   │  │
│  │  └────────┘ └────────┘ └────────┘ └─────────────────┘   │  │
│  │                                                            │  │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌─────────────────┐   │  │
│  │  │ Cron   │ │Security│ │ Media  │ │ TTS              │   │  │
│  │  │Service │ │ Audit  │ │Pipeline│ │ Pipeline          │   │  │
│  │  └────────┘ └────────┘ └────────┘ └─────────────────┘   │  │
│  └────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
        │                                        │
        ▼                                        ▼
┌─────────────────┐                ┌─────────────────────────────┐
│ LLM Backends    │                │ Messaging Channels           │
│ (Anthropic,     │                │ (WhatsApp, Telegram, Slack,  │
│  OpenAI, Google,│                │  Discord, Signal, iMessage,  │
│  Ollama, etc.)  │                │  Teams, Matrix, WebChat,     │
│                 │                │  Google Chat, etc.)           │
└─────────────────┘                └─────────────────────────────┘
```

---

### 1. Gateway Server — The Central Orchestrator

The gateway is the heart of OpenClaw. It's a Node.js HTTP/WebSocket server
that coordinates all subsystems.

**Entry:** `src/gateway/server.impl.ts` via `startGatewayServer(port, opts)`

**HTTP handler chain (processed in order):**
1. Hooks endpoint (`POST /hooks`)
2. Tools invoke HTTP (`/tools/invoke`)
3. Slack webhooks (`POST /slack/events`)
4. Plugin HTTP routes (`/api/channels/*`)
5. OpenAI-compatible API (`POST /v1/responses`, `POST /v1/chat/completions`)
6. Canvas host (if enabled)
7. Control UI (web dashboard at `/`)
8. 404 fallback

**WebSocket protocol (v6):** Binary-safe JSON frames:
- `RequestFrame` — `{ type: "req", id: UUID, method: string, params? }`
- `ResponseFrame` — `{ type: "res", id: UUID, ok: boolean, payload?, error? }`
- `EventFrame` — `{ type: "evt", seq?, event: string, payload? }`

**Authentication modes:** `none`, `token` (Bearer), `password`, `trusted-proxy`
(X-OpenClaw-User headers), `tailscale` (network identity), `device-token`
(Ed25519 signatures with nonce, 10-min rotation).

**Bind modes:** `loopback` (127.0.0.1), `lan` (0.0.0.0), `tailnet`, `auto`

**Authorization scopes:** `operator.admin`, `operator.read`, `operator.write`,
`operator.approvals`, `operator.pairing`

**Client connection flow:**
1. WebSocket open, 750ms queue delay
2. Server sends `connect.challenge` event with nonce
3. Client sends `connect` RPC with auth + capabilities
4. Server responds `hello-ok` with protocol version + metadata
5. Heartbeat ticks every 30s; stale detection triggers reconnect
6. Auto-reconnect with exponential backoff (1s to 30s)

**RPC methods (100+, by domain):**

| Domain | Key Methods |
|---|---|
| Agent | `agent`, `agent.wait`, `agents.list`, `agents.create`, `agents.update` |
| Chat | `chat.send`, `chat.history`, `chat.abort` |
| Sessions | `sessions.list`, `sessions.preview`, `sessions.patch`, `sessions.reset` |
| Channels | `channels.status`, `channels.logout` |
| Config | `config.get`, `config.set`, `config.patch`, `config.schema` |
| Nodes | `node.list`, `node.describe`, `node.invoke` |
| Cron | `cron.list`, `cron.add`, `cron.update`, `cron.run` |
| Wizard | `wizard.start`, `wizard.next`, `wizard.cancel` |
| System | `health`, `status`, `usage.status`, `models.list`, `skills.status` |

**Event broadcasting:** Gateway pushes events to all connected clients:
`agent`, `chat` (streaming), `presence`, `tick`, `talk.mode`, `shutdown`,
`cron`, `node.pair.*`, `device.pair.*`, `exec.approval.*`

---

### 2. Agent System — LLM Execution Engine

Agents are the intelligence layer. Each agent is an LLM-powered execution
context with its own workspace, skills, tools, and session history.

**Entry:** `src/agents/pi-embedded-runner/run.ts` via `runEmbeddedPiAgent()`
**Core:** `src/agents/pi-embedded-runner/run/attempt.ts` via `runEmbeddedAttempt()`

#### Agent Lifecycle (user message to response)

```
1. USER message arrives via channel
       ↓
2. GATEWAY routes to session (resolve-route.ts)
   binding lookup → session key derivation
       ↓
3. SESSION SETUP
   - Acquire exclusive write lock
   - Repair session file if needed
   - Open SessionManager + SettingsManager
       ↓
4. ENVIRONMENT PREPARATION
   - Resolve workspace dir + sandbox context
   - Load workspace skills → build skills prompt
   - Resolve bootstrap files (BOOTSTRAP.md, SOUL.md, IDENTITY.md, etc.)
       ↓
5. TOOL CREATION
   - Build core tools (exec, bash, file read/write/edit)
   - Build OpenClaw tools (message, sessions, memory, cron, browser)
   - Apply tool policies (profile-based, owner-only, depth-based)
   - Wrap with hooks + abort signal + param normalization
       ↓
6. SYSTEM PROMPT CONSTRUCTION
   - Runtime context (host, OS, arch, node, shell, timezone)
   - Channel capabilities (reactions, edit, threads)
   - Skills manifest (up to 150 skills, ~30KB)
   - Workspace notes (BOOTSTRAP.md)
   - Memory search hints, sandbox restrictions, model aliases
       ↓
7. SESSION HISTORY VALIDATION
   - Sanitize (strip invalid messages)
   - Provider-specific validation (Gemini, Anthropic)
   - Limit turns to fit context window
   - Repair tool_use/tool_result pairing
       ↓
8. PROMPT EXECUTION
   - Fire before_prompt_build hooks
   - Load prompt images for vision models
   - Call session.prompt(text, options) on LLM
       ↓
9. STREAMING + TOOL LOOP
   - Stream text deltas for real-time display
   - On tool_use → execute tool → inject tool_result → loop
   - Loop detection prevents infinite tool call cycles
   - Block replies chunked at paragraph boundaries
       ↓
10. FINALIZATION
    - Flush pending tool results
    - Persist session to disk, release write lock
    - Fire agent_end hooks, return usage stats
```

#### Agent Configuration

Per-agent config in `src/config/zod-schema.agents.ts`:
- `model` — Primary model + fallback chain
- `skills` — Allowlist (undefined = all eligible, [] = none)
- `workspace` — Isolated workspace directory
- `identity` — Name, avatar, personality
- `subagents` — `maxSpawnDepth` (default 1), `maxChildrenPerAgent` (default 5)
- `tools` — exec/fs policy, loop detection
- `sandbox` — Execution isolation
- `groupChat` — Multi-user settings

Resolution: Agent-specific → defaults → hardcoded fallbacks

#### Agent Workspace Layout

```
~/.openclaw/workspace/
├── BOOTSTRAP.md            # User instructions for agent
├── SOUL.md                 # Agent personality and values
├── IDENTITY.md             # Identity/profile information
├── AGENTS.md               # Subagent definitions
├── USER.md                 # User preferences
├── HEARTBEAT.md            # Periodic reminder prompts
├── MEMORY.md               # Persistent notes
├── TOOLS.md                # Tool usage guidelines
├── .openclaw/
│   └── workspace-state.json
├── skills/                 # Local skill definitions
└── ...                     # Project files
```

---

### 3. Skill System — Declarative Tool Definitions

Skills are declared as `SKILL.md` files with YAML frontmatter and Markdown body.

**Sources:** Bundled (`skills/`), workspace (`{workspace}/skills/`), plugin,
downloaded from community.

**SKILL.md format:**
```yaml
---
name: tmux
description: Remote-control tmux sessions
metadata:
  openclaw:
    emoji: "\U0001F9F5"
    os: ["darwin", "linux"]
    requires:
      bins: ["tmux"]
    install:
      - id: brew
        kind: brew
        formula: tmux
---
# Usage instructions (injected into system prompt)
```

**Loading:** Discover → parse frontmatter → filter by eligibility (OS, binaries,
agent allowlist) → build prompt (max 150 skills, ~30KB) → inject into system prompt.

---

### 4. Tool System — Agent Execution Capabilities

**Location:** `src/agents/tools/`

| Tool | Purpose |
|---|---|
| `exec` / `bash` | Shell commands (pty/non-pty, elevated) |
| `read` / `write` / `edit` | File system operations |
| `message` | Send messages across channels |
| `sessions_send` | Inter-session messaging |
| `sessions_spawn` | Spawn child agents (subagents) |
| `subagents` | List, kill, steer spawned subagents |
| `cron` | Schedule recurring tasks |
| `memory` | Search persistent knowledge base |
| `web_fetch` | HTTP requests |
| `web_search` | Web search |
| `browser` | Full browser automation |
| `image` | Image generation (DALL-E) |
| `nodes` | Gateway RPC to remote nodes |
| `discord_actions` | Discord-specific operations |
| `slack_actions` | Slack operations |
| `telegram_actions` | Telegram operations |

**Policy pipeline:** Profile-based allowlist → owner-only gating → granular
allowlists → depth-based restrictions (subagent tools gated by spawn depth).

**Wrapping chain:** Abort signal → before-hook → param normalization →
workspace boundary enforcement.

---

### 5. Subagent System — Child Agent Orchestration

Agents can spawn children for parallel or delegated work.

**Constraints:** `maxSpawnDepth` (default 1), `maxChildrenPerAgent` (default 5),
agent allowlist.

**Session isolation:** Each subagent gets unique session key
(`subagent:{parent}:{child}:{runId}`), separate workspace, restricted tools,
minimal system prompt.

**Management tools:** `sessions_spawn`, `subagents list/kill/steer`

**Registry:** Tracks active/completed runs, persists to disk, sweeps after
60-min TTL, handles announcement delivery with retry backoff.

---

### 6. Channel System — Messaging Integrations

Each channel is a plugin implementing adapter interfaces.

**ChannelPlugin interface** (`src/channels/plugins/types.plugin.ts`):
```
ChannelPlugin {
  id: ChannelId              // "telegram", "whatsapp", etc.
  meta: ChannelMeta          // Display name, docs, aliases
  capabilities               // Feature matrix
  config: ConfigAdapter      // Account management
  outbound?: OutboundAdapter // Send messages
  gateway?: GatewayAdapter   // Start/stop lifecycle
  setup?: SetupAdapter       // Wizard onboarding
  status?: StatusAdapter     // Health checks
  security?: SecurityAdapter // DM policy, allowlists
  auth?: AuthAdapter         // Login/logout (QR, password)
}
```

**Capabilities:** `chatTypes` (direct/group/channel/thread), `polls`,
`reactions`, `edit`, `unsend`, `reply`, `threads`, `media`, `blockStreaming`.

**Multi-account:** Each channel supports multiple accounts (e.g., 2 Telegram
bots), each independently startable/stoppable with its own config and status.

**Channel lifecycle:**
1. Initialize — Load plugin from registry
2. Start — `plugin.gateway.startAccount()` (webhook/polling/socket)
3. Running — Accept inbound messages
4. Restart — Exponential backoff on crash (5s to 5min)
5. Stop — `plugin.gateway.stopAccount()`

---

### 7. Message Flow — End-to-End

#### Inbound (Channel to Agent)

```
External message → Channel adapter receives → Dedup check →
Extract text/media → Access control (allowlist, group policy) →
Optional debounce → Build WebInboundMessage →
Gateway routing (resolve-route.ts, priority order):
  1. Binding peer match (direct DM)
  2. Binding peer parent (thread inherits)
  3. Binding guild+roles (Discord)
  4. Binding guild / team / account / channel
  5. Default agent (fallback)
→ Session key derivation → Agent execution → Response
```

**Session key format:**
`agent:{agentId}:channel:{channelId}:[group|direct]:{target}`

#### Outbound (Agent to Channel)

```
Agent reply → Outbound delivery (src/infra/outbound/deliver.ts) →
Resolve channel handler → Normalize payload →
Text chunking (channel-specific limits, markdown-aware) →
For each chunk: queue + dedup → channel.sendText/sendMedia →
OutboundDeliveryResult {messageId, timestamp} →
Fire message_sent hook → Record to session
```

#### Chat Streaming (WebSocket clients)

`chat.send` streams responses as EventFrames:
```
{ type: "evt", event: "chat", seq: N,
  payload: { sessionId, runId, delta?, text?, thinking?, done?, error? } }
```

`chat.abort` signals cancellation, saves partial output.

---

### 8. Plugin / Extension System

Plugins extend OpenClaw with channels, providers, services, tools, and hooks.

#### Discovery (4 sources, in order)

1. **Bundled** — Built-in (`resolveBundledPluginsDir()`)
2. **Global** — User-installed (`~/.config/openclaw/extensions/`)
3. **Workspace** — Project-scoped (`./.openclaw/extensions/`)
4. **Config** — Custom paths in config

#### Loading Pipeline

Manifest loading → Enable state check → Config validation (Zod) →
Dynamic import (Jiti) → `register(api)` call → Cache result

#### Plugin API (`OpenClawPluginApi`)

```
api.registerChannel(registration)       // ChannelPlugin
api.registerProvider(provider)          // Auth provider (OAuth, API key)
api.registerTool(tool, opts?)           // Agent tool
api.registerHook(events, handler)       // Lifecycle hook
api.registerHttpHandler(handler)        // Raw HTTP handler
api.registerHttpRoute({path, handler})  // Named HTTP route
api.registerGatewayMethod(method, fn)   // Gateway RPC method
api.registerCli(registrar, opts?)       // CLI command
api.registerService(service)            // Background service
api.registerCommand(command)            // Plugin command (no AI)
api.on(hookName, handler, opts?)        // Typed hook with priority
api.config                              // Full OpenClawConfig
api.pluginConfig                        // Plugin-specific validated config
api.runtime                             // PluginRuntime (logger, stash)
```

#### Extension Structure

```
extensions/my-extension/
├── index.ts               # Default exports plugin object
├── package.json           # Workspace package
├── openclaw.plugin.json   # Manifest (id, channels?, configSchema, uiHints?)
└── *.test.ts              # Tests
```

#### Plugin Registry Contents

`plugins[]`, `channels[]`, `providers[]`, `tools[]`, `hooks[]`,
`httpHandlers[]`, `httpRoutes[]`, `cliRegistrars[]`, `services[]`,
`commands[]`, `gatewayHandlers`, `diagnostics[]`

---

### 9. Hook System — Lifecycle Events

Hooks let plugins react to lifecycle events without modifying core code.

**Agent hooks:** `before_model_resolve`, `before_prompt_build`,
`before_agent_start`, `llm_input`, `llm_output`, `agent_end`

**Message hooks:** `message_received`, `message_sending` (cancellable),
`message_sent`

**Tool hooks:** `before_tool_call` (blockable), `after_tool_call`,
`tool_result_persist`

**Session hooks:** `session_start`, `session_end`, `before_compaction`,
`after_compaction`, `before_reset`

**Gateway hooks:** `gateway_start`, `gateway_stop`

**Sources:** Bundled (`src/hooks/`), managed (external), workspace
(`{workspace}/.hooks/`). Each hook has `HOOK.md` (frontmatter metadata)
and `handler.ts`.

---

### 10. Configuration System

**Format:** JSON5 (comments, trailing commas)

**Loading pipeline (4 layers):**
1. **Defaults** (`src/config/defaults.ts`) — Agent/model/session defaults,
   model alias map
2. **File** (`src/config/io.ts`) — JSON5 file with recursive includes,
   cycle detection, SHA256 integrity hash
3. **Env substitution** (`src/config/env-substitution.ts`) — `${VAR}` expansion
4. **Runtime overrides** (`src/config/runtime-overrides.ts`) — In-memory patches

**Validation:** Zod parsing → legacy detection → identity/avatar check →
agent dir validation → plugin config validation → custom issue collection.
Result: `{ ok, config } | { ok: false, issues[] }`

**Key env vars:** `OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH`,
`OPENCLAW_GATEWAY_PORT`, `OPENCLAW_PROFILE`, `OPENCLAW_SKIP_CHANNELS`

---

### 11. Session System — Conversation Persistence

**Storage:** `~/.openclaw/sessions/{sessionKey}.json` (file-based)

**Key formats:**
- Main: `agent:{agentId}:channel:{channelId}`
- Per-peer: `...:peer-kind:{peerId}`
- Per-guild: `...:guild:{guildId}`
- Subagent: `subagent:{parent}:{child}:{runId}`

**Features:** Exclusive write locking, session repair, history sanitization,
turn limiting, tool_use/tool_result pairing repair.

**Input provenance:** `external_user`, `inter_session`, `internal_system`

---

### 12. Memory System — Persistent Knowledge

**Backends:** Builtin (SQLite + sqlite-vec + FTS5), QMD (external tool)

**Embedding providers (auto-fallback):**
`openai → local (llama.cpp) → gemini → voyage`

**Hybrid search flow:**
1. Parallel vector search + BM25 FTS
2. Score normalization + merge
3. Keyword expansion (synonyms)
4. Fingerprint-based dedup
5. MMR ranking (diversity)

**Integration:** `MemoryIndexManager` per-agent on demand, global cache,
file watcher for auto-sync.

---

### 13. Cron System — Scheduled Tasks

**Job structure:** `{ id, agentId, name, enabled, schedule, sessionTarget, payload, delivery, state }`

**Schedule types:**
- `{ kind: "at", at: ISO8601 }` — One-shot
- `{ kind: "every", everyMs: number }` — Fixed interval
- `{ kind: "cron", expr: "0 9 * * MON", tz? }` — Cron expression

**Store:** `~/.openclaw/cron/jobs.json` (atomic writes)

**Execution:** Timer fires → isolated agent context → execute payload →
deliver results via channel/webhook → reaper prunes old sessions (24h).

---

### 14. Media and TTS

**Media:** Store at `~/.openclaw/media/` (5MB max, 2min TTL), SSRF-protected
fetch, `MEDIA:` token parsing from text.

**Media understanding:** `audio.transcription`, `video.description`,
`image.description` via OpenAI/Gemini.

**TTS providers:** OpenAI (`gpt-4o-mini-tts`), ElevenLabs, Edge (free).
Modes: `off`, `always`, `inbound`, `tagged`.
Formats: Opus (Telegram), MP3 (default), PCM (telephony).

---

### 15. Security

**Audit checks:** File permissions, secrets in config, plugin trust, sandbox
safety, tool policy, hook security, gateway auth/TLS probe, skill code scan.

**Auto-fix:** `chmod` (0o700 state, 0o600 config), Windows ACL resets.

---

### 16. State Directory Layout

```
~/.openclaw/
├── sessions/       # Per-agent session JSON files
├── oauth/          # OAuth tokens (0o700)
├── cron/           # Job store (jobs.json)
├── media/          # Media cache (2min TTL)
├── logs/           # Structured logs
└── [cache, state]  # Other runtime data
```

**Resolution:** `OPENCLAW_STATE_DIR` → legacy `CLAWDBOT_STATE_DIR` →
`~/.openclaw` → legacy dirs → default `~/.openclaw`.

**Migrations:** Auto-migrate legacy session keys, WhatsApp auth, session
canonicalization on startup.

---

### 17. Process Bootstrap

**`src/entry.ts` → `src/index.ts`:**
1. Set process title, install warning filter
2. Normalize environment, parse profile args
3. Load .env, ensure on PATH, enable console capture
4. Assert Node >= 22.12.0, install rejection handler
5. Parse CLI via Commander.js, execute command

**Gateway boot adds:**
- Load + validate config
- Initialize plugins (channel loader)
- Build CronService, heartbeat runner
- Memory manager (per-agent on demand)
- Internal hooks + session watchers
- Start HTTP/WS server

---

## Coding Conventions

### TypeScript

- **Strict mode** enforced. No `any` — `typescript/no-explicit-any` is error.
  Use `unknown` and narrow with type guards.
- **ESM with `.js` extensions.** All local imports include `.js`
  (e.g., `import { foo } from "./bar.js"`). Required by NodeNext resolution.
- **Target:** ES2023. Use modern JS (structuredClone, Array.at, etc.).
- **File size limit:** 500 lines max per `.ts` file (enforced by `check:loc`).

### Naming

- **Files:** kebab-case (`gateway-rpc.ts`, `parse-bytes.ts`)
- **Test files:** `filename.test.ts`, `filename.e2e.test.ts`, `filename.live.test.ts`
- **Functions:** camelCase with action verbs (`runAgent`, `parseConfig`, `resolveDefaults`)
- **Types/Classes:** PascalCase (`GatewayRpcOpts`, `ChannelPlugin`)
- **Error classes:** PascalCase + `Error` suffix (`WizardCancelledError`)
- **Constants:** UPPER_SNAKE_CASE (`DEFAULT_DELAY_MS`, `ADMIN_SCOPE`)

### Imports

oxfmt auto-sorts. Order: Node builtins (`node:fs`) → external packages →
local (with `.js`). Type-only: `import type { Foo } from "./foo.js"`.

### Error Handling

- Custom error classes extending `Error` for domain errors.
- Discriminated unions: `{ ok: true; result } | { ok: false; error }`.
- Wrap caught errors with `String(err)` for safe stringification.

### Logging

Use subsystem loggers, not `console.log`:
```ts
import { createSubsystemLogger } from "../logging/subsystem.js";
const log = createSubsystemLogger("my-feature");
log.info("message");
```
Levels: trace, debug, info, warn, error, fatal, raw, silent.

### Control UI (Lit)

Legacy decorators (`experimentalDecorators: true`):
```ts
@state() foo = "bar";
@property({ type: Number }) count = 0;
```
No standard `accessor` decorators — build tooling doesn't support them.

## Testing

```bash
pnpm test              # Parallel unit tests (all configs)
pnpm test:fast         # Unit tests only (no extensions/gateway)
pnpm test:e2e          # End-to-end tests
pnpm test:live         # Live API tests (requires credentials)
npx vitest run src/path/to/file.test.ts   # Single file
```

- **Framework:** Vitest with `describe`/`it` blocks.
- **Mocking:** `vi.hoisted()` for mock declarations, then `vi.mock("./module.js", ...)`.
- **Setup:** `test/setup.ts` — isolated HOME, plugin registry stubs, warning filters.
- **Cleanup:** `afterEach` clears mocks; fake timers auto-restored.
- **Six configs:** base, unit, e2e, gateway, live, extensions.
- **Coverage:** 70% lines, 70% functions, 55% branches (`pnpm test:coverage`).

## Linting & Formatting

```bash
pnpm format            # Auto-format with oxfmt
pnpm lint              # Run oxlint (type-aware)
pnpm lint:fix          # Auto-fix + format
pnpm check             # format:check + tsgo + lint (full CI)
```

Key rules: `typescript/no-explicit-any` (error), `curly` (error),
correctness/perf/suspicious (all error).
Suppress inline: `// oxlint-disable-next-line rule-name`

Pre-commit hook runs oxlint + oxfmt on staged files.
Swift: `pnpm format:swift`, `pnpm lint:swift`.

## Build System

```bash
pnpm build             # tsdown + plugin-sdk DTS + post-build scripts
pnpm ui:build          # Lit web UI
```

tsdown bundles multiple entry points (index, entry, daemon-cli, plugin-sdk,
hooks). Post-build copies assets, generates build info, writes CLI shims.

## PR Checklist

1. `pnpm check` passes (formatting, types, lint)
2. `pnpm test` passes
3. Files stay under 500 lines
4. No `any` types introduced
5. All imports use `.js` extensions
6. Tests added for new functionality
