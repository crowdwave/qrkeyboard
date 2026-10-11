# Embedded devices: advanced guidance

For a first run, use the [quick start](QUICKSTART.md). This guide describes the
updated checkout; check [release status](RELEASE_STATUS.md) before using features
that require the new hosted service.

The simplest integration is a web application using the browser SDK. Embedded
integration requires device-side networking, endpoint encryption, and input
handling. QRKeyboard does not ship firmware, bridges, or a USB-HID injector.

## What exists

- A Cloudflare HTTPS/WebSocket API for creating and connecting sessions.
- Owner-token authentication and hashed phone reconnect credentials.
- QR rendering as SVG, PNG, or a plain `0`/`1` bitmap grid.
- Single-socket provisioning that returns a new code and owner token, plus a QR
  grid for PIN instances.
- A hosted phone keyboard using live text, message entry, or named actions.
- Endpoint AES-256-GCM encryption, directional HKDF keys, and optional SPAKE2 PIN
  pairing in the browser implementation.

Provisioning and raw server QRs contain the join code alone, so the relay issues
them only for PIN instances (`/provision?pin=1&protocol=3`, or `pinRequired: true`
with `POST /i`). For no-PIN pairing the firmware generates the 32-byte key and
draws its own QR of `joinUrl` + `&k=<base64url key>`; the relay never sees that
key. Either way the firmware implements the pairing and framing (unchanged in protocol 3) as
described in the [API reference](DEVELOPER_GUIDE.md#advanced-application-messages-and-compatibility):
per-connection salts, SPAKE2 for PIN mode, counter nonces. The browser SDK
already does this for web screens.

## Hardware choices

| Hardware | Integration approach |
|---|---|
| ESP32-class with suitable crypto/TLS support | Implement secure WebSockets, pairing, and encrypted application frames |
| Small MCU with a capable companion | Companion handles TLS, pairing, and encryption; main MCU receives local input |
| Device without TLS or a companion | A gateway you build is required to reach the hosted relay |

Whether the bridge or the main MCU holds the endpoint keys determines where
plaintext exists. Choose and document that boundary. A TLS bridge alone does not
implement endpoint pairing or end-to-end encryption.

## Minimal device lifecycle

1. Create an instance through `POST /i` or `/provision`, declaring `protocol` 3.
2. Establish endpoint encryption and display the resulting usable QR with
   **Scan with your phone camera to type here**.
3. Connect as device using the owner token and participate in pairing.
4. Decrypt and handle `key`, `text`, `action`, or (with a `layout`) `down`/`up`
   messages. A bridge that drives a real TV can map remote-layout keys such as
   `ChannelUp` or `ColorF0Red` to IR codes or HDMI-CEC commands.
5. Send an encrypted `ack` only after handling the input.
6. On Finish, clear visitor state, call the authenticated end endpoint, and create
   a new instance/key before showing the next QR.

Send a `k` text message every 25 seconds to keep the connection warm; the relay
answers it without waking the session. See the [API reference](DEVELOPER_GUIDE.md)
for wire formats and frame limits.

## Separate approaches you would have to build

A local HTTP/WebSocket page served by your own MCU can implement a basic phone
keyboard without this service. A plaintext gateway can also forward traffic to a
TLS upstream. Neither a local-only phone page nor an insecure transport gateway
is a supported hosted-product feature. Keep those designs separate from the
ready-to-use browser integration.
