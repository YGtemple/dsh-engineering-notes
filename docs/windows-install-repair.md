# Repairing a portable Windows install that broke its own metadata

## Symptom

An Electron application (WorkBuddy AI) had been installed into a desktop folder (`Desktop\新建文件夹 (2)\WorkBuddyAI\`) rather than a standard location. The folder was later renamed manually.

After the rename, three classes of metadata still referenced the old path:

1. **Registry uninstall entries** — `DisplayIcon` and `UninstallString` pointed at `新建文件夹 (2)\...` and `新建文件夹 (4)\...`, neither of which existed
2. **Autostart entries** — both pointed at non-existent executables, one at a path (`C:\WorkBuddyAI\`) that had never existed
3. **Shortcuts** — Start Menu and Desktop links resolved to the same dead paths

The behavioural consequence: **every auto-update corrupted file state**. The installer locates its target through that metadata; finding nothing at the recorded path, it re-materialised files in inconsistent locations, and the version regressed from 5.8.4 to 5.8.3.

## Why this kept recurring

The application ships an `install-manifest.json` enumerating 111 files with an `appVersion` field. The update path validates the install directory against that manifest. When the recorded location and the actual location diverge, validation cannot succeed, and recovery behaviour is unspecified.

Nothing in the application *checks* that its own registry entries resolve. The breakage is silent until an update runs.

## Inventory before touching anything

The first step was establishing what actually existed, rather than trusting any single source:

```
Registry      WorkBuddy 5.8.3      → C:\WorkBuddy\WorkBuddy.exe            (valid)
Registry      WorkBuddy AI 5.8.4   → 新建文件夹 (2)\...                     (dead)
Autostart     WorkBuddy.WorkBuddy  → C:\WorkBuddyAI\WorkBuddy.exe          (dead)
Autostart     WorkBuddy.WorkBuddyAI→ 新建文件夹 (2)\...                     (dead)
Shortcut      Start Menu (AI)      → 新建文件夹 (2)\...                     (dead)
Shortcut      Public Desktop       → 新建文件夹 (4)\...                     (dead)
Shortcut      User Desktop         → C:\WorkBuddy\WorkBuddy.exe            (valid)
Filesystem    WorkBuddyAI.exe      → Desktop\WorkBuddyAI\WorkBuddyAI\       (real)
```

Two useful observations fell out of this:

- The domestic build had been reinstalled to `C:\WorkBuddy\` by an intervening auto-update, and *that* entry was valid.
- The User Desktop shortcut pointed at the valid install and had a modification time matching the update run — the installer had repaired it in passing.

## Repair sequence

Every step was preceded by a backup.

### 1. Backup

```
reg export HKLM\...\Uninstall            → uninstall-hklm.reg
reg export HKCU\...\Uninstall            → uninstall-hkcu.reg
enumerate related env vars               → env-backup.json
```

### 2. Relocate to a standard path

`Desktop\WorkBuddyAI\WorkBuddyAI\` → `%LOCALAPPDATA%\Programs\WorkBuddyAI\`

Before moving, verified the directory contained no user data — the top level held only `Assets`, `locales`, `resources`, `share-target`, and binaries. Application state lived under `%APPDATA%`, untouched.

`robocopy /E /MOVE` completed 6,751 files (1,295 MB) in 11 seconds. Plain `Move-Item` across the same volume would also work; `robocopy /MOVE` was chosen for its exit-code semantics and retry handling.

### 3. Rewrite registry entries

```
DisplayIcon     → %LOCALAPPDATA%\Programs\WorkBuddyAI\WorkBuddyAI.exe,0
UninstallString → "…\Uninstall WorkBuddyAI.exe" /currentuser
InstallLocation → %LOCALAPPDATA%\Programs\WorkBuddyAI
```

### 4. Rewrite autostart entries

```
WorkBuddy.WorkBuddy   → "C:\WorkBuddy\WorkBuddy.exe"
WorkBuddy.WorkBuddyAI → "%LOCALAPPDATA%\Programs\WorkBuddyAI\WorkBuddyAI.exe"
```

### 5. Rebuild shortcuts and rename for disambiguation

Both launch targets carried visually similar icons and one shared a display name. Renamed to `WorkBuddy 国内版` / `WorkBuddy 国际版` on both Desktop and Start Menu.

### 6. Remove the orphan shortcut

`C:\Users\Public\Desktop\WorkBuddy.lnk` was owned by `BUILTIN\Administrators`; the running context was an unelevated administrator. Deletion required an elevated child process (`Start-Process -Verb RunAs`).

It had also been *invisible* — a same-named file in the user Desktop shadowed it in the merged view. Renaming the visible shortcut unshadowed it, which is why it appeared to "newly" exist.

### 7. Clean the stale installer payload

```
%LOCALAPPDATA%\Temp\workbuddy-update-x64\WorkBuddy-Setup-5.8.3.exe   420 MB
```

A downgrade artifact — the installed build was already 5.8.4.

## Pitfalls worth recording

**Encoding, twice.**

- A `.bat` containing Chinese comments was rewritten as UTF-8. `cmd.exe` renders batch files in the OEM code page (936 on zh-CN), so the comments became mojibake. Rewriting with `[System.Text.Encoding]::GetEncoding(936)` fixed it.
- File content passed through PowerShell string operations and MCP with UTF-8 assumptions produced corrupt CJK in a source patch (`限时免费` → `闄愭椂鍏嶅垂`). Detected by asserting on known substrings after upload, not by inspection.

**Shadowed shortcuts.** Windows merges `C:\Users\Public\Desktop` and `%USERPROFILE%\Desktop`. Identically named entries resolve to one of them unpredictably; a broken entry can be present and invisible simultaneously.

**Registry entries are not authoritative.** `DisplayIcon` may point anywhere, including paths that never existed. Enumerate the filesystem independently.

**Version regression across components.** These three disagreed:

```
Registry DisplayVersion      5.8.4
install-manifest.json        5.8.4
version file                 37.10.3-42
```

Only the executing binary's `VersionInfo` matches what the application believes about itself.

## Verification

Every recorded path was tested with `Test-Path` after repair, and each executable's identity was confirmed independently:

```
WorkBuddy.exe      5.8.3.0    C:\WorkBuddy\
WorkBuddyAI.exe    5.8.4.0    %LOCALAPPDATA%\Programs\WorkBuddyAI\
```

Different product names, different versions, different paths — the two installations are genuinely distinct, which matters because they had appeared identical in the tray.

## Guidance

- Install to `%LOCALAPPDATA%\Programs\<Product>\`, not to the Desktop. Portable layouts are unsupported by installers that resolve paths via registry, and every future update inherits the mismatch.
- If a folder must be renamed, update registry `DisplayIcon` / `UninstallString` / `InstallLocation`, autostart entries, and every shortcut in the same operation.
- Keep a pre-repair registry export. `reg export` of the Uninstall hive is sufficient to roll back.
