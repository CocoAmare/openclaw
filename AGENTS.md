# Repository Guidelines

- Repo: https://github.com/openclaw/openclaw
- GitHub issues/comments/PR comments: use literal multiline strings or `-F - <<'EOF'` (or $'...') for real newlines; never embed "\\n".

## Project Structure & Module Organization

### Top-Level Layout

```
openclaw/
├── src/             # Main TypeScript source (ESM)
├── extensions/      # Plugin/extension workspace packages (29 extensions)
├── skills/          # Agent skill modules (53 skills)
├── apps/            # Native platform apps (Android, iOS, macOS, shared Swift kit)
├── ui/              # Web UI (Lit web components + Vite)
├── packages/        # Workspace shims (clawdbot/, moltbot/)
├── docs/            # Mintlify documentation site
├── scripts/         # Build, test, deploy, and utility scripts (~67 files)
├── test/            # Shared test setup, helpers, mocks, fixtures
├── patches/         # pnpm patches for dependencies
├── vendor/          # Vendored dependencies
├── git-hooks/       # Git hook templates
├── assets/          # Static assets
├── dist/            # Built output (JS, JSON, protocol schema)
└── .github/         # CI workflows, issue templates, labeler, dependabot
```

### Source Code (`src/`)

- **CLI & Commands:** `src/cli/` (CLI wiring), `src/commands/` (command implementations)
- **Gateway & Networking:** `src/gateway/` (server, control plane, protocol), `src/process/` (process management, RPC)
- **AI & Agents:** `src/agents/` (agent integration, tools), `src/providers/` (LLM provider abstractions), `src/acp/` (Agent Communication Protocol)
- **Messaging Channels (built-in):** `src/telegram/`, `src/discord/`, `src/slack/`, `src/signal/`, `src/imessage/`, `src/line/`, `src/web/` (WhatsApp web), `src/whatsapp/`, `src/channels/`, `src/routing/`
- **Media & Understanding:** `src/media/` (pipeline), `src/media-understanding/`, `src/link-understanding/`, `src/tts/`
- **Configuration & State:** `src/config/`, `src/sessions/`, `src/memory/`, `src/logging/`
- **UI Surfaces:** `src/tui/` (terminal UI), `src/control-ui/`, `src/canvas-host/` (canvas rendering)
- **Infrastructure:** `src/infra/` (logging, fetching, updates), `src/security/`, `src/shared/`, `src/utils/`
- **Extension Support:** `src/plugin-sdk/` (plugin SDK), `src/channels/` (channel plugin utilities)
- **Automation:** `src/hooks/`, `src/cron/`, `src/auto-reply/`, `src/wizard/` (onboarding), `src/pairing/`
- **Platform:** `src/macos/`, `src/browser/`, `src/node-host/`, `src/daemon/`
- **Entry Points:** `src/entry.ts`, `src/index.ts`, `src/runtime.ts`, `src/version.ts`, `src/globals.ts`
- Tests are colocated as `*.test.ts` alongside source files.

### Extensions (`extensions/`)

29 workspace extension packages:
- **Messaging Channels:** `bluebubbles`, `discord`, `googlechat`, `imessage`, `line`, `matrix`, `mattermost`, `msteams`, `nextcloud-talk`, `nostr`, `signal`, `slack`, `telegram`, `tlon`, `twitch`, `voice-call`, `whatsapp`, `zalo`, `zalouser`
- **Authentication:** `google-antigravity-auth`, `google-gemini-cli-auth`, `minimax-portal-auth`
- **Core Features:** `copilot-proxy`, `diagnostics-otel`, `llm-task`, `lobster`, `memory-core`, `memory-lancedb`, `open-prose`

Plugin rules:
- Keep plugin-only deps in the extension `package.json`; do not add them to the root `package.json` unless core uses them.
- Install runs `npm install --omit=dev` in plugin dir; runtime deps must live in `dependencies`. Avoid `workspace:*` in `dependencies` (npm install breaks); put `openclaw` in `devDependencies` or `peerDependencies` instead (runtime resolves `openclaw/plugin-sdk` via jiti alias).

### Skills (`skills/`)

53 agent skill modules including: `1password`, `apple-notes`, `apple-reminders`, `bear-notes`, `camsnap`, `canvas`, `coding-agent`, `discord`, `food-order`, `gemini`, `github`, `gifgrep`, `obsidian`, `notion`, and more. Each skill has its own directory with implementation files.

### Native Apps (`apps/`)

- **`apps/android/`** — Gradle-based Android app (`ai.openclaw.android`)
- **`apps/ios/`** — XcodeGen-based iOS app (`ai.openclaw.ios`)
- **`apps/macos/`** — SwiftUI macOS menubar/status bar app (Swift Package Manager)
- **`apps/shared/`** — OpenClawKit: shared Swift framework for iOS/macOS

### Web UI (`ui/`)

Lit web components built with Vite. Separate `package.json` and vitest config. Built via `pnpm ui:build`, dev server via `pnpm ui:dev`.

### Workspace Packages (`packages/`)

- `packages/clawdbot/` — Backward-compatibility shim
- `packages/moltbot/` — Backward-compatibility shim

pnpm workspace defined in `pnpm-workspace.yaml`: root `.`, `ui`, `packages/*`, `extensions/*`.

### Installers

- Served from `https://openclaw.ai/*`; live in the sibling repo `../openclaw.ai` (`public/install.sh`, `public/install-cli.sh`, `public/install.ps1`).

### Messaging Channels

Always consider **all** built-in + extension channels when refactoring shared logic (routing, allowlists, pairing, command gating, onboarding, docs).
- Core channel docs: `docs/channels/`
- Core channel code: `src/telegram`, `src/discord`, `src/slack`, `src/signal`, `src/imessage`, `src/web` (WhatsApp web), `src/channels`, `src/routing`
- Extensions (channel plugins): `extensions/*` (e.g. `extensions/msteams`, `extensions/matrix`, `extensions/zalo`, `extensions/zalouser`, `extensions/voice-call`)
- When adding channels/extensions/apps/docs, review `.github/labeler.yml` for label coverage.

## Docs Linking (Mintlify)

- Docs are hosted on Mintlify (docs.openclaw.ai).
- Internal doc links in `docs/**/*.md`: root-relative, no `.md`/`.mdx` (example: `[Config](/configuration)`).
- Section cross-references: use anchors on root-relative paths (example: `[Hooks](/configuration#hooks)`).
- Doc headings and anchors: avoid em dashes and apostrophes in headings because they break Mintlify anchor links.
- When Peter asks for links, reply with full `https://docs.openclaw.ai/...` URLs (not root-relative).
- When you touch docs, end the reply with the `https://docs.openclaw.ai/...` URLs you referenced.
- README (GitHub): keep absolute docs URLs (`https://docs.openclaw.ai/...`) so links work on GitHub.
- Docs content must be generic: no personal device names/hostnames/paths; use placeholders like `user@gateway-host` and “gateway host”.

## exe.dev VM ops (general)

- Access: stable path is `ssh exe.dev` then `ssh vm-name` (assume SSH key already set).
- SSH flaky: use exe.dev web terminal or Shelley (web agent); keep a tmux session for long ops.
- Update: `sudo npm i -g openclaw@latest` (global install needs root on `/usr/lib/node_modules`).
- Config: use `openclaw config set ...`; ensure `gateway.mode=local` is set.
- Discord: store raw token only (no `DISCORD_BOT_TOKEN=` prefix).
- Restart: stop old gateway and run:
  `pkill -9 -f openclaw-gateway || true; nohup openclaw gateway run --bind loopback --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &`
- Verify: `openclaw channels status --probe`, `ss -ltnp | rg 18789`, `tail -n 120 /tmp/openclaw-gateway.log`.

## Build, Test, and Development Commands

- Runtime baseline: Node **22+** (engine minimum: `>=22.12.0`), pnpm **10.23.0**. Keep Node + Bun paths working.
- Install deps: `pnpm install`
- Pre-commit hooks: `prek install` (runs same checks as CI)
- Also supported: `bun install` (keep `pnpm-lock.yaml` + Bun patching in sync when touching deps/patches).
- Prefer Bun for TypeScript execution (scripts, dev, tests): `bun <file.ts>` / `bunx <tool>`.
- Run CLI in dev: `pnpm openclaw ...` (bun) or `pnpm dev`.
- Node remains supported for running built output (`dist/*`) and production installs.
- Mac packaging (dev): `scripts/package-mac-app.sh` defaults to current arch. Release checklist: `docs/platforms/mac/release.md`.
- Type-check/build: `pnpm build` (runs `tsc` with `--noEmit false`, canvas bundle, hook metadata copy, build info). Type-checking also available via `pnpm tsgo` (TypeScript native preview compiler).
- Lint/format: `pnpm lint` (oxlint with `--type-aware`), `pnpm format` (oxfmt `--check`). Fix: `pnpm lint:fix`, `pnpm format:fix`.
- Swift lint/format: `pnpm lint:swift` (swiftlint), `pnpm format:swift` (swiftformat).
- Tests: `pnpm test` (vitest via parallel runner); coverage: `pnpm test:coverage`
- UI: `pnpm ui:build` (build web UI), `pnpm ui:dev` (dev server), `pnpm test:ui` (UI tests).
- Android: `pnpm android:run` (build + install + launch), `pnpm android:test` (unit tests).
- iOS: `pnpm ios:run` (xcodegen + build + launch simulator), `pnpm ios:open` (open Xcode project).
- macOS app: `pnpm mac:package` (package), `pnpm mac:open` (open built app).
- Protocol: `pnpm protocol:gen` (regenerate JSON schema), `pnpm protocol:gen:swift` (regenerate Swift models), `pnpm protocol:check` (verify protocol is up to date).

### CI Pipeline (`.github/workflows/ci.yml`)

The CI runs on every push and PR with these jobs:
- **install-check** — Verify `pnpm install --frozen-lockfile` succeeds.
- **checks** (matrix, Ubuntu) — `tsgo` type-check, `build + lint`, `test` (Node + Bun), `protocol:check`, `format`.
- **checks-windows** (matrix, Windows) — `build & lint`, `test`, `protocol:check`.
- **checks-macos** (PRs only) — `test`.
- **macos-app** (PRs only) — Swift `lint`, `build` (release), `test` (with coverage). Uses Xcode 26.1.
- **android** (Ubuntu) — Gradle `test` and `build` (assembleDebug). Java 21, Android SDK 36.
- **secrets** — `detect-secrets` scan against `.secrets.baseline`.
- **ios** — Currently disabled in CI (`if: false`).

## Coding Style & Naming Conventions

- Language: TypeScript (ESM, `"type": "module"`). Target: `es2023`. Module: `NodeNext`. Strict mode enabled.
- Formatting/linting via Oxlint (plugins: unicorn, typescript, oxc; categories: correctness, perf, suspicious all as errors) and Oxfmt; run `pnpm lint` before commits.
- Oxlint config: `.oxlintrc.json`. Oxfmt config: `.oxfmtrc.jsonc`.
- Add brief code comments for tricky or non-obvious logic.
- Keep files concise; extract helpers instead of "V2" copies. Use existing patterns for CLI options and dependency injection via `createDefaultDeps`.
- Aim to keep files under ~500 LOC; guideline only (not a hard guardrail). Split/refactor when it improves clarity or testability.
- Naming: use **OpenClaw** for product/app/docs headings; use `openclaw` for CLI command, package/binary, paths, and config keys.

## Key Dependencies

- **AI/Agent:** `@mariozechner/pi-agent-core`, `@mariozechner/pi-ai`, `@mariozechner/pi-coding-agent`, `@mariozechner/pi-tui` (Pi SDK), `@agentclientprotocol/sdk` (ACP)
- **Messaging:** `@whiskeysockets/baileys` (WhatsApp), `grammy` (Telegram), `@buape/carbon` (Discord — never update), `@slack/bolt` + `@slack/web-api`, `@line/bot-sdk`
- **Schema/Validation:** `@sinclair/typebox`, `zod`, `ajv`
- **Media:** `sharp`, `pdfjs-dist`, `@mozilla/readability`, `file-type`
- **Infrastructure:** `express` (v5), `ws`, `undici`, `commander` (CLI), `chalk`, `dotenv`, `yaml`, `jiti`
- **Build/Dev:** `typescript` (^5.9), `@typescript/native-preview` (tsgo), `rolldown`, `tsx`, `vitest`, `oxlint`, `oxfmt`
- **Notable constraints:** Never update the Carbon dependency. Dependencies with `pnpm.patchedDependencies` must use exact versions (no `^`/`~`).

## Versioning & Release Channels

- Version scheme: **date-based** (`YYYY.M.D`), e.g. `2026.1.30`. Tracked in `package.json` and platform-specific locations (see version locations in Agent-Specific Notes).
- Changelog: `CHANGELOG.md` — keep latest released version at top (no `Unreleased`); after publishing, bump version and start a new top section.
- stable: tagged releases only (e.g. `vYYYY.M.D`), npm dist-tag `latest`.
- beta: prerelease tags `vYYYY.M.D-beta.N`, npm dist-tag `beta` (may ship without macOS app).
- dev: moving head on `main` (no tag; git checkout main).

## Testing Guidelines

- Framework: Vitest with V8 coverage. Pool: `forks` (isolation). Workers: up to 16 local, 2–3 in CI.
- Coverage thresholds: **70%** lines/functions/statements, **55%** branches (defined in `vitest.config.ts`).
- Naming: match source names with `*.test.ts`; e2e in `*.e2e.test.ts`; live in `*.live.test.ts`.
- Test includes: `src/**/*.test.ts`, `extensions/**/*.test.ts`, `test/format-error.test.ts`.
- Test setup: `test/setup.ts` (global setup file).
- Run `pnpm test` (or `pnpm test:coverage`) before pushing when you touch logic.
- Do not set test workers above 16; tried already.
- Timeouts: 120s test timeout, 180s hook timeout on Windows.
- Live tests (real keys): `CLAWDBOT_LIVE_TEST=1 pnpm test:live` (OpenClaw-only) or `LIVE=1 pnpm test:live` (includes provider live tests). Docker: `pnpm test:docker:live-models`, `pnpm test:docker:live-gateway`. Onboarding Docker E2E: `pnpm test:docker:onboard`.
- Vitest configs: `vitest.config.ts` (main), `vitest.unit.config.ts` (unit-only), `vitest.e2e.config.ts` (e2e), `vitest.gateway.config.ts` (gateway), `vitest.live.config.ts` (live), `vitest.extensions.config.ts` (extensions).
- Full kit + what's covered: `docs/testing.md`.
- Pure test additions/fixes generally do **not** need a changelog entry unless they alter user-facing behavior or the user asks for one.
- Mobile: before using a simulator, check for connected real devices (iOS + Android) and prefer them when available.

## Commit & Pull Request Guidelines

- Create commits with `scripts/committer "<msg>" <file...>`; avoid manual `git add`/`git commit` so staging stays scoped.
- Follow concise, action-oriented commit messages (e.g., `CLI: add verbose flag to send`).
- Group related changes; avoid bundling unrelated refactors.
- Changelog workflow: keep latest released version at top (no `Unreleased`); after publishing, bump version and start a new top section.
- PRs should summarize scope, note testing performed, and mention any user-facing changes or new flags.
- PR review flow: when given a PR link, review via `gh pr view`/`gh pr diff` and do **not** change branches.
- PR review calls: prefer a single `gh pr view --json ...` to batch metadata/comments; run `gh pr diff` only when needed.
- Before starting a review when a GH Issue/PR is pasted: run `git pull`; if there are local changes or unpushed commits, stop and alert the user before reviewing.
- Goal: merge PRs. Prefer **rebase** when commits are clean; **squash** when history is messy.
- PR merge flow: create a temp branch from `main`, merge the PR branch into it (prefer squash unless commit history is important; use rebase/merge when it is). Always try to merge the PR unless it’s truly difficult, then use another approach. If we squash, add the PR author as a co-contributor. Apply fixes, add changelog entry (include PR # + thanks), run full gate before the final commit, commit, merge back to `main`, delete the temp branch, and end on `main`.
- If you review a PR and later do work on it, land via merge/squash (no direct-main commits) and always add the PR author as a co-contributor.
- When working on a PR: add a changelog entry with the PR number and thank the contributor.
- When working on an issue: reference the issue in the changelog entry.
- When merging a PR: leave a PR comment that explains exactly what we did and include the SHA hashes.
- When merging a PR from a new contributor: add their avatar to the README “Thanks to all clawtributors” thumbnail list.
- After merging a PR: run `bun scripts/update-clawtributors.ts` if the contributor is missing, then commit the regenerated README.

## Shorthand Commands

- `sync`: if working tree is dirty, commit all changes (pick a sensible Conventional Commit message), then `git pull --rebase`; if rebase conflicts and cannot resolve, stop; otherwise `git push`.

### PR Workflow (Review vs Land)

- **Review mode (PR link only):** read `gh pr view/diff`; **do not** switch branches; **do not** change code.
- **Landing mode:** create an integration branch from `main`, bring in PR commits (**prefer rebase** for linear history; **merge allowed** when complexity/conflicts make it safer), apply fixes, add changelog (+ thanks + PR #), run full gate **locally before committing** (`pnpm lint && pnpm build && pnpm test`), commit, merge back to `main`, then `git switch main` (never stay on a topic branch after landing). Important: contributor needs to be in git graph after this!

## Security & Configuration Tips

- Web provider stores creds at `~/.openclaw/credentials/`; rerun `openclaw login` if logged out.
- Pi sessions live under `~/.openclaw/sessions/` by default; the base directory is not configurable.
- Environment variables: see `~/.profile`.
- Never commit or publish real phone numbers, videos, or live configuration values. Use obviously fake placeholders in docs, tests, and examples.
- Release flow: always read `docs/reference/RELEASING.md` and `docs/platforms/mac/release.md` before any release work; do not ask routine questions once those docs answer them.

## Troubleshooting

- Rebrand/migration issues or legacy config/service warnings: run `openclaw doctor` (see `docs/gateway/doctor.md`).

## Agent-Specific Notes

- Vocabulary: "makeup" = "mac app".
- Never edit `node_modules` (global/Homebrew/npm/git installs too). Updates overwrite. Skill notes go in `tools.md` or `AGENTS.md`.
- Signal: "update fly" => `fly ssh console -a flawd-bot -C "bash -lc 'cd /data/clawd/openclaw && git pull --rebase origin main'"` then `fly machines restart e825232f34d058 -a flawd-bot`.
- When working on a GitHub Issue or PR, print the full URL at the end of the task.
- When answering questions, respond with high-confidence answers only: verify in code; do not guess.
- Never update the Carbon dependency.
- Any dependency with `pnpm.patchedDependencies` must use an exact version (no `^`/`~`).
- Patching dependencies (pnpm patches, overrides, or vendored changes) requires explicit approval; do not do this by default.
- CLI progress: use `src/cli/progress.ts` (`osc-progress` + `@clack/prompts` spinner); don’t hand-roll spinners/bars.
- Status output: keep tables + ANSI-safe wrapping (`src/terminal/table.ts`); `status --all` = read-only/pasteable, `status --deep` = probes.
- Gateway currently runs only as the menubar app; there is no separate LaunchAgent/helper label installed. Restart via the OpenClaw Mac app or `scripts/restart-mac.sh`; to verify/kill use `launchctl print gui/$UID | grep openclaw` rather than assuming a fixed label. **When debugging on macOS, start/stop the gateway via the app, not ad-hoc tmux sessions; kill any temporary tunnels before handoff.**
- macOS logs: use `./scripts/clawlog.sh` to query unified logs for the OpenClaw subsystem; it supports follow/tail/category filters and expects passwordless sudo for `/usr/bin/log`.
- If shared guardrails are available locally, review them; otherwise follow this repo's guidance.
- SwiftUI state management (iOS/macOS): prefer the `Observation` framework (`@Observable`, `@Bindable`) over `ObservableObject`/`@StateObject`; don’t introduce new `ObservableObject` unless required for compatibility, and migrate existing usages when touching related code.
- Connection providers: when adding a new connection, update every UI surface and docs (macOS app, web UI, mobile if applicable, onboarding/overview docs) and add matching status + configuration forms so provider lists and settings stay in sync.
- Version locations: `package.json` (CLI), `apps/android/app/build.gradle.kts` (versionName/versionCode), `apps/ios/Sources/Info.plist` + `apps/ios/Tests/Info.plist` (CFBundleShortVersionString/CFBundleVersion), `apps/macos/Sources/OpenClaw/Resources/Info.plist` (CFBundleShortVersionString/CFBundleVersion), `docs/install/updating.md` (pinned npm version), `docs/platforms/mac/release.md` (APP_VERSION/APP_BUILD examples), Peekaboo Xcode projects/Info.plists (MARKETING_VERSION/CURRENT_PROJECT_VERSION).
- **Restart apps:** “restart iOS/Android apps” means rebuild (recompile/install) and relaunch, not just kill/launch.
- **Device checks:** before testing, verify connected real devices (iOS/Android) before reaching for simulators/emulators.
- iOS Team ID lookup: `security find-identity -p codesigning -v` → use Apple Development (…) TEAMID. Fallback: `defaults read com.apple.dt.Xcode IDEProvisioningTeamIdentifiers`.
- A2UI bundle hash: `src/canvas-host/a2ui/.bundle.hash` is auto-generated; ignore unexpected changes, and only regenerate via `pnpm canvas:a2ui:bundle` (or `scripts/bundle-a2ui.sh`) when needed. Commit the hash as a separate commit.
- Release signing/notary keys are managed outside the repo; follow internal release docs.
- Notary auth env vars (`APP_STORE_CONNECT_ISSUER_ID`, `APP_STORE_CONNECT_KEY_ID`, `APP_STORE_CONNECT_API_KEY_P8`) are expected in your environment (per internal release docs).
- **Multi-agent safety:** do **not** create/apply/drop `git stash` entries unless explicitly requested (this includes `git pull --rebase --autostash`). Assume other agents may be working; keep unrelated WIP untouched and avoid cross-cutting state changes.
- **Multi-agent safety:** when the user says "push", you may `git pull --rebase` to integrate latest changes (never discard other agents' work). When the user says "commit", scope to your changes only. When the user says "commit all", commit everything in grouped chunks.
- **Multi-agent safety:** do **not** create/remove/modify `git worktree` checkouts (or edit `.worktrees/*`) unless explicitly requested.
- **Multi-agent safety:** do **not** switch branches / check out a different branch unless explicitly requested.
- **Multi-agent safety:** running multiple agents is OK as long as each agent has its own session.
- **Multi-agent safety:** when you see unrecognized files, keep going; focus on your changes and commit only those.
- Lint/format churn:
  - If staged+unstaged diffs are formatting-only, auto-resolve without asking.
  - If commit/push already requested, auto-stage and include formatting-only follow-ups in the same commit (or a tiny follow-up commit if needed), no extra confirmation.
  - Only ask when changes are semantic (logic/data/behavior).
- Lobster seam: use the shared CLI palette in `src/terminal/palette.ts` (no hardcoded colors); apply palette to onboarding/config prompts and other TTY UI output as needed.
- **Multi-agent safety:** focus reports on your edits; avoid guard-rail disclaimers unless truly blocked; when multiple agents touch the same file, continue if safe; end with a brief “other files present” note only if relevant.
- Bug investigations: read source code of relevant npm dependencies and all related local code before concluding; aim for high-confidence root cause.
- Code style: add brief comments for tricky logic; keep files under ~500 LOC when feasible (split/refactor as needed).
- Tool schema guardrails (google-antigravity): avoid `Type.Union` in tool input schemas; no `anyOf`/`oneOf`/`allOf`. Use `stringEnum`/`optionalStringEnum` (Type.Unsafe enum) for string lists, and `Type.Optional(...)` instead of `... | null`. Keep top-level tool schema as `type: "object"` with `properties`.
- Tool schema guardrails: avoid raw `format` property names in tool schemas; some validators treat `format` as a reserved keyword and reject the schema.
- When asked to open a “session” file, open the Pi session logs under `~/.openclaw/agents/<agentId>/sessions/*.jsonl` (use the `agent=<id>` value in the Runtime line of the system prompt; newest unless a specific ID is given), not the default `sessions.json`. If logs are needed from another machine, SSH via Tailscale and read the same path there.
- Do not rebuild the macOS app over SSH; rebuilds must be run directly on the Mac.
- Never send streaming/partial replies to external messaging surfaces (WhatsApp, Telegram); only final replies should be delivered there. Streaming/tool events may still go to internal UIs/control channel.
- Voice wake forwarding tips:
  - Command template should stay `openclaw-mac agent --message "${text}" --thinking low`; `VoiceWakeForwarder` already shell-escapes `${text}`. Don’t add extra quotes.
  - launchd PATH is minimal; ensure the app’s launch agent PATH includes standard system paths plus your pnpm bin (typically `$HOME/Library/pnpm`) so `pnpm`/`openclaw` binaries resolve when invoked via `openclaw-mac`.
- For manual `openclaw message send` messages that include `!`, use the heredoc pattern noted below to avoid the Bash tool’s escaping.
- Release guardrails: do not change version numbers without operator’s explicit consent; always ask permission before running any npm publish/release step.

## NPM + 1Password (publish/verify)

- Use the 1password skill; all `op` commands must run inside a fresh tmux session.
- Sign in: `eval "$(op signin --account my.1password.com)"` (app unlocked + integration on).
- OTP: `op read 'op://Private/Npmjs/one-time password?attribute=otp'`.
- Publish: `npm publish --access public --otp="<otp>"` (run from the package dir).
- Verify without local npmrc side effects: `npm view <pkg> version --userconfig "$(mktemp)"`.
- Kill the tmux session after publish.
