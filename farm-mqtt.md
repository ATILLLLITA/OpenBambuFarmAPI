# Farm MQTT

The farm broker (`mochi-mqtt`, hosted by the server on `1883/TLS`) carries all printer control
and telemetry. The printer connects **out** to it (farm mode), authenticates as `BambuFarm`
(see [farm-tls.md](./farm-tls.md)), then subscribes to its request topic.

## Topics

| Topic | Direction | Payload |
|-------|-----------|---------|
| `device/<serial>/request` | server → printer | commands (`print`, `info`, `system`, `bind`, …) |
| `device/<serial>/report`  | printer → server | telemetry & command results (mostly `push_status`) |

Every command object carries a `command` and a `sequence_id`; the server assigns the
`sequence_id` (the broker injects one if a producer omits it) and matches it against the report.

## Commands the server actually sends

Across a full **unattended** bind → print → finish session (no operator interaction), the
**only** things the server published to the printer were:

| Namespace.command | Purpose | Frequency |
|-------------------|---------|-----------|
| `print.project_file` | start a print, including one selected from the printer's Farm screen | once per job |
| `system.sntp` | time sync | periodic |
| `info.get_version` | module/firmware versions | occasional |
| `bind.unbind` | unbind (see [farm-bind.md](./farm-bind.md)) | on unbind |

Crucially, the server sends **nothing at the end of the print** — no `stop`/`finish`/
"collected"/`bed_clean` is auto-issued; the Collected prompt is printer-native (see
*Print lifecycle* below). Operator actions add on-demand controls on top of this baseline —
see *Device control commands* below (those include `stop`, `pause`, `bed_clean`, etc., but only
when a human clicks them).

### Printer-screen queue launch ordering

For a job selected on the printer's own Farm screen, the REST ACK and MQTT start are separate
steps. The observed order is:

```text
POST /device/launchtask {"cmd":"sub_start", ...}
<- {"cmd":"sub_start","err_code":0}
... about 1.26 seconds ...
MQTT print.project_file
GET /file/f3mf/<f3mf-id>/f3mffile
```

The ACK alone does not start anything. A clone that returns success but omits the later MQTT
publish leaves the panel at `Waiting to print`. This printer-screen path did **not** include an
extra `bed_clean` command. See [farm-tasks.md](./farm-tasks.md) for the REST task list, ID
relationships, and per-copy state.

## `print.project_file` — start a print

The farm server sends a **minimal** form (fewer fields, and slightly different field names,
than the cloud/SD form in OpenBambuAPI). Observed payload on `device/<serial>/request`:

```json
{
  "print": {
    "sequence_id": "10009",
    "command": "project_file",
    "param": "Metadata/plate_1.gcode",
    "project_id": "0",
    "profile_id": "0",
    "task_id": "2502",
    "subtask_id": "2503",
    "subtask_name": "«job name»",
    "url": "hpart://file/f3mf/2501/f3mffile",
    "timelapse": false,
    "bed_leveling": true,
    "flow_cali": false,
    "vibration_cali": false,
    "ams_mapping2": [ { "ams_id": 255, "slot_id": 0 } ],
    "use_ams": false,
    "size": 53356,
    "md5": "«file md5»",
    "plate_md5": "«plate md5»",
    "bed_temp": 70
  }
}
```

| Field | Notes |
|-------|-------|
| `param` | plate g-code path inside the 3MF, `Metadata/plate_<N>.gcode` |
| `task_id` / `subtask_id` | farm task identifiers — what makes the job a **managed task** |
| `url` | **`hpart://file/f3mf/<id>/f3mffile`** — farm scheme; the printer fetches it via the server's REST `:8888` (see [farm-rest.md](./farm-rest.md)). The `<id>` is the **f3mf file id** (`2501` above), **not** the task id — note `task_id:2502` / `subtask_id:2503` differ from the url's `2501`. |
| `ams_mapping2` | array of `{ams_id, slot_id}`; `ams_id:255` = external spool |
| `use_ams` | `false` with the `255` external mapping |
| `size` / `md5` / `plate_md5` | file size + hashes for integrity |
| `bed_temp` | initial bed target (°C) |

### Field-name differences vs. OpenBambuAPI (cloud/SD form)

| OpenBambuAPI (generic) | Farm (observed) |
|------------------------|-----------------|
| `bed_levelling` (two L's) | `bed_leveling` (one L) |
| `layer_inspect` present | absent |
| `bed_type: "auto"` present | absent |
| `ams_mapping` (string) | `ams_mapping2` (array of `{ams_id,slot_id}`) |
| `url: file:///mnt/sdcard` | `url: hpart://file/f3mf/<id>/f3mffile` |

Field-name spelling appears firmware/path-dependent; the farm values above are what this
C12 / `01.06.30.01` printer received and accepted.

## Device control commands (`opt` → MQTT)

Interactive controls from the Farm Manager client arrive as REST `POST /device/<sn>/opt`
([farm-rest.md](./farm-rest.md)) and are bridged 1:1 to a command on `device/<sn>/request`.
Each row below is one observed control, decrypted from live GUI actions (namespace in **bold**):

| Command | Namespace | Key params | Effect |
|---------|-----------|-----------|--------|
| `print_speed` | **print** | `param`: `"1"`–`"4"` | speed preset: 1 silent · 2 standard · 3 sport · 4 ludicrous |
| `ledctrl` | **system** | `led_node` (`chamber_light`), `led_mode` (`on`/`off`/`flashing`) | chamber light control |
| `gcode_line` | **print** | `param`: raw G-code line(s), e.g. `"M140 S60\n"` | run arbitrary G-code (incl. heater set) |
| `ipcam_record_set` | **camera** | `control`: `enable`/`disable` | timelapse/recording toggle |
| `calibration` | **print** | `option`: bitmask | run calibrations: `2` bed-leveling · `4` vibration · `8` flow (OR'd together) |
| `pause` / `resume` / `stop` | **print** | — | pause / resume / abort the active job |
| `bed_clean` | **print** | — | bed-clean / post-print routine |

Example (speed → ludicrous):

```json
{ "print": { "sequence_id": "10021", "command": "print_speed", "param": "4" } }
```

```json
{ "system": { "sequence_id": "10022", "command": "ledctrl",
              "led_node": "chamber_light", "led_mode": "on" } }
```

```json
{ "print": { "sequence_id": "10023", "command": "calibration", "option": 6 } }
```
*(`option: 6` = `2|4` = bed-leveling + vibration.)*

## Filament / external spool

A spool is configured with **`print.ams_filament_setting`** — there is **no** separate
`external_filament_setting` command; the *external* spool is just this command with
**`ams_id: 255`** (the same `255` that appears in `project_file.ams_mapping2`). Observed shape:

```json
{
  "print": {
    "sequence_id": "10031",
    "command": "ams_filament_setting",
    "ams_id": 255,
    "tray_id": 0,
    "tray_info_idx": "«filament preset id»",
    "tray_type": "PLA",
    "tray_color": "«RRGGBBAA»",
    "nozzle_temp_min": 190,
    "nozzle_temp_max": 240
  }
}
```

| Field | Meaning |
|-------|---------|
| `ams_id` | `255` = external spool (no AMS); `0..` = an AMS unit |
| `tray_id` | slot within the AMS (`0` for external) |
| `tray_info_idx` | Bambu filament-preset id |
| `tray_type` / `tray_color` | material + ARGB colour |
| `nozzle_temp_min/max` | temperature window (°C) |

## `report` — `push_status`

The printer streams `push_status` (hundreds per job) on `device/<serial>/report`. A full
status snapshot is requested with `pushing.pushall`; subsequent messages are deltas. Selected
fields (consistent with OpenBambuAPI):

```json
{
  "print": {
    "command": "push_status",
    "gcode_state": "RUNNING",
    "mc_percent": 87,
    "mc_print_stage": "2",
    "print_type": "cloud",
    "nozzle_temper": 238.5,
    "bed_temper": 69.6,
    "wifi_signal": "-48dBm",
    "sequence_id": "971",
    "msg": 1
  }
}
```

| Field | Meaning |
|-------|---------|
| `gcode_state` | job state machine (below) |
| `mc_percent` | progress 0–100 |
| `mc_print_stage` | coarse stage (`1` idle/setup, `2` printing) |
| `print_type` | becomes **`cloud`** for a farm-managed task; `idle` otherwise |
| `msg` | `0` = full snapshot, `1` = delta |

## Print lifecycle & the Collected / Reprint / Cancel screen

Observed `gcode_state` progression for one managed job:

```
IDLE → PREPARE → RUNNING → FINISH → IDLE
```

At completion the printer reports:

```json
{ "print": { "command": "push_status", "gcode_state": "FINISH",
             "mc_percent": 100, "mc_print_stage": "1", "print_type": "idle" } }
```

**Key finding:** the server sends **no end-of-print command**. The printer's on-screen
**Collected / Reprint / Cancel** prompt is **firmware-native**, raised when `gcode_state`
reaches `FINISH` for a job that was started as a **managed task** (i.e. via `project_file` with
valid `task_id` + `subtask_id` and the `hpart://` `url`, which the firmware tracks as
`print_type: cloud`). Pressing **Collected** on the panel does **not** emit a distinct
printer→server command — it only advances the next `push_status`.

Practical consequence for a reimplementation: to make the panel show Collected/Reprint/Cancel,
start the job with the **exact managed `project_file` above** and let the printer reach
`FINISH`. Do not expect (or need) a server-sent "collected" trigger.

**Confirmed by reimplementation:** a server emitting this minimal `project_file` (same field
names, no extra fields) and letting the printer reach `FINISH` reproduces the on-printer
Collected/Reprint/Cancel screen; a bloated/renamed `project_file` (extra `print_type`,
calibration flags under different names, etc.) starts the job as a plain print and the screen
does not appear. The managed-task shape is the trigger.
