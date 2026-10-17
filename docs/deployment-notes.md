# Suppressing auto-update without guessing

## The problem with "just block the URL"

When an auto-updater corrupts state repeatedly, the instinct is to block its endpoint. That instinct is reasonable but needs verification first: a mis-targeted block can produce an unstartable application, or silently mask a genuine security update.

Three questions must be answered before touching anything:

1. **Is there an official toggle?** If yes, use it. Anything else is fighting the vendor.
2. **How is the endpoint selected?** Env var, config file, or hardcoded?
3. **What happens on failure?** Does the request failure surface as a modal, block startup, or degrade quietly?

## Reading the actual control flow

`app.asar` is a packed archive, but it is not compressed — string search over the raw bytes reveals the source. This is faster and more reliable than reading the published documentation, which describes *intent*.

### Is there an official toggle?

```
startupForceAutoUpdate
startupForceAutoUpdateSystemDefault
StartupForceAutoUpdateDefault
```

Resolution order in the vendor build:

```
1. user setting        startupForceAutoUpdate                (local config store)
2. remote feature      StartupForceAutoUpdateDefault         (server-delivered)
3. cached system default
```

And the consumer:

```js
r && e.checkStartupForceAutoUpdate({ enabled: true, ... })
```

So a local boolean gates the whole startup path. A log line in the same module even checks `hasShowSetting=${keys.includes('ShowStartupForceAutoUpdateSetting')}` — the UI entry exists in the code but was not delivered to this build by the server.

**Conclusion:** an official toggle exists; it is simply not exposed. But it lives behind a `readBooleanFromConfig(store, key)` abstraction over an internal config store, not a plain JSON file. Setting it by hand is not tractable without reverse-engineering the store format.

### How is the endpoint selected?

```js
process.env.WORKBUDDY_UPDATE_URL
  || this.updateBaseUrlProvider?.getUpdateBaseUrl()
  || 'https://copilot.tencent.com'
```

An environment variable takes precedence over the provider-resolved base. That is a documented, supported override intended for private deployments — not a hack.

### What happens on failure?

The check is wrapped:

```js
try {
  await this.checkForUpdatesWithCustomUpdater(url)
} catch (e) {
  this.setState('error', undefined, undefined, { message, code: 'CHECK_ERROR' })
  reportTelemetry({ kind: 'update', outcome: 'failed', stage: 'check' })
}
```

The state machine moves to `error` and a telemetry event is emitted. **Startup is not blocked.** The failure is contained.

## Chosen approach

Point the update base at an address that cannot resolve a server:

```powershell
[Environment]::SetEnvironmentVariable(
  'WORKBUDDY_UPDATE_URL', 'http://127.0.0.1:1', 'User')
```

`127.0.0.1:1` is guaranteed closed on a normal Windows host; no listener can bind port 1 without elevation.

Verified before committing to it:

```powershell
Invoke-WebRequest 'http://127.0.0.1:1/v2/update?platform=win32-x64&version=5.8.4'
# → 无法连接到远程服务器   (connection refused, as intended)
```

Chosen over alternatives for these reasons:

| Approach | Verdict |
| --- | --- |
| Hand-edit config store | Store format undocumented; edits are overwritten on next write |
| Patch `app.asar` | Breaks the install manifest's file hashes; next verification pass fails |
| `hosts` file entry | Requires elevation, affects all processes, hard to attribute later |
| Env var override | Reversible, scoped to the user, uses a supported code path |

## Scoping

The variable is user-level, so it applies to both installed builds (domestic and international) — which is what was wanted here.

If per-build control is needed, drop the user-level variable and set it inside a launcher script instead:

```bat
@echo off
set WORKBUDDY_UPDATE_URL=http://127.0.0.1:1
start "" "%LOCALAPPDATA%\Programs\WorkBuddyAI\WorkBuddyAI.exe"
```

Environment set in a batch file applies only to processes it spawns.

**Note on batch encoding:** this file must be written in the OEM code page (936 for zh-CN). A UTF-8 batch file with non-ASCII content renders as mojibake, and a stray byte in a `REM` line can silently swallow the following line.

## Rollback

```powershell
[Environment]::SetEnvironmentVariable('WORKBUDDY_UPDATE_URL', $null, 'User')
```

Removing the variable restores the provider-resolved default with no residue.

## Tradeoff

Suppressed updates mean no security fixes and no new features until the update is restored. For a build that has demonstrably corrupted its own installation metadata, the tradeoff is defensible — but it should be a deliberate, time-boxed decision rather than a permanent state.

A workable pattern: leave updates enabled in general, and update manually on a schedule — back up first, update, then re-verify the registry, autostart, and shortcut paths that the installer may have rewritten. See [windows-install-repair.md](windows-install-repair.md) for what to re-check.
