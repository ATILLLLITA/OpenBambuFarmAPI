# Printer-side farm task queue

Farm Manager 2.4.0 exposes queued jobs on the printer's own **Farm** screen. This is not a
single REST start call: REST supplies the task cards and acknowledges the operator's choice,
then the server sends the actual `print.project_file` over MQTT.

The complete observed exchange is:

```text
desktop POST /task                         create server-side task
printer POST /device/sync                  pull Farm-screen cards
printer POST /device/launchtask            select one task copy
server  -> exact REST ACK                  accept the selection
server  -> MQTT print.project_file         dispatch after the ACK
printer GET /file/f3mf/<id>/f3mffile       download the full 3MF
printer -> MQTT push_status                report the print lifecycle
```

## 1. Pull the task cards

While the Farm screen is open, the printer polls `POST /device/sync` over HTTPS. The observed
request was:

```http
POST /device/sync
User-Agent: ESP32 HTTP Client/1.0
Content-Type: application/json

{"cmd":"task_list","cur_status":"IDLE","cur_task_id":"","cur_subtask_id":""}
```

The request also carries the authenticated printer identity in transport metadata; its value is
omitted here.

The response is a plain JSON object, not a `code`/`result`/`data` envelope:

```json
{
  "status": 0,
  "is_suspend": false,
  "wait_reason": 0,
  "tasks": [
    {
      "task_id": "<task id>",
      "task_name": "Example",
      "plate": 1,
      "slice_info": "hpart://file?asset_path=3mf/f3mf_<asset id>/Metadata/slice_info.config",
      "thumbnail": "hpart://file?asset_path=3mf/f3mf_<asset id>/Metadata/plate_1.png"
    }
  ]
}
```

The response additionally echoes the authenticated caller in `dev_id`; the identity value is
omitted from the example.

Observed `status` mapping: `IDLE` -> `0`, `FAILED` -> `2`. Polls were approximately 45 seconds
apart while the view remained open; leaving and re-entering the Farm screen forced a fresh
request and replaced the cached cards.

`slice_info` and `thumbnail` are preview assets. They resolve through `GET /file?asset_path=...`
and are not the printable archive. The displayed `task_name` does not need to include the
physical `.gcode.3mf`/`.3mf` filename suffix.

## 2. Start one copy from the panel

Pressing **Start** on a card sends:

```http
POST /device/launchtask
Content-Type: application/json

{"cmd":"sub_start","task_id":"<task id>"}
```

The exact success response is:

```json
{"cmd":"sub_start","err_code":0}
```

The official server returned this ACK first. About 1.26 seconds later it published the managed
`print.project_file` described in [farm-mqtt.md](./farm-mqtt.md). The printer then downloaded
the complete archive from:

```http
GET /file/f3mf/<f3mf record id>/f3mffile
User-Agent: ESP32 HTTP Client/1.0
```

That response was `200 application/octet-stream`. The panel advanced through `Downloading`,
`Extracting 3MF`, and `Printing`.

## 3. Four independent identifiers

Four IDs participate in one launch and must not be collapsed:

| Identifier | Where it appears | Meaning |
|------------|------------------|---------|
| **task ID** | task card and `sub_start.task_id` | logical queued task |
| **subtask ID** | `project_file.subtask_id` | one physical copy/run of that task |
| **F3MF record ID** | `/file/f3mf/<id>/f3mffile` | server database record for the archive |
| **asset directory ID** | preview `asset_path` | unpacked thumbnail/slice-info directory |

The values may be numerically close but are not interchangeable. In particular, the ID in the
full-file URL is the F3MF record ID, not the task ID, and `subtask_id` identifies one reserved
copy rather than the whole task.

The observed firmware normalized all-digit task IDs by removing leading zeroes. Treating `0024`
and `24` as the same ID is therefore useful, but only for values that are entirely numeric;
opaque string IDs should retain exact comparison.

## 4. Per-copy state and timestamps

The official server stores one subtask record for each printable copy. `create_time` records the
real creation event. A newly queued copy uses Unix zero only as the sentinel for lifecycle
events that have not happened:

```json
{
  "state": "queue",
  "sch_state": "not_sched",
  "print_start_time": "1970-01-01T00:00:00Z",
  "print_end_time": "1970-01-01T00:00:00Z",
  "sch_time": "1970-01-01T00:00:00Z",
  "sch_leave_time": "1970-01-01T00:00:00Z"
}
```

On a panel launch the server assigns the printer, changes `sch_state` to `sched`, records
`sch_time`, and writes the original spelling:

```text
launch_from = "lanch_subtask_from_device"
```

An actual `RUNNING` report replaces `print_start_time`. A terminal finish, failure, or stop
replaces `print_end_time` and `sch_leave_time`. Observed timestamps use second-precision RFC3339.
The zero values are deliberate sentinels, not fabricated creation dates.

Implementation implication *(inferred from the per-copy records and the panel error)*: each
accepted panel start must reserve an unused subtask copy. Advertising a task with no copy left,
or losing it between the list and start requests, causes the printer to report that the task
count ran out or the task was removed.

## 5. Reimplementation result

A supervised clean-room implementation reproduced the complete exchange on a controlled
printer: the panel displayed the task, accepted Start, downloaded and extracted the full 3MF,
printed to 100%, and produced terminal per-copy accounting with real start/end/leave times.
The task ended with zero remaining copies and one finished copy.

No `bed_clean` preceded `project_file`. A separately configured AutoEject workflow sent
`bed_clean` only after `FINISH` as its Collected action and returned the printer to `IDLE`.
That automation is optional post-finish behavior, not part of the task-pull launch contract.

## 6. Reimplementation checklist

- Authenticate the printer identity from its TLS client certificate and/or device header.
- Return only tasks targeted at that bound printer and backed by an available 3MF.
- Serve preview assets separately from the full F3MF record.
- ACK `sub_start` first, then publish exactly one managed `project_file` after a short delay.
- Keep task, subtask, F3MF-record, and asset-directory IDs independent.
- Do not prepend `bed_clean` to the printer-screen launch path.
- Update per-copy times from printer reports; retain Unix zero until the event occurs.
- Sign `project_file` for secured printers as described in
  [farm-command-security.md](./farm-command-security.md).

Common failure signatures:

| Symptom | Likely contract violation |
|---------|---------------------------|
| Farm screen has no cards | wrong response envelope, identity/target mismatch, or missing preview-backed task |
| `Waiting to print` after a successful ACK | post-ACK MQTT `project_file` was not sent or was rejected |
| task count ran out / task removed | no unused per-copy subtask, exhausted count, or stale task ID |
| download starts but print does not | wrong F3MF ID/URL, TLS/port-8888 mismatch, hash/size mismatch, or invalid archive |
