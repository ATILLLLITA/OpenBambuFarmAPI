# OpenBambuFarmAPI

Independent, interoperability-focused documentation of the **LAN "farm" protocol** used
between *Bambu Farm Manager* (a self-hosted Go server by Bambu Lab for managing many
printers) and Bambu Lab 3D printers.

This is a companion to [Doridian/OpenBambuAPI](https://github.com/Doridian/OpenBambuAPI),
which documents the cloud/LAN *single-printer* MQTT/HTTP/FTP protocol. That project does
**not** cover farm mode. This one documents the farm-specific layer: the TCP-3002 bind
handshake, the farm TLS trust chain, the farm MQTT broker, the REST control API, the
command-security signature layer that secured printers require, and the end-of-print behaviour — all recovered by observing a real server talking to a real printer.

> **Disclaimer.** Independent interoperability research. Not affiliated with, authorized by,
> or endorsed by Bambu Lab. Contains **no** Bambu source code, binaries, decompilation, or
> secret cryptographic material — only observed protocol facts and original prose. See
> [DISCLAIMER.md](./DISCLAIMER.md).

## Scope & method

All findings come from passively capturing and decrypting traffic between **one** server and
**one** printer (model C12, firmware `01.06.30.01`) on a private LAN, then cross-checking the
decrypted JSON against the independently-written OpenBambuAPI and against the server's own
decompiled symbols. Where a fact is inferred rather than directly observed it is marked
*(inferred)*.

Secret values (private keys, the shared broker password, TLS session secrets, device certs)
are **redacted** throughout and shown only as `«REDACTED»` placeholders.

## Architecture

```
                 SSDP (discovery)              TCP 3002 / TLS  (bind: server dials printer)
  ┌────────────┐  239.255.255.250:2021   ┌───────────────────┐  login/detect → login_report
  │ Farm       │ ◀──────────────────────▶│ Printer           │
  │ Manager    │                          │ 01P00C5725xxxxxx  │  MQTT 1883 / TLS  (printer dials server)
  │ Server     │ ◀──────────────────────▶│ (MQTT *client*)   │  device/<sn>/{request,report}
  │ (Go, :8888 │   MQTT 1883 / TLS        └───────────────────┘
  │  REST +    │   REST 8888 / TLS         liveview 6000/TLS, FTPS 990 (printer-hosted)
  │  :1883     │ ◀──────────────────────▶
  │  broker)   │
  └────────────┘
   Farm Manager Client (Electron) ──REST :8888/TLS──▶ Server
```

Key inversion vs. cloud mode: in **farm mode the printer is the MQTT _client_** and connects
*out* to the server's broker on `1883`; the printer's own `8883` broker is closed. Binding is
the one exception — the **server** dials the **printer** on `3002` to hand it the farm
endpoints.

## Ports

| Port | Host | Transport | Purpose |
|------|------|-----------|---------|
| **3002** | printer | TLS 1.2 (`0xc030`) | Bind handshake: server → printer `login`/`detect` |
| **1883** | server | TLS 1.2 (`0xc02f`), mutual | Farm MQTT broker (`mochi-mqtt`); printer connects out |
| **8888** | server | **HTTP + TLS 1.2** (`0xc02f`) | Farm REST API (control, files, liveview, login). Dual-protocol: plain HTTP for the client/liveview, HTTPS for the printer's job download — see [farm-rest.md](./farm-rest.md) |
| **6000** | printer | TLS | Liveview stream (printer-hosted) |
| **990**  | printer | FTPS | File access (printer-hosted) |
| **2021** | multicast | UDP | SSDP farm discovery |
| 8883 | printer | — | **Closed** in farm mode (printer is a client, not a broker) |

## Connection modes

`lan` · `cloud` · `farm`. This document covers **`farm`** (a.k.a. `LAN_FARM`).

## Documents

| File | Topic |
|------|-------|
| [farm-discovery.md](./farm-discovery.md) | SSDP discovery, NOTIFY headers, URNs |
| [farm-bind.md](./farm-bind.md) | TCP-3002 `detect`/`login` handshake, bind & unbind |
| [farm-tls.md](./farm-tls.md) | TLS versions, ciphers, the farm certificate chain, why decryption needs the session secret |
| [farm-mqtt.md](./farm-mqtt.md) | Broker auth, `device/<sn>/{request,report}`, `project_file`, `push_status`, print lifecycle, the Collected/Reprint/Cancel trigger |
| [farm-rest.md](./farm-rest.md) | The `:8888` REST surface used by the client and printer |
| [farm-command-security.md](./farm-command-security.md) | MQTT-Sec: the per-command signature layer that **secured** printers require |

## License

Documentation © 2026 the OpenBambuFarmAPI contributors, licensed under the
**GNU Free Documentation License, Version 1.3** ([LICENSE](./LICENSE)), with **no Invariant
Sections, no Front-Cover Texts, and no Back-Cover Texts** — matching OpenBambuAPI so material
may flow freely between the two projects.
