# Farm REST API — complete route reference (`:8888`)

[farm-rest.md](./farm-rest.md) documents the `:8888` routes **observed in use** on the wire,
with request/response detail. This file is the **complete surface**: every method+path pair the
farm server answers, grouped by area, with a one-line purpose.

The set of routes is a protocol fact — the paths the `:8888` server responds to, confirmable
against your own server. Wire-level bodies for the actively-used subset live in
[farm-rest.md](./farm-rest.md), [farm-mqtt.md](./farm-mqtt.md) and [farm-tasks.md](./farm-tasks.md);
error codes are in [farm-errors.md](./farm-errors.md). Path parameters are shown in the server's
own `:name` form. Purposes for rarely-exercised routes are *(inferred)* from the path and the
server's error vocabulary rather than from captured traffic.

**Surface at a glance:** 109 routes — GET 37, POST 31, PUT 22, DELETE 13, PATCH 6.

## Health

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/ping` | Liveness probe (returns a pong body). |

## License & activation

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/license` | Install / activate the server license. |
| PUT | `/license` | Update the installed license. |
| PUT | `/license/crl` | Install a certificate-revocation list for license certificates. |
| GET | `/license/owner` | Read the licensed-owner identity. |

## Captain (farm session & aggregate state)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/captain` | Client heartbeat / session keepalive plus aggregate farm state (frequent, heavy). |
| PUT | `/captain` | Update captain configuration. |
| GET | `/captain/size` | Report the encrypted backup/database size. |
| POST | `/captain/up_cloud` | Enable the farm's cloud uplink. |
| DELETE | `/captain/up_cloud` | Disable the farm's cloud uplink. |

## SDK (third-party integration)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/sdk/login/ticket` | Obtain an SDK login ticket. |
| PUT | `/sdk/login` | Establish an SDK session. |
| GET | `/sdk/login` | Query the current SDK session. |
| DELETE | `/sdk/login` | End the SDK session. |
| POST | `/sdk/license` | Submit an SDK license. |
| PUT | `/sdk/license` | Update the SDK license. |
| PUT | `/sdk/device/bind` | Bind a device through the SDK. |

## Users

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/users` | List users. |
| POST | `/users` | Create a user. |
| GET | `/users/current_user` | Current user's profile. |
| PATCH | `/users/current_user/passwd` | Change own password. |
| PUT | `/users/admin/reset` | Administrative password reset. |
| PUT | `/users/:user_name/passwd` | Set a user's password. |
| PATCH | `/users/:user_name/role` | Change a user's role. |
| PATCH | `/users/:user_name/name` | Rename a user. |
| DELETE | `/users/:user_name` | Delete a user. |

## Login (tickets & tokens)

Two-step challenge/ticket → token exchange (see [farm-rest.md](./farm-rest.md)). `local` is the
farm-native path; `bambu` is the cloud-account path (refused in `LAN_FARM` with error `1027`,
see [farm-errors.md](./farm-errors.md)).

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/login/local/tickets` | Local login challenge (issue ticket). |
| POST | `/login/local/tokens` | Exchange a local ticket for a token. |
| POST | `/login/bambu/tickets` | Bambu-account login challenge. |
| POST | `/login/bambu/tokens` | Exchange a Bambu ticket for a token. |

## Devices & binding

| Method | Path | Purpose |
|--------|------|---------|
| PUT | `/bind2` | Bind a printer — triggers the server's TCP-3002 `login` ([farm-bind.md](./farm-bind.md)). |
| DELETE | `/bind` | Unbind a printer (`{ "dev_ids": [ … ] }` → MQTT `bind.unbind`). |
| GET | `/devices` | List devices (basic). |
| GET | `/devices2` | List bound devices (rich model; `?use_lite=true`). |
| GET | `/device/:dev_id` | One device's full state. |
| GET | `/local_printers` | Enumerate locally-known printers. |
| GET | `/scan_printers` | SSDP scan for printers on the LAN ([farm-discovery.md](./farm-discovery.md)). |
| GET | `/device/tags` | Device→tag associations. |
| PUT | `/device/:dev_id/tags` | Set a device's tags. |
| PUT | `/device/:dev_id/name` | Rename a device. |
| POST | `/device/:dev_id/opt` | Client→printer command bridge (→ MQTT `device/<sn>/request`, [farm-mqtt.md](./farm-mqtt.md)). |
| POST | `/device/:dev_id/taskopt` | Task-scoped device operation. |
| POST | `/device/:dev_id/launchtask` | Client-directed start of a task on a specific device. |
| PUT | `/device/:dev_id/print` | Device-scoped print control. |
| POST | `/device/bulkopt` | Bulk operation across many devices. |
| GET | `/device/bulkopt/:bulk_id` | Poll a bulk operation's status/result. |
| PUT | `/device/firmware` | Start a device firmware upgrade (→ MQTT `upgrade.offline_upgrade`, [farm-firmware.md](./farm-firmware.md)). |
| PUT | `/device/ams_filament_settings` | Set AMS / external-spool filament settings. |
| PUT | `/devices/info/up_cloud` | Enable cloud uplink for device info. |
| DELETE | `/devices/info/up_cloud` | Disable cloud uplink for device info. |

## Tags

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/tags` | List tags. |
| POST | `/tags` | Create a tag. |
| GET | `/tags/:tag_id` | Read a tag. |
| PUT | `/tags/:tag_id` | Update a tag. |
| DELETE | `/tags/:tag_id` | Delete a tag. |
| PUT | `/tags/:tag_id/devices` | Add devices to a tag. |
| DELETE | `/tags/:tag_id/devices` | Remove devices from a tag. |

## Filaments

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/filaments` | Filament presets / catalog. |

## Files & 3MF jobs

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/file/upload3mf` | Upload a 3MF job to the server. |
| GET | `/file/f3mf` | List / read 3MF records. |
| DELETE | `/file/f3mf` | Delete 3MF records. |
| GET | `/file/f3mf/:f3mf_id/f3mffile` | **Download the full job archive** — the `hpart://file/f3mf/<id>/f3mffile` target ([farm-mqtt.md](./farm-mqtt.md)). |
| PATCH | `/file/f3mf/:f3mf_id` | Update a 3MF record. |
| DELETE | `/file/f3mf/:f3mf_id` | Delete a 3MF record. |
| POST | `/file/f3mffolder` | Create a 3MF folder. |
| GET | `/file/f3mffolder` | List 3MF folders. |
| PATCH | `/file/:folder_id/f3mffolder` | Update a folder. |
| DELETE | `/file/:folder_id/f3mffolder` | Delete a folder. |
| POST | `/file/:folder_id/pinopt` | Pin / unpin a folder item. |
| PUT | `/file/f3mffolder/root/order` | Reorder root folders. |
| POST | `/file/move` | Move files / folders. |
| GET | `/file` | Fetch an asset by `asset_path` (liveview JPEG, thumbnails, slice-info previews). |

## Firmware catalog

Server-side offline-OTA package store; the printer-facing download uses the `hpart://` scheme.
Full flow in [farm-firmware.md](./farm-firmware.md).

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/file/firmwares` | Upload / register a firmware package (multipart). |
| GET | `/file/firmwares` | List the firmware catalog. |
| GET | `/file/firmwares/:firmware_name/info` | Firmware-package metadata. |
| DELETE | `/file/firmwares` | Delete catalog entries. |
| GET | `/file/firmwares/:firmware_name` | **Printer-facing** firmware download (`hpart://file/firmwares/<name>` target). |

## Tasks

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/task` | Create a task. |
| GET | `/task` | List tasks. |
| GET | `/task/:task_id` | Read a task. |
| POST | `/task/:task_id` | Update a task. |
| POST | `/task/:task_id/terminate` | Terminate one task. |
| POST | `/task/:task_id/launchprinter` | Assign / launch a task to a printer. |
| POST | `/task/sortongoing` | Reorder ongoing tasks. |
| POST | `/task/terminatetasks` | Terminate multiple tasks. |
| POST | `/task/finished/delete` | Delete finished tasks. |

## Support logs

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/support/logs` | Request a support-log collection task. |
| GET | `/support/logs` | List support-log tasks. |
| GET | `/support/logs/:log_task_id/logfile` | Download a collected log file. |
| DELETE | `/support/logs/:log_task_id` | Delete a support-log task. |
| PUT | `/support/:log_task_id/logfile` | Upload a log file (agent/printer-facing). |

## Statistics & export

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/stat/uptime` | Device / farm uptime statistics. |
| POST | `/stat/export` | Start a statistics export. |
| GET | `/stat/export/latest` | Latest export status. |
| GET | `/stat/export/latest/file` | Download the latest export (CSV). |

## Notifications

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/notification/config` | Read notification configs. |
| POST | `/notification/config` | Create a notification config. |
| PATCH | `/notification/config/:config_id` | Update a config. |
| DELETE | `/notification/config/:config_id` | Delete a config. |
| GET | `/notification/config_test` | Test a notification config. |
| GET | `/notification/status` | Notification worker / delivery status. |

## Alarms

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/alarm/events` | Farm alarm-event list / stream. |

## Printer-facing routes

These are called by the **printer** (UA `ESP32 HTTP Client/1.0`) against the `apix_v` base it
received at bind time, not by the desktop client. Detail in [farm-tasks.md](./farm-tasks.md) and
[farm-mqtt.md](./farm-mqtt.md).

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/device/sync` | Printer pulls its Farm-screen task cards (`cmd: "task_list"`). |
| POST | `/device/launchtask` | Printer-side Start (`cmd: "sub_start"`). |
| PUT | `/device/:dev_id/liveview` | Liveview control / JPEG upload. |
| PUT | `/device/:dev_id/task_report` | Printer reports task progress. |
| PUT | `/device/:dev_id/unbind` | Single-device unbind alias (response carries `unbind_results`, [farm-bind.md](./farm-bind.md)). |

> Note that `POST /device/launchtask` (printer-side, unscoped) and
> `POST /device/:dev_id/launchtask` (client-directed, device-scoped) are **distinct** routes with
> the same leaf name — do not collapse them.

## Debug (operator caution)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/debug/dbkeys` | Enumerate the server's internal Badger database keys. |
| GET | `/debug/dbvalue` | Read a raw database value by key. |

> **Operator note.** In the observed build these two `/debug` routes are registered **without**
> the authentication middleware that guards the rest of the API, so any client that completes the
> `:8888` transport can enumerate and read internal database entries. The farm server is intended
> for a **trusted LAN**: keep `:8888` firewalled to your management network and do not expose it
> to the internet. This note is defensive guidance for self-hosters running their own server; it
> is a property of the route registration, not an exploit or a way past the transport.
