# Plugin Structure & API Reference

> Source of truth: `src/plugins/types.ts` in the openclaw repo.

## Directory Layout

```
my-plugin/
├── openclaw.plugin.json     ← manifest (required)
├── package.json
├── index.ts                 ← plugin entry point
├── src/
│   ├── config.ts            ← config parsing + types
│   ├── commands.ts          ← bot command handlers
│   ├── tools.ts             ← agent tool handlers
│   ├── hooks.ts             ← event hook handlers
│   ├── api-handlers.ts      ← HTTP route handlers
│   └── runtime.ts           ← runtime bridge
└── dist/                    ← built assets if needed
```

## openclaw.plugin.json Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | ✅ | Canonical plugin id. Used as config key. |
| `configSchema` | ✅ | Inline JSON Schema for plugin config. |
| `name` | optional | Display name. |
| `description` | optional | Short summary. |
| `version` | optional | Informational version string. |
| `channels` | optional | Channel ids registered by this plugin. |
| `providers` | optional | Provider ids registered by this plugin. |
| `skills` | optional | Skill directories to load (relative paths). |
| `uiHints` | optional | UI labels/placeholders/sensitive flags per config field. |
| `kind` | optional | Plugin kind — only valid value is `"memory"`. |

**configSchema** must be a valid JSON Schema. Empty plugin:
```json
{ "type": "object", "additionalProperties": false, "properties": {} }
```

---

## Plugin Object Interface (`OpenClawPluginDefinition`)

```ts
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";

const plugin = {
  id: "my-plugin",           // should match openclaw.plugin.json id
  name: "My Plugin",
  description: "...",
  version: "1.0.0",          // optional
  kind: "memory" as const,   // optional, only for memory-slot plugins

  // Optional: parse config in plugin object (alternative: parse manually in register())
  configSchema: {
    parse(value: unknown): MyConfig {
      // validate + return typed config
    },
  },

  register(api: OpenClawPluginApi): void {
    // all registration happens here
  },
};

export default plugin;
```

Both function-style and object-style exports work:
```ts
// Also valid (OpenClawPluginModule supports both):
export default function register(api: OpenClawPluginApi) { ... }
export default plugin  // ← preferred (enables kind, configSchema, etc.)
```

---

## Config Parsing (Two Patterns)

**Pattern A — in `configSchema.parse` (plugin object):**
```ts
configSchema: {
  parse(value: unknown) {
    const raw = (value && typeof value === "object" && !Array.isArray(value))
      ? (value as Record<string, unknown>) : {};
    return {
      externalUrl: typeof raw.externalUrl === "string" ? raw.externalUrl : undefined,
    };
  },
},
register(api: OpenClawPluginApi) {
  const config = api.pluginConfig as ReturnType<typeof plugin.configSchema.parse>;
}
```

**Pattern B — manually in `register()` (used by memory-lancedb-pro):**
```ts
register(api: OpenClawPluginApi) {
  const config = parseMyConfig(api.pluginConfig); // your own function
}
```

Both are correct. Pattern A makes the config typed and validated early; Pattern B is more flexible.

---

## `OpenClawPluginApi` — All Methods

### `registerTool`

Tools use **TypeBox** for parameter schemas. Import `Type` from `@sinclair/typebox` and `AnyAgentTool` from the sdk.

```ts
import { Type } from "@sinclair/typebox";
import type { AnyAgentTool, OpenClawPluginApi } from "openclaw/plugin-sdk";
import { stringEnum } from "openclaw/plugin-sdk"; // for string enums

api.registerTool({
  name: "my_tool",
  label: "My Tool",            // display name shown in UI
  description: "What it does",
  parameters: Type.Object({
    query: Type.String({ description: "Search query" }),
    limit: Type.Optional(Type.Number({ description: "Max results" })),
    category: Type.Optional(stringEnum(["fact", "preference", "decision"] as const)),
  }),
  async execute(_toolCallId, params) {
    const { query, limit = 5 } = params as { query: string; limit?: number };
    return {
      content: [{ type: "text", text: `Result for: ${query}` }],
      details: { count: 1 },  // optional structured data
    };
  },
} as AnyAgentTool);
```

Optional second arg pins the tool name:
```ts
api.registerTool(myTool, { name: "my_tool" });
// or for multiple names:
api.registerTool(toolFactory, { names: ["tool_a", "tool_b"] });
```

### `registerCommand`

Register a bot command (e.g. `/files`). Bypasses the LLM — use for simple state/status operations.

```ts
api.registerCommand({
  name: "files",                // becomes /files in chat
  description: "Open file manager",
  acceptsArgs: true,            // true if command takes text after it
  requireAuth: true,            // default true; false = anyone can use
  handler: async (ctx) => {
    // ctx.args — string after command name
    // ctx.config — current OpenClawConfig
    // ctx.channel — "telegram", "discord", etc.
    // ctx.from, ctx.to, ctx.accountId
    return { text: "Reply text" };
    // Telegram buttons: return { text: "...", webAppUrl: "https://..." }
  },
});
```

### `registerHttpRoute` / `registerHttpHandler`

```ts
// Preferred: named route (path is relative to plugin's base path)
api.registerHttpRoute({
  path: "/api/ls",
  handler: async (req, res) => {
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ files: [] }));
  },
});

// Legacy: no path, catches all requests to plugin's HTTP space
api.registerHttpHandler(async (req, res) => {
  if (req.url?.includes("/api/ls")) { ... }
  return false; // return false to pass through
});
```

### `on` — Typed Lifecycle Hooks

```ts
// All valid hook names:
// "before_model_resolve" | "before_prompt_build" | "before_agent_start"
// "llm_input" | "llm_output" | "agent_end"
// "before_compaction" | "after_compaction" | "before_reset"
// "message_received" | "message_sending" | "message_sent"
// "before_tool_call" | "after_tool_call" | "tool_result_persist"
// "before_message_write" | "session_start" | "session_end"
// "subagent_spawning" | "subagent_delivery_target" | "subagent_spawned" | "subagent_ended"
// "gateway_start" | "gateway_stop"

// Inject context before agent runs:
api.on("before_prompt_build", (event, ctx) => {
  // event.prompt, event.messages
  // ctx.agentId, ctx.sessionKey
  return { prependContext: "<memories>...</memories>" };
});

// Override model per-request:
api.on("before_model_resolve", (event, ctx) => {
  return { modelOverride: "claude-haiku-4-5" };
});

// React after agent finishes:
api.on("agent_end", async (event, ctx) => {
  // event.messages, event.success, event.durationMs
});
```

### `registerHook` — Untyped Hook (for non-standard events)

```ts
// For custom/internal events not in PluginHookName:
api.registerHook("command:new", async (event) => {
  // handle /new command
});
```

Use `on()` for standard events (type-safe). Use `registerHook()` only for custom event strings.

### `registerService` — Background Services

```ts
api.registerService({
  id: "my-service",
  start: async (ctx) => {
    // ctx.config, ctx.workspaceDir, ctx.stateDir, ctx.logger
    // Start timers, connections, etc.
    // Do NOT await slow startup here — use setTimeout(..., 0) to avoid blocking gateway
    setTimeout(() => void runStartupChecks(), 0);
  },
  stop: async (ctx) => {
    // Clean up timers and connections
  },
});
```

### `registerCli` — CLI Commands

```ts
import type { Command } from "commander";

api.registerCli((ctx) => {
  // ctx.program — commander Command instance
  // ctx.config, ctx.workspaceDir, ctx.logger
  ctx.program
    .command("memory-pro list")
    .description("List memories")
    .action(async () => { ... });
}, { commands: ["memory-pro"] });
```

### `registerChannel`

```ts
api.registerChannel({ plugin: myChannelPlugin, dock: myDock });
// or shorthand:
api.registerChannel(myChannelPlugin);
```

### `registerProvider`

```ts
api.registerProvider({
  id: "my-provider",
  label: "My LLM Provider",
  auth: [{ id: "api_key", label: "API Key", kind: "api_key", run: async (ctx) => { ... } }],
});
```

### Other helpers

```ts
api.resolvePath("~/some/path");  // resolves ~ and relative paths
api.config;                       // current OpenClawConfig
api.pluginConfig;                 // raw plugin config from openclaw.json
api.runtime;                      // PluginRuntime (TTS, etc.)
api.logger.debug("...");          // debug? (optional)
api.logger.info("...");
api.logger.warn("...");
api.logger.error("...");
```

---

## Runtime Bridge Pattern

For plugins with multiple files, store the API ref so all modules can access it:

```ts
// src/runtime.ts
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";
let _api: OpenClawPluginApi | null = null;
export function setRuntime(api: OpenClawPluginApi) { _api = api; }
export function getRuntime(): OpenClawPluginApi {
  if (!_api) throw new Error("Runtime not initialized");
  return _api;
}
```

```ts
// index.ts
import { setRuntime } from "./src/runtime.js";
import { registerAll } from "./src/register.js";

const plugin = {
  id: "my-plugin",
  register(api: OpenClawPluginApi) {
    setRuntime(api);
    registerAll(api);
  },
};
export default plugin;
```

---

## package.json for Standalone (npm-publishable) Plugins

```json
{
  "name": "my-openclaw-plugin",
  "version": "1.0.0",
  "type": "module",
  "description": "What it does",
  "exports": { ".": "./index.ts" },
  "dependencies": {
    "@sinclair/typebox": "0.34.48"
  },
  "openclaw": {
    "extensions": ["./index.ts"]
  }
}
```

Notes:
- `"type": "module"` required for ESM
- `exports` needed for npm consumers
- **Do NOT add `openclaw` to devDependencies** — `import type` from `openclaw/plugin-sdk` is stripped by jiti at runtime
- For monorepo extensions: `"openclaw": "workspace:*"` in devDeps gives IDE types
- Real runtime deps (e.g. `@lancedb/lancedb`, `openai`) go in `dependencies`, NOT devDependencies
- `@sinclair/typebox` must be in `dependencies` if you use `Type.Object(...)` in tools
- **Never** add `openclaw` to `dependencies`

## Common Pitfalls

| Wrong | Correct |
|-------|---------|
| `inputSchema: { type: "object", ... }` | `parameters: Type.Object(...)` with TypeBox |
| `handler(params)` | `execute(_toolCallId, params)` |
| `registerHttpHandler("GET", "/path", fn)` | `registerHttpRoute({ path: "/path", handler: fn })` |
| `export default function register(api)` | `export default plugin` (object) |
| Hand-writing `type OpenClawPluginApi = {...}` | `import type { OpenClawPluginApi } from "openclaw/plugin-sdk"` |
| `@sinclair/typebox` in devDependencies | Put in `dependencies` (needed at runtime) |
