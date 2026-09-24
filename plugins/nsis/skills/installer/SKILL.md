---
name: installer
description: Architecture, conventions and pitfalls of the Firefox Windows installers (stub, full, helper/uninstaller, MSI, MSIX) built with NSIS. Use when reading, writing, reviewing or debugging .nsi/.nsh files, anything under browser/installer/windows/ or toolkit/mozapps/installer/windows/nsis/, NSIS plugins in other-licenses/nsis/, or installer/uninstall telemetry.
---

# Firefox Windows Installer

The Firefox installer gets users running Firefox as quickly and reliably as
possible while integrating it into the Windows system. It is built on the
NSIS (Nullsoft Scriptable Install System) framework, which compiles scripts
in its custom language (plus native C++ plugins) into executable installers.

We build four installer types: Stub Installer, Full Installer, MSI package
(wraps the full installer), and MSIX package. There is also helper.exe
(uninstaller + post-update + shortcuts).

Read `references/build-and-test.md` before building or testing changes,
`references/telemetry.md` before touching install/uninstall pings,
`references/full-installer-options.md` for the full installer's CLI/INI
surface, `references/file-map.md` to locate a script/include/doc by topic,
`references/uac.md` before touching elevation logic, and
`references/stub-installer.md` before touching the stub's HTML/CSS/JS UI or
the WebBrowser plugin.

---

# 1. Architecture

## 1.1 Stub installer

The default installer most users see. A tiny download (200-300 KB) that
downloads and runs the full installer silently. It offers no options except
an optional profile cleanup prompt for returning users.

- Main script: `browser/installer/windows/nsis/stub.nsi`
- Structure is unusual for NSIS: one empty section, no predefined pages; all
  work is in two custom pages (`createProfileCleanup` and `createInstall`)
- GUI is built with HTML/CSS/JS rendered via the Windows `IWebBrowser2`
  control (IE-based), driven by the custom WebBrowser NSIS plugin

### Execution flow
1. `.onInit` checks system requirements, selects 32/64-bit, looks for
   existing install to pave over, displays UAC prompt, initializes variables
   and GUI
2. `createProfileCleanup` optionally offers profile cleanup (same as
   about:support refresh)
3. `createInstall` draws the download/install UI, starts a blurb rotation
   timer, runs `StartDownload`
4. `StartDownload` kicks off the InetBgDl plugin for background download,
   starts `OnDownload` timer (200ms)
5. `OnDownload` checks download status, retries on failure, updates progress
   bar, verifies and runs the full installer when complete
6. `CheckInstall` waits for the full installer to exit, deletes the file
7. `FinishInstall` copies post-signing data, waits for the app to launch
   (requesting profile cleanup if selected)
8. Once the app is running and has shown a window, the stub sends its ping
   and exits

### Architecture selection
64-bit is selected only if: (1) 64-bit OS, (2) strictly more than 2 GB RAM,
(3) no incompatible third-party software. Otherwise 32-bit.

### Install scope
Automatically selected based on privileges. Rejecting the UAC prompt results
in a per-user installation. No UI for this choice.

### Install location
Looks for existing same-channel installation with matching architecture to
pave over. If none found, uses a hard-coded default based on architecture,
scope, and channel.

### Profile cleanup
Shown when helpful. Two variants: "paveover" (overwriting existing install)
and "reinstall" (new install). Triggers the same refresh as about:support.
Decision logic:
1. Find default Firefox profile; if none, skip
2. Look for existing same-channel installation via registry
3. Check if default profile's last-used version is 2+ versions behind
   current; if so, show prompt

### GUI and WebBrowser plugin
IE8 rendering mode, limited image/font formats, fixed window size, and the
`ShowPage`/`RegisterCustomFunction`/`CreateTimer` WebBrowser plugin API used
to drive the HTML/CSS/JS UI — see `references/stub-installer.md`.

## 1.2 Full installer

The "real" installer that does the actual work. Uses a traditional wizard
interface. Can be launched by the stub or standalone. Produces setup.exe
(30-40 MB).

- Main script: `browser/installer/windows/nsis/installer.nsi`
- Heavy lifting in `toolkit/mozapps/installer/windows/nsis/common.nsh`
- Writes `installation_telemetry.json` to the install location (read by
  Firefox for telemetry) — see `references/telemetry.md`
- If not launched by the stub, sends an install ping on exit — see
  `references/telemetry.md`
- Can access NSIS plugins written in C++
- Full CLI/INI option surface: `references/full-installer-options.md`

## 1.3 Helper (helper.exe)

Contains the uninstaller, a post-update routine, and default browser/shortcut
utilities. Two main files: `uninstaller.nsi` (entry point + uninstall logic)
and `shared.nsh` (other functions).

### Uninstaller
Philosophy: remove everything the installer creates, nothing it doesn't.
- Removes all installer-created registry entries (including very old
  versions)
- Deletes known application files using the `precomplete` file
  (auto-generated list of all app files)
- Leaves the install directory if any unknown files remain
- Runs maintenance service uninstaller if this was the only Firefox using it
- Cancels any BITS jobs for this installation
- Uploads the uninstall ping — see `references/telemetry.md`
- Profiles and user-generated files are NOT removed

### PostUpdate
Runs after an update via `/PostUpdate` switch. Maintains system integration
objects (version numbers in registry, shortcut renaming, maintenance service
updates, DLL registration).

Key facts:
- Runs after its own code has been updated, so changes take effect
  immediately in the first build containing a patch
- Runs twice: once elevated (HKLM entries) and once as regular user (HKCU
  entries)
- Windows-only; cannot fix things up on other platforms

### Default browser and shortcut handling
- `/ShowShortcuts` — add application to Open With
- `/HideShortcuts` — remove application from Open With
- `/SetAsDefaultAppGlobal` — make default browser (system-wide)
- `/SetAsDefaultAppUser` — make default browser (invoked by Firefox
  preferences)

On Windows 10+, SetAsDefault commands only write necessary registry entries
since the settings app controls defaults. ShowShortcuts/HideShortcuts are
never called (SPAD control panel no longer exists).

## 1.4 MSI package

A Windows Installer package for enterprise deployment. NOT a "true" MSI — it
wraps the full installer. The full installer runs inside the MSI just as if
launched normally. Built with WiX tools from
`browser/installer/windows/msi/installer.wxs`.

- Build: `./mach repackage msi`
- Most of the WiX file passes MSI properties through as CLI arguments to the
  full installer
- Cannot uninstall Firefox via standard MSI tools (limitation of the wrapper
  approach)

## 1.5 MSIX package

A full participant in the Windows modern app packaging system. Distributed,
installed, updated, repaired, and uninstalled entirely via that system.
Firefox's built-in updater is always disabled in MSIX. Better than MSI for
Windows 10+ enterprise deployment.

- Build: `./mach repackage msix`
- Can build unsigned packages with `--unsigned` (Windows 11 only)
- Can sign locally with `--sign` or `./mach repackage sign-msix`
- Channel-specific paths and assets (branding-aware)
- Langpacks included as distribution extensions in shippable builds
- `resources.pri` generated with `makepri.exe` from Windows SDK

---

# 2. File locations

Installer scripts and includes live under
`browser/installer/windows/nsis/` (browser-level) and
`toolkit/mozapps/installer/windows/nsis/` (toolkit-level shared utilities in
`common.nsh`), with NSIS plugins in `other-licenses/nsis/Plugins/`,
branding in `browser/branding/*/branding.nsi`, and docs in
`browser/installer/windows/docs/`. See `references/file-map.md` for the
full breakdown by script, include, branding, build config, localization,
plugins, UI content, repackaging, and tests.

---

# 3. NSIS language guide

## 3.1 Variables

- Declared with `Var VarName` at the top of files, outside any section or
  function
- Avoid `Var /GLOBAL` inside sections/functions; it works but is
  non-idiomatic
- All variables are global, mutable, and only known at install-time
- Access with `$VarName` syntax
- Key built-in variables: `$INSTDIR` (install directory), `$EXEDIR`
  (executable directory)

## 3.2 Defines (symbols)

- Compile-time constants, accessed with `${DefineName}` syntax
- `!undef` a define when no longer needed to prevent accidental reuse
- Key defines: `${BrandFullName}` (product name), `${FileMainEXE}` (main
  exe name)
- `${GetLongPath}` returns the correctly formatted full path for a file

## 3.3 Registers

- NSIS has registers `$0`-`$9` and `$R0`-`$R9` for temporary storage
- Registers are global — a value set in a macro/function persists after
  execution
- CRITICAL: always push registers before use and pop after, in FILO order,
  to avoid clobbering
- Use registers in ascending order: `$0`, then `$1`, `$2`, `$3`, etc. (same
  for `$R0`, `$R1`, `$R2`, ...)
- Convention: use `$R9` for macro return values

## 3.4 Functions vs macros

- **Functions**: for code invoked repeatedly; cannot take arguments
  directly; use slightly less disk space
- **Macros**: copy-paste code at invocation site; can take arguments; beware
  of variable clobbering since the code is inlined
- Prefer macros when arguments are needed
- Convention: define a `!define MacroName "!insertmacro MacroName"`
  shorthand so callers use `${MacroName}` syntax
- Functions cannot take arguments natively; use stack Push/Pop or global
  variables instead

## 3.5 Parameter passing and return values

- Stack-based: parameters pushed onto stack, functions use `Exch` and `Pop`
- Push/Pop must follow FILO (First In Last Out) order
- Return values: assign to a register (convention: `$R9`) or a user
  variable
- Macros have no built-in return mechanism; use registers or global
  variables

## 3.6 LogicLib and conditional logic

`${If}`, `${ElseIf}`, `${Else}`, `${EndIf}`, `${Unless}` are LogicLib macros
that simplify conditional logic (replacing raw IntCmp/StrCmp gotos).

Key constraint: LogicLib does NOT work recursively — a macro meant to be
used as a right-hand operand of `${If}` cannot itself use `${If}`
internally. For such macros, use raw `IntCmp`, `StrCmp`, or similar
comparison instructions instead.

## 3.7 Conditional compilation

- `!ifdef`, `!ifndef`, `!if`, `!else`, `!endif` for compile-time
  conditionals
- `IfFileExists` for runtime file checks
- `defines.nsi.in` uses `@VARIABLE@` substitution (Mozilla build system
  preprocessor)
- Key defines: `MOZ_MAINTENANCE_SERVICE`, `MOZ_BITS_DOWNLOAD`,
  `MOZ_DEFAULT_BROWSER_AGENT`, `HAVE_64BIT_BUILD`, `_ARM64_`

## 3.8 Error handling

- Check errors with `${If} ${Errors}` after operations
- Use `IfFileExists` before file operations
- Call `ClearErrors` at the start of functions/macros that check the error
  flag
- `ClearErrors` can mask bugs if used carelessly; use with caution
- Registry operations should handle missing keys gracefully
- Common pattern: try HKLM, check errors, fall back to HKCU

---

# 4. Codebase conventions

## 4.1 Naming

- Standard macros: `${MacroName}` (e.g., `${SetHandlers}`,
  `${TouchStartMenuShortcut}`)
- Uninstaller-only variants MUST use the `un.` prefix (e.g.,
  `un.CheckForFilesInUse`)
- If a macro is used in both installer and uninstaller, place it in
  `shared.nsh` and create both prefixed and unprefixed variants
- Functions are called with `Call FunctionName` or `Call un.FunctionName`

## 4.2 Registry operations

- Two main root keys: HKLM (system-wide, admin) and HKCU (current user
  only)
- NSIS auto-creates parent keys when writing subkeys/values
- Common value types: REG_SZ (string) and REG_DWORD (32-bit integer)
- Write default value of a key by using empty string as value name
- HKLM writes may fail in unelevated installs; always write fallback code
  for HKCU
- Use `ClearErrors` before operations that may fail, then check
  `${If} ${Errors}`
- Prefer commands that take an explicit root key argument over relying on
  `SetShellVarContext`

## 4.3 SetShellVarContext

- `SetShellVarContext all` makes shell folder constants (`$DESKTOP`,
  `$SMPROGRAMS`, etc.) resolve to all-users paths AND sets `SHCTX` to HKLM
  for registry operations
- `SetShellVarContext current` makes them resolve to current-user paths AND
  sets `SHCTX` to HKCU
- It affects BOTH file system paths and registry operations (via the
  `SHCTX` root key)
- Misreading or forgetting to restore the context is a common source of
  bugs
- When possible, favor registry commands that explicitly take a root key
  (HKLM/HKCU) over relying on `SHCTX`
- If you must use it, always restore it to the expected value after your
  operation
- `control_utils.nsh` provides helpers: `${SetShellVarContextToValue}` and
  `${SwapShellVarContext}`

## 4.4 Code style

- Keep lines reasonable in length
- Use `; comment` style for inline comments (not `# comment`)
- Section organization follows: PreInit -> Init -> InstallTypes -> Pages ->
  Sections -> Functions
- Delete files with `Delete`, not `RMDir` (which is for directories)

---

# 5. UAC and elevation

All installer scripts declare `RequestExecutionLevel user` — they do NOT
request admin at startup. Elevation is lazy, via the UAC plugin
(`${ElevateUAC}`/`${UnloadUAC}` in common.nsh), and degrades gracefully to
HKCU if declined or unavailable. After elevation, scripts test-write to
HKLM to decide `$RegHive` (HKLM or HKCU) and set `SetShellVarContext`
accordingly. When elevated code needs to run a user-level operation (e.g.
shortcuts), it uses the dual-context pattern: detect the `/UAC:` flag with
`${GetOptions}`, and if present, call `UAC::ExecCodeSegment` on a
`GetFunctionAddress` of the user-level function instead of calling it
directly.

See `references/uac.md` for the full UAC plugin function reference, the
elevation flow, the dual-context code pattern, and how elevation differs
between the full installer, stub installer, and uninstaller.
