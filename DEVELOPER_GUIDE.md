# QRKeyboard API reference

Use this reference after the [copy-and-run quick start](QUICKSTART.md).
To publish your receiving page, use [Deploy your page](DEPLOY.md). There is no
QRKeyboard backend to deploy for an ordinary browser integration.

This reference describes the updated checkout. Message mode, named actions,
delivery acknowledgements, and Finish are [pending service deployment](RELEASE_STATUS.md).
The starter explicitly uses the existing live-key API.

Your application displays the QR and handles the input. The phone opens our
hosted page; both endpoints use the internet relay at `https://qrkeyboard.com`.
The phone does not need to reach your development server.

## Browser setup

Load `https://qrkeyboard.com/v1.js`, then call `QRKeyboard.create(options)`.
Create the keyboard after the QR/output elements exist, and register event
handlers immediately. The returned object is available before network setup
finishes. In the updated SDK, `keyboard.ready` resolves when the instance/QR has
been prepared and rejects on setup failure; listen for `error` to show a useful
message. Creating a keyboard does not wait for a phone to scan it.

```js
const keyboard = QRKeyboard.create({ el: '#qr', name: 'My screen', caps: ['keyboard'] });
keyboard.on('key', handleInput);
keyboard.on('error', showConnectionError);
```

`handleInput` and `showConnectionError` are your application callbacks; the
[starter](examples/index.html) supplies complete implementations. In a framework,
create once on mount, then call `close()` on unmount. The classic script exposes
`window.QRKeyboard`; no package import is required.

| Option | Default | Meaning |
|---|---|---|
| `el` | none | CSS selector or element for the QR, instructions, status, and PIN |
| `name` | `This screen` | Destination shown on the phone |
| `mode` | `live` | `live` streams edits; `text` sends a complete message |
| `submitLabel` | `Send` | Text on the submit button; also describes Enter's action |
| `caps` | `['keyboard']` | Supported hosted inputs: `keyboard`, `buttons` |
| `actions` | none | Up to 16 `{ id, label }` named actions; adds buttons if caps are omitted |
| `pin` | none | `true` generates/displays a PIN; a string sets one |
| `region` | automatic | Optional relay placement hint: `wnam`, `enam`, `sam`, `weur`, `eeur`, `apac`, `oc`, `afr`, `me` |
| `cellSize`, `margin` | `4`, `4` | QR rendering options |
| `target` | none | Advanced: dispatch synthetic keydown events to this element; does not insert text automatically |

Buttons-only setup uses `caps: ['buttons']`. Without explicit action labels,
buttons are named **Previous** and **Next**. Prefer labels specific to your app.

The QR panel emits its own accessible status and instructions. Style
`.qrkeyboard-pairing` and `.qrkeyboard-code` to fit your screen. Keep the QR dark
on white, leave its quiet zone intact, and keep it large enough to scan.

## Input events and delivery confirmation

```js
keyboard.on('text', async ({ text }) => {
  await saveMessage(text);
});
```

The phone reports **Sent** after every handler for that event completes. Returning
a Promise delays acknowledgement until the operation finishes. Throwing or
rejecting reports failure to the phone. With no input handler, the SDK rejects
the input rather than claiming delivery. This confirms handler completion, not
any downstream effect the handler neglected to await.

| Event | Payload | When |
|---|---|---|
| `key` | `{ text, id }` or `{ code, id }` | Live text edits or special keys |
| `text` | `{ text, action, id }` | Complete-message submission; `action` is the submit label |
| `action` | `{ action, label, id }` | Named button activated; `action` is its configured action ID |
| `state` | `{ state, message }` | `preparing`, `scan`, `connected`, `lost`, `finished`, `error` |
| `ready` | `{ code }` | A fresh instance and QR have been prepared |
| `connect`, `disconnect` | `{}` | Phone pairing established or lost |
| `finish` | `{ code }` | Old session ended; clear application state before a fresh QR is created |
| `reset` | `{}` | Finish handlers completed |
| `pinfail` | `{ attemptsLeft }` | PIN confirmation failed |
| `locked` | `{}` | Five failed PIN confirmations; create a fresh session |
| `error` | Error | Network, input-handler, pairing, or reset failure |

Special key codes: `8` Backspace, `9` Tab, `10` Enter, `27` Escape, `37–40` arrows.
In live mode the phone diffs committed text and emits suffix backspaces/text.
The receiving application should remove one Unicode code point per Backspace.
Bulk deletions travel as one counted key message; the SDK expands them into
individual Backspace events without flooding the relay.

Messages must fit the relay's 1024-byte encrypted-frame limit. New JSON envelopes
are limited to 950 plaintext bytes. Oversized complete messages are rejected on
the phone with a request to shorten them; live text is split into small chunks.
Duplicate message IDs are acknowledged without repeating the handler while they
remain in the current SDK session's bounded cache (256 messages).

Register the handler matching the mode: `key` for live typing, `text` for complete
messages, `action` for named buttons. Setting `mode: 'text'` alone is insufficient
if your app only listens for `key`. The [examples](examples/README.md) show all
three working setups.

## Session lifecycle

`await keyboard.finish()` (also `restart()`) ends the current instance through
the authenticated API, closes connections, awaits `finish` handlers, and prepares
a fresh QR/key/PIN as applicable. Old links and credentials cease to work.
If your reset handler fails, the SDK does not admit the next visitor; fix or retry
the reset. Always clear application-owned state in `finish` for shared screens.

The phone's **Finish** requests this same operation over the encrypted connection.
Its **Disconnect** only stops that phone connection and clears its local
credential. A new visitor can join after it closes. Only one active phone is
accepted per session; a credential reconnect replaces its previous connection.

`close()` stops SDK timers and sockets. It does not invalidate the server session.
The relay expires unused sessions after 30 days. Reconnection uses a short
exponential backoff. Pending unconfirmed inputs are not automatically replayed;
users are asked to check the receiving screen before sending again.

## Pairing and privacy

No-PIN QRs contain a random 32-byte key in their URL fragment. The phone removes
the fragment from the address bar after reading it. Anyone who captures that QR
can possess its key; keep private-input QRs out of broadcasts and finish sessions
between visitors. PIN mode omits the key and uses SPAKE2 with explicit confirmation.
The SDK's five-attempt PIN limit applies to the current running screen session.

Both modes derive directional AES-256-GCM keys using HKDF. Input, acknowledgements,
and finish requests are encrypted at the endpoints. The relay forwards opaque
binary frames and holds only hashed owner/phone credentials, public configuration,
and session metadata. Displaying a PIN beside a public QR aids pairing, but does
not restrict access against people who can read both.

## Advanced: raw HTTP and WebSocket API

### Create

`POST /i` with JSON configuration (`caps`, `name`, `mode`, `submitLabel`, `actions`,
`pinRequired`, optional `region`) returns `code`, `joinUrl`, `ownerToken`, `qrUrl`,
and the device-role `relay` URL. Creation is limited to 20 requests per IP/minute.
Store the owner token privately. The raw API does not generate an endpoint key.
To use the hosted phone page, implement key generation/QR assembly or PIN pairing.

### Resolve

`GET /j/<code>` returns public `config` and the phone-role relay URL. Unknown or
finished instances return 404. Possession of a code permits requesting a phone
join, but encrypted input requires the endpoint pairing secret.

### QR formats

`GET /i/<code>/qr` requires `Authorization: Bearer <ownerToken>`. Formats are
`svg` (default), `png`, and `bits` (a newline-separated grid of `0`/`1`). Render
options include `ecc=L|M|Q|H`, `margin`, `scale`, `size`, `dark`, `light`, and
`shape=square|rounded|dots`. Raw server QRs contain only the join code, not a
client-generated encryption key. The browser SDK generates the usable keyed QR
locally. Every custom QR display should say **Scan with your phone camera to type
here** and show a destination name.

### Connect

`GET /s/<code>?role=device|phone` upgrades to a WebSocket. Within 10 seconds:

- Device sends `{ "t": "auth", "owner": "<ownerToken>" }`.
- First phone sends `{ "t": "join" }`, receives `{ "type": "credential", "value": "..." }`.
- Reconnecting phone sends `{ "t": "auth", "cred": "..." }`.

The relay sends `hello`, `ready`, and `peer` presence messages as JSON strings.
Authenticated input is binary. Limits are 120 frames/sec and 32 KiB/sec per
socket, 1024 bytes/frame, and 60 edge connections per IP/minute.

### End

`POST /i/<code>/end`, with `Authorization: Bearer <ownerToken>`, invalidates the
session and closes every socket with code `4010`. Missing/incorrect owner tokens
return 401/403; unknown or ended instances return 404. Session reset allocates a
new instance rather than reusing an ended code.

### Provision a device over one socket

`/provision` allocates an instance and pushes a JSON `provision` message containing
`code`, `joinUrl`, `ownerToken`, and a QR bitmap, followed by `ready`. Persist the
owner token to reconnect as the device. Firmware still needs endpoint encryption
and pairing; this is an advanced transport API, not a ready-made phone keyboard
for an arbitrary MCU. See [embedded guidance](LOW_END_MCU.md).

## Advanced: application messages and compatibility

Encrypted frame: `e1 | 12-byte random nonce | AES-GCM ciphertext + tag`.
New plaintext: `10 | UTF-8 JSON`. Message types:

- `key`: `{ type, id, text }` or `{ type, id, code }`; Backspace may include a
  `count` (expanded into individual events, capped at 10000).
- `text`: `{ type, id, text }`.
- `action`: `{ type, id, action }` (configured action ID).
- `finish`: `{ type, id }`.
- `ack`: `{ type, id, ok }` (screen to phone).

The SDK continues to decode older binary `KEY` (`04`), `BUTTON` (`02`), `MOVE`
(`01`), `SCROLL` (`03`), and `SETTLE` (`05`) frames for custom clients. Those legacy
frames do not carry acknowledgement IDs. Pointer/scroll interfaces are not
features of the current hosted phone keyboard. Local design experiments and
historical plans are kept separately from this API.
