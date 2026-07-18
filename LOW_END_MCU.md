# qrkeyboard — Making it doable on low-end MCUs

Goal: help a developer run a qrkeyboard device on cheap, small microcontrollers —
down to 8-bit AVR-class parts (ATmega328: 2 KB RAM, 32 KB flash) and Wi-Fi
modules that can't do TLS reliably (ESP8266, or an AVR with an ESP-01 modem).

> **Scope.** This is *guidance*, not components. We provide only the QR + events
> API and the hosted phone controller. All device-side code — firmware, any
> bridge, any local server — is yours. Nothing here is something we ship; these
> are patterns you can implement to reach the API from constrained hardware.

## The single real blocker

Everything the device does is cheap **except transport TLS**:
- The frame protocol is tiny (3–5 byte deltas / key events) — trivial to parse.
- The QR is **pushed to the device as a bitmap** (single-socket provisioning,
  built) — the device needs no QR library and no HTTP client; it blits ~106–200
  bytes onto its display.
- The E2E crypto is **ChaCha20-Poly1305**, deliberately chosen because it runs
  fine on 8-bit MCUs (a few hundred bytes of code, no AES hardware needed).

TLS is the outlier: Cloudflare's edge is HTTPS-only and the phone needs a secure
context for the gyro, so the *wire* is `wss://`. AVR can't do TLS at all; ESP8266
can but painfully. So the whole problem reduces to: **get TLS off the low-end
MCU.** Three ways, pick by hardware.

## The key insight that unlocks it

**End-to-end encryption is independent of transport TLS.** If the device does the
ChaCha20 itself (feasible on AVR), the input frames are already authenticated
ciphertext before they hit the wire — so transport TLS becomes *redundant for
input confidentiality*. That lets a low-end device use a **plaintext transport**
without exposing keystrokes: the relay and network see ciphertext either way.
Transport TLS would then only hide metadata (which the relay sees regardless) and
protect the handshake (which the app-layer pairing already authenticates).

This decouples "secure" from "TLS" and is what makes the tiers below possible.

## The tiers (device picks by capability)

### Tier 0 — Direct `wss://` (ESP32-class) — built, secure by default
ESP32/-S3/-C3 do TLS fine. Single-socket provisioning already reduces the
firmware to: open one `wss://`, receive the QR bitmap, blit it, receive frames.
This is the reference path and needs nothing further.

### Tier 1 — Bridge / network coprocessor (recommended for low-end) — keeps full security
The low-end MCU talks a **dead-simple plaintext link** (UART / SPI / plain TCP)
to a cheap companion that does TLS + `wss://` on its behalf:
- The companion is a ~$2 ESP32/ESP-01 you program to speak our provisioning
  socket upstream — often it is *already there* as the device's Wi-Fi module.
- The low-end MCU sees only two things: "here are QR bytes, blit them" and "here
  is an input event." It never touches TLS or WebSockets.
- **Full security preserved** (the bridge holds the TLS leg), full internet
  reach, stable QR. This is the best answer for AVR-class main MCUs. Both halves
  (bridge and main MCU) are your code; we provide the upstream API it speaks.

### Tier 2 — Plaintext-LAN (ESP8266 / AVR+Ethernet), keyboard/d-pad only
For a device on the same network as the phone, with **no gyro** (a QR-navigated
keyboard/menu — teletext, a thermostat, a jukebox), you can bypass our cloud
entirely:
- Your MCU runs a tiny **plain `http` + `ws` server** and serves your own minimal
  controller page + its QR. No TLS, no relay, no cloud — runs on an ESP8266.
- Works because a keyboard/d-pad controller needs **no secure context** (only the
  gyro does); the phone loads `http://<device-lan-ip>` and opens `ws://`.
- Trade-off: LAN-only, no gyro, unencrypted on the local network — fine for
  low-stakes home use; label it "local, unencrypted" on both screens.
- Note: this path does not use our hosted controller (that one needs HTTPS); the
  controller page here is yours. It's a fully self-contained alternative, not a
  qrkeyboard integration.

### Tier 3 — Plaintext transport to a gateway you run (opt-in, low-stakes)
For a low-end MCU that needs internet reach, can't do TLS, and has no bridge, you
can run a small plaintext gateway that fronts our API:
- Your MCU connects **plaintext** (`ws://` / raw TCP) to your gateway (a
  lightweight server you host — *not* Cloudflare, which is TLS-only); the gateway
  does the TLS leg to our relay.
- If your MCU does app-layer ChaCha20 (see the key insight), **input stays
  confidential** despite plaintext transport. If not, it's an explicitly insecure
  low-stakes setup — never for HID/console-type input.

## What the API gives you (so any of the above is possible)

**Live + tested:** single-socket provisioning (QR pushed over one socket,
reboot-reclaim for a stable QR), the plain 0/1 QR grid format, the tiny frame
protocol, ownerToken/phone-join auth. That's the whole surface your device code
targets; Tier 1's bridge speaks exactly this upstream. **We do not provide the
bridge, the self-hosted controller, or the gateway** — those are your code.

## Recommendation

For AVR-class hardware, the **bridge** pattern (Tier 1) is the cleanest: it keeps
full security and internet reach while making the main-MCU code trivial, and the
bridge chip is usually the Wi-Fi module already present. The **plaintext-LAN**
pattern (Tier 2) is the simplest zero-cloud option for same-network keyboard use
(but it's self-contained, not a qrkeyboard integration). Keep confidentiality on
wherever you can (ChaCha20 is nearly free even on 8-bit); reserve plaintext for
genuinely low-stakes, never for HID/console input.
