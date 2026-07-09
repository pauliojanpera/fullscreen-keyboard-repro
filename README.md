# Fullscreen keyboard / phantom-navbar repro

A minimal, self-contained reproduction of a Chromium-on-Android bug that shows up in **fullscreen**
(entered via the Fullscreen API) when the top-level container is sized to the **dynamic viewport**
(`100dvh`). Installing as a PWA is **not** required.

## Live demo (test on a phone)

Published to GitHub Pages over HTTPS — open on an Android phone in a Chromium browser:

- **Custom-elements build:** <https://pauliojanpera.github.io/fullscreen-keyboard-repro/>
- **No-custom-elements build:** <https://pauliojanpera.github.io/fullscreen-keyboard-repro/no-custom-elements.html>

Both pages cross-link to each other. The two builds are identical in behaviour; the second one uses a
plain `<div>` in the light DOM instead of a `<test-app>` custom element with shadow DOM, so it rules
out custom elements / shadow DOM as a factor in the bug.

The site is deployed by the `Deploy to GitHub Pages` workflow (`.github/workflows/deploy-pages.yml`)
on every push to `master` (and to the deploy branch). **One-time setup** by the repo owner:

1. In **Settings → Pages → Build and deployment**, set the **Source** to **GitHub Actions**.
2. Because this repository is **private**, GitHub Pages requires a paid plan (GitHub Pro / Team /
   Enterprise). On the Free plan, make the repository **public** first.

After that, re-run the workflow (Actions tab → *Deploy to GitHub Pages* → *Run workflow*) or push a
commit, and the URLs above go live.

## Symptom

When the on-screen keyboard is dismissed (especially with a quick tap), the OS navigation bar
flickers back in "phantom" — it isn't really drawn, but the browser briefly reports a viewport
shrunk *as if* it were there. The `100dvh` container faithfully shrinks with it, so a strip of the
**root canvas** (the page's default background) is exposed along the bottom edge. With a default
white canvas this reads as a bright white bar. Dismissing with a **long press** usually avoids it.

The underlying viewport update is **interaction-gated, not time-based**: `innerHeight`,
`visualViewport`, and `VirtualKeyboard.boundingRect` sit frozen at the stale value for as long as you
wait, then snap only on the next real touch. So no amount of waiting lets the page "catch up".

### `interactive-widget` mode matters

The magenta strip appears in **`resizes-visual`** and **`overlays-content`**, but **not** in
**`resizes-content`**. In `resizes-content` the keyboard resizes the layout viewport, so the `100dvh`
container is laid out against the actually-available area and never exposes the canvas; in the other
two the layout viewport stays "full" and the transient phantom-navbar shrink is what leaks the strip.

But `resizes-content` is not a clean escape either: the bottom bar at `bottom: 0` still sits about a
**navbar height too low**, so the keyboard overlaps it by that much. Same offset the phantom navbar
is responsible for — the resized layout viewport's bottom sits below the keyboard's top by the navbar
height. So the navbar-height error is present in *every* mode; only its outward form differs (canvas
strip vs. occluded input).

### The lag is not fullscreen-specific

Even **outside fullscreen** (a plain browser tab) with `resizes-visual`, focusing the field shows it
**pop up from under the keyboard with a visible delay** — the browser's "scroll the focused input
into view" reaction lands late, the same interaction-gated lag. Fullscreen only adds the phantom
navbar on top; the underlying delayed viewport reaction is present in an ordinary tab.

Two smaller, related quirks the HUD also exposes:

- In `VirtualKeyboard` overlay mode, `geometrychange` fires **one interaction late** — on open it
  still reports the closed geometry; the real height arrives at the next focus change (blur).
- `VirtualKeyboard.boundingRect.height` is measured from the layout-viewport bottom, so it **omits
  the navigation bar** — it is short by the navbar height versus the OS IME window.

### An exit-FS / re-FS cycle appears to sidestep the strip

Observed on-device: with **Exit FS on focus: ON** *and* **Re-FS on outside-tap: ON** at a ~**200 ms**
delay, the magenta strip stays away. The cycle keeps fullscreen and the open keyboard from ever
overlapping:

1. Focusing the field **exits fullscreen**, so the keyboard opens in an ordinary (non-fullscreen)
   context where there is no phantom-navbar shrink to leak the canvas.
2. Tapping outside to dismiss **re-requests fullscreen** ~200 ms later (the dismiss tap is the user
   gesture that authorises it).

So fullscreen is only ever active while the keyboard is closed, and the phantom-navbar shrink never
coincides with `fullscreen + 100dvh` — which is exactly why the strip stays away. This targets the
actual trigger (fullscreen and the keyboard being up at the same time) instead of papering over the
symptom, so it is a genuine mitigation.

## What's in the page

- A `<test-app>` **custom element** (shadow DOM) sized to `100dvh` with an opaque dark background —
  the app container. The **root canvas is painted magenta** so any uncovered strip is unmissable.
  (`no-custom-elements.html` is the same page built from a plain `<div>` in the light DOM instead.)
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
