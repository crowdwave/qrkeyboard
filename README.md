# QRKeyboard

**Put a QR code on your web page. Someone scans it and types on their phone.
Your page receives the text.**

Useful for a TV search box, kiosk, or presentation. You decide what the input does
in your application. QRKeyboard supplies the QR, hosted phone page, pairing, and
encrypted connection between the two browsers.

## Try it before writing code

Open the [hosted demo](https://qrkeyboard.com/#demo) on your computer. Scan the QR
with your phone camera, connect, and type. No download or account is needed.
The demo runs the deployed version; the simplified phone flow and new message /
action / Finish features in this checkout are [awaiting deployment](RELEASE_STATUS.md).

## Build your first version

Follow the [quick start](QUICKSTART.md): copy one complete HTML file, open it,
and scan its QR. No framework, npm package, API key, or backend setup is needed.
The starter uses the existing live keyboard API so you can begin before the new
service release.

The integration is a script, a QR container, and an input handler:

```html
<script src="https://qrkeyboard.com/v1.js"></script>
<div id="qr"></div>
<pre id="text"></pre>
<script>
  const output = document.querySelector('#text');
  const keyboard = QRKeyboard.create({ el: '#qr', name: 'My screen', caps: ['keyboard'] });
  keyboard.on('key', event => {
    if (event.text !== undefined) output.textContent += event.text;
    else if (event.code === 8) output.textContent = [...output.textContent].slice(0, -1).join('');
    else if (event.code === 10) output.textContent += '\n';
  });
  keyboard.on('finish', () => { output.textContent = ''; });
</script>
```

Your page and the phone both use the hosted relay over the internet. The phone
never needs access to your localhost server; different networks are fine. Input
is encrypted between endpoints. You host your application; we host the phone
page and relay.

## Deploy the same file

Upload `index.html` to an HTTPS static host. There is no build step for the starter
and no QRKeyboard service to deploy yourself. [Deployment instructions](DEPLOY.md)
include a drag-and-drop path and a CLI path.

## Choose the interaction after your first success

| Application | Phone experience | Example |
|---|---|---|
| Live field entry | Type and see edits immediately | [Starter](examples/index.html) |
| TV search or kiosk message | Compose, then Search / Send | [Message entry](examples/message.html) |
| Presentation | Previous slide / Next slide | [Presentation](examples/presentation.html) |
| TV, set-top or kiosk navigation | Remote with D-pad, volume, channel, colour keys | [TV remote](examples/remote.html) |

Message entry, named actions and remote controls require the updated hosted
service. Start with the live keyboard example while that release is pending.

## Where to go next

- [Quick start](QUICKSTART.md): from an empty folder to your first working page.
- [Deploy your page](DEPLOY.md): publish it and verify the result.
- [Example guide](examples/README.md): runnable files and how to choose one.
- [API reference](DEVELOPER_GUIDE.md): options, events, controls, delivery, and session cleanup.
- [Use cases](USE_CASES.md): the interaction patterns.
- [Embedded devices](LOW_END_MCU.md): advanced firmware/bridge integration.
- [Release status](RELEASE_STATUS.md): implemented versus deployed behaviour.
