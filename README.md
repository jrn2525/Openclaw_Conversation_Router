# openclaw-conversation-router

A native **[OpenClaw](https://docs.openclaw.ai) plugin** that acts as a **stateful inbound conversational workflow router**.

> Not a model router. Not a skill loader. Not a planner. This intercepts trigger phrases (e.g. `New Event`), drives a multi-turn dialog, persists state across separate inbound messages, and routes completed structured payloads to configurable downstream handlers.

---

## Target transcript (proven live on Telegram)

```
User:      New Event
Assistant: What is the event title?
User:      Lunch With Scott
Assistant: What day is it?
User:      04/28/2026
Assistant: What time?
User:      9:00 am
Assistant: Where is it?
User:      Panera Bread, Knoxville, North Peters
Assistant: Title: Lunch With Scott
           Date: 04/28/2026
           Time: 9:00 am
           Location: Panera Bread, Knoxville, North Peters

           Create it?
User:      yes
Assistant: Got it. The event details are saved.
```

A byte-exact test ([test/engine.test.js](test/engine.test.js) — "byte-exact target transcript") replays every line and asserts each assistant reply against the expected string verbatim.

---

## Install

```bash
openclaw plugins install ./
# or local link for development
openclaw plugins install -l ./
```

`prepare` builds TypeScript automatically. Manual build:

```bash
npm install
npm run build
```

The compiled entrypoint is `./dist/index.js`, declared in [`package.json`](package.json) under `openclaw.extensions` per [/plugins/building-plugins](https://docs.openclaw.ai/plugins/building-plugins).

### Build robustness

The `build` / `lint` scripts invoke TypeScript via `node`:

```
node ./node_modules/typescript/bin/tsc -p tsconfig.json
```

Not via `tsc` or `npx tsc`. This deliberately sidesteps a real failure mode we hit in a prior packaging pass: when a repo is zipped/transferred between filesystems, `node_modules/.bin/tsc` can lose its exec bit (mode `0644` / `0666`), producing `sh: 1: tsc: Permission denied` on `npm run build`. Calling the file via `node` doesn't depend on the exec bit. A test in [test/smoke.test.js](test/smoke.test.js) asserts this.

---

## The live interception contract

This plugin's correctness standard is **"it actually claims the turn in a real OpenClaw deployment"** — not "it matches the docs." Earlier drafts that optimised for doc-shape centred on `message:preprocessed` / `message:received` and mutated `event.messages`; that path did not actually intercept in the runtimes we tested.

### Final hook strategy

| Hook                 | Role                 | Source of confidence                                                                  |
|----------------------|----------------------|---------------------------------------------------------------------------------------|
| `before_dispatch`    | **Primary**          | **Proven in live OpenClaw runtime testing** with `{ handled: true, text }` return.    |
| `before_agent_reply` | Defensive fallback   | Docs: "claim the turn and return a synthetic reply". ([agent-loop](https://docs.openclaw.ai/concepts/agent-loop)) |
| `reply_dispatch`     | Defensive fallback   | Docs: `{ handled: true, ... }` is the terminal contract. ([sdk-overview](https://docs.openclaw.ai/plugins/sdk-overview)) |

Registration order in [src/hooks/register.ts](src/hooks/register.ts) is primary-first (`before_dispatch`, then the two fallbacks) so a host that runs handlers in registration order hits the proven seam first.

**Terminal return contract** across all three:

```ts
return { handled: true, text: "<reply text>" };
```

This short-circuits the default agent reply flow AND carries the synthetic reply back to the host for delivery on the same channel.

**Non-intercepting turns** return `undefined` so the agent handles them normally.

**We never mutate `event.messages`.** The return value is the authoritative contract.

### Dedup across multiple firings

If a host fires more than one of these hooks for the same turn (e.g. both `before_dispatch` and `reply_dispatch`), the handler would otherwise process the same inbound text twice and advance the workflow state twice. A small TTL cache keyed by `(sessionKey, text, coarse-timestamp)` returns the cached terminal result for any duplicate firing within 5 seconds. See `DedupCache` in [src/hooks/register.ts](src/hooks/register.ts). A regression test (`dedup: same turn firing through two hooks only advances state once`) guards it.

---

## Configuration — `plugins.entries.conversation-router.config`

The plugin reads `api.pluginConfig`, which OpenClaw populates from:

```jsonc
{
  "plugins": {
    "entries": {
      "conversation-router": {
        "config": {
          "enabled": true,
          "workflows": [ /* ... */ ]
        }
      }
    }
  }
}
```

A full working example lives at [example.config.json](example.config.json). Merge the `plugins.entries.conversation-router.config` block into your OpenClaw config.

### Zero workflows = silent no-op?

**No.** If the plugin is `enabled: true` but `workflows` is missing or empty, the plugin logs at ERROR:

```
plugin-router: enabled but NO workflows configured — the plugin will not intercept any inbound message.
Define at least one workflow under plugins.entries.conversation-router.config.workflows[].
```

A test covers this: `zero workflows: plugin logs error but registers hooks and returns undefined`.

---

## Workflow definition

```jsonc
{
  "id": "new_event",
  "displayName": "New Event",
  "triggers": ["New Event", "Add Event", "Schedule Meeting"],
  "channels": ["telegram", "webchat"],
  "confirmBeforeComplete": true,
  "completeMessage": "Got it. The event details are saved.",
  "cancelMessage": "Okay, cancelled the event.",
  "steps": [
    { "field": "title",    "type": "text",     "prompt": "What is the event title?", "required": true, "transform": "trim" },
    { "field": "date",     "type": "date",     "prompt": "What day is it?",          "required": true },
    { "field": "time",     "type": "time",     "prompt": "What time?",               "required": true },
    { "field": "location", "type": "location", "prompt": "Where is it?",             "required": false, "skippable": true }
  ]
}
```

### Step types

`text` · `boolean` · `date` · `time` · `datetime` · `number` · `choice` · `location` · `email` · `phone`.

Every step supports `required`, `default`, `validate` (min/max/minLength/maxLength/pattern), `transform` (trim/lowercase/uppercase/titlecase), `retryPrompt`, `skippable`, and branching via `next` or `branches[]`.

Date parser accepts ISO (`2026-04-28`), slash forms (`04/28/2026`, `5/3`), weekdays (`Friday`, `next Monday`), and relative words (`today`, `tomorrow`). Time parser accepts `1 PM`, `13:00`, `9:00 am`, `noon`, `midnight`.

---

## Commands during an active workflow

| Utterance              | Effect                           |
|------------------------|----------------------------------|
| `cancel` / `stop`      | Abort the workflow.              |
| `start over`           | Restart from the first step.     |
| `skip`                 | Skip a `skippable` step.         |
| `help` / `?`           | Show current step prompt + help. |
| `yes` / `no`           | Used in confirmation steps.      |

All vocabularies are configurable under `config.commands`.

---

## Completion handlers

Configure under `config.completion.handlers[]`:

| `type`         | Effect                                                                 |
|----------------|------------------------------------------------------------------------|
| `chat-confirm` | No-op — the engine already returned `completeMessage` as the turn's reply. |
| `log`          | Emits the payload via `api.logger.info`.                               |
| `file`         | Appends a JSONL record to `options.path` (resolved under the plugin state dir). |
| `callback`     | Reserved — plugin-emit bus not yet in public SDK surface.              |
| `command`      | Reserved — public inter-plugin command invocation not yet documented.  |

Emitted payload:

```json
{
  "workflow": "new_event",
  "confirmed": true,
  "values": {
    "title": "Lunch With Scott",
    "date":  { "iso": "2026-04-28", "raw": "04/28/2026", "year": 2026, "month": 4, "day": 28 },
    "time":  { "iso": "09:00", "raw": "9:00 am", "hour": 9, "minute": 0, "period": "am" },
    "location": "Panera Bread, Knoxville, North Peters"
  },
  "startedAt":   "2026-04-21T19:12:00.000Z",
  "completedAt": "2026-04-21T19:15:47.000Z",
  "conversation": { "channel": "telegram", "chatId": "...", "userId": "..." }
}
```

---

## Storage

Default backend: **atomic JSON** under `api.runtime.state.resolveStateDir()`.

- Writes use `fs.writeFile` to a sibling `.tmp` path + `fs.rename` for atomic swap.
- A serialised internal write queue prevents interleaved partial updates.
- Loads are tolerant of a missing or empty file.
- `cleanupIntervalMs` reaps workflows past `expiresAt`.
- Keyed by `conversationKeyStrategy` (default `channel+chat+user`).

The `memory` backend is available for ephemeral deployments and unit tests. A restart test verifies state survives a plugin reload using the JSON backend on disk.

---

## Module shape — NodeNext ESM

- `package.json` has `"type": "module"` and NodeNext-style `exports`.
- `tsconfig.json` uses `"module": "NodeNext"` and `"moduleResolution": "NodeNext"`.
- Every relative import in `src/**/*.ts` uses an explicit `.js` extension (e.g. `from './plugin.js'`). Verified by a smoke test.
- `__dirname` is derived via `fileURLToPath(import.meta.url)` — no CJS-specific globals.
- `dist/index.js` is the published entrypoint; a smoke test imports it live and asserts the default export + named exports load cleanly.

---

## Package metadata (`openclaw.*` in `package.json`)

Per [/plugins/sdk-setup#package-metadata](https://docs.openclaw.ai/plugins/sdk-setup#package-metadata), the OpenClaw host reads the following fields from `package.json`:

```jsonc
{
  "openclaw": {
    "extensions": ["./dist/index.js"],
    "compat": {
      "pluginApi":         ">=0.1.0",
      "minGatewayVersion": ">=0.1.0"
    },
    "install": {
      "npmSpec":                      "openclaw-conversation-router",
      "localPath":                    ".",
      "defaultChoice":                "local",
      "minHostVersion":               ">=0.1.0",
      "allowInvalidConfigRecovery":   true
    }
  }
}
```

| Field | Purpose |
|---|---|
| `extensions` | Entry point files, relative to the package root. The host loads these on activation. |
| `compat.pluginApi` | Minimum Plugin-SDK API version the plugin supports. |
| `compat.minGatewayVersion` | Minimum OpenClaw gateway version the plugin supports. |
| `install.npmSpec` | Canonical npm spec for install/update flows. |
| `install.localPath` | Local path the host uses for development/bundled installs. |
| `install.defaultChoice` | `"npm"` or `"local"` — preferred install source when both are available. |
| `install.minHostVersion` | Minimum OpenClaw host version the plugin is willing to install under. Enforced at install time. |
| `install.allowInvalidConfigRecovery` | When `true`, the host may attempt to repair a broken config rather than refusing to load. |

Version floors are intentionally permissive (`>=0.1.0`) so the plugin installs on any recent OpenClaw build. Tighten them in your fork if you need hard compatibility guarantees.

### What we deliberately *don't* ship

Per the same doc page:

- **`setupEntry`** — this is a channel-plugin-only pattern (the setup entry receives a `ChannelPlugin` object). This plugin is hook-based (it subscribes to `before_dispatch` across existing channels) rather than owning a channel, so there's no `setup-entry.ts`.
- **`channel` / `channels` / `kind: "channel"`** — we don't own a channel, so we don't declare one.
- **`providers`** — we don't register an LLM provider.
- **`build.openclawVersion` / `build.pluginSdkVersion`** — these are required only for ClawHub-published plugins. Add them if you publish to ClawHub.

## Legacy plugin ids

The manifest declares:

```json
"legacyPluginIds": ["openclaw-plugin-router"]
```

This exists because an earlier revision of this plugin used the id `openclaw-plugin-router`. Per [/plugins/manifest](https://docs.openclaw.ai/plugins/manifest), listing the old id in `legacyPluginIds` tells the OpenClaw host to normalise any existing user config still referencing `plugins.entries.openclaw-plugin-router.config` into `plugins.entries.conversation-router.config` automatically — so upgraders don't silently lose their workflow definitions.

## Import convention

Per [/plugins/building-plugins](https://docs.openclaw.ai/plugins/building-plugins), plugins must import from **focused SDK subpaths**, not the monolithic root:

```ts
// ✔ Use focused subpaths
import { definePluginEntry } from 'openclaw/plugin-sdk/plugin-entry';
import type { OpenClawPluginApi } from 'openclaw/plugin-sdk/plugin-entry';

// ✘ Deprecated — do not use
import type { OpenClawPluginApi } from 'openclaw/plugin-sdk';
```

Every source file in `src/**` uses the focused-subpath form. A smoke test (`src imports use focused openclaw/plugin-sdk/<subpath> paths`) fails the build if anyone reintroduces a bare-root import.

## Schema `$id`

The config schema uses a URN (`urn:openclaw:plugin:conversation-router:config-schema:v1`) rather than an `openclaw.dev` HTTPS URL, so it cannot collide with any first-party schema the OpenClaw project may ship under that namespace. Guarded by a smoke test.

---

## Project layout

```
.
├── openclaw.plugin.json          # Plugin manifest (inline configSchema)
├── package.json                  # ESM pkg; declares openclaw.extensions; tsc via node
├── tsconfig.json                 # NodeNext
├── schema/config.schema.json     # Standalone copy of the config schema
├── example.config.json           # Wrapped under plugins.entries.conversation-router.config
├── workflows/new_event.json      # Reusable workflow definition
├── src/
│   ├── index.ts                  # definePluginEntry default export
│   ├── plugin.ts                 # startRouter(api) — wires everything up
│   ├── sdk-shim.d.ts             # Ambient types for openclaw/plugin-sdk/*
│   ├── types.ts                  # Plugin-internal types
│   ├── engine/                   # Workflow state machine
│   ├── triggers/matcher.ts       # Exact + regex + channel gating
│   ├── commands/handler.ts       # cancel/help/skip/confirm vocab
│   ├── parsers/                  # text/date/time/number/choice/...
│   ├── storage/                  # jsonStore (atomic) + memoryStore
│   ├── completion/handlers.ts    # log / file / chat-confirm
│   ├── hooks/register.ts         # before_dispatch (primary) + two defensive fallbacks + dedup
│   └── util/                     # atomic write, conversation keying
└── test/                         # 56 tests incl. e2e + smoke + byte-exact transcript
```

---

## Tests

```bash
npm test
```

59 tests:

- **Trigger matching** — case-insensitive, aliases, channel gating, regex triggers.
- **Parsers** — text/boolean/number/choice/email/phone + date/time/datetime including `04/28/2026` and `9:00 am`.
- **Storage** — memory + json, atomic concurrent writes, restart survival, relative path resolution.
- **Engine** — happy path, cancel, start over, help, skip, validation retry, concurrent conversations, duplicate trigger safety, state persists across plugin restart.
- **Hooks** — all three interception hooks registered; each returns the documented `{ handled: true, text }` shape; non-trigger returns `undefined`; duplicate firings of the same turn are deduped to a single outcome; zero-workflow config triggers an error log.
- **Engine byte-exact** — the full target transcript is replayed and every reply is compared verbatim against the expected text.
- **Smoke** — `dist/index.js` imports cleanly; `package.json` is a valid ESM package; build script invokes `tsc` via `node` (no bin-exec-bit dependency); `openclaw.plugin.json` id matches; `example.config.json` is wrapped under `plugins.entries.conversation-router.config`; schema `$id` doesn't use the reserved `openclaw.dev` namespace; no `*.tmp*` artifacts at repo root; no relative import in `dist/` missing `.js`.

---

## SDK compatibility

- The plugin imports `openclaw/plugin-sdk/plugin-entry` and `openclaw/plugin-sdk` types; the real OpenClaw runtime provides them. A dev-only stub at [test/stubs/openclaw/](test/stubs/openclaw/) lets `npm test` run standalone.
- Ambient types live at [src/sdk-shim.d.ts](src/sdk-shim.d.ts), matching [/plugins/sdk-overview](https://docs.openclaw.ai/plugins/sdk-overview), [/plugins/sdk-entrypoints](https://docs.openclaw.ai/plugins/sdk-entrypoints), [/concepts/agent-loop](https://docs.openclaw.ai/concepts/agent-loop), and [/automation/hooks](https://docs.openclaw.ai/automation/hooks).
- If the host surfaces inbound events under non-standard field names, extend `normalizeInboundFromEvent` in [src/hooks/register.ts](src/hooks/register.ts) — it already tries `context.message`, `context.inbound`, `context.turn.message`, and flat `context`, with aliases for `text`/`content`/`body`, `channel`/`channelId`/`platform`, etc.

---

## License

MIT — see [LICENSE](LICENSE).
