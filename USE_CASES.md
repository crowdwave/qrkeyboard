# qrkeyboard — Use Cases

The primitive: **scan a QR on any screen → your phone becomes an input device
for it (gyro pointer, keyboard, buttons, multi-user), with no app install.** We
provide the mechanism (controller + relay + pairing); integrators build the
screen. Everything below is that one primitive dressed for a context.

Each entry notes: who the *screen* is, what input model, connection path
(LAN-direct / relay / either), single vs multi-user, and the security-sensitive
angle (informs the QR-exposure guidance below).

> ⚠️ **Public-QR flag.** Use cases whose QR is
> displayed where it can be *photographed, streamed, or physically swapped* carry
> two risks that are inherent, not bugs: a captured QR yields the E2E key for the
> session's life, and a swapped sticker (quishing) defeats "sight of the
> screen". These are flagged **[PUBLIC-QR]** below and require: short
> TTL + per-join rotation, the ticketed ECDH mode for confidential input, screen
> name/nonce confirmation in the controller, and tamper-evident installs.
> Use cases with a private, in-hand QR do not carry these.

---

## A. Screens without keyboards (the core problem)

1. **Smart TV / set-top / media centre** — screen = TV app; keyboard + gyro
   pointer replace an awful remote. Single-user, LAN-direct ideal.
   *Sensitive:* people type streaming-service passwords → keystroke
   confidentiality is paramount.
2. **Digital signage / menu boards** **[PUBLIC-QR]** — screen = the sign; staff
   update text from a phone. Single or few trusted users, relay (often not
   co-located). *Sensitive:* the pairing QR is visible to the public and
   physically reachable for swapping — use ticketed rotation; never show a
   confidential-input QR on the sign itself.
3. **Kiosks / self-service terminals** **[PUBLIC-QR]** — screen = kiosk; visitor
   uses own phone instead of a grubby touchscreen. Single-user, ephemeral, LAN
   or relay. *Sensitive:* unattended → prime quishing target (tamper-evidence
   needed); session must not leak to the *next* user; no persistence.
4. **Conference / event display** **[PUBLIC-QR]** — screen = stage or booth
   screen; presenter or attendees drive it. Single or multi, relay. A QR on a
   projected/streamed screen is broadcast to everyone — rotate per join.
5. **Hotel-room TV** — guest pairs, controls, and pairing must fully expire at
   checkout. *Sensitive:* credential lifetime vs room turnover.

## B. Touchless / hygiene

6. **Hospital / cleanroom displays** — own-phone input avoids shared surfaces.
   *Sensitive:* clinical context, but we hold no PHI (E2E, no persistence) —
   that's a feature to state explicitly.
7. **Food-service POS / order screens** — staff or customer input by phone.
8. **Public information terminals** (libraries, transit, government)
   **[PUBLIC-QR]** — accessibility + hygiene. *Sensitive:* untrusted public;
   unattended quishing target; abuse of the pipe; the phone as a
   personalized assistive device.

## C. Presentation & AV / creative

9. **Slide clicker + laser-pointer gyro cursor** — screen = presentation app.
   Single-user, LAN-direct (latency matters). The flagship gyro demo.
10. **OBS / stream-deck / production control** — screen = control surface;
    buttons + sliders via layouts-as-data.
11. **Projection mapping / art installations** — gyro + multi-cursor; possibly
    OSC bridge downstream (integrator's side).
12. **Live-event audience participation** — screen = venue display; N phones;
    reactions/pointers. Multi-user, relay, *ticketed* pairing.

## D. Makers & embedded

13. **ESP32 / Raspberry Pi projects** needing occasional input without a
    dedicated keyboard — screen = the device. LAN-direct.
14. **Headless-server rescue input** — the flashable USB-HID dongle: plug into
    any host, scan, type. *Sensitive:* this injects OS-level input into a
    trusted machine → **highest-consequence integration**; a QR-observer
    takeover here means arbitrary keystrokes at a console. Keep the QR
    private (in-hand, never on a shared/streamed screen); prefer LAN-direct.
15. **3D printer / CNC / appliance consoles** — screen = machine UI.
16. **Robot / gimbal / RC teleoperation** — gyro as an intuitive controller.
    LAN-direct (latency + safety). *Sensitive:* input drives physical motion;
    loss-of-signal behavior matters (the SETTLE/keepalive design).

## E. Games

17. **Casual motion games in any browser tab** — screen = web game; gyro/buttons.
18. **Couch / party gaming, arcade cabinets** — N phones as controllers.
    Multi-user, LAN-direct, open pairing.
19. **Jackbox-style quiz / trivia** — screen = host display; N phones buzz/answer
    via structured layouts. Multi-user, relay, ticketed.

## F. Multi-user / shared-screen (mechanism = N phones, one screen)

20. **Party / wedding guestbook wall** — N guests post to a shared display.
    Open pairing (physical presence is the gate).
21. **Classroom / lecture response board** — students answer/annotate. Ticketed
    or roster-gated (integrator's app).
22. **Collaborative whiteboard / brainstorming wall** — multi-cursor + text.
23. **Public "crowd input" screens** (times-square-style) **[PUBLIC-QR]** — the
    highest-risk multi-user case; *ticketed*, rotating QR, integrator
    moderation. The screen broadcasts its own QR — rotation is mandatory,
    and content shown is the integrator's responsibility (mechanism-only).

## G. Accessibility

24. **Own-phone as input for public terminals** — a device already tuned to the
    user's abilities (font size, switch access, voice) drives an untailored
    screen. A genuine accessibility win, not a side effect.
25. **Gyro pointing for users who struggle with trackpads/mice** — coarse motor
    control mapped to an adaptive-gain pointer.

---

## Cross-cutting dimensions

- **QR exposure**: private (in-hand device, hotel room) vs public/broadcast
  (signage, TV on a stream, times-square screen). Public ⇒ pairing-token and
  room-key exposure is the dominant risk.
- **Consequence of input**: cosmetic (a guestbook) vs OS-level (HID dongle,
  #14) vs physical motion (robot, #16). Higher consequence ⇒ stricter pairing
  and loss-of-signal semantics.
- **Trust among users**: single trusted owner vs cooperating group vs hostile
  public. Drives open vs ticketed pairing and rate/kick needs.
- **Persistence expectation**: ephemeral (kiosk, hotel) vs durable pairing
  (personal TV, maker device). Drives credential TTL.
- **Connection path**: LAN-direct (lowest latency, offline-capable) vs relay
  (universal) — changes the attack surface (LAN sniffing vs relay operator).
- **Who hosts the screen**: our hosted screen vs integrator page vs
  hardware. Changes where plaintext lives and who is responsible for content.
