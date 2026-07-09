# Fullscreen keyboard / phantom-navbar repro

A minimal, self-contained reproduction of a Chromium-on-Android bug that shows up in **fullscreen**
(entered via the Fullscreen API) when the top-level container is sized to the **dynamic viewport**
(`100dvh`). Installing as a PWA is **not** required.

## Symptom

When the on-screen keyboard is dismissed (especially with a quick tap), the OS navigation bar
flickers back in "phantom" — it isn't really drawn, but the browser briefly reports a viewport
shrunk *as if* it were there. The `100dvh` container faithfully shrinks with it, so a strip of the
**root canvas** (the page's default background) is exposed along the bottom edge. With a default
white canvas this reads as a bright white bar. Dismissing with a **long press** usually avoids it.

The underlying viewport update is **interaction-gated, not time-based**: `innerHeight`,
`visualViewport`, and `VirtualKeyboard.boundingRect` sit frozen at the stale value for as long as you
wait, then snap only on the next real touch. So no amount of waiting lets the page "catch up".

Two smaller, related quirks the HUD also exposes:

- In `VirtualKeyboard` overlay mode, `geometrychange` fires **one interaction late** — on open it
  still reports the closed geometry; the real height arrives at the next focus change (blur).
- `VirtualKeyboard.boundingRect.height` is measured from the layout-viewport bottom, so it **omits
  the navigation bar** — it is short by the navbar height versus the OS IME window.

## What's in the page

- A `<test-app>` **custom element** (shadow DOM) sized to `100dvh` with an opaque dark background —
  the app container. The **root canvas is painted magenta** so any uncovered strip is unmissable.
- A fixed bottom bar with a text input (the thing that must ride above the keyboard).
- A live **HUD** (top of screen) sampling every frame: `innerHeight`, `visualViewport`,
  `VirtualKeyboard.boundingRect`, `--kbtop`, display-mode, fullscreen element.
- Toggle buttons that show their current state and isolate variables:
  - **Fullscreen (API)** — enter/exit element-fullscreen (the trigger; no install needed).
  - **VK overlaysContent** — flip the VirtualKeyboard overlay mode.
  - **Container height 100dvh / 100lvh** — switch between the dynamic and the **large** viewport.
    On `100lvh` the container always covers the physical screen, so the magenta strip cannot appear —
    this is the "remove the hole" fix.
  - **Bar anchor bottom:0 / top (--kbtop)** — position the bar with `bottom: 0` vs
    `bottom: calc(100% - var(--kbtop))` (top-referenced, immune to the bottom drifting).
  - **iw** — set `interactive-widget` (`resizes-visual` / `resizes-content` / `overlays-content`).

## How to run

Serve over **HTTPS** or `http://localhost` (the Fullscreen API and VirtualKeyboard API need a secure
context), open on an Android phone in a Chromium browser, then:

```sh
# any static server works, e.g.:
python3 -m http.server 8080
```

1. Tap **Enter fullscreen (API)**.
2. Focus the input → keyboard opens.
3. Dismiss with a **short tap** on the card → watch for the magenta strip and the frozen HUD numbers.
4. Compare a **long-press** dismiss (usually clean).
5. Toggle **100lvh** and repeat — the strip should be gone.

## Environment where observed

- Android, Chromium-based browser, fullscreen, 3-button navigation bar.
