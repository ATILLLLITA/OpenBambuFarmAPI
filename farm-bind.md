# Binding & unbinding a printer (TCP 3002 + MQTT)

Binding is the one place where the **server dials the printer**. The server opens a TLS 1.2
connection to the printer on **port 3002**, identifies it, then hands it the farm's endpoints.
After a successful bind the printer connects *out* to the farm MQTT broker (`1883`) and starts
streaming telemetry.

Messages on 3002 are JSON objects wrapped in the printer's framing (`a5 a5 <len-LE> … <crc16>`)
inside the TLS stream, one logical command per record.

## 1. `detect` — identify the printer

Server → printer:

```json
{ "login": { "command": "detect" } }
```

The printer replies with its identity (serial, model, firmware, current bind state). The
server uses this to confirm the target and to fill placeholders in the next message.

## 2. `login` — hand over the farm endpoints

Server → printer (the key message). The firmware substitutes its own runtime values for the
`VAR_*` placeholders — `VAR_SIP` = server IP, `VAR_MPROTS` = MQTT scheme, `VAR_PSN` = printer
serial:

```json
{
  "login": {
    "sequence_id": 2021,
    "command": "login",
    "apix_v":   "http://VAR_SIP:8888/",
    "emqx_v":   "VAR_MPROTS://VAR_SIP:1883/",
    "timezone": "UTC-07:00",
    "mode":     "LAN_FARM",
    "server_id": "«REDACTED — the farm server's id»",
    "liveview_v": "http://VAR_SIP:8888/device/VAR_PSN/liveview"
  }
}
```

| Field | Meaning |
|-------|---------|
| `apix_v` | REST API base the printer will call (`:8888`) |
| `emqx_v` | MQTT broker URL the printer will connect out to (`:1883`) |
| `mode` | `LAN_FARM` — selects farm behaviour (printer = MQTT client, no cloud) |
| `server_id` | identifies the owning farm; the printer remembers it (see *unbind* below) |
| `liveview_v` | URL the printer PUSHes liveview JPEGs to (see [farm-rest.md](./farm-rest.md)) |
| `timezone` | POSIX-style TZ pushed to the printer |

Printer → server:

```json
{ "login_report": { "command": "login", "sequence_id": 2021, "status": "SUCCESS" } }
```

If the printer is already bound to a different farm, it refuses:

```
"server id not same, need unbind"
```

→ unbind from the previous farm first, then bind.

After `SUCCESS`, the printer opens its outbound MQTT session to `VAR_SIP:1883` and the
device appears in [`push_status`](./farm-mqtt.md) telemetry.

> The client-facing trigger for this whole flow is REST `PUT /bind2` on `:8888`
> (see [farm-rest.md](./farm-rest.md)); `bind2` is what makes the server perform the 3002 login.

## 3. Unbind

Unbind does **not** use port 3002. The client calls REST `DELETE /bind` (body
`{ "dev_ids": [ "<serial>" ] }`) on `:8888`, and the server then publishes an `unbind` command
to the printer over the **existing MQTT session**:

Server → printer, topic `device/<serial>/request`:

```json
{ "bind": { "sequence_id": "12021", "command": "unbind" } }
```

Namespace is **`bind`** (not `print`). The printer emits a few more `push_status` reports and
then drops its MQTT connection.

### Asymmetry summary

| Action | Direction | Channel |
|--------|-----------|---------|
| **bind** | server → printer | TCP 3002 `login`/`detect` |
| **unbind** | server → printer | MQTT `device/<sn>/request` `bind.unbind` |
