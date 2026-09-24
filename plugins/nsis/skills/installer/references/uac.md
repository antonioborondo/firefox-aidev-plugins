# UAC and elevation

All installer scripts declare `RequestExecutionLevel user` — they do NOT
request admin at startup. Elevation is lazy: it only happens when needed,
via the UAC plugin. Both installer and uninstaller define
`!define NONADMIN_ELEVATE`.

## Core macros (in common.nsh)

- `${ElevateUAC}` — attempts elevation if not already admin; checks
  `UAC::IsAdmin`, `UAC::SupportsUAC`, and `UAC::GetElevationType`
- `${UnloadUAC}` — cleans up the UAC plugin before exit; only unloads if
  `/UAC:` flag is in the command line

## UAC plugin functions

| Function | Purpose | Return |
|----------|---------|--------|
| `UAC::IsAdmin` | Check if process is elevated | 0=no, 1=yes |
| `UAC::SupportsUAC` | Check if system has UAC | 0=no, 1=yes |
| `UAC::GetElevationType` | Get elevation type | 1=limited, 2=elevated, 3=split token |
| `UAC::RunElevated` | Attempt elevation or init plugin | 0=success |
| `UAC::Unload` | Release plugin resources | N/A |
| `UAC::ExecCodeSegment` | Run a function in the non-elevated context | N/A |

## Elevation flow

1. Installer starts as non-elevated user process
2. `${ElevateUAC}` checks if admin; if not:
   - Checks UAC support and elevation type
   - If split token (type 3 = admin user not running elevated): calls
     `UAC::RunElevated`, then quits the non-elevated instance
3. If elevation denied or unavailable: continues with non-elevated
   privileges (graceful degradation to HKCU)

## Registry hive detection after elevation

- Test write to HKLM: `WriteRegStr HKLM "Software\Mozilla"
  "${BrandShortName}InstallerTest" "Write Test"`
- If write succeeds: `$RegHive = "HKLM"`, `SetShellVarContext all`
- If write fails: `$RegHive = "HKCU"`, `SetShellVarContext current`
- All subsequent registry operations respect `$RegHive`

## Dual-context pattern

When the process is elevated but needs to perform user-level operations
(e.g., user shortcuts), use `UAC::ExecCodeSegment` to run code in the
non-elevated context:
```nsis
${GetParameters} $0
${GetOptions} "$0" "/UAC:" $0
${If} ${Errors}
  ; Not elevated — call directly
  Call UserLevelFunction
${Else}
  ; Elevated — delegate to non-elevated context
  GetFunctionAddress $0 UserLevelFunction
  UAC::ExecCodeSegment $0
${EndIf}
```
Used in installer.nsi, stub.nsh, uninstaller.nsi, and shared.nsh for
shortcut updates, app launch, and URL opening.

## Differences per installer type

- **Full installer** (installer.nsi): `${ElevateUAC}` early in init;
  determines `$RegHive` via test write; uses dual-context pattern for user
  shortcuts
- **Stub installer** (stub.nsh): `${ElevateUAC}` very early before any
  complex operations; uses dual-context pattern for LaunchApp and profile
  cleanup
- **Uninstaller** (uninstaller.nsi): smarter elevation — only elevates when
  necessary based on: write access to INSTDIR, registry access to
  maintenance service keys, and whether in silent mode
