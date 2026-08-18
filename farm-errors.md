# Farm error codes

The `:8888` REST server ([farm-rest.md](./farm-rest.md)) returns application errors as a small
JSON envelope carrying a numeric **`code`** and a human **`message`**, with a matching HTTP
status. The same numeric codes appear in client- and printer-facing responses, so a
reimplementation that wants the official Farm Manager client (and the printer's Farm screen) to
react correctly should reproduce the **code**, not just the HTTP status.

This is the complete code/message/HTTP-status set the server can emit (the server's internal
`xerr` table). Codes are protocol facts: each one is confirmable by triggering the matching
condition against your own server. Messages are quoted verbatim as they appear on the wire,
including their original spelling and grammar.

> Envelope shape *(inferred from the code/message pairing and observed responses)*: an error
> response is a JSON object of the form `{ "code": <n>, "message": "<text>" }` (some routes wrap
> it as `{ "code", "message", "data" }`). Success uses `code: 0`. The printer-side task routes
> use a **different** envelope with `err_code` rather than `code` — see
> [farm-tasks.md](./farm-tasks.md) (`{"cmd":"sub_start","err_code":0}`).

## Reused codes (important)

A handful of numeric codes are **reused** for two different conditions with different HTTP
statuses. Treat `(code, message)` — or `(code, http)` — as the identifying pair, not the code
alone:

| Code | Meaning A | Meaning B |
|------|-----------|-----------|
| `1018` | `invalid license` (400) | `invalid activate uid` (409) |
| `1022` | `launch task err device is suspended` (409) | `launch task err for task is launched` (400) |
| `1043` | `sdk invalid login ticket` (400) | `sdk too many requests` (429) |
| `1063` | `user not exist` (400) | `too many devices` (400) |

## Core codes (`1`–`23`)

General server, auth, session, versioning, upgrade and security errors.

| Code | HTTP | Message |
|------|------|---------|
| `1` | 500 | internal server error |
| `2` | 500 | internal db error |
| `3` | 403 | not activated |
| `4` | 400 | invalid request |
| `5` | 400 | bad request |
| `6` | 401 | invalid token |
| `7` | 401 | token expired |
| `8` | 426 | client upgrade required |
| `9` | 426 | server upgrade required |
| `10` | 400 | server and client mismatch |
| `11` | 400 | parameter out of range |
| `12` | 400 | 3mf not exist in from folder |
| `13` | 408 | support log timeout |
| `14` | 500 | file system error |
| `15` | 400 | upgrade device not ready |
| `16` | 400 | upgrade device in sch |
| `17` | 400 | upgrade failed in downloading |
| `18` | 408 | upgrade downloading timeout |
| `19` | 408 | upgrade timeout |
| `20` | 500 | upgrade failed |
| `21` | 400 | upgrade unexpected state |
| `22` | 400 | security failed |
| `23` | 400 | parameter error |

The `426 Upgrade Required` pair (`8`/`9`) and `server and client mismatch` (`10`) are the
server↔client version gate: the official client and server refuse to interoperate across an
incompatible protocol version. The `upgrade *` cluster (`15`–`21`) is the **device firmware**
upgrade state machine — see [farm-firmware.md](./farm-firmware.md).

## Files, folders & 3MF

| Code | HTTP | Message |
|------|------|---------|
| `1000` | 400 | folder delete not empty |
| `1001` | 400 | folder name exist |
| `1002` | 400 | folder not exist |
| `1003` | 400 | is not gcode 3mf |
| `1004` | 400 | is not one plate gcode 3mf |
| `1014` | 400 | bad 3mf |
| `1055` | 416 | file out of range |
| `1075` | 400 | folder order mismatch |

`is not one plate gcode 3mf` (`1004`) reflects the farm requirement that a launched job be a
single-plate sliced 3MF (consistent with `param: "Metadata/plate_1.gcode"` in
[farm-mqtt.md](./farm-mqtt.md)). `416` with `file out of range` is the range-request boundary on
the job/asset download routes.

## Devices, MQTT & operations

| Code | HTTP | Message |
|------|------|---------|
| `1005` | 408 | mqttx request failed |
| `1006` | 400 | unbinded device |
| `1035` | 400 | device offline |
| `1049` | 400 | operation not execute |
| `1050` | 400 | operation not support |
| `1051` | 400 | device busy |
| `1052` | 400 | device internal error, check sd card or other |
| `1054` | 400 | operation hardware not support |
| `1072` | 400 | extruder no filament |

`mqttx request failed` (`1005`, **408**) is what a REST caller sees when the server's bridged
MQTT command ([farm-rest.md](./farm-rest.md) `/opt`) does not get a printer reply within the
request window.

## Tasks & launch

| Code | HTTP | Message |
|------|------|---------|
| `1009` | 400 | task ongoing |
| `1019` | 409 | opt outdated ongoing task |
| `1020` | 409 | launch task err finished or terminated |
| `1021` | 409 | launch task err no subtask |
| `1022` | 409 | launch task err device is suspended |
| `1022` | 400 | launch task err for task is launched |
| `1030` | 400 | update not ongoing task |
| `1031` | 400 | update task bad task cnt |

`launch task err no subtask` (`1021`) is the code behind the "task count ran out / task removed"
failure signature in [farm-tasks.md](./farm-tasks.md): a launch was accepted but no unused
per-copy subtask remained.

## Firmware

| Code | HTTP | Message |
|------|------|---------|
| `1008` | 400 | no firmware |
| `1010` | 400 | firmware not exist |
| `1012` | 400 | firmware invalid |
| `1013` | 409 | firmware exist |
| `1069` | 400 | firmware is too low |

`firmware is too low` (`1069`) is the farm minimum-version rule; the client also enforces an
equivalent check locally before upload (see [farm-firmware.md](./farm-firmware.md)).

## Bulk operations

| Code | HTTP | Message |
|------|------|---------|
| `1015` | 400 | unsupported option |
| `1016` | 400 | bulk actions ongoing |
| `1017` | 404 | bulk actions not found |

Backs `POST /device/bulkopt` and `GET /device/bulkopt/:bulk_id`.

## Support logs

| Code | HTTP | Message |
|------|------|---------|
| `1032` | 409 | too much support logs |
| `1033` | 404 | support log not exist |
| `1034` | 409 | too much ongoing logs |

Back the `/support/logs` route group; the core code `13 support log timeout` (408) also belongs
to this flow.

## Users & authentication

| Code | HTTP | Message |
|------|------|---------|
| `1023` | 400 | wrong user name or password |
| `1024` | 400 | user name duplicated |
| `1025` | 403 | user not authorized |
| `1026` | 429 | login too frequent |
| `1027` | 403 | bambu login not supported |
| `1028` | 400 | bambu login failed |
| `1056` | 400 | username must be greater than 1 character and less than  128 byte |
| `1057` | 400 | password must contain: numbers, letters, special characters, at least 8 byte long, can not over 64 byte |
| `1062` | 400 | user exist |
| `1063` | 400 | user not exist |
| `1063` | 400 | too many devices |

`bambu login not supported` (`1027`) is the `LAN_FARM` refusal of the cloud-account login path —
consistent with the farm being a self-hosted, cloud-independent mode. The verbatim double space
in `1056` is preserved from the wire.

## Binding

| Code | HTTP | Message |
|------|------|---------|
| `1029` | 400 | printer bind failed |
| `1036` | 400 | bind device info invalid |
| `1037` | 400 | device restriction limit |
| `1038` | 400 | bind network error |
| `1039` | 400 | bind no device response |

These are the REST-visible failures of the `PUT /bind2` → TCP-3002 handshake in
[farm-bind.md](./farm-bind.md). `bind no device response` (`1039`) corresponds to the printer
never answering the server's 3002 `detect`/`login`; `device restriction limit` (`1037`) is the
per-license device-count cap.

## SDK

| Code | HTTP | Message |
|------|------|---------|
| `1041` | 400 | sdk request with invalid parameter |
| `1042` | 500 | sdk request request failed |
| `1043` | 400 | sdk invalid login ticket |
| `1043` | 429 | sdk too many requests |
| `1044` | 401 | sdk not logged |
| `1045` | 400 | sdk activation failed |
| `1046` | 400 | sdk dev login failed |

Backs the `/sdk/*` route group (third-party/SDK integration login, license and device-bind).

## License & activation

| Code | HTTP | Message |
|------|------|---------|
| `1018` | 400 | invalid license |
| `1018` | 409 | invalid activate uid |
| `1070` | 400 | server activated |
| `1071` | 400 | server activate failed |
| `1073` | 400 | time out of cert valid date |

`time out of cert valid date` (`1073`) is a licensing/clock-validity guard tied to the license
certificate window.

## Notification, statistics, HMS & validation

| Code | HTTP | Message |
|------|------|---------|
| `1047` | 404 | notification config not found |
| `1048` | 400 | notification config exist |
| `1040` | 400 | statistics out of range |
| `1076` | 409 | stat export ongoing |
| `1077` | 404 | stat export not found |
| `1066` | 400 | Insufficient available space |
| `1067` | 400 | Hms model not support |
| `1068` | 400 | Hms info not found |
| `1007` | 400 | no tag found |
| `1053` | 400 | invalid filament setting |
| `1058` | 400 | date and timestamp are both empty |
| `1059` | 400 | date must format '2025-01-01 15:00:00' |
| `1060` | 400 | timestamp out of range, is seconds level |
| `1061` | 400 | time zone invalid, need like: 'UTC+08:00' |
| `1064` | 400 | region need CN \| COM |
| `1065` | 400 | env need QA\|DEV\|PRE\|PREUS\|CN\|COM |
| `1074` | 400 | tag name exist |

`Hms*` (`1067`/`1068`) refers to Bambu's **HMS** (Health Management System) diagnostic codes.
The `region`/`env` validators (`1064`/`1065`) enumerate the server's accepted deployment regions
and environments.

## RTSP liveview relay

| Code | HTTP | Message |
|------|------|---------|
| `1078` | 429 | rtsp relay server full |
| `1079` | 429 | rtsp relay device full |
| `1080` | 408 | rtsp relay init timeout |
| `1081` | 502 | rtsp relay init failed |

These four are the only wire-visible surface of a server-side **RTSP relay** for liveview
(beyond the JPEG PUT/GET path in [farm-rest.md](./farm-rest.md)). Their existence shows the farm
server can broker an RTSP stream and enforces per-server and per-device relay capacity
*(inferred; the relay's media path itself was not captured on the wire in this research)*.

## Special / transport

| Code | HTTP | Message |
|------|------|---------|
| `10001` | 403 | not allowed |
| `10002` | 500 | no response |
| `10003` | 500 | negtive response |

The `negtive response` spelling (`10003`) is verbatim.

## MQTT command-security codes

Separate from the REST `xerr` table above, a **secured** printer answers a bad or missing
per-command signature over MQTT with its own codes — see
[farm-command-security.md](./farm-command-security.md):

| Code | Meaning |
|------|---------|
| `84033543` | MQTT command signature verification failed (bad or absent signature) |
| `84033545` | signature / task-id verification failed |
