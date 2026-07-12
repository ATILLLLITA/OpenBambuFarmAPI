# Farm command security (MQTT-Sec) — secured printers

[farm-tls.md](./farm-tls.md) notes that the reference printer (model `C12`) reports
`device insecure`, so its MQTT payloads are **plaintext JSON** inside the TLS tunnel — once the
TLS is decrypted there is no second layer. **Secured** printers (observed on an H-class machine
whose state carries `sec: "secure"`) add exactly that second layer: **every server → printer
command must carry a cryptographic signature**, or the printer refuses to execute it. This is
Bambu's "MQTT-Sec" / command-security scheme.

This document records the *shape* of that layer for interoperability. It contains **no** signing
key, no bootstrap secret, and no procedure for obtaining them; it does **not** let you sign
commands for a printer you do not control.

## When it applies

A printer advertises the requirement in its state / bind record:

| Signal | Meaning |
|--------|---------|
| `sec: "secure"` (in the bind record / `push_status`) | printer enforces signed commands |
| `sec: ""` (empty) | insecure — plaintext commands accepted (the `C12` reference case) |

*(inferred)* The requirement is also carried in a firmware capability/feature bit; the exact
bit is not reproduced here.

## What a secured printer rejects

Sending an **unsigned** command (a `bind.unbind`, a `print` command, …) to a secured printer on
`device/<serial>/request` is answered with a failure report instead of being executed:

| Error code | Meaning |
|------------|---------|
| `84033543` | MQTT command signature verification failed (bad or absent signature) |
| `84033545` | signature / task-id verification failed |

So a farm reimplementation that must drive a *secured* printer has to attach the signature layer
below; an insecure printer needs none.

## Signed-command shape

A signed command adds a `header` sibling next to the command namespace(s):

```json
{
  "print": { "command": "…", "sequence_id": "…", "…": "…" },
  "header": {
    "sign_ver":    "v1.0",
    "sign_alg":    "RSA_SHA256",
    "sign_string": "«REDACTED — base64 signature»",
    "cert_id":     "«issuer + ':' + serial (lower-case hex) of the signing cert»",
    "payload_len": 123
  }
}
```

| `header` field | Meaning |
|----------------|---------|
| `sign_ver` | signature-scheme version (`v1.0` observed) |
| `sign_alg` | `RSA_SHA256` — RSA PKCS#1 v1.5 signature over a SHA-256 digest |
| `sign_string` | base64 of the signature (`«REDACTED»`) |
| `cert_id` | identifies the signing certificate: issuer name + `:` + serial (lower-case hex) |
| `payload_len` | UTF-8 byte length of the signed content (see *Signing scope*) |

### Signing scope

The signature covers the **entire command object with the `header` field removed** — i.e. the
JSON of every top-level namespace/sibling *except* `header`, **including** any `user_id` or other
top-level fields, not merely the inner `{ "print": … }` object. `payload_len` is the UTF-8 byte
length of exactly that content, and the digest is taken over it.

> A common reimplementation bug is to sign only the inner `{ "print": … }` object and drop
> sibling fields such as `user_id`; a secured printer then answers `84033543`. Any namespace
> that carries a `command` is signable this way — `print`, `system`, `bind`/`unbind`, etc.

Field-level encryption is a *separate*, narrower mechanism: only a `project_file` `url`
(→ `url_enc`) and a `gcode_line` `param` (→ `param_enc`) are encrypted, using the **printer's**
device certificate. When present it is applied to those fields *before* the outer signature.

## The signing certificate chain (public) — and the key that is **not** here

Signing uses Bambu's **application** certificate: an **install-shared** (not per-device)
leaf → intermediate → root chain rooted at `application_root.bambulab.com` — the same
`application_root` that anchors the farm TLS chain in [farm-tls.md](./farm-tls.md). The **public**
certificates are presentable material; the corresponding **application private key** is Bambu's
secret.

- The private key is **not** included here (`«REDACTED»`) and is **not** derivable from this
  document. Independent research reports it is not extractable from the shipping software
  (statically-linked, obfuscated OpenSSL, anti-debug), and the only vendor renewal path is gated
  by a per-install bootstrap secret. Reproducing or distributing that secret or key is **out of
  scope** for this documentation (see [DISCLAIMER.md](./DISCLAIMER.md)) — exactly as the farm-leaf
  TLS private key is out of scope in [farm-tls.md](./farm-tls.md).
- Practical consequence for self-hosting: an **insecure** printer is fully drivable by a clone
  farm ([farm-mqtt.md](./farm-mqtt.md)); a **secured** printer will still complete the farm
  session — bind, subscribe, stream `push_status` telemetry — but will **ignore sensitive
  commands** unless they are signed by the vendor key. That is a deliberate limit of open
  interoperability today, not a defect of the clone.

## Interaction with unbind

Because `bind.unbind` is a command like any other, on a secured printer it must be signed as
well; an unsigned `bind.unbind` is rejected with `84033543` and the printer stays bound to its
current `server_id` (see the asymmetry table and *"server id not same, need unbind"* refusal in
[farm-bind.md](./farm-bind.md)). *(inferred)* The single-printer LAN `8883` path, where a
secured printer exposes one, authenticates with the per-device access code rather than this
command signature; that path is outside farm scope.
