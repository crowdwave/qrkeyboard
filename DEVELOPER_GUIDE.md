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
| `el` | none | CSS selector or element. Its children are replaced by a panel with status, the QR, an instruction, the PIN and a **Try again** button (shown after an error). Without `el` no QR is drawn; render `keyboard.joinUrl` yourself. |
| `name` | `This screen` | Destination shown on the phone; at most 120 characters (the relay refuses longer) |
| `mode` | `live` | `live` streams edits; `text` sends a complete message |
| `submitLabel` | `Send` | Text on the submit button; also describes Enter's action; cut to 40 characters |
| `caps` | `['keyboard']`, or `['keyboard', 'buttons']` with `actions` | Hosted inputs: `keyboard`, `buttons`. Ignored with `layout`/`layouts`, which set it from their blocks |
| `actions` | none | 1 to 16 named actions: `{ id, label }` (each 1–60 characters, ids unique) or plain strings, whose id is their position (`'0'`, `'1'`, …) |
| `pin` | none | `true` generates a 4-digit PIN shown under the QR; a string sets your own (the phone accepts up to 64 characters) |
| `rejoinSeconds` | `300` | How long a phone that drops (locked, switched apps, lost signal) keeps the screen before another phone may join; 0 to 3600. `0` frees the screen the moment the phone drops. |
| `region` | automatic | Optional relay placement hint: `wnam`, `enam`, `sam`, `weur`, `eeur`, `apac`, `oc`, `afr`, `me`; any other value is rejected |
| `cellSize`, `margin` | `4`, `4` | QR rendering options |
| `layout` | none | Phone controls: `'keyboard'`, `'buttons'`, `'remote'`, `'teletext'`, `'gamepad'`, or a list of blocks; see [Controls](#controls-remotes-keypads-and-game-buttons) |
| `layouts`, `context` | none; the first layout named | Instead of `layout`: named layouts the screen switches between, and the one shown first; see [Switching layouts](#switching-layouts-as-your-app-changes) |
| `target` | none | Advanced: dispatch synthetic `keydown` (and, for layout keys, `keyup`) events to this element; does not insert text automatically. With `target`, input counts as handled even without a listener. |

Buttons-only setup uses `caps: ['buttons']`. Without explicit action labels,
buttons are named **Previous** and **Next**. Prefer labels specific to your app.

The QR panel emits its own accessible status and instructions, so your page
needs neither. Style `.qrkeyboard-pairing` and `.qrkeyboard-code` to fit your
screen. Keep the QR dark on white, leave its quiet zone intact, and keep it
large enough to scan.

The keyboard object also exposes, read-only: `state` (as in the `state` event),
`code` (the join code), `joinUrl` (what the QR encodes), `pin`, `config` (the
configuration sent to the relay), `context` (with `layouts`; settable) and
`ownerToken` (keep it private).

## Input events and delivery confirmation

```js
keyboard.on('text', async ({ text }) => {
  await saveMessage(text);
});
```

The phone confirms an input after every handler for that event completes:
text, live keys and Enter show **Sent**; layout keys and action buttons glow
instead. Returning a Promise delays the confirmation until the operation
finishes. Throwing or rejecting reports failure to the phone. With no input
handler (and no `target`), the SDK rejects the input rather than claiming
delivery. This confirms handler completion, not any downstream effect the
handler neglected to await.

| Event | Payload | When |
|---|---|---|
| `key` | `{ text, id }` or `{ code, id }` | Live text edits or special keys |
| `text` | `{ text, action, id }` | Complete-message submission; `action` is the submit label |
| `action` | `{ action, label, id }` | Named button activated; `action` is its configured action ID |
| `state` | `{ state, message }` | `preparing`, `scan`, `connected`, `waiting` (a dropped phone may return), `lost`, `finished`, `error` |
| `ready` | `{ code }` | A fresh instance and QR have been prepared |
| `connect`, `disconnect` | `{}` | Phone pairing established or lost (an ended session moves to `finished` without `disconnect`) |
| `finish` | `{ code }` | Old session ended; clear application state before a fresh QR is created (`code` may be `null`) |
| `reset` | `{}` | Finish handlers completed |
| `pinfail` | `{ attemptsLeft }` | A PIN attempt ended without pairing (wrong PIN, or the phone left mid-check) |
| `locked` | `{}` | Five PIN attempts without pairing; state becomes `error`; call `restart()` for a fresh session |
| `error` | Error | Network, input-handler, pairing, or reset failure |
| `press` | `{ key, label, id }` | A layout key went down (once per press). `action:<id>` keys raise `action` instead of `press`/`keydown`/`keyup`. |
| `keydown` | `{ key, repeat, id }` | A layout key went down; repeats with `repeat: true` while a `repeat` key is held |
| `keyup` | `{ key, id }` | A layout key was released |

Special key codes: `8` Backspace, `9` Tab, `10` Enter, `27` Escape, `37–40` arrows.
In live mode the phone diffs committed text and emits suffix backspaces/text.
The receiving application should remove one Unicode code point per Backspace.
Bulk deletions travel as one counted key message; the SDK expands them into
individual Backspace events without flooding the relay.

Messages must fit the relay's 1024-byte frame limit. After the 29 bytes of
framing and authentication tag, 995 plaintext bytes remain (a 1-byte tag and
994 bytes of JSON),
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
- `{ type: 'keys', columns, keys }`: a grid of 1 to 20 keys, `columns` 1–5
  (default: the number of keys, up to 3). Each key
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
- `'action:<id>'` for one of your `actions`; it raises the `action` event (and
  not `press`, `keydown` or `keyup`).

Each key appears once per layout, and a layout holds at most 40 keys. A D-pad
counts as its five keys, so a layout with a D-pad cannot also list the arrows
or `Enter`. The relay rejects any layout outside these rules with a 400 that
names the problem.

Every press sends the key down and, on release, up. The SDK raises `press` and
`keydown`, repeats `keydown` (after 400 ms, every 80 ms) for keys marked `repeat`,
and raises `keyup` on release. A held key is released if the phone disconnects
or after 15 seconds without a release. With `target`, the same keys arrive as
ordinary `keydown`/`keyup` events, so an existing TV web app that already handles
remote keys needs no QRKeyboard-specific code.

The phone confirms each key with a brief glow when the screen's handler has
finished; failures are reported on the phone. Layouts need the protocol 3 SDK.

### Switching layouts as your app changes

Give `layouts` instead of `layout`: up to 6 named layouts (names of 1–32
letters, digits, `-` or `_`, starting with a letter or digit), and the
`context` to show first (default: the first one named). Set
`keyboard.context` whenever your app moves on, and the phone follows straight
away:

```js
const keyboard = QRKeyboard.create({
  el: '#qr', name: 'Arcade',
  layouts: { menu: 'remote', play: 'gamepad', name: [{ type: 'text' }] },
  context: 'menu',
});
keyboard.on('press', ({ key }) => {
  if (keyboard.context === 'menu' && key === 'Enter') { startGame(); keyboard.context = 'play'; }
});
onGameOver(() => { keyboard.context = 'name'; }); // no key press needed
```

The phone receives every layout when it pairs, and the configuration hash
covers them all, so only those layouts can ever appear. It keeps one question
open, asking the screen to say when the context differs from the layout it
shows, and the screen answers that question the moment you set `context`. The
answer is an acknowledgement carrying the layout's name, nothing else. Every
other acknowledgement carries the current context too, so a press that changes
it switches the phone in its own reply. A press from a layout the screen has
left, sent before the phone caught up, is refused and switches the phone
instead of running your handler. Keys still held when the context changes are
released, with `keyup`. Setting a name not in `layouts` throws.

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

If a phone that has joined drops without Disconnect (locked, switched apps, lost
signal), only that phone may rejoin for `rejoinSeconds` (default 5 minutes).
With `rejoinSeconds: 0` the dropout loses control at once: the next phone to scan
takes over, and the dropped phone's credential stops working. This applies even if it never
finished pairing, because the relay cannot see pairing. The screen shows the
`waiting` state meanwhile, and Finish ends the reservation at any time.

`close()` stops the SDK for good: it tells the relay the screen has left, which
ends the server session, and the phone asks its visitor to scan the screen's new
QR. The SDK does the same on `pagehide`, so a closed or reloaded page never
leaves a phone waiting. After `close()`, `finish()` and `restart()` do nothing;
call `QRKeyboard.create()` again. To serve the next visitor instead, use
`finish()`. A screen that drops without leaving (network loss) is expected back;
the phone waits for it. The relay expires unused sessions after 30 days. Reconnection uses a short
exponential backoff. Pending unconfirmed inputs are not automatically replayed;
users are asked to check the receiving screen before sending again.

## Pairing and privacy

No-PIN QRs contain a random 32-byte key in their URL fragment. The phone removes
the fragment from the address bar after reading it. Anyone who captures that QR
can possess its key; keep private-input QRs out of broadcasts and finish sessions
between visitors. PIN mode omits the key and uses SPAKE2 with explicit confirmation.
Every PIN attempt counts toward a limit of five consecutive failures, including
attempts a phone abandons before confirming; a successful pairing or Finish
resets the count. Once locked, only `finish()` or `restart()` unlocks.

On every connection both endpoints contribute a fresh random salt, and HKDF
derives directional AES-256-GCM keys from the pairing secret and both salts.
Frames carry an increasing counter as their nonce, and each receiver accepts a
counter only once and only in order, so a frame cannot be replayed, reordered, or
carried into a later connection. The relay forwards opaque binary frames and
holds only hashed owner/phone credentials, public configuration, and session
metadata. Displaying a PIN beside a public QR aids pairing, but does not restrict
access against people who can read both.

The phone reads the screen's name and controls from the relay, so the QR also
carries a hash of the configuration (`h` in the fragment). The phone refuses a
configuration that does not match, and in PIN mode the hash is part of the
SPAKE2 transcript as well, so pairing fails if the relay altered it. The
configuration is authenticated, not secret: the relay stores it and serves it
to anyone with the code, so put nothing private in names or labels.

## Advanced: raw HTTP and WebSocket API

### Create

`POST /i` with JSON configuration (`caps`, `name`, `mode`, `submitLabel`, `actions`,
`pinRequired`, `protocol`, `layout` or `layouts` with `context`, `rejoinSeconds`, optional `region`) returns `code`, `joinUrl`,
`ownerToken`, `qrUrl`, and the device-role `relay` URL. Any other top-level
field is refused with 400 (fields inside layouts, keys and actions are not
checked beyond the layout rules). Set `protocol: 3`; the hosted phone refuses
screens that declare anything else. Configuration bodies are limited to 16 KiB
(413 above that). Creation is limited to 20 requests per client per minute
(IPv6 clients are counted per /64). Store the owner token privately. The raw API does not generate an endpoint
key. To use the hosted phone page, implement key generation/QR assembly or PIN
pairing.

### Resolve

`GET /j/<code>` returns public `config` and the phone-role relay URL. Unknown or
finished instances return 404. Lookups are limited to 60 per client per minute. Possession of a code permits requesting a phone
join, but encrypted input requires the endpoint pairing secret.

### QR formats

`GET /i/<code>/qr` requires `Authorization: Bearer <ownerToken>` and is
available for PIN instances (`pinRequired: true`). A server QR contains only the
join code; without a PIN the phone also needs the screen's key in the QR, which
the relay never sees, so keyed instances return `409` and must render their QR
locally (the browser SDK does). Formats are `svg` (default), `png`, and `bits`
(a newline-separated grid of `0`/`1`). Render options include `ecc=L|M|Q|H`,
`margin` (0–16), `scale` (1–64), `size` (32–4096 px), `dark`, `light`, and
`shape=square|rounded|dots`; out-of-range numbers are clamped. Responses may be
cached for 5 minutes. A server QR carries only `c=`, so the phone checks the
configuration through the PIN transcript instead of `h`.
Every custom QR display should say **Scan with your phone camera to type here**
and show a destination name.

### Connect

`GET /s/<code>?role=device|phone` upgrades to a WebSocket. Within 10 seconds:

- Device sends `{ "t": "auth", "owner": "<ownerToken>" }`.
- First phone sends `{ "t": "join" }`, receives `{ "type": "credential", "role": "phone", "value": "..." }`.
- Reconnecting phone sends `{ "t": "auth", "cred": "..." }`.

A request without `Upgrade: websocket` gets 426, and one session accepts at
most 30 upgrades per 10 seconds (429). Close codes:

| Code | Meaning |
|---|---|
| `4001` | Relay lost the socket's state |
| `4003` | Authentication failed, or a binary frame arrived before authentication |
| `4004` | No such instance |
| `4005` | The screen is in use by another phone, or this phone reconnected elsewhere |
| `4008` | Not authenticated within 10 seconds, or a rate limit was exceeded |
| `4009` | Frame larger than 1024 bytes |
| `4010` | Session finished (`/end` or Finish) |
| `4011` | The screen left (`leave` from the device) |

The relay sends `hello`, `ready`, and `peer` presence messages as JSON strings;
a `peer` message with `up: false` may carry `reservedForMs` while a dropped phone
may return. After authentication, the only text messages accepted are `k`
(keepalive; the relay replies `k` itself) and `leave`: from the phone it releases
the session; from the device it ends the session and closes every socket with
`4011`. Clients ignore text they cannot parse and message types they do not
know. Input is binary. Limits are 120 frames/sec and 32 KiB/sec per
socket, 16 MiB per session per hour, 1024 bytes/frame, and 60 connections per
client per minute.

### End

`POST /i/<code>/end`, with `Authorization: Bearer <ownerToken>`, invalidates the
session and closes every socket with code `4010`. Missing/incorrect owner tokens
return 401/403; unknown or ended instances return 404. Session reset allocates a
new instance rather than reusing an ended code.

### Provision a device over one socket

`/provision?protocol=3` (a WebSocket upgrade) allocates an instance and pushes a
JSON `provision` message containing `code`, `joinUrl` and `ownerToken`, followed
by `ready`; the socket is already authenticated as the device. Only `caps`
(comma-separated, default `keyboard`), `pin=1` and `protocol` can be set: name,
mode, labels and layouts need `POST /i`. With `pin=1` the message also carries
`qr`, a bit grid of the join URL, because only a PIN QR works without a key the
device adds itself. Provisioning shares the creation limit of 20 per client per
minute. Persist the owner token, and reconnect as the device at
`/s/<code>?role=device`. Firmware still needs
endpoint encryption and pairing; this is an advanced transport API, not a
ready-made phone keyboard for an arbitrary MCU. See [embedded guidance](LOW_END_MCU.md).

## Advanced: application messages and compatibility

Audited against the relay and SDK source on 2026-10-11.

Protocol 3 frames, each a binary WebSocket message:

- `e3 | 16-byte salt`: sent by both endpoints when the peer is reported up.
- `e0 | 65-byte SPAKE2 share` and `e2 | 32-byte confirmation`: PIN mode only.
  P-256 with the RFC 9382 §6 M and N; the device uses M, the phone N. The
  password scalar is SHA-256(`"qrkb/spake2/v1:" + code + ":" + pin`) mod n (0
  becomes 1). The transcript is, each with an 8-byte little-endian length
  prefix: the label `qrkb/spake2/v2`, the join code, the configuration hash, the
  device share, the phone share, the shared point and the password scalar (32
  bytes, big-endian). Ke and Kc are HKDF-SHA-256 over SHA-256(transcript), empty
  salt, info `qrkb/ke` and `qrkb/kc`; the device confirms with HMAC(Kc, "CA"),
  the phone with HMAC(Kc, "CB"). The screen, not the relay, limits PIN attempts.
- `e1 | 12-byte nonce | AES-GCM ciphertext + 16-byte tag`: everything else. The
  nonce is four zero bytes then a 64-bit big-endian counter starting at 0 for
  each key; receivers drop any counter not greater than the last accepted one.

Keys: HKDF-SHA-256 over the pairing secret (the QR key, or the SPAKE2 secret),
salt = device salt ‖ phone salt, info `qrkb/p2d/v2` (phone to device) and
`qrkb/d2p/v2` (device to phone). Plaintext: `10 | UTF-8 JSON`. Every message
`id` is 8 random bytes as unpadded base64url (11 characters); the screen ignores
any other shape. Message types:

- `key`: `{ type, id, text }` or `{ type, id, code }`; Backspace may include a
  `count` (expanded into individual events, capped at 10000).
- `text`: `{ type, id, text }`.
- `action`: `{ type, id, action }` (configured action ID). The SDK accepts it;
  the hosted phone sends action buttons as `down`/`up` with `action:<id>`.
- `down` / `up`: `{ type, id, key }`, layout keys (`action:<id>` keys included).
- `finish`: `{ type, id }`. Success is not acknowledged: the phone learns of it
  from close code `4010`. Only a failed finish gets `ok: false`.
- `context`: `{ type, id, have }`, with `layouts` only. `have` is the layout the
  phone shows. The screen holds the ack until its context differs from `have`
  (at once if it already does); the phone asks again after each answer, so it
  holds one question at a time. A screen without `layouts` answers `ok: false`.
- `ack`: `{ type, id, ok }` (screen to phone, the only message in that direction).
  With `layouts`, every ack also carries `context`, the screen's current layout
  name; the phone switches only to names in its configuration. A `down`/`up`
  key from another of the screen's layouts gets `ok: false` with `context`.
  The screen answers a type it does not know with `ok: false`. A known type it
  cannot use gets no ack at all, and the phone reports the delivery unconfirmed
  after 8 seconds: a key in none of its layouts, `key` in message mode, `text`
  in live mode, or an unknown action ID. The hosted phone never sends these.

The phone allows 12 seconds for pairing to complete after the screen is up.

The configuration hash is SHA-256 over the configuration as JSON with object
keys sorted at every level and no whitespace, truncated to 16 bytes, carried in
the QR fragment as `h=<base64url>` after `c=` (and `k=` without a PIN). Hash the
configuration exactly as the phone will receive it from `/j/<code>`: every
field explicit (`caps`, `mode`, `name`, `submitLabel`, and `actions`,
`pinRequired: true`, `protocol`, `layout` or `layouts` and `context`, `rejoinSeconds` when used), without `region`.

Protocol 3 adds `layout`, `layouts` and `context`, the `down`/`up` and `context`
messages, and the configuration hash.
It is the only protocol the phone accepts: an older screen has no hash, so
accepting one would let the relay pass off any screen as old to skip the
check.

Protocol 2 replaced the earlier binary `KEY`/`BUTTON`/`MOVE`/`SCROLL`/`SETTLE`
payloads; the SDK no longer decodes them. Pointer/scroll interfaces are not
features of the current hosted phone keyboard.
