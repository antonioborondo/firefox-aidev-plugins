# Full installer command-line options

Source of truth: `browser/installer/windows/docs/FullConfig.md`. Any CLI
option implicitly enables silent mode. Options use `/` prefix only (not
`-` or `--`, except `/S` which also accepts `-ms` for backwards
compatibility). `=true` can be omitted.

| Option | Default | Description |
|--------|---------|-------------|
| `/S` (or `-ms`) | — | Silent install with all defaults |
| `/InstallDirectoryPath=[path]` | — | Absolute install path (ignored if `InstallDirectoryName` is set) |
| `/InstallDirectoryName=[name]` | — | Directory name within Program Files (overrides Path) |
| `/TaskbarShortcut={true,false}` | true | Pin shortcut to taskbar |
| `/DesktopShortcut={true,false}` | true | Create desktop shortcut (no effect if `DesktopLauncher=true`) |
| `/DesktopLauncher={true,false}` | false | Install desktop launcher app (overrides DesktopShortcut) |
| `/StartMenuShortcut={true,false}` | true | Create Start menu shortcut (also spelled `/StartMenuShortcuts`, plural, for back-compat) |
| `/PrivateBrowsingShortcut={true,false}` | true | Create private browsing shortcut |
| `/MaintenanceService={true,false}` | true | Install Mozilla Maintenance Service |
| `/RemoveDistributionDir={true,false}` | true | Remove distribution dir from existing install |
| `/PreventRebootRequired={true,false}` | false | Avoid actions requiring reboot |
| `/OptionalExtensions={true,false}` | true | Install bundled extensions |
| `/RegisterDefaultAgent={true,false}` | true | Create scheduled task for default browser agent |
| `/INI=[path]` | — | Read config from .ini file (`[Install]` section) |
| `/ExtractDir=[dir]` | — | Extract files and exit without installing (no other options may be combined with this) |

INI file settings use the same names as the CLI options (minus `/S` and
`/INI`). CLI values override INI values when both are present.

Example INI:
```ini
[Install]
; Semicolons can be used to add comments
InstallDirectoryName=Firefox Release
DesktopShortcut=false
StartMenuShortcuts=true
MaintenanceService=false
OptionalExtensions=false
```

This option set is valid for Firefox 62+. Before Firefox 62, only `/S` and
`/INI` were accepted, and `/StartMenuShortcut` (singular) didn't exist —
only the plural `/StartMenuShortcuts` worked.

Note: this is the `.exe` full installer only. The MSI package
(`browser/installer/windows/msi/installer.wxs`) has its own property
surface — see `browser/installer/windows/docs/MSI.md`.
