# openclaw-plugin-router

A production-ready native [OpenClaw](https://openclaw.dev) plugin that acts as an **inbound conversational workflow router and engine**.

> This is **not** a model router, **not** a skill loader, and **not** a planner.
> It is a **stateful inbound conversation router** that intercepts trigger phrases, runs multi-turn dialogs, persists state across separate inbound messages, and emits structured results to configurable downstream handlers.

---

## What it does

When a user types a trigger phrase like `New Event` in Telegram or webchat, the plugin:

1. Intercepts the inbound message before normal agent handling.
2. Starts a named workflow instance.
3. Asks the next question in chat.
4. Saves the workflow state atomically.
5. On the next inbound reply from the same user/chat/session, resumes the workflow.
6. Continues until all required fields are collected and validated.
7. Emits a structured payload to any number of configured completion handlers.
8. Optionally hands off to a downstream tool, command, task, or callback.

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
Assistant: Got it — here's what I have:
              title:    Lunch with Caleb
              date:     Friday
              time:     1 PM
              location: Panera on North Peters
           Create it?
User:      Yes
Assistant: Event captured.
```

Structured emitted result:

```json
{
  "workflow": "new_event",
  "title": "Lunch with Caleb",
  "date": "Friday",
  "time": "1 PM",
  "location": "Panera on North Peters",
  "confirmed": true
}
```

---

## Install

```bash
openclaw plugins install ./openclaw-plugin-router
# or, install locally (link mode)
openclaw plugins install -l ./openclaw-plugin-router
```

The plugin entrypoint is `dist/index.js`. Run `npm run build` once after cloning, or install from a tarball produced by `npm pack` — `prepare` builds automatically.

### Manual install

```bash
cd openclaw-plugin-router
npm install
npm run build
openclaw plugins install ./
```

---

## Runtime contract

The plugin expects the host to invoke the exported `activate(context)` with a minimal plugin context:

```ts
interface PluginContext {
  config: RouterConfig;              // validated against schema/config.schema.json
  logger?: Logger;                   // optional; falls back to console
  dataDir?: string;                  // absolute path for plugin storage
  on(event: string, handler: Function): void;
  off?(event: string, handler: Function): void;
  reply(msg: InboundMessage, text: string): Promise<void> | void;
  emit?(event: string, payload: unknown): void;
  runCommand?(cmd: string, args: Record<string, unknown>): Promise<unknown>;
}
```

Supported lifecycle hooks (first available wins):

| Event                  | Used for                                   |
|------------------------|--------------------------------------------|
| `message:preprocessed` | Preferred — intercept before agent reply.  |
| `message:received`     | Fallback.                                  |
| `session:ended`        | Clean up any active workflow for session.  |
| `plugin:deactivated`   | Final flush of state to disk.              |

If the host does not expose `reply`, the plugin falls back to returning `{ reply: string, intercept: true }` from the handler so the host can render it.

---

## Configuration

Config is validated against [`schema/config.schema.json`](./schema/config.schema.json). Full defaults live in [`example.config.json`](./example.config.json).

```jsonc
{
  "enabled": true,
  "storage": {
    "backend": "json",
    "path": "./data/workflows.json",
    "cleanupIntervalMs": 300000
  },
  "defaults": {
    "timeoutMs": 1800000,
    "caseInsensitiveTriggers": true,
    "allowOneActiveWorkflowPerConversation": true
  },
  "commands": {
    "cancel":     ["cancel", "stop", "never mind"],
    "restart":    ["start over", "restart"],
    "skip":       ["skip"],
    "help":       ["help"],
    "confirmYes": ["yes", "confirm"],
    "confirmNo":  ["no"]
  },
  "workflows": [ /* see workflows/new_event.json */ ]
}
```

### Step types (parsers)

`text`, `boolean`, `date`, `time`, `datetime`, `number`, `choice`, `location`, `email`, `phone`.

Each step supports: `required`, `default`, `validate` (min/max/minLength/maxLength/pattern), `transform` (trim/lowercase/uppercase/titlecase), `retryPrompt`, `skippable`, branching via `next` or `branches[]`.

---

## Completion handlers

Configure one or many handlers under `completion.handlers[]`:

| `type`         | Effect                                                                 |
|----------------|------------------------------------------------------------------------|
| `chat-confirm` | Sends the configured `completeMessage` (or a default) to the user.     |
| `log`          | Writes the structured payload via the plugin logger.                   |
| `file`         | Appends a JSONL record to `options.path`.                              |
| `callback`     | Emits `workflow:completed` on the plugin context for other listeners.  |
| `command`      | Invokes `context.runCommand(options.command, payload)` if supported.   |

The structured payload always includes `workflow`, `confirmed`, collected fields, and metadata (`startedAt`, `completedAt`, `conversation`).

---

## Storage

Default backend: **atomic JSON** at `./data/workflows.json`.

- Writes use `fs.writeFile` to a sibling `.tmp` path + `fs.rename` for atomic swap.
- Loads are tolerant of a missing or empty file.
- `cleanupIntervalMs` reaps workflows whose `expiresAt` has passed.
- Keyed by `conversationKeyStrategy` (default `channel+chat+user`).

The `memory` backend is available for tests and ephemeral deployments.

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
├── openclaw.plugin.json          # Plugin manifest
├── package.json                  # npm package
├── tsconfig.json                 # TypeScript compiler config
├── schema/config.schema.json     # JSON Schema for config
├── example.config.json           # Default config
├── workflows/new_event.json      # Reusable workflow definition
├── src/
│   ├── index.ts                  # Plugin entrypoint (activate/deactivate)
│   ├── plugin.ts                 # Router orchestrator
│   ├── types.ts                  # All TypeScript interfaces
│   ├── engine/
│   │   ├── engine.ts             # Workflow engine
│   │   └── instance.ts           # Active workflow instance helpers
│   ├── triggers/matcher.ts       # Trigger matcher
│   ├── commands/handler.ts       # cancel/help/skip/etc
│   ├── parsers/                  # text/date/time/number/etc
│   ├── storage/                  # JSON + memory backends
│   ├── completion/handlers.ts    # completion action dispatcher
│   ├── hooks/register.ts         # Lifecycle hook registration
│   └── util/                     # keying, logger, atomic I/O
└── test/                         # node:test unit tests
```

---

## Tests

```bash
npm test
```

Covers: trigger matching, parsers, storage atomicity, engine state transitions, cancel/restart/skip/confirm flows, and an end-to-end `new_event` happy path.

---

## License

MIT — see [LICENSE](./LICENSE).
