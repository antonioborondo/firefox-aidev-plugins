# Telemetry and pings

The installer sends telemetry data via "pings" to track installation
success/failure and gather system information. There are two install
pings — stub and full — that now share the same mechanism, plus a separate
uninstall ping. Additionally, `installation_telemetry.json` is written to
disk for Firefox to read on first launch.

Documentation: `toolkit/components/telemetry/docs/data/install-ping.md` and
`uninstall-ping.md`.

## Install ping (unified mechanism)

The stub and full installer used to send different pings (the stub used a
GET request to a dedicated DSMO endpoint). **That was unified**: both now
build and send the same JSON ping through
`browser/installer/windows/nsis/telemetry.nsh`, included by both
`stub.nsh` (`!define TELEMETRY_STUB_INSTALLER`) and `installer.nsi`
(`!define TELEMETRY_FULL_INSTALLER`).

- **Protocol:** HTTP POST to
  `${TELEMETRY_BASE_URL}/${TELEMETRY_NAMESPACE}/${TELEMETRY_INSTALL_PING_DOCTYPE}/${TELEMETRY_INSTALL_PING_VERSION}/<uuid>`
  (`TELEMETRY_BASE_URL` = `https://incoming.telemetry.mozilla.org/submit`,
  namespace `firefox-installer`, doctype `install`), sent via
  `nsJSON::Set /http ping`. Defines live in
  `browser/installer/windows/nsis/defines.nsi.in`.
- **Format:** JSON document (common Telemetry ping format)
- **Sent:** just before the installer exits
  - Full installer: `SendPingIfApplicable` (installer.nsi) sends it, but
    **only if NOT launched from the stub** (`/LaunchedFromStub` absent),
    to avoid double-counting
  - Stub installer: `SendPing` (stub.nsh) always sends it when the stub
    exits

### How it's built

1. `Function PrepareTelemetryPing` (telemetry.nsh) fills in the fields
   common to both installers: build/update channel, locale, app version,
   build ID, distribution ID/version, Windows UBR, from_msi, 64bit_os,
   os_version/service_pack, server_os, admin_user, new_default/old_default,
   attribution, and (if `/TelemetryDebug:<id>` was passed) an
   `X-Debug-ID` header for the
   [Glean Debug Ping Viewer](https://debug-ping-preview.firebaseapp.com/).
2. It then calls a callback (passed on the stack via
   `GetFunctionAddress`) that adds the installer-type-specific fields:
   - `PrepareFullInstallPing` (under `!ifdef TELEMETRY_FULL_INSTALLER`) —
     installer_type "full", installer_version, 64bit_build,
     had_old_install, default_path, set_default, phase timings
     (intro/options/install/finish time), succeeded/user_cancelled,
     new_launched, silent, desktop_launcher_status
   - `PrepareStubInstallPing` (under `!ifdef TELEMETRY_STUB_INSTALLER`) —
     installer_type "stub", stub_build_id, funnelcake version, download
     phase timings/retries/bytes, exit codes, profile cleanup prompt
     type/response, etc.
3. `Function SendTelemetryPing` calls `PrepareTelemetryPing` (with the
   callback) then does `nsJSON::Set /http ping` to actually send it. This
   call blocks until a response, but by then no installer windows should
   still be open.

### Modifying the install ping

1. Decide whether the new field applies to both installers (add it in
   `PrepareTelemetryPing`) or only one (add it in `PrepareFullInstallPing`
   or `PrepareStubInstallPing`), using
   `nsJSON::Set /tree ping "Data" "[FIELD]" /value '[DATA]'`.
2. Add a corresponding unit test in
   `browser/installer/windows/nsis/test_telemetry.nsh` (included by
   `test_stub.nsi`; run via the `${UnitTest}` macro and helpers like
   `${AssertTelemetryData}`). `${GenerateUUID}` can be mocked for
   deterministic tests.
3. Update the
   [template](https://github.com/mozilla-services/mozilla-pipeline-schemas/blob/main/templates/firefox-installer/install/install.1.schema.json)
   in `mozilla-pipeline-schemas` so the field lands in the BigQuery table;
   a CMake build there propagates it into `schemas/`.
4. Document the field in
   `toolkit/components/telemetry/docs/data/install-ping.md`.
5. You can inspect a ping manually by running either installer with
   `/TelemetryDebug:<something>`, which submits it visibly to the Glean
   Debug Ping Viewer.

### Historical note

The stub installer used to send a separate GET-based ping to
`download-stats.mozilla.org` (DSMO), parsed server-side by gcp-ingestion's
`StubUri` class. That endpoint and its URL-encoded format are gone — both
installers now go through the unified JSON POST mechanism above. If you
find references to `StubURLVersion`, DSMO, or a `SendPing` that builds a
URL by hand, they're either stale or belong to the legacy code path history
— the current `SendPing` in `stub.nsh` just prepares tick-count/timing data
and calls `SendTelemetryPing`.

## Uninstall ping

- **Type:** `"uninstall"` (opt-out ping), common ping format with
  `clientId`, `profileGroupId`, and the Telemetry Environment
- **Written to disk by Firefox itself**, not the installer: on delayed
  Telemetry init (~1 minute into a run), if opt-out telemetry is enabled.
  One ping per install; old ones are replaced when a new one is written,
  and the ping is deleted if opt-out telemetry is later disabled.
  (`toolkit/components/telemetry/app/TelemetryStorage.sys.mjs`)
- **Sent by the uninstaller:** `un.SendUninstallPing` in
  `browser/installer/windows/nsis/uninstaller.nsi` finds the ping file
  (named `uninstall_ping_$AppUserModelID_<uuid>.json` in the common
  telemetry directory), POSTs it directly with `HttpPostFile::Post` to
  `${TELEMETRY_BASE_URL}/${TELEMETRY_UNINSTALL_PING_NAMESPACE}/<uuid>/${TELEMETRY_UNINSTALL_PING_DOCTYPE}/...`,
  then deletes it (and any stray extras). Sent from `un.onGUIEnd` (or
  directly from `un.onUninstSuccess` in silent mode) to avoid freezing the
  uninstaller GUI.

Payload:
```json
{
  "type": "uninstall",
  "clientId": "<UUID>",
  "profileGroupId": "<UUID>",
  "environment": { "...": "..." },
  "payload": {
    "otherInstalls": <integer 0-11>
  }
}
```

`otherInstalls` counts other Firefox installations on the system (capped at
11 for privacy), derived from `Software\Mozilla\Firefox\TaskBarIDs` in both
HKCU/HKLM and both 32/64-bit registry views, excluding this install.

## installation_telemetry.json

Written to the install directory by `WriteInstallationTelemetryData` in
`installer.nsi`, right after a successful install (reads
`$INSTDIR\application.ini`). Unlike the install ping — which the full
installer only sends when not launched by the stub — this file is always
written by the full installer, to reduce duplication and keep the
in-profile telemetry consistent regardless of installer type. It's deleted
by the uninstaller and read by
`browser/installer/windows/nsis/desktop_launcher_helpers.nsh` to determine
desktop-launcher status for the next `PrepareFullInstallPing` call.

Fields: `version`, `build_id`, `admin_user`, `install_existed`,
`profdir_existed`, `installer_type` ("stub"/"full"), `silent`, `from_msi`,
`default_path`, `install_timestamp` (Windows file time as string).

The `install_timestamp` is saved in the profile to detect new installations
but is NOT sent in the telemetry event below.

## installation.first_seen_* Glean events

Fired by Firefox on first launch after a new/changed installation is
detected (no longer an Events.yaml legacy event — it's Glean now).

- **Metrics:** `installation.first_seen_full`, `installation.first_seen_stub`,
  `installation.first_seen_msix` — defined in `browser/modules/metrics.yaml`
  under the `installation` category
- **Collection:** opt-out (all channels), `expires: never`

Shared `extra_keys`: `version`, `build_id`, `admin_user`, `install_existed`,
`other_inst`, `other_msix_inst`, `profdir_existed`; `silent`, `from_msi`,
`default_path` are present only on the `full` variant.

## Data access

BigQuery tables via Redash (default "Telemetry (BigQuery)" data source):
- `firefox_installer.install` — install pings (stub + full)
- `telemetry.uninstall` — uninstall pings

Columns marked [DEPRECATED] in the install-ping schema involve features
removed when the stub installer was streamlined in Firefox 55 (bug
1328445); they're kept for compatibility with old data but shouldn't be
used going forward.
