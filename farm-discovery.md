# Farm discovery (SSDP)

Both sides announce themselves on the LAN with SSDP-style multicast so the Farm Manager
client/server and the printers can find each other without configuration.

## Server / farm service

- **URN:** `urn:bambulab-com:farm:server:1`
- **Multicast group:** `239.255.255.250:2021`
- **Custom location scheme:** `bambu-farm-client://f` *(used in the discovery location header)*
- **Farm server hostname:** `farm_server.bambulab.com` — note this is the **TLS SNI / cert
  name** the printer validates (see [farm-tls.md](./farm-tls.md)); in `LAN_FARM` it resolves to
  the local server's IP, not a cloud host.

## Printer announcements

The printer periodically emits `NOTIFY * HTTP/1.1` datagrams to:

- `239.255.255.250:1990`
- `255.255.255.255:2021`

with:

- **NT:** `urn:bambulab-com:3dprinter:1`

and a set of vendor headers carrying inventory/state. The header **names** are themselves of
the form `Dev*.bambu.com` (this is a naming convention for the headers — **not** cloud
domains the printer contacts):

| Header (name) | Meaning | Example value |
|---------------|---------|---------------|
| `DevName.bambu.com` | friendly name | `test` |
| `DevModel.bambu.com` | model code | `C12` |
| `DevSignal.bambu.com` | Wi-Fi RSSI | `-50dBm` |
| `DevConnect.bambu.com` | connection kind | — |
| `DevBind.bambu.com` | bind state | `free` / bound |
| `DevVersion.bambu.com` | firmware version | `01.06.30.01` |
| `DevCap.bambu.com` | capabilities bitmask | — |

> **Correction to a common misreading:** `Dev*.bambu.com` strings found in the binary are
> these SSDP **header names**, not outbound cloud endpoints. In `LAN_FARM` the printer makes no
> cloud connections.

`DevBind = free` is the signal the Farm Manager uses to offer an unbound printer for binding
(see [farm-bind.md](./farm-bind.md)).
