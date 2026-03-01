---
name: openclaw-plugin-dev
description: Guide for writing, structuring, installing, and publishing OpenClaw plugins. Use this skill whenever the user wants to create a new OpenClaw plugin, fix or improve an existing plugin, publish a plugin to npm, or install a plugin (local, link, or npm). Triggers on: openclaw plugin, openclaw extension, openclaw插件, plugin-sdk, openclaw.plugin.json, plugins install, registerCommand, registerTool, api.on hook, before_prompt_build, openclaw publish.
---

# OpenClaw Plugin Development

Covers everything from writing to publishing. Read the relevant reference files as needed.

## Quick Reference

- **Plugin structure + API** → [references/structure.md](references/structure.md)
- **Publishing to npm** → [references/publish.md](references/publish.md)

---

## Minimal Correct Plugin (template)

A plugin is a directory with three files:

```
my-plugin/
├── openclaw.plugin.json   ← manifest (required)
├── package.json
└── index.ts               ← entry point
```

### `openclaw.plugin.json`

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "What it does",
  "version": "1.0.0",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "myOption": {
        "type": "string",
        "description": "Example config field"
      }
    }
  }
}
```

`id` and `configSchema` are **required**. Everything else is optional but recommended.

### `package.json`

```json
{
  "name": "my-openclaw-plugin",
  "version": "1.0.0",
  "type": "module",
  "description": "Short description",
  "devDependencies": {
    "openclaw": "workspace:*"
  },
  "openclaw": {
    "extensions": ["./index.ts"]
  }
}
```

For **npm publishing**, remove `private: true` and add `exports`:
```json
{
  "name": "my-openclaw-plugin",
  "version": "1.0.0",
  "type": "module",
  "exports": { ".": "./index.ts" },
  "openclaw": {
    "extensions": ["./index.ts"]
  }
}
```

### `index.ts`

```ts
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";

const plugin = {
  id: "my-plugin",
  name: "My Plugin",
  description: "What it does",
  configSchema: {
    parse(value: unknown) {
      const raw = value && typeof value === "object" && !Array.isArray(value)
        ? (value as Record<string, unknown>)
        : {};
      return {
        myOption: typeof raw.myOption === "string" ? raw.myOption : undefined,
      };
    },
  },
  register(api: OpenClawPluginApi) {
    const config = api.pluginConfig as ReturnType<typeof plugin.configSchema.parse>;
    // register commands, hooks, tools here
  },
};

export default plugin;
```

**Key point:** always `export default plugin` as an **object**, not a function. OpenClaw expects `{ id, configSchema, register }`.

---

## Common Non-Standard Patterns to Avoid

| Wrong | Correct |
|-------|---------|
| `export default function register(api)` | `export default plugin` (object) |
| Hand-writing `type OpenClawPluginApi = {...}` | `import type { OpenClawPluginApi } from "openclaw/plugin-sdk"` |
| No `openclaw` in devDependencies | Add `"openclaw": "workspace:*"` (monorepo) or `"openclaw": "^2026.x.x"` (standalone) |
| `private: true` in publishable package | Remove `private` for npm-published plugins |
| Missing `exports` field | Add `"exports": { ".": "./index.ts" }` for npm |

---

## Plugin API Cheatsheet

```ts
// Register a bot command (e.g. /files)
api.registerCommand({
  name: "files",
  description: "Open file manager",
  handler: async (ctx) => ({ text: "..." }),
});

// Register an agent tool
api.registerTool({
  name: "read_file",
  description: "Read a file",
  inputSchema: { type: "object", properties: { path: { type: "string" } } },
  handler: async ({ path }) => ({ content: "..." }),
});

// Register an HTTP route on the gateway
api.registerHttpHandler("GET", "/api/ls", async (req, res) => {
  res.json({ files: [] });
});

// Hook into the event pipeline
api.on("before_prompt_build", (event) => {
  // modify event.prompt or inject prependContext
  return { prependContext: "extra context" };
});
```

Full API surface: read [references/structure.md](references/structure.md).

---

## Installation Methods

```bash
# 1. From npm (production — recommended)
openclaw plugins install my-openclaw-plugin
openclaw plugins install my-openclaw-plugin --pin   # lock exact version

# 2. Local directory (dev/testing — copies files)
openclaw plugins install ./path/to/my-plugin

# 3. Link (dev — adds to plugins.load.paths, no copy)
openclaw plugins install --link ./path/to/my-plugin

# After any install, restart gateway:
openclaw restart
```

**npm package name vs plugin id:** The npm package name (e.g. `openclaw-telegram-files`) and the plugin's `id` field in `openclaw.plugin.json` (e.g. `telegram-files`) can differ. Config always uses the **manifest id** as the key: `plugins.entries.<manifest-id>.config.*`.

---

## Verifying Installation Source

After installing, confirm the plugin is properly tracked as an npm install (not a leftover local copy):

```bash
# Quick check — source should be "npm", not missing
cat ~/.openclaw/openclaw.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
print(json.dumps(d.get('plugins', {}).get('installs', {}), indent=2))
"
```

Key fields to verify:
- `"source": "npm"` ✅
- `"integrity"` hash present ✅ (matches npm registry — proves no tampering)
- `"spec"` includes exact version ✅

```bash
# Also confirm the install directory has no .git (would mean it's a git clone)
ls ~/.openclaw/extensions/<plugin-id>/   # should NOT have .git/

# List all plugins with source/version
openclaw plugins list
```

If you see this in the gateway log, the plugin is **not** npm-tracked:
```
[plugins] <id>: loaded without install/load-path provenance
```
→ Uninstall and reinstall from npm. Full steps: [references/publish.md](references/publish.md#replacing-a-localcloned-install-with-npm).

---

## Publishing to npm

Read [references/publish.md](references/publish.md) for the full flow. Quick path:

```bash
# 1. Verify version and build
cat package.json | grep version
npm pack --dry-run

# 2. Get OTP from 1Password
op read 'op://Private/Npmjs/one-time password?attribute=otp'

# 3. Publish
npm publish --access public --otp="<otp>"

# 4. Verify
npm view my-openclaw-plugin version --userconfig "$(mktemp)"
```
