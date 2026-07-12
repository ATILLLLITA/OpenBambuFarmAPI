# Farm TLS & trust chain

Every farm channel is TLS 1.2. The interesting part is the **trust anchor**: in `LAN_FARM` the
printer does *not* trust a public CA — it validates the server against a private **Bambu farm
chain** and a specific SNI, which is why a self-signed `server.bambulab.farm` cert is rejected
with `bad_certificate`.

## Versions & cipher suites (observed)

| Channel | TLS | Cipher suite | PRF / AEAD |
|---------|-----|--------------|------------|
| MQTT `1883` | 1.2 | `0xc02f` `ECDHE_RSA_WITH_AES_128_GCM_SHA256` | SHA-256 / AES-128-GCM |
| REST `8888` | 1.2 | `0xc02f` | SHA-256 / AES-128-GCM — **but also serves plain HTTP**, see note |
| Bind `3002` | 1.2 | `0xc030` `ECDHE_RSA_WITH_AES_256_GCM_SHA384` | SHA-384 / AES-256-GCM |

- Signature algorithm in the handshake: `rsa_pkcs1_sha256`.
- **SNI** the printer requires on the broker/REST: `farm_server.bambulab.com`.
- The MQTT `1883` ClientHello carries the **`extended_master_secret`** extension (RFC 7627).
- `1883` and `3002` are **mutual TLS** (the peer presents a client certificate).
- **`8888` is dual-protocol:** it answers both plain **HTTP** (client polling, liveview) and
  **HTTPS** (the printer's job download, SNI `farm_server.bambulab.com`) on the same port. See
  [farm-rest.md](./farm-rest.md) — a clone that serves HTTP-only there leaves the printer stuck
  at `IDLE` after the `project_file` ACK.

## The farm certificate chain

The server presents a 3-cert chain (recovered from the ServerHello `Certificate` message and
cross-checked against the server binary). Subject ← Issuer:

```
farm_server.bambulab.com   (leaf, SAN: farm_server.bambulab.com)
        ↑ issued by
farm_root.bambulab.com
        ↑ issued by
application_root.bambulab.com
        ↑ issued by
BBL CA   (O = "BBL Technologies Co., Ltd")
```

The printer's `CertificateRequest` (mutual TLS) advertises acceptable client CAs including
**`BBL CA`**, **`BBL CA2 RSA`**, and **`BBL CA2 ECC`**. The printer authenticates with its
**factory BBL device certificate** (`CN = <printer serial>`).

> A clone server that wants a real printer to trust it must present a leaf with SNI
> `farm_server.bambulab.com` chaining to `farm_root → application_root`, speak TLS 1.2 `0xc02f`,
> and use `rsa_pkcs1_sha256`. The **private key** for the farm leaf is Bambu's secret and is
> **not** included here (`«REDACTED»`); recovering or distributing it is out of scope for this
> documentation (see [DISCLAIMER.md](./DISCLAIMER.md)).

## MQTT broker authentication

Inside the `1883` TLS tunnel, the MQTT `CONNECT` carries:

| Field | Value |
|-------|-------|
| `clientid` | `<serial>_<N>` (e.g. `01P00C5725xxxxxx_4116`) |
| `username` | `BambuFarm` |
| `password` | a 49-char string prefixed `Bambu…` — a **shared farm secret** (`«REDACTED»`), held in the broker's `mochi-mqtt` auth ledger; **not** a per-device access code |

Because this printer has **no MQTT-Sec** layer (it reports `device insecure`), the MQTT
payloads inside TLS are **plaintext JSON** — once the TLS is decrypted there is no second
encryption layer (see [farm-mqtt.md](./farm-mqtt.md)). **Secured** printers (state
`sec: "secure"`) add a per-command signature layer on top of that plaintext — see
[farm-command-security.md](./farm-command-security.md).

## Why decryption needs the session secret (not just a key)

All suites are **ECDHE** → forward secrecy → the server's private key alone cannot decrypt a
recorded session. To analyse your **own** traffic you need that session's TLS **master secret**.
Conceptually (this is standard TLS research, described at a high level only):

1. Capture the full handshake **and** force a fresh one (the printer's MQTT link is long-lived;
   a re-bind produces a new `ClientHello`/`clientRandom`).
2. Obtain the master secret for that session from the endpoint you control while the session is
   **still open** (the secret is only resident in memory for the life of the connection).
3. Build a standard NSS key-log line `CLIENT_RANDOM <clientRandom> <masterSecret>` and decrypt
   with any TLS-aware tool.

The 48-byte master secret and the key-log are session-specific secrets and are **redacted**
here. Note also that some tools fail to decrypt these sessions despite a correct key-log
(observed: an `extended_master_secret` session where one tool reported `no decoder available`);
deriving keys directly from the master secret with the TLS 1.2 PRF works regardless.
