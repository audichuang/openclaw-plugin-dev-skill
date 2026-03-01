# Publishing an OpenClaw Plugin to npm

## Pre-publish Checklist

```bash
# 1. Confirm package.json has no private: true
grep private package.json   # should return nothing

# 2. Confirm version is correct
grep '"version"' package.json

# 3. Dry run — see what will be in the package
npm pack --dry-run
```

Make sure the tarball includes:
- `openclaw.plugin.json`
- `index.ts` (or entry listed in `openclaw.extensions`)
- `src/**` (source files)
- Built assets if needed (e.g. `dist/webapp/`)

Make sure the tarball **excludes**:
- `node_modules/`
- `.git/`
- Test files (use `.npmignore` or `package.json` `files` field)

### Recommended `package.json` `files` field

```json
{
  "files": [
    "index.ts",
    "src/",
    "dist/",
    "openclaw.plugin.json"
  ]
}
```

## Publishing with 1Password OTP

All npm publish commands require an OTP. Run inside tmux to avoid hangs:

```bash
# Start a tmux session
tmux new -s publish

# Sign into 1Password
eval "$(op signin --account my.1password.com)"

# Get OTP
OTP=$(op read 'op://Private/Npmjs/one-time password?attribute=otp')

# Publish
npm publish --access public --otp="$OTP"
```

## Verifying the Publish

```bash
# Check published version (avoids local .npmrc side effects)
npm view <package-name> version --userconfig "$(mktemp)"

# Check full package info
npm view <package-name> --userconfig "$(mktemp)"
```

## Installing from npm (user perspective)

```bash
# Basic install
openclaw plugins install <package-name>

# Pin exact version (recommended — prevents auto-upgrade)
openclaw plugins install <package-name> --pin
```

After install, restart the gateway:
```bash
openclaw restart
```

### npm package name vs plugin id

The npm package name (e.g. `openclaw-telegram-files`) and `openclaw.plugin.json` `id` (e.g. `telegram-files`) can differ. OpenClaw uses the **manifest id** as the config key:

```
plugins.entries.<manifest-id>.config.*
```

Not the npm package name. Document this in your README if they differ.

## Verifying Installation Source

After installing, confirm the plugin really came from npm (not a lingering local copy):

### 1. Check the install record in config

```bash
cat ~/.openclaw/openclaw.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
print(json.dumps(d.get('plugins', {}).get('installs', {}), indent=2))
"
```

A proper npm install looks like:
```json
{
  "telegram-files": {
    "source": "npm",
    "spec": "openclaw-telegram-files@0.1.0",
    "installPath": "/home/user/.openclaw/extensions/telegram-files",
    "version": "0.1.0",
    "resolvedName": "openclaw-telegram-files",
    "resolvedVersion": "0.1.0",
    "resolvedSpec": "openclaw-telegram-files@0.1.0",
    "integrity": "sha512-...",
    "shasum": "...",
    "resolvedAt": "2026-02-28T23:56:51.277Z",
    "installedAt": "2026-02-28T23:56:51.301Z"
  }
}
```

**What to confirm:**
- `source` is `"npm"` (not `"local"` or missing)
- `integrity` hash matches what `npm view <pkg>` shows — proves the tarball wasn't tampered with
- `spec` includes the exact version

### 2. Check the install directory has no `.git`

A git-cloned local install has a `.git` directory. An npm install does not:

```bash
ls ~/.openclaw/extensions/<plugin-id>/
# Should NOT contain: .git/
# Should contain: index.ts, openclaw.plugin.json, package.json, src/
```

### 3. Check via openclaw CLI

```bash
openclaw plugins list
# Shows id, version, source, and enabled status for all plugins

openclaw plugins info <plugin-id>
# Shows detailed info including install source
```

### 4. Check startup log for provenance warning

If the plugin loaded without proper npm tracking, the gateway log will warn:

```
[plugins] telegram-files: loaded without install/load-path provenance;
treat as untracked local code and pin trust via plugins.allow or install records
```

If you see this warning, the plugin is **not** properly tracked as an npm install. Uninstall and reinstall from npm.

---

## Replacing a local/cloned install with npm

```bash
# 1. Uninstall the old local version
openclaw plugins uninstall <plugin-id>

# 2. If the directory still exists (e.g. was a git clone), remove it
rm -rf ~/.openclaw/extensions/<plugin-id>

# 3. Install from npm
openclaw plugins install <npm-package-name> --pin

# 4. Re-apply any config you had before
openclaw config set plugins.entries.<plugin-id>.config.someKey "value"

# 5. Restart
openclaw restart
```

## Updating a Published Plugin

```bash
# 1. Bump version in package.json
# 2. Publish new version
npm publish --access public --otp="$OTP"

# 3. Users update with:
openclaw plugins update <plugin-id>
# or
openclaw plugins update --all
```

## Version Conventions

OpenClaw core uses date-based versions: `YYYY.M.D` (e.g. `2026.2.17`).

Community plugins can use either `YYYY.M.D` or standard semver (`0.1.0`, `1.0.0`, etc.).
