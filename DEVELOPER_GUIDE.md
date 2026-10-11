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

### Pin a version for production

`/v1.js` always serves the latest SDK. To pin an exact, immutable build, load
a versioned file with Subresource Integrity. Current paths and hashes are listed
in [`https://qrkeyboard.com/sdk/manifest.json`](https://qrkeyboard.com/sdk/manifest.json)
(`latest`, plus every published version):

```html
<script src="https://qrkeyboard.com/sdk/v1-<hash>.js"
        integrity="sha384-<digest>" crossorigin="anonymous"></script>
```

Versioned files never change and stay online. Each records the protocol version
it speaks; if the hosted phone page stops supporting that version, phones that
scan your QR are told the screen needs updating. Move to a newer pinned file
before then.

| Option | Default | Meaning |
|---|---|---|
| `el` | none | CSS selector or element for the QR, instructions, status, and PIN |
| `name` | `This screen` | Destination shown on the phone |
| `mode` | `live` | `live` streams edits; `text` sends a complete message |
| `submitLabel` | `Send` | Text on the submit button; also describes Enter's action |
| `caps` | `['keyboard']` | Supported hosted inputs: `keyboard`, `buttons` |
| `actions` | none | Up to 16 `{ id, label }` named actions; adds buttons if caps are omitted |
| `pin` | none | `true` generates/displays a PIN; a string sets one |
| `region` | automatic | Optional relay placement hint: `wnam`, `enam`, `sam`, `weur`, `eeur`, `apac`, `oc`, `afr`, `me`; any other value is rejected |
| `cellSize`, `margin` | `4`, `4` | QR rendering options |
| `layout` | none | Phone controls: `'keyboard'`, `'buttons'`, `'remote'`, `'teletext'`, `'gamepad'`, or a list of blocks; see [Controls](#controls-remotes-keypads-and-game-buttons) |
| `target` | none | Advanced: dispatch synthetic `keydown` (and, for layout keys, `keyup`) events to this element; does not insert text automatically |

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
| `state` | `{ state, message }` | `preparing`, `scan`, `connected`, `waiting` (a dropped phone may return), `lost`, `finished`, `error` |
| `ready` | `{ code }` | A fresh instance and QR have been prepared |
| `connect`, `disconnect` | `{}` | Phone pairing established or lost |
| `finish` | `{ code }` | Old session ended; clear application state before a fresh QR is created |
| `reset` | `{}` | Finish handlers completed |
| `pinfail` | `{ attemptsLeft }` | A PIN attempt ended without pairing (wrong PIN, or the phone left mid-check) |
| `locked` | `{}` | Five PIN attempts without pairing; create a fresh session |
| `error` | Error | Network, input-handler, pairing, or reset failure |
| `press` | `{ key, label, id }` | A layout key went down (once per press) |
| `keydown` | `{ key, repeat, id }` | A layout key went down; repeats with `repeat: true` while a `repeat` key is held |
| `keyup` | `{ key, id }` | A layout key was released |

Special key codes: `8` Backspace, `9` Tab, `10` Enter, `27` Escape, `37–40` arrows.
In live mode the phone diffs committed text and emits suffix backspaces/text.
The receiving application should remove one Unicode code point per Backspace.
Bulk deletions travel as one counted key message; the SDK expands them into
individual Backspace events without flooding the relay.

Messages must fit the relay's 1024-byte frame limit. After the 29 bytes of
framing and authentication tag, a JSON envelope may use 995 plaintext bytes,
roughly 900 ASCII characters or 300 CJK characters of message text. In message
mode the phone shows how much to shorten while the user types; live text is
split into small chunks. A repeated message ID is acknowledged again without
repeating the handler while it remains in the SDK's bounded cache (256 messages).

Register the handler matching the mode: `key` for live typing, `text` for complete
messages, `action` for named buttons. Setting `mode: 'text'` alone is insufficient
if your app only listens for `key`. The [examples](examples/README.md) show all
three working setups.

## Controls: remotes, keypads and game buttons

Set `layout` to give the phone more than a keyboard. Use a preset, or compose
your own from three primitives. Input only travels from the phone to the screen.

```js
const keyboard = QRKeyboard.create({ el: '#qr', name: 'Living room TV', layout: 'remote' });
keyboard.on('press', ({ key }) => { if (key === 'Enter') openSelected(); });
keyboard.on('keydown', ({ key }) => { if (key === 'ArrowDown') moveFocus(+1); }); // repeats while held
```

| Preset | Phone shows |
|---|---|
| `keyboard` | The text entry (the default) |
| `buttons` | Your named `actions`, one per row |
| `remote` | D-pad with OK; Back, Home, Menu; Vol −, Mute, Vol +; Ch −, Play/Pause, Ch +; four colour keys |
| `teletext` | Four colour keys; digits 0–9 with Text and TV; Sub −, Page −, Page +, Sub + |
| `gamepad` | D-pad with OK; A and B |

Your own layout is a list of up to 6 blocks:

```js
layout: [
  { type: 'dpad', repeat: true },
  { type: 'keys', columns: 3, keys: [
    { key: 'MediaRewind' }, { key: 'MediaPlayPause' }, { key: 'MediaFastForward' },
    { key: 'action:like', label: 'Like', tone: 'accent' },
  ] },
  { type: 'text' },
]
```

- `{ type: 'text' }`: the text entry, using `mode` and `submitLabel`. At most one.
- `{ type: 'dpad', repeat }`: arrows and OK, sent as `ArrowUp`, `ArrowDown`,
  `ArrowLeft`, `ArrowRight` and `Enter`. `repeat: true` repeats the arrows while held.
- `{ type: 'keys', columns, keys }`: a grid of 1 to 20 keys, `columns` 1–5. Each key
  has a `key`, an optional `label` (up to 20 characters), an optional `tone`
  (`plain`, `accent`, `red`, `green`, `yellow`, `blue`) and an optional `repeat`.

A key is one of:

- A standard key value from the [W3C UI Events key list](https://www.w3.org/TR/uievents-key/):
  `ArrowUp` `ArrowDown` `ArrowLeft` `ArrowRight` `Enter` `Backspace` `Escape`
  `ContextMenu` `Power` `ChannelUp` `ChannelDown` `MediaPlayPause` `MediaStop`
  `MediaRewind` `MediaFastForward` `MediaTrackPrevious` `MediaTrackNext`
  `AudioVolumeUp` `AudioVolumeDown` `AudioVolumeMute` `BrowserBack` `BrowserHome`
  `ColorF0Red` `ColorF1Green` `ColorF2Yellow` `ColorF3Blue` `Teletext` `Info`
  `Guide` `Exit` `Settings` `Subtitle`. Colour keys are coloured automatically.
- One printable character, such as `'a'` or `'7'`.
- `'action:<id>'` for one of your `actions`; it raises the `action` event as before.

Each key appears once per layout, and a layout holds at most 40 keys. The relay
rejects any layout outside these rules with a 400 that names the problem.

Every press sends the key down and, on release, up. The SDK raises `press` and
`keydown`, repeats `keydown` (after 400 ms, every 80 ms) for keys marked `repeat`,
and raises `keyup` on release. A held key is released if the phone disconnects
or after 15 seconds without a release. With `target`, the same keys arrive as
ordinary `keydown`/`keyup` events, so an existing TV web app that already handles
remote keys needs no QRKeyboard-specific code.

The phone confirms each key with a brief glow when the screen's handler has
finished; failures are reported on the phone. Layouts need the protocol 3 SDK.

## Session lifecycle

`await keyboard.finish()` (also `restart()`) ends the current instance through
the authenticated API, closes connections, awaits `finish` handlers, and prepares
a fresh QR/key/PIN as applicable. Old links and credentials cease to work.
If your reset handler fails, the SDK does not admit the next visitor; fix or retry
the reset. Always clear application-owned state in `finish` for shared screens.

The phone's **Finish** requests this same operation over the encrypted connection.
Its **Disconnect** stops that phone, clears its local credential and releases the
session, so a new visitor can join straight away. Only one phone is accepted per
session; a credential reconnect replaces its previous connection.

If a paired phone drops without Disconnect (locked, switched apps, lost signal),
only that phone may rejoin for 5 minutes. The screen shows the `waiting` state
meanwhile, and Finish ends the reservation at any time.

`close()` stops SDK timers and sockets. It does not invalidate the server session.
The relay expires unused sessions after 30 days. Reconnection uses a short
exponential backoff. Pending unconfirmed inputs are not automatically replayed;
users are asked to check the receiving screen before sending again.

## Pairing and privacy

No-PIN QRs contain a random 32-byte key in their URL fragment. The phone removes
the fragment from the address bar after reading it. Anyone who captures that QR
can possess its key; keep private-input QRs out of broadcasts and finish sessions
between visitors. PIN mode omits the key and uses SPAKE2 with explicit confirmation.
Every PIN attempt counts toward the five-attempt limit of the running screen
session, including attempts a phone abandons before confirming.

On every connection both endpoints contribute a fresh random salt, and HKDF
derives directional AES-256-GCM keys from the pairing secret and both salts.
Frames carry an increasing counter as their nonce, and each receiver accepts a
counter only once and only in order, so a frame cannot be replayed, reordered, or
carried into a later connection. The relay forwards opaque binary frames and
holds only hashed owner/phone credentials, public configuration, and session
metadata. Displaying a PIN beside a public QR aids pairing, but does not restrict
access against people who can read both.

## Advanced: raw HTTP and WebSocket API

### Create

`POST /i` with JSON configuration (`caps`, `name`, `mode`, `submitLabel`, `actions`,
`pinRequired`, `protocol`, optional `region`) returns `code`, `joinUrl`,
`ownerToken`, `qrUrl`, and the device-role `relay` URL. Set `protocol: 3` (or 2,
without layouts); the hosted phone refuses screens that do not declare a protocol
it supports. Configuration bodies are limited to 16 KiB. Creation
is limited to 20 requests per client per minute (IPv6 clients are counted per
/64). Store the owner token privately. The raw API does not generate an endpoint
key. To use the hosted phone page, implement key generation/QR assembly or PIN
pairing.

### Resolve

`GET /j/<code>` returns public `config` and the phone-role relay URL. Unknown or
finished instances return 404. Possession of a code permits requesting a phone
join, but encrypted input requires the endpoint pairing secret.

### QR formats

`GET /i/<code>/qr` requires `Authorization: Bearer <ownerToken>` and is
available for PIN instances (`pinRequired: true`). A server QR contains only the
join code; without a PIN the phone also needs the screen's key in the QR, which
the relay never sees, so keyed instances return `409` and must render their QR
locally (the browser SDK does). Formats are `svg` (default), `png`, and `bits`
(a newline-separated grid of `0`/`1`). Render options include `ecc=L|M|Q|H`,
`margin`, `scale`, `size`, `dark`, `light`, and `shape=square|rounded|dots`.
Every custom QR display should say **Scan with your phone camera to type here**
and show a destination name.

### Connect

`GET /s/<code>?role=device|phone` upgrades to a WebSocket. Within 10 seconds:

- Device sends `{ "t": "auth", "owner": "<ownerToken>" }`.
- First phone sends `{ "t": "join" }`, receives `{ "type": "credential", "value": "..." }`.
- Reconnecting phone sends `{ "t": "auth", "cred": "..." }`.

A socket that has not authenticated within 10 seconds is closed with `4008`.
The relay sends `hello`, `ready`, and `peer` presence messages as JSON strings;
a `peer` message with `up: false` may carry `reservedForMs` while a dropped phone
may return. After authentication, the only text messages accepted are `k`
(keepalive; the relay replies `k` itself) and, from the phone, `leave` (release
the session). Input is binary. Limits are 120 frames/sec and 32 KiB/sec per
socket, 16 MiB per session per hour, 1024 bytes/frame, and 60 connections per
client per minute.

### End

`POST /i/<code>/end`, with `Authorization: Bearer <ownerToken>`, invalidates the
session and closes every socket with code `4010`. Missing/incorrect owner tokens
return 401/403; unknown or ended instances return 404. Session reset allocates a
new instance rather than reusing an ended code.

### Provision a device over one socket

`/provision?protocol=3` allocates an instance and pushes a JSON `provision`
message containing `code`, `joinUrl` and `ownerToken`, followed by `ready`. Add
`pin=1` for a PIN instance; only then does the message also carry `qr`, a bit
grid of the join URL, because only a PIN QR works without a key the device adds
itself. Persist the owner token to reconnect as the device. Firmware still needs
endpoint encryption and pairing; this is an advanced transport API, not a
ready-made phone keyboard for an arbitrary MCU. See [embedded guidance](LOW_END_MCU.md).

## Advanced: application messages and compatibility

Protocol 2 and 3 frames, each a binary WebSocket message:

- `e3 | 16-byte salt`: sent by both endpoints when the peer is reported up.
- `e0 | 65-byte SPAKE2 share` and `e2 | 32-byte confirmation`: PIN mode only.
- `e1 | 12-byte nonce | AES-GCM ciphertext + 16-byte tag`: everything else. The
  nonce is four zero bytes then a 64-bit big-endian counter starting at 0 for
  each key; receivers drop any counter not greater than the last accepted one.

Keys: HKDF-SHA-256 over the pairing secret (the QR key, or the SPAKE2 secret),
salt = device salt ‖ phone salt, info `qrkb/p2d/v2` (phone to device) and
`qrkb/d2p/v2` (device to phone). Plaintext: `10 | UTF-8 JSON`. Message types:

- `key`: `{ type, id, text }` or `{ type, id, code }`; Backspace may include a
  `count` (expanded into individual events, capped at 10000).
- `text`: `{ type, id, text }`.
- `action`: `{ type, id, action }` (configured action ID).
- `down` / `up`: `{ type, id, key }`, protocol 3 layout keys (`action:<id>` keys
  included). Keys not in the screen's layout are ignored.
- `finish`: `{ type, id }`.
- `ack`: `{ type, id, ok }` (screen to phone, the only message in that direction).

Protocol 3 adds `layout` and the `down`/`up` messages. The phone still serves
protocol 2 screens, rendering their keyboard and `actions` and sending `action`
messages, so pinned protocol 2 SDK files keep working.

Protocol 2 replaced the earlier binary `KEY`/`BUTTON`/`MOVE`/`SCROLL`/`SETTLE`
payloads; the SDK no longer decodes them. Pointer/scroll interfaces are not
features of the current hosted phone keyboard.
