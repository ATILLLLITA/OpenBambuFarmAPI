# Farm REST API (`:8888`)

The server exposes a TLS REST API on `8888` (Go, `go-zero` framework). Two clients use it:

- the **Farm Manager client** (Electron, UA `BambuFarmManagerClient/2.3.0 … Electron/33.2.1`)
  — control + heavy polling, from the LAN IP and from `127.0.0.1`;
- the **printer** (UA `ESP32 HTTP Client/1.0`) — file download + liveview upload, using the
  `apix_v` base it received during bind.

Endpoints below are those **observed in use** (from request logs / decrypted traffic); the
surface is larger. Request bodies are JSON unless noted.

## ⚠️ Port 8888 is dual-protocol (HTTP **and** HTTPS) — critical for reimplementation

The bind `login` hands the printer `apix_v: "http://<server>:8888/"` and
`liveview_v: "http://<server>:8888/…"` — i.e. **plain HTTP**. But the printer's actual **job
download** on `8888` was observed as **TLS** (`ClientHello` with SNI
`farm_server.bambulab.com`). So `8888` must answer **both** schemes on the same port:

| Traffic | Scheme on `:8888` |
|---------|-------------------|
| Client polling, liveview JPEG PUT/GET | **HTTP** (plain) |
| Printer fetching the job (`hpart://` → `/file/f3mf/.../f3mffile`) | **HTTPS** (TLS, SNI `farm_server.bambulab.com`) |

**Verified against a clone:** if `:8888` serves *plain HTTP only*, the printer **ACKs**
`project_file` over MQTT but then **never downloads the file and stays `IDLE`**. Serving HTTPS
(with the farm chain / SNI from [farm-tls.md](./farm-tls.md)) on `:8888` is required for the
printer to fetch the model and actually start. A reimplementation must therefore terminate both
HTTP and TLS on `8888` (protocol-sniff the first bytes, or run both and route by client).

## Printer binding & inventory

| Method & path | Purpose |
|---------------|---------|
| `PUT /bind2` | Bind a printer — triggers the server's TCP-3002 `login` ([farm-bind.md](./farm-bind.md)). ~40 s timeout. |
| `DELETE /bind` | Unbind. Body `{ "dev_ids": [ "<serial>" ] }` → server publishes `bind.unbind` over MQTT. |
| `PUT /device/{serial}/unbind` | Unbind alias for a single device — same lifecycle as `DELETE /bind`; response carries `unbind_results`. |
| `GET /captain` | Client heartbeat / session keepalive (frequent, heavy). |
| `GET /devices2?use_lite=true` | List bound devices (lite view). |
| `GET /device/{serial}` | One device's full state. |
| `POST /device/sync` | Push/refresh device state. |
| `POST /device/bulkopt` | Bulk operation across devices. |
| `PUT /device/{serial}/name` | Rename a device. |

## Tasks & printing

| Method & path | Purpose |
|---------------|---------|
| `GET /task` | List tasks. |
| `POST /device/{serial}/opt` | Client→printer bridge: an `opt` request is forwarded as the matching MQTT command (e.g. the `project_file` in [farm-mqtt.md](./farm-mqtt.md)). |
| `POST /device/{serial}/launchtask` | Start a task on a device. |
| `POST /task/{id}/launchprinter` | Assign/launch a task to a printer. |

The `/opt` request body is effectively the decrypted MQTT command — the REST layer is a thin
bridge to `device/<sn>/request`.

## Files & liveview

| Method & path | Purpose |
|---------------|---------|
| `POST /file/upload3mf` | Upload a 3MF to the server. |
| `GET /file?asset_path=…` | Fetch an asset by server-side path. |
| `GET /file/f3mf` , `GET /file/f3mffolder` | 3MF listing/metadata. |
| `GET /file/f3mf/{id}/f3mffile` | **Download the job file.** The `hpart://file/f3mf/{id}/f3mffile` URL in `project_file` resolves here. |
| `PUT /device/{serial}/liveview` | Liveview control. |

**Liveview path:** the printer **PUT**s JPEG frames to the server; the client then **GET**s
`/file?asset_path=liveview\{serial}\liveview.jpeg`. (The `liveview_v` URL handed to the printer
at bind time is `http://<server>:8888/device/<serial>/liveview`.)

**`hpart://` scheme:** a farm indirection — `hpart://file/f3mf/{id}/f3mffile` tells the printer
to download the job from the server's REST `:8888` rather than from cloud or SD.

## Login / auth

| Method & path | Purpose |
|---------------|---------|
| `POST /login/local/{tokens,tickets}` | Local (farm) login — challenge/salt ticket scheme. |
| `POST /login/bambu/{tokens,tickets}` | Bambu-account login path. |
| `PUT /users/bind2`, `PUT /users/{user}/{passwd,role,name}`, `POST /users/admin/reset` | User management. |

Login uses a challenge/salt **ticket** exchange; credentials and tickets are secrets and are
not reproduced here.

## `GET /devices2` response model (client parity)

For a clone to satisfy the official Electron client, the device list must carry the fields the
client filters/renders on. Key ones observed (non-exhaustive):

| Field | Why the client needs it |
|-------|-------------------------|
| `bind: "bind"` | marks the device as bound (else hidden / "Total: 0") |
| `devicePool[].schable` | schedulability flag used by the queue UI |
| `upgrade_info.current_ver` | firmware version shown; gates "Need Upgrade Firmware" |
| `task_info.f3mf_info.model_setting_config.cur_thumbnail_url` | job thumbnail |
| tag objects (`/tags`) | grouping/filtering |
| `task.statistics` (`finished` / `stopped` / `printing`) | dashboard counters |

This is server-model detail (what the server returns), not wire protocol — included for
reimplementation parity; field set varies by client version.

## Notes
- Response bodies are not logged by default; the framework can be made to log them
  (`go-zero` verbose mode) for analysis — a server-operator action on your own server.
- The printer model in responses for this device is **`C12`**.
