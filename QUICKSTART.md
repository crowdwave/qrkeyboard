# Your first phone keyboard

**Goal:** put a QR code on your page and see text arrive from your phone.
You need a computer, a phone with a camera, and internet on both. There is no
QRKeyboard account, API key, package installation, or backend to configure for
this browser integration.

Want to experience it before writing code? Open the [hosted demo](https://qrkeyboard.com/#demo)
on your computer and scan its QR with your phone. It runs the deployed version;
see [release status](RELEASE_STATUS.md) for features still awaiting deployment.

## 1. Save this as `index.html`

This is the complete app. Copy it unchanged, or use the identical
[starter file](examples/index.html).

```html
<!doctype html>
<html lang="en">
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>My first phone keyboard</title>
<style>
  body { font: 18px/1.5 system-ui; max-width: 640px; margin: 40px auto; padding: 20px; }
  #qr svg { width: 220px; max-width: 100%; height: auto; }
  #text { min-height: 100px; padding: 16px; background: #f4f7f2; white-space: pre-wrap; overflow-wrap: anywhere; }
  [hidden] { display: none; }
</style>
<h1>Type on your phone. See it here.</h1>
<p>Scan with your phone camera to type here.</p>
<div id="qr"></div>
<p id="status" role="status">Waiting for your phone…</p>
<pre id="text" aria-label="Text received from your phone"></pre>
<script src="https://qrkeyboard.com/v1.js"></script>
<script>
  const output = document.querySelector('#text');
  const status = document.querySelector('#status');
  const keyboard = QRKeyboard.create({ el: '#qr', name: 'My first demo', caps: ['keyboard'] });
  keyboard.on('key', event => {
    if (event.text !== undefined) output.textContent += event.text;
    else if (event.code === 8) output.textContent = [...output.textContent].slice(0, -1).join('');
    else if (event.code === 10) output.textContent += '\n';
  });
  keyboard.on('connect', () => { status.textContent = 'Phone connected — start typing.'; });
  keyboard.on('disconnect', () => { status.textContent = 'Waiting for your phone to reconnect…'; });
  keyboard.on('error', () => { status.textContent = 'Could not connect. Check your internet connection and reload.'; });
  keyboard.on('finish', () => { output.textContent = ''; status.textContent = 'Waiting for the next phone…'; });
</script>
</html>
```

There are three important lines: load `v1.js`, call `QRKeyboard.create(...)`, and
handle `key`. The other lines make the demo readable and show connection status.
The explicit `caps: ['keyboard']` also keeps the starter keyboard-only when used
with the previous hosted SDK.

## 2. Open it on your computer

Try opening the saved file directly in your browser first. If the browser blocks
local-file access, run this in the folder containing `index.html` (Python 3):

```bash
python3 -m http.server 8080 --bind 127.0.0.1
```

Then open **http://127.0.0.1:8080/** on the computer. If you already have a local
dev server, put the file there instead. The browser needs Web Crypto; HTTPS and
loopback development addresses are trusted contexts. Browser treatment of direct
file URLs can vary. [Secure-context specification](https://www.w3.org/TR/secure-contexts/#is-origin-trustworthy).

**Your phone does not open this local URL.** It scans the QR and opens the hosted
phone page. Both browsers connect to the hosted relay over the internet. You do
not need a tunnel, router changes, or the same Wi-Fi network.

## 3. Scan and type

1. Keep the computer page open; it displays a QR code.
2. Scan it using your phone's camera and open the link.
3. Check the destination name: **My first demo**.
4. Tap **Start typing**. On the earlier hosted phone page, tap **Connect**, then
   **keyboard** if needed.
5. Type `hello screen`. Those words appear on the computer. Try Backspace and Enter.

That is the successful first run. If it does not happen, use the
[short troubleshooting table](DEPLOY.md#troubleshooting).

## 4. Put it online

Upload the same `index.html` to your existing HTTPS website. There is no build
step or server process to deploy for this example. If you need a host, follow
[Deploy your page](DEPLOY.md) for drag-and-drop or a two-command upload.

Open the deployed page and scan its newly generated QR. Do not copy the demo's
QR image into your app: each running page creates its own connection.

## 5. Replace the demo handler with your app's behaviour

Keep `QRKeyboard.create(...)` and change the `key` handler to update your own
field, search box, or application state. A received event has either `text` or a
special-key `code`. In this example, `8` deletes one Unicode code point and `10`
inserts a newline; your app can use Enter to submit instead.

Create one keyboard per receiving screen. Both endpoints need internet; keep
the receiving page open while the phone is connected.

## Next: compose a message or add presentation buttons

These features require the updated service in [release status](RELEASE_STATUS.md).
Use a complete example rather than adding options without an input handler:

| You want | Example | Event your app handles |
|---|---|---|
| Typing appears immediately | [Starter](examples/index.html) | `key` |
| Write, review, then Send | [Message entry](examples/message.html) | `text` |
| Previous / Next slide | [Presentation](examples/presentation.html) | `action` |
| A TV-style remote | [TV remote](examples/remote.html) | `press`, `keydown` |

For message entry, set `mode: 'text'`. Label its action with `submitLabel: 'Search'`
or `'Send'`, and give the destination a useful `name`. The phone displays **Sent**
after your handler completes; return a Promise to await a save or search.

For buttons, supply named `actions`; use `caps: ['buttons']` if you do not need a
keyboard. For a remote control, set `layout: 'remote'` (or `'teletext'`,
`'gamepad'`) and handle `press` and `keydown`; see
[Controls](DEVELOPER_GUIDE.md#controls-remotes-keypads-and-game-buttons). For a PIN,
add `pin: true`; the generated PIN appears beside the QR.

On a shared screen, clear visitor data in the `finish` handler. **Finish** ends
the old pairing and creates a fresh QR after cleanup. **Disconnect** only stops
that phone and leaves the screen session open. The starter includes cleanup for
Finish; the earlier deployment does not implement that lifecycle yet.

Use the [API reference](DEVELOPER_GUIDE.md) when you need more options, lifecycle
methods, or device-level APIs.
