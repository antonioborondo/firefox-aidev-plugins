# File locations

## Installer scripts
- `browser/installer/windows/nsis/installer.nsi` — full installer entry
  point
- `browser/installer/windows/nsis/stub.nsi` — stub installer entry point
- `browser/installer/windows/nsis/uninstaller.nsi` — uninstaller entry point
  (helper.exe)
- `browser/installer/windows/nsis/maintenanceservice_installer.nsi` —
  maintenance service installer
- `browser/installer/windows/nsis/test_stub.nsi` — test harness for stub

## Shared includes (.nsh)
- `toolkit/mozapps/installer/windows/nsis/common.nsh` — toolkit-level shared
  utilities (file ops, registry, UAC, silent install, MUI)
- `browser/installer/windows/nsis/shared.nsh` — browser-level shared macros
  (shortcuts, file associations, registry, protocol handlers)
- `browser/installer/windows/nsis/installer_helpers.nsh` — full installer
  helpers
- `browser/installer/windows/nsis/uninstaller_helpers.nsh` — uninstaller
  helpers
- `browser/installer/windows/nsis/stub_helpers.nsh` — stub installer helpers
- `browser/installer/windows/nsis/stub.nsh` — stub installer includes
- `browser/installer/windows/nsis/stub_shared_defs.nsh` — stub shared
  definitions
- `browser/installer/windows/nsis/control_utils.nsh` — shell context
  utilities (SetShellVarContextToValue, SwapShellVarContext)
- `browser/installer/windows/nsis/desktop_launcher_helpers.nsh` — desktop
  launcher status detection (reads `installation_telemetry.json`)
- `browser/installer/windows/nsis/install_dir_helpers.nsh` — install
  directory resolution helpers
- `browser/installer/windows/nsis/telemetry.nsh` — shared install-ping
  logic used by both the stub and full installer; see
  `references/telemetry.md`
- `browser/installer/windows/nsis/test_telemetry.nsh` — unit tests for
  `telemetry.nsh`, included by `test_stub.nsi`
- `browser/installer/windows/nsis/postupdate_helper.nsh` — post-update
  routines
- `toolkit/mozapps/installer/windows/nsis/overrides.nsh` — NSIS function
  overrides
- `toolkit/mozapps/installer/windows/nsis/locale-fonts.nsh` — locale font
  definitions

## Branding
- `browser/branding/{official,nightly,aurora,unofficial}/branding.nsi` —
  per-channel branding defines
- `browser/branding/*/stubinstaller/` — stub installer background images
  and CSS

## Build configuration
- `browser/installer/windows/nsis/defines.nsi.in` — preprocessed defines
  (@APP_VERSION@, @MOZ_BUILDID@, the `TELEMETRY_*` endpoint defines, etc.)
- `browser/installer/windows/Makefile.in` — browser-specific installer
  build rules
- `toolkit/mozapps/installer/windows/nsis/makensis.mk` — NSIS compilation
  rules

## Localization
- `browser/locales/{locale}/installer/` — per-locale translations
- `browser/locales/en-US/installer/` — English fallback strings
- `toolkit/mozapps/installer/windows/nsis/preprocess-locale.py` — converts
  UTF-8 .properties to UTF-16LE .nsh
- `toolkit/mozapps/installer/windows/nsis/locales.nsi` — RTL locale
  definitions (ar, he, fa, ug, ur)
- Property files: `override.properties`, `mui.properties`,
  `custom.properties`

## NSIS plugins
- `other-licenses/nsis/Plugins/` — custom DLLs used by the installers
  (AccessControl, AppAssocReg, ApplicationID, BitsUtils, CertCheck,
  CityHash, ExecInExplorer, HttpPostFile, InetBgDL, InvokeShellVerb,
  liteFirewallW, nsJSON, PinToTaskbar, ServicesHelper, ShellLink, UAC,
  WebBrowser)
- Plugin source: `other-licenses/nsis/Contrib/`

## UI content
- `browser/installer/windows/nsis/content/` — HTML/CSS/JS for stub
  installer UI (installing.html, profile_cleanup.html, stub_common.css,
  stub_common.js)

## Repackaging
- `python/mozbuild/mozbuild/repackaging/installer.py` — archive_exe(),
  repackage_installer()
- `python/mozbuild/mozbuild/action/exe_7z_archive.py` — 7z SFX creation

## Tests
- `browser/installer/windows/nsis/test/xpcshell/test_stub_installer.js` —
  xpcshell test
- `browser/installer/windows/nsis/test_stub.nsi` — NSIS test script with
  mock functions and assertion macros (includes `test_telemetry.nsh`)

## Documentation
- `browser/installer/windows/docs/` — InstallerBuild.md, StubInstaller.md,
  StubConfig.md, StubArch.md, StubGUI.md, FullInstaller.md, FullConfig.md,
  Helper.md, MSI.md, MSIX.md, NSISPlugins.md, WebBrowserAPI.md
- `toolkit/components/telemetry/docs/data/install-ping.md` and
  `uninstall-ping.md` — telemetry ping reference
