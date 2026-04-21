# openclaw-plugin-router

A production-ready **native [OpenClaw](https://docs.openclaw.ai) plugin** that acts as an **inbound conversational workflow router and engine**.

> This is **not** a model router, **not** a skill loader, and **not** a planner.
> It is a **stateful inbound conversation router** that intercepts trigger phrases, runs multi-turn dialogs, persists state across separate inbound messages, and emits structured results to configurable downstream handlers.

---

## What it does

When a user types a trigger phrase like `New Event` in Telegram, webchat, or any other OpenClaw channel, the plugin:

1. Intercepts the inbound message via the `message:preprocessed` hook.
2. Starts a named workflow instance.
3. Pushes the next question onto `event.messages` so the host delivers it.
4. Atomically persists the workflow state.
5. On the next inbound reply from the same user/chat/session, resumes the workflow.
6. Continues until all required fields are collected and validated.
7. Emits a structured payload to any number of configured completion handlers.

### Example

```
User:      New Event
Assistant: What is the event title?
User:      Lunch with Caleb
Assistant: What day is it?
User:      Friday
Assistant: What time?
User:      1 PM
Assistant: Where is it?
User:      Panera on North Peters
Assistant: Here's what I have:
              title:    Lunch with Caleb
              date:     Friday
              time:     1 PM
              location: Panera on North Peters
           Create it?
User:      Yes
Assistant: Event captured.
```

Emitted payload:

```json
{
  "workflow": "new_event",
  "confirmed": true,
  "values": {
    "title": "Lunch with Caleb",
    "date":  { "iso": "2026-04-24", "raw": "Friday", "year": 2026, "month": 4, "day": 24 },
    "time":  { "iso": "13:00", "raw": "1 PM", "hour": 13, "minute": 0, "period": "pm" },
    "location": "Panera on North Peters"
  },
  "startedAt":   "2026-04-21T16:00:00.000Z",
  "completedAt": "2026-04-21T16:03:12.000Z",
  "conversation": { "channel": "telegram", "chatId": "...", "userId": "..." }
}
```

---

## Install

```bash
openclaw plugins install ./openclaw-plugin-router
# or, install as a local link for development
openclaw plugins install -l ./openclaw-plugin-router
```

`prepare` builds the TypeScript automatically on install. To build by hand:

```bash
npm install
npm run build
```

The compiled entrypoint is `dist/index.js`, declared in [`package.json`](package.json) under `openclaw.extensions` as documented at [/plugins/building-plugins](https://docs.openclaw.ai/plugins/building-plugins).

---

## Plugin contract

The entry module uses the documented SDK:

```ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "openclaw-plugin-router",
  name: "Conversation Workflow Router",
  description: "...",
  configSchema: () => loadConfigSchema(),
  register(api) {
    // Subscribes hooks, sets up engine + storage + completion dispatcher.
  },
});
```

At activation, `register(api)` receives an `OpenClawPluginApi` ([SDK overview](https://docs.openclaw.ai/plugins/sdk-overview)) and uses:

| API                                     | Purpose                                                  |
|-----------------------------------------|----------------------------------------------------------|
| `api.pluginConfig`                      | This plugin's config from `plugins.entries.<id>.config`. |
| `api.logger`                            | Scoped logger (`debug`/`info`/`warn`/`error`).           |
| `api.runtime.state.resolveStateDir()`   | Absolute path where this plugin may persist state.       |
| `api.resolvePath(p)`                    | Resolve paths relative to the plugin root.               |
| `api.registerHook(events, handler)`     | Subscribes to inbound message events.                    |

### Hook subscription

Subscribes to both:

- `message:preprocessed` — *"After all media and link understanding completes"* — the preferred event.
- `message:received` — *"Inbound message from any channel"* — fallback for text-only channels.

Handler signature (from [/automation/hooks](https://docs.openclaw.ai/automation/hooks)):

```ts
async (event) => {
  // event: { type, action?, sessionKey?, timestamp?, messages?, context }
  // Push replies onto event.messages; the host delivers them.
  event.messages.push({ text: "What is the event title?" });
  return { handled: true };  // best-effort terminal signal
}
```

`event.context` is mapped to an `InboundMessage` by trying `context.message.*`, then `context.inbound.*`, then flat `context.*`, accepting field aliases (`text`/`content`/`body`, `channel`/`channelId`/`platform`, `chatId`/`roomId`/`conversationId`, etc).

---

## Configuration

Config is validated by the host against the inline `configSchema` in [`openclaw.plugin.json`](openclaw.plugin.json). Invalid config causes the install to fail closed — run `openclaw doctor --fix` to recover.

A full example lives in [`example.config.json`](example.config.json).

```jsonc
{
  "enabled": true,
  "storage": {
    "backend": "json",
    "path": "workflows.json",       // resolved under api.runtime.state.resolveStateDir()
    "cleanupIntervalMs": 300000
  },
  "defaults": {
    "timeoutMs": 1800000,
    "caseInsensitiveTriggers": true,
    "allowOneActiveWorkflowPerConversation": true,
    "summarizeBeforeConfirm": true,
    "conversationKeyStrategy": "channel+chat+user"
  },
  "commands": {
    "cancel":     ["cancel", "stop", "never mind"],
    "restart":    ["start over", "restart"],
    "skip":       ["skip"],
    "help":       ["help"],
    "confirmYes": ["yes", "confirm"],
    "confirmNo":  ["no"]
  },
  "completion": {
    "handlers": [
      { "type": "chat-confirm" },
      { "type": "log" },
      { "type": "file", "options": { "path": "completions.jsonl" } }
    ]
  },
  "workflows": [ /* see workflows/new_event.json */ ]
}
```

Set it in your OpenClaw config under `plugins.entries.openclaw-plugin-router.config`.

### Step types (parsers)

`text`, `boolean`, `date`, `time`, `datetime`, `number`, `choice`, `location`, `email`, `phone`.

Each step supports: `required`, `default`, `validate` (min/max/minLength/maxLength/pattern), `transform` (trim/lowercase/uppercase/titlecase), `retryPrompt`, `skippable`, branching via `next` or `branches[]`.

---

## Completion handlers

Configure one or many handlers under `completion.handlers[]`:

| `type`         | Effect                                                                               |
|----------------|--------------------------------------------------------------------------------------|
| `chat-confirm` | No-op: the engine already queued the `completeMessage` on the hook event.            |
| `log`          | Emits `workflow completed` via `api.logger.info`.                                     |
| `file`         | Appends a JSONL record to `options.path` (resolved under the plugin state dir).      |
| `callback`     | Reserved — plugin-emit bus not yet in the public SDK surface.                        |
| `command`      | Reserved — public inter-plugin command invocation not yet documented.                |

The payload includes `workflow`, `confirmed`, collected `values`, `startedAt`, `completedAt`, and the `conversation` identity snapshot.

---

## Storage

Default backend: **atomic JSON** at `<stateDir>/workflows.json` where `stateDir = api.runtime.state.resolveStateDir()`.

- Writes use `fs.writeFile` to a sibling `.tmp` path followed by `fs.rename` — atomic on POSIX; safe under concurrent puts on Windows.
- A serialised internal write queue prevents interleaved partial updates.
- Loads are tolerant of a missing or empty file.
- `cleanupIntervalMs` reaps workflows whose `expiresAt` has passed.
- Keyed by `conversationKeyStrategy` (default `channel+chat+user`).

The `memory` backend is available for ephemeral deployments and tests.

---

## Commands during an active workflow

| Utterance           | Effect                           |
|---------------------|----------------------------------|
| `cancel` / `stop`   | Abort the workflow.              |
| `start over`        | Restart from the first step.     |
| `skip`              | Skip a `skippable` step.         |
| `help`              | Show current step prompt + help. |
| `yes` / `no`        | Used in confirmation steps.      |

All command vocabularies are configurable.

---

## Project layout

```
.
├── openclaw.plugin.json          # Plugin manifest (inline configSchema)
├── package.json                  # npm package; declares openclaw.extensions
├── tsconfig.json
├── schema/config.schema.json     # Standalone copy of the config schema
├── example.config.json           # Documented config with defaults
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
│   ├── completion/handlers.ts    # log / file / chat-confirm / ...
│   ├── hooks/register.ts         # registerHook + event normalisation
│   └── util/                     # atomic write, conversation keying
└── test/                         # node:test — 40 unit + e2e tests
```

---

## Tests

```bash
npm test
```

40 tests covering triggers, parsers, storage atomicity, engine transitions, cancel/restart/skip/help flows, validation retry, concurrent conversations, duplicate-trigger safety, state survival across plugin restart, the `normalizeInboundFromEvent` mapping, and an **end-to-end run through the real OpenClaw hook surface** (fires `message:preprocessed` events against a fake api mirroring the documented shape).

---

## SDK compatibility notes

- The plugin imports `openclaw/plugin-sdk/plugin-entry` and `openclaw/plugin-sdk` (types). These modules are provided by the OpenClaw runtime at install time; they are declared as an optional peer dependency so `npm install` does not require a host checkout.
- Ambient type shims live at [src/sdk-shim.d.ts](src/sdk-shim.d.ts). They reflect the surface documented at [/plugins/sdk-overview](https://docs.openclaw.ai/plugins/sdk-overview), [/plugins/sdk-entry-points](https://docs.openclaw.ai/plugins/sdk-entry-points), and [/automation/hooks](https://docs.openclaw.ai/automation/hooks). If the SDK adds or renames fields, update the shim.
- For hook events the docs describe `event.context` generically without enumerating fields. We handle several common shapes; if your channel plugin uses non-standard field names, extend [src/hooks/register.ts:normalizeInboundFromEvent](src/hooks/register.ts).

---

## License

MIT — see [LICENSE](LICENSE).
