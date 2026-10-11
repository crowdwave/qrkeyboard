# Run an example

**Quickest way to try:** once the updated service is live, every example here is
hosted. Open one on the screen and scan it with your phone; there is nothing to
download:

- Billboard demos: **https://qrkeyboard.com/examples/billboards/**
- Starter: https://qrkeyboard.com/examples/ · TV remote: https://qrkeyboard.com/examples/remote.html

Start with **[index.html](index.html)**. It is the same complete page as the
[quick start](../QUICKSTART.md): QR on the computer, typing on the phone, text on
the computer. It uses the existing live-key API.

Save the file in an empty folder and open it in your browser. If direct-file
access is blocked, run this in that folder (Python 3):

```bash
python3 -m http.server 8080 --bind 127.0.0.1
```

Open **http://127.0.0.1:8080/** on the computer, then scan its QR on your phone.
The phone opens the hosted phone page; it does not connect to localhost. Keep
both devices online and the receiving page open.

If you already have this repository, run the same command from `examples/`.
Open `/` for the starter, `/message.html` for a message, or `/presentation.html`
for slide buttons. All examples load the SDK from the hosted service, not from
an unpublished local build.

| File | What to try | Release requirement |
|---|---|---|
| [index.html](index.html) | Type, Backspace, Enter | Existing live-key API |
| [web-basic.html](web-basic.html) | Live entry plus an explicit screen Finish button | Updated lifecycle |
| [message.html](message.html) | Compose a message, then Send | Updated message protocol |
| [presentation.html](presentation.html) | PIN, Previous slide, Next slide | Updated named actions |
| [remote.html](remote.html) | D-pad, OK, volume and channel on a TV-style menu | Protocol 3 controls |

## Billboard demos

[billboards/](billboards/index.html) holds big-screen experiences, each a single
self-contained HTML file. Every one plays an attract animation when idle, gives
one visitor a timed turn, and ends an idle turn so the next person gets a fresh
code.

| Demo | Phone controls | Screen |
|---|---|---|
| [Neon Wall](billboards/neon.html) | Live keyboard and colour buttons | Words as flickering neon tubes on a brick wall |
| [Skywriter](billboards/skywriter.html) | Message, then Launch | A rocket bursts into the message over a skyline |
| [Light Trail](billboards/light-trail.html) | Turn left / Turn right | LED light-cycle game against two bots |
| [Departures](billboards/departures.html) | Message, then Post | A split-flap station board |
| [Flow Mural](billboards/mural.html) | Mood buttons | A generative flow-field painting |
| [QRTEXT](billboards/teletext.html) | `teletext` remote preset | A teletext service with pages, Fastext and subpages |

Public screens show what visitors type. A real deployment needs moderation
suited to its audience; these demos have none.

Check [release status](../RELEASE_STATUS.md) before trying the updated features.
To publish an example as your own app, copy it to a public folder as `index.html`
and follow [deployment instructions](../DEPLOY.md). Each page creates its own QR;
there is no QR URL or token to paste into its source.
