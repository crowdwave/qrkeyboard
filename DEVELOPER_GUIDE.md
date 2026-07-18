# qrkeyboard — Developer Guide

Everything you need to put a QR code on your screen so that when your end user
scans it, their phone becomes an input device — with exactly the capabilities you
choose (keyboard, gyro mouse, buttons, scroll) — and the input arrives in your
code. No app for them to install; nothing to host but your own screen.

> **Base URL.** Production is `https://qrkeyboard.com`. Pre-launch (until the
> custom domain activates) it is `https://qrkeyboard-relay.videopaintbox.workers.dev`
> — substitute that in the examples for now. WebSocket URLs use `wss://` on the
> same host.

---

## 1. How it works (the whole model in one minute)

```
   You create an instance ─► you get a short joinUrl + a QR
   End user scans the QR   ─► their phone opens OUR controller (app.qrkeyboard.com)
   Controller shows exactly the inputs YOU declared (keyboard / gyro / buttons…)
   End user's input        ─► arrives in YOUR code as events (web) or frames (MCU)
```

Three parties:
- **You / your screen** — a web page, or a microcontroller, or a game. You show a
  QR and consume input. This is all you build.
- **qrkeyboard (us)** — the phone controller UI, the relay, pairing, encryption.
  You never build the phone side.
- **The end user's phone** — scans, gets the controller, sends input. No install.

> **Scope of what we provide.** Just two things: an API to **get a QR** and a
> stream to **receive events** (plus the hosted phone controller the QR opens).
> We do **not** ship MCU firmware, device SDKs, or bridge code — you talk to the
> HTTP/WebSocket API below from whatever your screen is. The optional `v1.js`
> web helper is a thin convenience; the raw API works without it.

You interact with the system in exactly **two moments**: (1) create an instance
and obtain its QR, (2) receive input events. Everything below fills those in.

---

## 2. The three choices you make

| Choice | Options |
|---|---|
| **Capabilities** you offer | `keyboard`, `gyro` (motion pointer), `touch` (trackpad), `buttons`, `scroll` |
| **How you show the QR** | web SDK auto-renders it · fetch it as SVG/PNG · MCU gets it pushed over one socket |
| **Where input goes** | high-level events (web SDK) · raw binary frames (MCU) |

You declare capabilities once, at instance creation. The controller the end user
sees renders **only** those inputs, and you only receive events for what you
offered.

---

## 3. Quick start — web page (2 minutes)

```html
<script src="https://qrkeyboard.com/v1.js"></script>
<div id="qrkb"></div>
<script>
  const kb = QRKeyboard.create({
    el: '#qrkb',
    caps: ['keyboard', 'gyro', 'buttons'],   // what the phone will offer
    name: 'My Dashboard'                      // shown on the phone to confirm
  });

  kb.on('key',    e => insertText(e.text));           // keyboard
  kb.on('move',   e => moveCursor(e.dx, e.dy));        // gyro / touch pointer
  kb.on('button', e => click(e.button, e.down));       // buttons
  kb.on('scroll', e => scrollBy(e.dy));                // scroll
  kb.on('connect',    () => {});                       // a phone paired
  kb.on('disconnect', () => {});                       // it left
</script>
```

`QRKeyboard.create` renders the QR into `#qrkb`, pairs through our relay, and
emits events. Deploy your page over HTTPS (any host) and it works.

**Zero-code variant** — drive the DOM directly (phone acts like a real
mouse/keyboard on your page):
```js
QRKeyboard.create({ el: '#qrkb', caps: ['gyro','keyboard'], target: document.body });
// dispatches native pointermove / keydown / wheel on the target
```

---

## 4. Quick start — microcontroller (single socket)

The device opens **one** WebSocket; the relay creates the instance and **pushes
the QR bitmap back**. No QR library, no HTTP, no tokens to manage.

```
1. Open  wss://qrkeyboard.com/provision?role=device&caps=keyboard,gyro,buttons
2. Receive JSON  { "type":"provision", "code", "joinUrl", "ownerToken", "qr" }
       → "qr" is a 0/1 grid (§8): blit a filled cell for each '1' onto your display
       → persist ownerToken in flash (to reclaim the same QR after reboot)
3. Receive JSON  { "type":"ready" }
4. Loop: receive BINARY input frames (§7) → act on them; send a keepalive ping ≤120 s
```
Everything you need is in the one `provision` message — the QR grid included — so
there's nothing binary to demultiplex and no QR library required.

Reboot with a stable QR: connect `wss://…/s/<code>?role=device`, send
`{"t":"auth","owner":"<ownerToken>"}`, and you're back on the same code.

That's the entire contract — you implement it on your device in whatever
language/stack you use. Can't do TLS (AVR-class)? **LOW_END_MCU.md** collects
patterns *you* can use (a Wi-Fi coprocessor that does TLS for you, or a
plaintext-LAN setup); the frame contract below is identical regardless.

---

## 5. Declaring capabilities

Capabilities are the config you set when creating the instance. They are public
to the phone (via `/j/<code>`) so the controller renders exactly them.

```json
{
  "layout": "auto",                       // auto | keyboard | pointer | gamepad
  "caps":   ["keyboard", "gyro", "buttons", "scroll"],
  "name":   "My Device",                  // shown on the controller to confirm
  "rotate": false                         // true → rotate the QR for public screens
}
```

| Capability | Phone shows | You receive |
|---|---|---|
| `keyboard` | native keyboard + extras bar | `key` events / `KEY` frames |
| `gyro` | clutch-to-aim motion pointer | `move` + `button` |
| `touch` | trackpad surface | `move` + `button` |
| `buttons` | on-screen buttons / d-pad | `button` events / `BUTTON` frames |
| `scroll` | scroll gesture | `scroll` events / `SCROLL` frames |

Offer only what you handle. A phone that scans a `["keyboard"]` instance never
shows a pointer, and you never get `move` events.

---

## 6. Getting the QR — all the ways

### a) Web SDK (auto) — §3. It renders the QR for you; nothing else to do.

### b) HTTP API — fetch the QR yourself (web asset, print, custom UI)
```
POST /i                       body = your config (§5)   →  { code, joinUrl, ownerToken, qrUrl, relay }
GET  /i/<code>/qr?format=…    header: Authorization: Bearer <ownerToken>
```
`ownerToken` is shown once and is the **only** key that can fetch this instance's
QR — keep it server-side / in device flash; never send it to the phone.

| `format` | Content-Type | For |
|---|---|---|
| `svg` (default) | image/svg+xml | web, print, any size |
| `png` | image/png | fixed-size raster |
| `bits` | text/plain | MCUs — a 0/1 grid you draw yourself (§8) |

Style params: `size=<px>` or `scale=<px/module>` · `margin=<modules>` ·
`dark=<hex>` · `light=<hex>` · `shape=square|rounded|dots` (svg) · `ecc=L|M|Q|H`.

```js
// embed without leaking the owner token into an <img> URL:
const r = await fetch(qrUrl + '?format=svg&shape=rounded&dark=1f6feb',
                      { headers: { Authorization: `Bearer ${ownerToken}` } });
document.querySelector('#qr').src = URL.createObjectURL(await r.blob());
```

### c) MCU provisioning — the QR is pushed over the socket. §4.

---

## 7. Receiving input

### Web (SDK events)
| Event | Fields | Meaning |
|---|---|---|
| `key` | `{ text }` or `{ code }` | text typed, or a special key (Enter/Backspace/arrows) |
| `move` | `{ dx, dy }` | pointer delta (screen-space, already fused from the gyro) |
| `button` | `{ button, down }` | button id, pressed/released |
| `scroll` | `{ dx, dy }` | scroll delta |
| `connect` / `disconnect` | — | a phone paired / left |
| `member` (multi-user) | on every event | which participant (see §9) |

### MCU / raw (binary frames)
Every input is a small binary WebSocket frame; **byte 0 is the type**. Multi-byte
integers are big-endian. (The web SDK parses these for you.)

| Frame | Bytes | Layout |
|---|---|---|
| `MOVE` | 5 | `01  dxHi dxLo  dyHi dyLo` (int16 dx, int16 dy) |
| `BUTTON` | 3 | `02  buttonId  down(1/0)` |
| `SCROLL` | 3 | `03  deltaHi deltaLo` (int16) |
| `KEY` (text) | 2+n | `04  00  <utf-8 bytes>` |
| `KEY` (special) | 3 | `04  01  keycode` (8=Bksp, 9=Tab, 10=Enter, 27=Esc, 37-40=arrows) |
| `SETTLE` | 1 | `05` — motion ended; last position is final (loss repair) |

Pointer frames are latest-wins: if you fall behind, jump to the newest and smear
2–3 frames rather than snapping. Keyboard frames are reliable — apply in order.

---

## 8. The QR grid format (`bits`) — for MCUs that render it themselves

`format=bits` (and the `qr` field in the provision message) is a plain text grid:
```
one line per row, '1' = dark module, '0' = light (a 0-border quiet zone included)
e.g.
    0000000000...
    0011111110...
    0010000010...
    ...
```
Split on newline; for every `'1'` draw a filled cell on your display. No
bit-packing, no format to learn, parses in any language. A small QR is ~29–49
lines; each line is `size + 2·margin` characters.

---

## 9. Connection lifecycle & multi-user

- **Pairing:** end user scans → controller loads → pairs. You get `connect`.
- **Presence:** you're told when the peer is up/down (`connect`/`disconnect`;
  raw: `{"type":"peer","up":bool}`).
- **Reconnect:** the phone keeps a credential and rejoins automatically; the
  device reclaims its code with the `ownerToken`.
- **Multiple phones (a shared screen):** create with `{ "room": true }`. Every
  event then carries a stable `member` id; you also get join/leave and can
  `kick(id)`. What appears on screen is your app — we deliver attributed input,
  nothing more.

---

## 10. Security — what you get, and the one rule

**Free & automatic:** pairing requires physical sight of your QR; per-session
credentials; single-use, time-limited join; **end-to-end encryption** (AES-256-GCM,
key carried only in the QR) so even we can't read the input. Capabilities are
enforced controller-side — a phone can't send input types you didn't offer.

**The one rule:** a QR is a bearer capability for its lifetime. **Don't display
the pairing QR where it can be photographed or streamed if the input is
confidential** (e.g. someone typing a password). For public/broadcast screens set
`"rotate": true`, and never use qrkeyboard for OS-level/console input on a
publicly-visible QR. Per-scenario guidance: **USE_CASES.md**.

---

## 11. Reference

### Endpoints
| Method · Path | Auth | Purpose |
|---|---|---|
| `POST /i` | none | create instance (body = config §5) → `{code, joinUrl, ownerToken, qrUrl, relay}` |
| `GET /i/<code>/qr` | `Bearer ownerToken` | QR as svg/png/bin (§6) |
| `GET /provision` (WS, `role=device`) | connection = capability | single-socket create + QR push (§4) |
| `GET /j/<code>` | none (code = permission) | phone config incl. declared caps |
| `GET /s/<code>?role=device\|phone` (WS) | first-message (below) | the relay |
| `GET /healthz` | none | health |

### Relay WebSocket handshake (`/s/<code>`)
1. Server → `{"type":"hello","v":1}`.
2. Client → auth as its **first** message:
   - device: `{"t":"auth","owner":"<ownerToken>"}`
   - phone first join: `{"t":"join"}` → server issues `{"type":"credential","role":"phone","value":"<cred>"}`
   - phone reconnect: `{"t":"auth","cred":"<cred>"}`
3. Server → `{"type":"ready"}`, then `{"type":"peer","up":bool}` on presence changes.
4. Data plane: binary frames (§7), forwarded verbatim device↔phone.

Secrets travel in messages, never in URLs (so they aren't logged). Send a
keepalive ≤120 s idle.

### Close codes
| Code | Meaning |
|---|---|
| 4001 | internal / no attachment |
| 4003 | bad auth (wrong owner token, bad credential, malformed) |
| 4004 | no such instance (create it first) |
| 4008 | auth timeout (didn't authenticate in 10 s) or rate limit |
| 4009 | frame too large (>1024 bytes) |

### Limits
Frame ≤1024 B · ≤120 msgs/s and ≤32 KB/s per socket · 30-day idle session
expiry · session id (code) is 16-char base32 (~80-bit, server-allocated).

---

## 12. What's live vs pending (2026-07-18)

**Live & tested now:** the relay, instance creation, owner-gated QR
(svg/png/bits), single-socket provisioning with QR push + reboot-reclaim,
capability config flowing to `/j/<code>`, the full WebSocket handshake and
framing, the hosted phone controller, the `v1.js` web helper, and end-to-end
encryption (verified through a real browser). During pre-launch the service runs
at `qrkeyboard-relay.videopaintbox.workers.dev`; it moves to `qrkeyboard.com`
(shorter QR URLs) when the domain goes live — no code change for you.

**Pending:** the custom short domain (`app.qrkeyboard.com`) and a service-worker
offline mode. Neither changes the developer contract above.

**Out of scope (we do not provide):** MCU firmware, device/embedded SDKs, bridge
firmware, prebuilt flashable binaries. You reach the API above from your own
device code. We provide the QR and the events; the rest of your screen is yours.
