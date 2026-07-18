# qrkeyboard — Developer Quickstart

Turn any screen into something a phone can type on and point at, by showing a QR
code. You build the **screen**; qrkeyboard hosts everything else — the phone
controller, the relay, pairing, and encryption. No app for your users to install,
no account for you to create, no server for you to run.

> **The one thing to understand:** you never build the phone side. Your users
> scan your QR, their phone opens `app.qrkeyboard.com` (our hosted controller),
> and input events arrive in your code. You only (1) show a QR and (2) handle
> events.

---

## Step 0 — One decision: what is your screen?

| Your screen is… | Follow |
|---|---|
| A web page (dashboard, kiosk, game, sign) | **Path A** (2 minutes) |
| A microcontroller / embedded display (ESP32, Pi, teletext) | **Path B** |
| Nothing yet — you just want it to work | Use our hosted screen: open **screen.qrkeyboard.com**, done |

Everything else (relay URL, pairing, crypto, the controller UI) is the same and
handled for you.

---

## Path A — Web page (the simplest)

### 1. Add one script tag and a container
```html
<script src="https://qrkeyboard.com/v1.js"></script>
<div id="qrkb"></div>
```

### 2. Create it — this renders the QR
```js
const kb = QRKeyboard.create({ el: '#qrkb' });
```
A styled QR appears in `#qrkb`. When a phone scans it and pairs, the QR morphs
away and events start flowing.

### 3. Handle events
```js
kb.on('key',    e => console.log(e.text));      // typed text / keys
kb.on('move',   e => moveCursor(e.dx, e.dy));   // pointer deltas
kb.on('button', e => click(e.button, e.down));  // taps / clicks
kb.on('connect',    () => show('phone connected'));
kb.on('disconnect', () => show('phone left'));
```

That's the whole integration. Deploy your page over HTTPS (any host) and it works.

### Even simpler: let it drive the DOM for you
If you just want the phone to act like a real mouse/keyboard on your page, pass a
`target` and skip event handling entirely — qrkeyboard dispatches native
`keydown` / `pointermove` / `wheel` events onto it:
```js
QRKeyboard.create({ el: '#qrkb', target: document.body });
```

### Pick what the phone shows (optional)
```js
QRKeyboard.create({ el: '#qrkb', caps: ['keyboard'] });               // keyboard only
QRKeyboard.create({ el: '#qrkb', caps: ['gyro','buttons'] });          // pointer + buttons
// default caps: ['keyboard','gyro','buttons']
```

### Restrict who can connect with a PIN (optional)
```js
QRKeyboard.create({ el: '#qrkb', pin: '4271' });  // your password (tell users out of band)
QRKeyboard.create({ el: '#qrkb', pin: true });    // random PIN in kb.pin — show it on screen
```
The QR can then be fully public — only someone with the PIN can connect, and the
PIN can't be brute-forced (SPAKE2, 5-try lockout). See the Developer Guide.

---

## Path B — Microcontroller / embedded

You implement this against our API in your own device code — **we don't ship
firmware or a device library.** It's one WebSocket:

```
1. Open  wss://qrkeyboard.com/provision?role=device&caps=keyboard,gyro,buttons
2. Receive JSON  { "type":"provision", "code", "joinUrl", "ownerToken", "qr" }
       → "qr" is a 0/1 text grid: draw a filled cell for each '1' (no QR library)
       → persist ownerToken in flash (to reclaim the same QR after reboot)
3. Receive JSON  { "type":"ready" }
4. Loop: receive BINARY input frames → act on them; ping ≤120 s to stay alive
```

The relay creates the instance and puts the QR (as a plain 0/1 grid) right in the
provision message, so the device needs no QR library and no HTTP. Frame formats
and the grid layout are in **DEVELOPER_GUIDE.md** (§7–§8).

**Hardware note:** the device needs Wi-Fi + TLS, so an ESP32-class chip (or a
Pi). A sub-TLS microcontroller reaches the API through a Wi-Fi coprocessor that
does TLS for it, or via a plaintext-LAN setup — patterns you can use are in
**LOW_END_MCU.md** (guidance only; no firmware from us).

---

## Getting the QR code from our API

If you don't use the web SDK's auto-rendered QR (e.g. you're on an MCU, or you
want the QR as a file/asset), fetch it from our API. **Two calls.**

### 1. Create an instance — returns your short URL + owner token
```
POST https://qrkeyboard.com/i
→ { "code": "ZFCHCX7VKPRGNEL8",
    "joinUrl": "https://qrkeyboard.com/j/ZFCHCX7VKPRGNEL8",   ← goes in the QR
    "ownerToken": "…",     ← SECRET. store it. only this fetches your QR.
    "qrUrl": "https://qrkeyboard.com/i/ZFCHCX7VKPRGNEL8/qr",
    "relay": "wss://qrkeyboard.com/s/ZFCHCX7VKPRGNEL8" }
```
The `ownerToken` is shown once and is the **only** capability that can retrieve
this instance's QR — keep it server-side / in device flash, never ship it to the
phone.

### 2. Fetch the QR — owner token required, your choice of format & style
```
GET  /i/<code>/qr?format=svg|png|bin  &  <style params>
Header:  Authorization: Bearer <ownerToken>
```

| `format` | Returns | Use for |
|---|---|---|
| `svg` (default) | `image/svg+xml`, any size | web, print, scalable |
| `png` | `image/png` raster | fixed-size image asset |
| `bits` | `text/plain` 0/1 grid | **MCUs** — draw a cell per '1' |

Style params: `size=<px>` or `scale=<px/module>` (png) · `margin=<modules>` ·
`dark=<hex>` · `light=<hex>` · `shape=square|rounded|dots` (svg) · `ecc=L|M|Q|H`.

```js
// web: fetch as a blob and show it (owner token stays out of the <img> URL)
const r = await fetch(qrUrl + '?format=svg&shape=rounded&dark=1f6feb',
                      { headers: { Authorization: `Bearer ${ownerToken}` } });
img.src = URL.createObjectURL(await r.blob());
```

The **`bits`** format is a plain text grid — one line per row, `'1'` = dark,
`'0'` = light, quiet-zone border included. An MCU splits on newline and draws a
cell per `'1'` onto its display (teletext, e-ink, OLED) with no QR library.

> Short URLs: on `qrkeyboard.com` the join URL is ~41 chars → a small QR
> (version 3, ~29 modules) that stays crisp on low-res/CRT screens. (During
> pre-launch the workers.dev origin makes it longer.)

## Concepts you may need (all optional)

- **Multiple phones (a shared screen):** pass `room: true`. Every event then
  carries `e.member` (a stable id + color); you also get `join`/`leave` and a
  `kb.kick(id)`. What appears on screen — moderation, layout, who-can-do-what —
  is entirely your code; qrkeyboard only delivers attributed input.
- **Connection state:** `kb.on('connect'|'disconnect')`, and `kb.status` reports
  `direct` (LAN) vs `relay` (internet) so you can show a quality dot.
- **Your own infrastructure (escape hatch):** everything defaults to our hosted
  relay. To self-host, set `relay: 'wss://your-relay'` — the protocol is open and
  the relay is a small deployable. You never *have* to.

---

## What qrkeyboard handles so you don't

The phone controller UI and the gyro/keyboard UX · the QR contents and the
pairing flow · session credentials (issue, store, revoke) · end-to-end
encryption · the relay and its scaling · direct-LAN vs internet fallback · iOS
sensor-permission prompts · offline/PWA behavior.

## What you handle

Showing the QR (one call) · doing something with the events · **on a public
screen, what content you display** (qrkeyboard is the mechanism, not a content
manager).

---

## Security you get for free — and the one rule

Free: physical-presence pairing (only someone who can see your QR can connect),
per-session credentials, single-use time-limited pairing tokens, and
**end-to-end encryption** (AES-256-GCM, key carried only in the QR) so even we
can't read the input.

The one rule you must follow: **don't display the pairing QR anywhere it can be
photographed or streamed if the input is confidential** — a captured QR can pair
for the session's lifetime. For public screens, enable rotation
(`rotate: true`). Flagged per scenario in **USE_CASES.md**.

---

> **Status (2026-07-18):** the relay, hosted phone controller, `v1.js` web
> helper, and end-to-end encryption are live and tested (including through a real
> browser). Pre-launch the service runs at
> `qrkeyboard-relay.videopaintbox.workers.dev`; it moves to `qrkeyboard.com` when
> the domain activates. We ship no MCU firmware — you reach the API from your
> own device code.
