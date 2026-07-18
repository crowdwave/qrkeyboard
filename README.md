# qrkeyboard

**Scan a QR code on any screen and your phone becomes its keyboard, gyro mouse,
and buttons — no app install.** You build the screen; qrkeyboard hosts the phone
controller, the relay, pairing, and end-to-end encryption. You get a QR and a
stream of input events.

```html
<script src="https://qrkeyboard.com/v1.js"></script>
<div id="qrkb"></div>
<script>
  const kb = QRKeyboard.create({ el: '#qrkb', caps: ['keyboard', 'gyro', 'buttons'] });
  kb.on('key',    e => console.log(e.text));
  kb.on('move',   e => moveCursor(e.dx, e.dy));
  kb.on('button', e => click(e.button, e.down));
</script>
```

Your users scan, their phone opens our controller, and their input arrives in
your code — keystrokes and pointer motion encrypted end-to-end (we can't read
them; the key lives only in the QR).

## Docs

- **[QUICKSTART.md](QUICKSTART.md)** — get running in two minutes (web or MCU).
- **[DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md)** — the complete API: endpoints,
  capabilities, event formats, QR retrieval, handshake, reference tables.
- **[USE_CASES.md](USE_CASES.md)** — what people build with it, and which
  scenarios need care with a publicly-visible QR.
- **[LOW_END_MCU.md](LOW_END_MCU.md)** — patterns for reaching the API from
  constrained microcontrollers.
- **[examples/](examples/)** — a minimal working web page.

## What we provide vs. what you build

We provide the QR-getting API, the event stream, and the hosted phone
controller. You build your screen (a web page, a microcontroller, a game) and
reach the API from it. We ship no device firmware or SDK beyond the optional
`v1.js` web helper — the raw HTTP/WebSocket API works from anything.

## Status

Live at **https://qrkeyboard.com**. End-to-end encryption is always on; an
optional PIN adds access control on a public QR. Free to start.
