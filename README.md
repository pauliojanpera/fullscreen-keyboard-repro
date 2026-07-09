# Fullscreen keyboard / phantom-navbar repro

A minimal, self-contained reproduction of a Chromium-on-Android bug that shows up in **fullscreen**
(entered via the Fullscreen API) when the top-level container is sized to the **dynamic viewport**
(`100dvh`). Installing as a PWA is **not** required.

## Live demo (test on a phone)

Published to GitHub Pages over HTTPS — open on an Android phone in a Chromium browser:

<https://pauliojanpera.github.io/fullscreen-keyboard-repro/>

The page is a plain `<div>` in the light DOM. (An earlier custom-element / shadow-DOM build behaved
identically, ruling custom elements out as a factor in the bug, so only the plain page is kept.)

The site is deployed by the `Deploy to GitHub Pages` workflow (`.github/workflows/deploy-pages.yml`)
on every push to `master`. Pages source is **GitHub Actions** (Settings → Pages), and the repo is
public so Pages is available on the free plan.

## Symptom

When the on-screen keyboard is dismissed (especially with a quick tap), the browser reports a viewport
shrunk by the height of the OS navigation bar *as if* the bar were there — a "phantom" navbar that
isn't actually drawn. This is **not a flicker**: the shrink stays put indefinitely, and the exposed
area does **not** recover on its own — it sits there until the next touch (see below). The `100dvh`
container faithfully shrinks with the stale value, so a strip of the **root canvas** (the page's
default background) is exposed along the bottom edge and stays exposed. With a default white canvas
this reads as a bright white bar. Dismissing with a **long press** usually avoids it.

The underlying viewport update is **interaction-gated, not time-based**: `innerHeight`,
`visualViewport`, and `VirtualKeyboard.boundingRect` sit frozen at the stale value for as long as you
wait, then snap only on the next real touch. So no amount of waiting lets the page "catch up".

### `interactive-widget` mode matters

The magenta strip appears in **`resizes-visual`** and **`overlays-content`**, but **not** in
**`resizes-content`**. In `resizes-content` the keyboard resizes the layout viewport, so the `100dvh`
container is laid out against the actually-available area and never exposes the canvas; in the other
two the layout viewport stays "full" and the stale phantom-navbar shrink is what leaks the strip.

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

**Re-FS on blur** is the cleaner form of the same idea: it re-requests fullscreen straight from the
field's `blur` — no outside tap and no timer. `blur` fires on time (it is *not* interaction-gated
the way the VK geometry is), so the only constraint is authorisation. Since `requestFullscreen` needs
transient activation, the handler fires it only while `navigator.userActivation.isActive` — true when
the blur was caused by a gesture (tap elsewhere, `Tab`, the **done** key) and false otherwise, so it
succeeds exactly when it can and no-ops silently when it can't (e.g. a dismissal via the keyboard's
own hide button or the system Back gesture, which deliver no activation).

So fullscreen is only ever active while the keyboard is closed, and the phantom-navbar shrink never
coincides with `fullscreen + 100dvh` — which is exactly why the strip stays away. This targets the
actual trigger (fullscreen and the keyboard being up at the same time) instead of papering over the
symptom, so it is a genuine mitigation.

## The same bug, a different moving part: omnibox vs. navbar

The "viewport stays stale until you tap" behaviour is already known — but every public write-up pins
it on the **omnibox / top controls**: Chrome's own URL bar and toolbar that hide and show as you
scroll. In an ordinary tab those are the only browser chrome that moves the layout viewport, so that
is where the bug was found and named. The reported cause is the dynamic URL bar — `window.innerHeight`
reports a height that no longer matches what is on screen (typically the URL-bar-hidden "full" value)
and **does not correct until a tap forces a reflow** that re-reads the top-controls state
([w3tutorials write-up][w3t], Chromium [*Top Controls change layout viewport height*][tc],
[Next.js #63724][next]). The trigger is the omnibox; the update is interaction-gated.

This repro reproduces the *same* failure with the omnibox taken out of the picture. **Fullscreen has
no top controls** — no URL bar, no toolbar, nothing at the top edge left to move. Yet dismissing the
keyboard still produces a stale shrink of **exactly one navigation-bar height**, and it still refuses
to recover until the next touch. The only chrome that can account for a navbar-height shrink here is
the **bottom OS navigation bar** returning in "phantom".

| Aspect | Known reports (ordinary tab) | This repro (fullscreen) |
|---|---|---|
| Which chrome moves | Omnibox / top controls (top edge) | OS navigation bar (bottom edge) |
| Its state | Hides/shows on scroll | Hidden by fullscreen; returns as a phantom |
| `innerHeight` / `dvh` stale until next touch | Yes | Yes |
| Surface symptom | Misplaced fixed elements, wrong `100vh` | `100dvh` container leaves a canvas strip at the bottom |
| Escape | `visualViewport` API / `dvh` instead of `vh` | `100lvh`, or never overlap fullscreen + keyboard |

So this is **not a second, unrelated bug**: it is the same interaction-gated viewport update, but the
moving part has shifted from the omnibox to the navbar. Fullscreen is what makes that provable — by
removing the omnibox it rules out the usual suspect and leaves the navigation bar as an independent
trigger of the same class of bug. The omnibox variant is well documented; the **navbar framing is the
part not separately reported**, and it is exactly what fullscreen isolates.

[w3t]: https://www.w3tutorials.net/blog/why-is-window-innerheight-incorrect-until-i-tap-chrome-android/
[tc]: https://issues.chromium.org/issues/40391415
[next]: https://github.com/vercel/next.js/discussions/63724

## What's in the page

- A plain `<div>` container sized to `100dvh` with an opaque dark background — the app container. The
  **root canvas is painted magenta** so any uncovered strip is unmissable.
- A fixed bottom bar with a text input (the thing that must ride above the keyboard).
- A live **HUD** (top of screen) sampling every frame: `innerHeight`, `visualViewport`,
  `VirtualKeyboard.boundingRect`, `--kbtop`, `navigator.userActivation` (`isActive` /
  `hasBeenActive`), display-mode, fullscreen element.
- Toggle buttons that show their current state and isolate variables:
  - **Fullscreen (API)** — enter/exit element-fullscreen (the trigger; no install needed).
  - **VK overlaysContent** — flip the VirtualKeyboard overlay mode.
  - **Container height 100dvh / 100lvh** — switch between the dynamic and the **large** viewport.
    On `100lvh` the container always covers the physical screen, so the magenta strip cannot appear —
    this is the "remove the hole" fix.
  - **Bar anchor bottom:0 / top (--kbtop)** — position the bar with `bottom: 0` vs
    `bottom: calc(100% - var(--kbtop))` (top-referenced, immune to the bottom drifting).
  - **Exit FS on focus** — leave fullscreen when the field is focused (so the keyboard opens outside
    fullscreen). Pair it with a re-FS toggle to run the exit/re-enter cycle above.
  - **Re-FS on blur** — re-request fullscreen straight from the field's `blur`, gated on live
    transient activation.
  - **Re-FS on outside-tap** — re-request fullscreen after a delay when an outside tap dismisses the
    keyboard.
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
