# Farm firmware upgrade (offline OTA)

Farm Manager can push a firmware package to a bound printer **without** the printer reaching
Bambu's cloud: the operator supplies an official offline-OTA `.zip`, the server stores it, and
the printer downloads it from the farm server over the same `:8888` channel it uses for job
files. This is the **offline OTA** path.

Unlike the job-launch flow ([farm-tasks.md](./farm-tasks.md)), the firmware command is one of the
**sensitive** commands a **secured** printer requires to be signed — the reference here is a
secured P2S-class machine (model token `N7`), so its `upgrade` command travels through the
MQTT-Sec layer in [farm-command-security.md](./farm-command-security.md).

## Flow

```text
client  GET  https://…/v1/iot-device-service/api/device/offlineota?dev_model=<model>   (official Bambu catalog)
client  (validate filename/model + farm minimum-version rules — local)
client  GET  /file/firmwares/<firmware_name>/info      check server catalog
client  download the official package (.zip, ~1 GB)
client  POST /file/firmwares                           upload package to the farm server (multipart)
client  PUT  /device/firmware                          request the upgrade
server  -> MQTT  upgrade.offline_upgrade  (signed)     dispatch to the printer
printer GET  /file/firmwares/<firmware_name>           download the package from the farm server
printer -> MQTT  print.upgrade_state                   report flashing progress
```

Two catalogs are involved and should not be confused: the **official Bambu** offline-OTA catalog
(`…/iot-device-service/api/device/offlineota`, queried by the *client* to obtain the correct
package for a model) and the **farm server's own** catalog under `/file/firmwares` (what the
*printer* downloads from). The client bridges one to the other by uploading the package it fetched.

## Routes involved

| Step | Route | Notes |
|------|-------|-------|
| Client fetches the package list | official Bambu `GET …/device/offlineota?dev_model=<model>` | Outbound to Bambu; returns the model's published packages. Not a farm-server route. |
| Server catalog query | `GET /file/firmwares`, `GET /file/firmwares/:firmware_name/info` | Returns persisted metadata; survives a server restart. |
| Catalog upload | `POST /file/firmwares` | Multipart fields `firmware` (the `.zip` bytes) and `name`, optional MD5. Validates the offline-firmware filename, stores metadata in the database and the bytes under the server's `assets/firmwares` store. |
| Request upgrade | `PUT /device/firmware` | Triggers the signed MQTT dispatch below. |
| Printer download | `GET /file/firmwares/:firmware_name` | Printer-facing authenticated delivery; supports full and range responses. |

Firmware-related error codes (`no firmware`, `firmware not exist`, `firmware invalid`,
`firmware exist`, `firmware is too low`, and the `upgrade *` state cluster) are in
[farm-errors.md](./farm-errors.md).

## The MQTT command — `upgrade.offline_upgrade`

`PUT /device/firmware` causes the server to publish an `upgrade` command on
`device/<serial>/request`. The command content, **before** the MQTT-Sec envelope is applied, is:

```json
{
  "upgrade": {
    "command": "offline_upgrade",
    "sequence_id": "<10000..13999>",
    "url": "hpart://file/firmwares/<firmware_name>",
    "version": "<version parsed from the package filename>"
  }
}
```

| Field | Meaning |
|-------|---------|
| `command` | `offline_upgrade` — the offline (farm-hosted) OTA variant |
| `sequence_id` | server-assigned, from the external command range (`10000`–`13999`) |
| `url` | **`hpart://file/firmwares/<firmware_name>`** — same farm indirection scheme as job files ([farm-rest.md](./farm-rest.md)); the printer resolves it to `GET /file/firmwares/<name>` on `:8888` |
| `version` | the target version, parsed from the package filename |

- The command carries an internal source id of `src_id = 4` and expects a correlated printer ACK
  within about **10 seconds**.
- On a **secured** printer this command is **signed** via MQTT-Sec
  ([farm-command-security.md](./farm-command-security.md)); an unsigned firmware command is
  rejected like any other sensitive command. The signing key is Bambu's and is **not** in this
  repository (`«REDACTED»`).

## Version rules & downgrade

The client enforces a **farm minimum-version** rule locally, before any upload or dispatch. An
attempted downgrade below that floor is refused on the client with:

```
The firmware version is below the adaptation requirements of the Farm Manager
```

The server has a matching guard — error `1069 firmware is too low`
([farm-errors.md](./farm-errors.md)). The filename's model token must also match the target
printer's model. Historical/older packages are selectable in the client only when they still meet
the farm floor.

## Progress reporting

Once flashing begins the printer reports progress through `print.upgrade_state` messages on
`device/<serial>/report` (alongside its normal `push_status`, [farm-mqtt.md](./farm-mqtt.md)).
The server's `upgrade *` error codes (`15`–`21`) mirror this state machine: `upgrade device not
ready`, `upgrade device in sch` (busy with a scheduled task), `upgrade failed in downloading`,
`upgrade downloading timeout`, `upgrade timeout`, `upgrade failed`, `upgrade unexpected state`.

*(inferred)* A partial `upgrade_state` report can omit fields; a consumer should merge reports by
`sequence_id` rather than assuming every report is a full snapshot.

## Observed vs. pending

Directly observed against a bound, secured P2S (`N7`) and the recovered farm server: the client's
official-catalog query, the `GET /file/firmwares` / `…/info` server catalog reads, the package
upload and ~1 GB clone-side storage, the `PUT /device/firmware` request (HTTP 200), the signed
`offline_upgrade` MQTT dispatch, and a correlated positive printer acceptance. The captured run
was a **same-version reinstall** (the selected package matched the printer's current firmware).

**Not yet confirmed end-to-end** *(pending)*: terminal flash success, printer reboot, MQTT
reconnect, and the new version appearing in `GET /devices2` — and, separately, an actual
cross-version downgrade (blocked by the version floor above). Nothing here should be read as proof
that an arbitrary version can be flashed; it documents the transport and command contract only.

## Reimplementation checklist

- Accept a multipart upload at `POST /file/firmwares`; validate the model token in the filename
  and persist metadata separately from the bytes so the catalog survives a restart.
- Serve the package to the printer at `GET /file/firmwares/<name>` with range support.
- On `PUT /device/firmware`, publish exactly one `upgrade.offline_upgrade` with an
  `hpart://file/firmwares/<name>` url and the parsed target version.
- Enforce the farm minimum-version floor (`firmware is too low`) before dispatch.
- **Sign** the command for secured printers ([farm-command-security.md](./farm-command-security.md));
  an insecure printer needs no signature.
- Track `print.upgrade_state` and surface the `upgrade *` errors on failure.
