# Firefox NSIS Plugin

Skills for NSIS development.

## Skills

### `installer`

Architecture, conventions, and pitfalls of the Firefox Windows installers
(stub, full, helper/uninstaller, MSI, MSIX): script/file layout, the NSIS
language (variables, defines, registers, macros vs. functions, LogicLib),
registry and `SetShellVarContext` conventions, UAC/elevation, and the
unified install/uninstall telemetry pings.

Use it when reading, writing, reviewing, or debugging `.nsi`/`.nsh` files,
anything under `browser/installer/windows/` or
`toolkit/mozapps/installer/windows/nsis/`, NSIS plugins in
`other-licenses/nsis/`, or installer/uninstaller telemetry. See
[`skills/installer/SKILL.md`](skills/installer/SKILL.md) for
details.
