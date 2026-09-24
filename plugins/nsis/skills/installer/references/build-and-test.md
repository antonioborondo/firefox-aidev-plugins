# Building and testing

## Building the installers

Must be on a Windows machine. Artifact builds are supported for
installer-only work. `$OBJDIR` is typically `obj-x86_64-pc-windows-msvc` on
Windows.

Steps in order:
1. Add `export MOZ_STUB_INSTALLER=1` to `mozconfig` if you need the stub
   installer (not built by default)
2. `./mach build` — builds Firefox and the uninstaller (helper.exe at
   `$OBJDIR/dist/bin/uninstall/helper.exe`)
3. `./mach package` — packages the application and builds the full
   installer (and stub if enabled); output in `$OBJDIR/dist/`
4. `./mach repackage installer ...` — (optional) manually repackage the
   .exe installer from the zip package
5. `./mach repackage msi ...` — (optional) wrap the .exe installer into an
   MSI package
6. `./mach repackage msix` — (optional) create an MSIX package

### Creating an installer manually (repackage)
After `./mach build` and `./mach package`, create the .exe installer with:
```bash
./mach repackage installer \
  --tag browser/installer/windows/app.tag \
  --setupexe $OBJDIR/browser/installer/windows/instgen/setup.exe \
  --sfx-stub other-licenses/7zstub/firefox/7zSD.Win32.sfx \
  --package-name firefox \
  --package $OBJDIR/dist/firefox-<version>.en-US.win64.zip \
  -o $OBJDIR/dist/firefox-<version>.en-US.win64.installer.exe
```

### Creating an MSI installer manually
Requires the .exe installer from the previous step:
```bash
./mach repackage msi \
  --wsx browser/installer/windows/msi/installer.wxs \
  --version <version> \
  --locale en-US \
  --arch x86_64 \
  --setupexe $OBJDIR/dist/firefox-<version>.en-US.win64.installer.exe \
  -o $OBJDIR/dist/firefox-<version>.en-US.win64.installer.msi
```

### Mozconfig options for installer work
Add these to `mozconfig` in the repo root as needed:
- `export MOZ_STUB_INSTALLER=1` — required to build the stub installer (not
  built by default in local builds)
- `ac_add_options --with-branding=browser/branding/official` — build with
  official branding, producing an installer identical to the real release
  one (other options: `nightly`, `aurora`, `unofficial` which is the
  default)
- `ac_add_options --with-redist` — include redistributable files for builds
  intended for distribution

## Build process

1. Application is packaged via `mach package` into `$OBJDIR/dist/firefox`
2. All required files copied to instgen directory (.nsi, .nsh, plugins,
   images, icons, 7-zip SFX module)
3. NSIS scripts compiled → setup.exe and setup-stub.exe (if enabled)
4. 7-zip SFX module run through UPX
5. Application files + setup.exe compressed into one 7z archive
6. Stub installer compressed into its own 7z archive
7. SFX module + config + 7z concatenated → final full installer
8. SFX module + config + stub 7z concatenated → final stub installer

Build logic lives in `toolkit/mozapps/installer/windows/nsis/makensis.mk`
and the `mach repackage` command.

## Testing

- There is no comprehensive automated test suite for NSIS scripts
- To test logic: use `MessageBox` to display variable/register values
  (pauses execution until OK), then remove before landing
- The stub installer has a limited xpcshell test via `test_stub.nsi` /
  `test_stub_installer.js`; the shared telemetry logic in `telemetry.nsh`
  is unit-tested via `test_telemetry.nsh`, which `test_stub.nsi` includes
- Always build and manually run the installer to verify changes
- You can debug C++ plugins by compiling in debug mode and attaching a
  debugger to the installer process
- To debug tests running in silent mode, temporarily insert a breakpoint
  that pauses execution so you can inspect the system state:
```nsis
SetSilent normal
MessageBox MB_OK "Breakpoint: inspect system now"
SetSilent silent
```
This switches out of silent mode to show the message box, waits for the
developer to click OK, then resumes silent mode. Remove before landing.

## Building NSIS plugins

Plugins are C++ DLLs, not integrated with the build system (pending bug
1771192). Built manually with Visual Studio.

Key requirements:
- **Must be 32-bit (x86)** — NSIS only works with 32-bit plugins
- Use NSIS 3.07 ExDLL headers: `pluginapi.h`, `pluginapi.c`,
  `nsis_tchar.h`
- Set output directory to `$SRCDIR/other-licenses/nsis/Plugins`
- Disable precompiled headers
- Register in `makensis.mk` under `CUSTOM_NSIS_PLUGINS`

Plugin function signature:
```cpp
extern "C" void __declspec(dllexport)
MyFunc(HWND, int string_size, TCHAR* variables,
       stack_t** stacktop, void*) {
  EXDLL_INIT();
  // popstringn() to read args, pushstring() to return
}
```

Called from NSIS as: `MyPlugin::MyFunc "$arg"`

May need `./mach clobber` for new DLLs to be recognized. Libraries from
`mfbt/` and `toolkit/` are usually safe to use.
