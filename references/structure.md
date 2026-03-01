# Plugin Structure & API Reference

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
│   ├── api-handlers.ts      ← HTTP route handlers (if needed)
│   └── runtime.ts           ← runtime bridge (store api.runtime ref)
├── dist/                    ← built assets (webapp, etc.)
└── webapp/                  ← frontend Mini App (if applicable)
```

## openclaw.plugin.json Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | ✅ | Canonical plugin id. Used as config key. |
| `configSchema` | ✅ | JSON Schema for plugin config. |
| `name` | optional | Display name. |
| `description` | optional | Short summary. |
| `version` | optional | Informational version string. |
| `channels` | optional | Channel ids registered by this plugin. |
| `providers` | optional | Provider ids registered by this plugin. |
| `skills` | optional | Skill directories to load (relative paths). |
| `uiHints` | optional | UI labels/placeholders for config fields. |
| `kind` | optional | Plugin kind, e.g. `"memory"`. |

**configSchema** must be a valid JSON Schema. Even if the plugin takes no config, include:
```json
{ "type": "object", "additionalProperties": false, "properties": {} }
```

## Plugin Object Interface

```ts
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";

const plugin = {
  id: "my-plugin",           // must match openclaw.plugin.json id
  name: "My Plugin",
  description: "...",
  configSchema: {
    parse(value: unknown): MyConfig {
      // parse and validate config; return typed config object
      // called at gateway startup with plugins.entries.<id>.config value
    },
  },
  register(api: OpenClawPluginApi): void {
    // called once at plugin load time
    // register commands, tools, hooks, HTTP handlers here
  },
};

export default plugin;
```

## OpenClawPluginApi Methods

### Commands

```ts
api.registerCommand({
  name: "files",              // becomes /files in chat
  description: "Short help text shown in /help",
  acceptsArgs: true,          // true if command takes text after it
  requireAuth: false,         // true to require pairing auth
  handler: async (ctx) => {
    // ctx.args — string after command name
    // ctx.accountId — sender's account id
    return { text: "Reply text" };
    // or: return { text: "...", buttons: [...] }  (Telegram Mini App button)
  },
});
```

### Agent Tools

```ts
api.registerTool({
  name: "read_workspace_file",
  description: "Read a file from the workspace",
  inputSchema: {
    type: "object",
    required: ["path"],
    properties: {
      path: { type: "string", description: "File path to read" },
    },
  },
  handler: async (input) => {
    const { path } = input as { path: string };
    // return tool result
    return { content: "file contents..." };
  },
});
```

### HTTP Handlers

```ts
// Mount under /plugins/<plugin-id>/
api.registerHttpHandler("GET", "/api/ls", async (req, res) => {
  res.json({ files: [] });
});

api.registerHttpHandler("POST", "/api/write", async (req, res) => {
  const { path, content } = req.body;
  res.json({ ok: true });
});
```

### Event Hooks

```ts
// Intercept prompt before it goes to the AI
api.on("before_prompt_build", (event: { prompt: string }) => {
  // return { prependContext: "..." } to inject context
  // return undefined to pass through unchanged
  return undefined;
});
```

### Runtime Helpers

```ts
// Access gateway runtime (logger, config, etc.)
const runtime = api.runtime;

// TTS (telephony)
const audio = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello",
  cfg: api.config,
});

// Logger
api.logger.info("Plugin started");
api.logger.error("Something failed");
```

### Plugin Config

```ts
// Access parsed config (after configSchema.parse())
const config = api.pluginConfig as MyConfig;
```

## Runtime Bridge Pattern

For plugins with multiple files, use a runtime bridge so all modules can access the API:

```ts
// src/runtime.ts
let _api: OpenClawPluginApi | null = null;

export function setRuntime(api: OpenClawPluginApi) {
  _api = api;
}

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
  // ...
  register(api: OpenClawPluginApi) {
    setRuntime(api);
    registerAll(api);
  },
};
```

## Config Schema Best Practices

Keep config parsing in `configSchema.parse()`:

```ts
configSchema: {
  parse(value: unknown) {
    const raw = (value && typeof value === "object" && !Array.isArray(value))
      ? (value as Record<string, unknown>)
      : {};
    return {
      externalUrl: typeof raw.externalUrl === "string" ? raw.externalUrl : undefined,
      allowedPaths: Array.isArray(raw.allowedPaths)
        ? (raw.allowedPaths as unknown[]).filter((p): p is string => typeof p === "string")
        : [],
    };
  },
},
```

Never use `zod` or external validation libraries unless they're listed in `dependencies` (not devDependencies).

## package.json for Standalone (npm-publishable) Plugins

```json
{
  "name": "my-openclaw-plugin",
  "version": "2026.2.17",
  "type": "module",
  "description": "What it does",
  "exports": { ".": "./index.ts" },
  "openclaw": {
    "extensions": ["./index.ts"]
  }
}
```

Notes:
- `"type": "module"` is required for ESM plugins
- `exports` is needed for npm consumers to resolve the entry
- **Do NOT add `openclaw` to devDependencies.** `import type { OpenClawPluginApi } from "openclaw/plugin-sdk"` is a type-only import — jiti strips it at runtime and never resolves the module. Adding `openclaw` as a devDep forces unnecessary large installs.
- For monorepo extensions (inside the openclaw repo): use `"openclaw": "workspace:*"` in devDependencies to get types during development. For standalone plugins outside the monorepo: omit it entirely.
- **Never** add `openclaw` to `dependencies`.
