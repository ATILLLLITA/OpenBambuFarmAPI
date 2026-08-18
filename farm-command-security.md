# Farm command security (MQTT-Sec) â secured printers

[farm-tls.md](./farm-tls.md) notes that the reference printer (model `C12`) reports
`device insecure`, so its MQTT payloads are **plaintext JSON** inside the TLS tunnel â once the
TLS is decrypted there is no second layer. **Secured** printers (observed on an H-class machine
whose state carries `sec: "secure"`) add exactly that second layer: **every server â printer
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
| `sec: ""` (empty) | insecure â plaintext commands accepted (the `C12` reference case) |

*(inferred)* The requirement is also carried in a firmware capability/feature bit; the exact
bit is not reproduced here.

## What a secured printer rejects

Sending an **unsigned** command (a `bind.unbind`, a `print` command, â¦) to a secured printer on
`device/<serial>/request` is answered with a failure report instead of being executed:

| Error code | Meaning |
|------------|---------|
| `84033543` | MQTT command signature verification failed (bad or absent signature) |
| `84033545` | signature / task-id verification failed |

(These MQTT-Sec codes, and the full REST error table, are collected in
[farm-errors.md](./farm-errors.md).)

So a farm reimplementation that must drive a *secured* printer has to attach the signature layer
below; an insecure printer needs none.

## Signed-command shape

A signed command adds a `header` sibling next to the command namespace(s):

```json
{
  "print": { "command": "â¦", "sequence_id": "â¦", "â¦": "â¦" },
  "header": {
    "sign_ver":    "v1.0",
    "sign_alg":    "RSA_SHA256",
    "sign_string": "Â«REDACTED â base64 signatureÂ»",
    "cert_id":     "Â«issuer + ':' + serial (lower-case hex) of the signing certÂ»",
    "payload_len": 123
  }
}
```

| `header` field | Meaning |
|----------------|---------|
| `sign_ver` | signature-scheme version (`v1.0` observed) |
| `sign_alg` | `RSA_SHA256` â RSA PKCS#1 v1.5 signature over a SHA-256 digest |
| `sign_string` | base64 of the signature (`Â«REDACTEDÂ»`) |
| `cert_id` | identifies the signing certificate: issuer name + `:` + serial (lower-case hex) |
| `payload_len` | UTF-8 byte length of the signed content (see *Signing scope*) |

### Signing scope

The signature covers the **entire command object with the `header` field removed** â i.e. the
JSON of every top-level namespace/sibling *except* `header`, **including** any `user_id` or other
top-level fields, not merely the inner `{ "print": â¦ }` object. `payload_len` is the UTF-8 byte
length of exactly that content, and the digest is taken over it.

> A common reimplementation bug is to sign only the inner `{ "print": â¦ }` object and drop
> sibling fields such as `user_id`; a secured printer then answers `84033543`. Any namespace
> that carries a `command` is signable this way â `print`, `system`, `bind`/`unbind`, etc.

Field-level encryption is a *separate*, narrower mechanism: only a `project_file` `url`
(â `url_enc`) and a `gcode_line` `param` (â `param_enc`) are encrypted, using the **printer's**
device certificate. When present it is applied to those fields *before* the outer signature.

## The signing certificate chain (public) â and the key that is **not** here

Signing uses Bambu's **application** certificate: an **install-shared** (not per-device)
leaf â intermediate â root chain rooted at `application_root.bambulab.com` â the same
`application_root` that anchors the farm TLS chain in [farm-tls.md](./farm-tls.md). The **public**
certificates are presentable material; the corresponding **application private key** is Bambu's
secret.

- The private key is **not** included here (`Â«REDACTEDÂ»`) and is **not** derivable from this
  document. Independent research reports it is not extractable from the shipping software
  (statically-linked, obfuscated OpenSSL, anti-debug), and the only vendor renewal path is gated
  by a per-install bootstrap secret. Reproducing or distributing that secret or key is **out of
  scope** for this documentation (see [DISCLAIMER.md](./DISCLAIMER.md)) â exactly as the farm-leaf
  TLS private key is out of scope in [farm-tls.md](./farm-tls.md).
- Practical consequence for self-hosting: an **insecure** printer is fully drivable by a clone
  farm ([farm-mqtt.md](./farm-mqtt.md)) with no credentials at all. A **secured** printer is
  **also fully controllable — the same way the official software controls it — once you supply
  the application signing credentials yourself.** With a valid signing key in place the farm signs
  each server → printer command and the full sensitive set works: bind/unbind, `print` control and
  firmware push. This was observed end-to-end — a signed `upgrade.offline_upgrade` was **accepted
  and acted on by a secured P2S** ([farm-firmware.md](./farm-firmware.md)). Without a signature the
  same command is rejected with `84033543`, so a secured printer still completes its farm session
  (bind, subscribe, `push_status` telemetry) but ignores sensitive commands until they are signed.
- This documentation does **not** ship Bambu's application private key, will **not** give it out,
  and gives **no** instructions for obtaining it (`«REDACTED»`) — the same posture other
  interoperability projects take for these credentials (e.g.
  [ClusterM/open-bamboo-networking](https://github.com/ClusterM/open-bamboo-networking), which
  signs `print` commands only when the user supplies their own Bambu slicer key). If you supply
  your own credentials it is on you to ensure your use complies with the applicable terms and law.
  The signature layer is a lock, not a wall: the only barrier for an *open* reimplementation is
  possession of that key — nothing else in the protocol.


## Interaction with unbind

Because `bind.unbind` is a command like any other, on a secured printer it must be signed as
well; an unsigned `bind.unbind` is rejected with `84033543` and the printer stays bound to its
current `server_id` (see the asymmetry table and *"server id not same, need unbind"* refusal in
[farm-bind.md](./farm-bind.md)). *(inferred)* The single-printer LAN `8883` path, where a
secured printer exposes one, authenticates with the per-device access code rather than this
command signature; that path is outside farm scope.
