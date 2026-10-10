# What QRKeyboard is for

For a first run, use the [quick start](QUICKSTART.md). This guide describes the
updated checkout; check [release status](RELEASE_STATUS.md) before using features
that require the new hosted service.

**Scan the code. Type on your phone. It appears on the screen.**

Start with one of these three patterns. Each requires integration into the
receiving application; QRKeyboard does not control arbitrary desktop apps.

## TV search

The TV app displays a QR beside its search field. The phone says **Typing into:
TV search** and offers a message box with a **Search** button. Use `mode: 'text'`
and `submitLabel: 'Search'`; handle `text` by performing the search. Return its
Promise so the phone confirms delivery after the application accepts the query.

## Kiosk text entry

A visitor scans to enter a name or message using their own keyboard. Give the
kiosk a recognisable destination name. Use `mode: 'text'`, handle `text`, and
clear all visitor data in the `finish` handler. Finish invalidates the previous
QR and creates a fresh session before the next visitor joins.

For restricted input, use PIN pairing and choose where to display the PIN based
on who should have access. Avoid broadcasting confidential-input pairing codes.

## Presentation buttons

The screen offers a QR for the presenter. Configure `caps: ['buttons']` and named
**Previous slide** and **Next slide** actions. The phone shows only those controls,
with no keyboard. Add a PIN if attendees should not be able to advance slides.

## Picking a mode

| Need | Use |
|---|---|
| See edits immediately on another screen | Default live keyboard; `key` handler |
| Compose, review, then send once | `mode: 'text'`; `text` handler |
| Perform a few clear commands | Named `actions`; `action` handler |

These are the supported interaction patterns. Multi-user rooms, local-only
transport, desktop injection, and experimental media/trackpad layouts are outside
the current hosted phone keyboard. See the [quick start](QUICKSTART.md) to build
one of the supported patterns.
