# Bug report — draft for the Brave issue tracker

Paste-ready for <https://github.com/brave/brave-browser/issues/new/choose>. Fill in the **`<…>`**
placeholders (device, versions) before filing. This is a **functional** viewport bug — not a security
report.

Chrome on the same device does not reproduce it, so this is filed against Brave rather than Chromium.
Before filing: search the tracker and, if a matching issue exists, comment instead of opening a
duplicate. The closest Chromium issues are [40391415 — *Top Controls change layout viewport
height*](https://issues.chromium.org/issues/40391415) and [40924170 — *Popup soft keyboard sizing bug
in Chrome on Android*](https://issues.chromium.org/issues/40924170); both concern the omnibox rather
than the navigation bar (see “Relationship”).

---

## Title

Fullscreen + `100dvh`: dismissing the on-screen keyboard leaves a stale, navbar-height viewport shrink that persists until the next touch (Chrome unaffected)

## Summary

In element-fullscreen on Android, dismissing the virtual keyboard shrinks the reported viewport by
exactly one navigation-bar height and **keeps it shrunk until the next touch**. A top-level container
sized to `100dvh` follows the stale value, exposing a persistent strip of the page canvas along the
bottom edge.

While the keyboard is up, `navigator.virtualKeyboard.boundingRect.height` stays **0**; the real height
is reported only at dismissal. Chrome reports it as soon as the keyboard is up, and shows no strip.

## Environment

- Device: `<e.g. Pixel 7>`
- OS: `<e.g. Android 16>`
- Browser + version: **Brave** `<e.g. 1.71.x (Chromium 141.0.xxxx.xx)>`
- Control: **Chrome** `<e.g. 141.0.xxxx.xx>` on the same device — **not affected**
- Navigation mode: **3-button navigation bar** (`<confirm; gesture nav may differ>`)
- Not a PWA install — reproduced in an ordinary tab via the Fullscreen API.

## Live reproduction

<https://pauliojanpera.github.io/fullscreen-keyboard-repro/>

Minimal, self-contained page (single HTML file, source:
<https://github.com/pauliojanpera/fullscreen-keyboard-repro>). In fullscreen — the repro condition —
the root canvas turns dark magenta, so any uncovered strip is unmistakable (while windowed it matches
the bottom bar and an ordinary strip blends into the UI). A one-line readout shows `window.innerHeight`
sitting frozen at the stale value until the next touch, and `geometryWithheld`, which latches `true`
once the browser has been caught leaving `boundingRect.height` at 0 for a whole focused period. A
**Workaround** checkbox toggles the mitigation described below.

## Steps to reproduce

1. Open the live URL on the device above.
2. Tap **“Enter fullscreen”**.
3. Tap the text field at the bottom → the on-screen keyboard opens.
4. Dismiss the keyboard with a tap on the page body.
5. Repeat in Chrome as a control.

## Expected

After the keyboard is dismissed, the reported viewport returns to the full fullscreen height and the
`100dvh` container covers the screen — no canvas strip. This is what Chrome does.

## Actual

- `boundingRect.height` reads **0** for the whole time the keyboard is up; `geometryWithheld` latches.
- A **magenta strip one navigation-bar tall** appears along the bottom and **stays there
  indefinitely**. It does not recover on its own.
- `innerHeight`, `visualViewport.height` and `VirtualKeyboard.boundingRect` remain frozen at the
  stale (shrunk) value in the HUD for as long as you wait.
- The very next real touch anywhere snaps all of them to the correct values and the strip disappears.

Entering the stale state looks duration-sensitive: dismissing with a **long press** instead of a tap
seems to avoid the strip. Leaving it is not — the recovery is **interaction-gated, not time-based**,
and no amount of waiting lets the page catch up.

## Root cause, as observed

Two facts, in the order they can be measured:

1. The keyboard's geometry is reported **one interaction late** — zero while it is open, real at
   dismissal.
2. The reported viewport metrics then **stay stale after the keyboard hides and only update on the
   next touch.**

Everything visible follows from the second — `100dvh` faithfully renders the stale height, the
“phantom navbar” is the missing navbar-height difference, and fullscreen merely makes it visible as a
canvas leak.

## Relationship to existing reports

Public write-ups of “`innerHeight` is wrong until you tap” attribute it to the **omnibox / top
controls** (the URL bar / toolbar that hide and show on scroll) — e.g. issue 40391415. Those are
reported against Chrome. This report has the omnibox removed (fullscreen has no top controls), a stale
shrink of exactly navbar height, and a Chrome that behaves correctly. The navigation bar is therefore
implicated as a separate trigger, and whatever Brave does differently from Chrome as the cause.

| | Known reports (ordinary tab, Chrome) | This report (fullscreen, Brave) |
|---|---|---|
| Moving chrome | Omnibox / top controls (top) | OS navigation bar (bottom) |
| Stale `innerHeight`/`dvh` until next touch | Yes | Yes |
| Surface symptom | Misplaced fixed elements, wrong `100vh` | `100dvh` leaves a persistent canvas strip |

## `interactive-widget` mode dependence

Observed across `interactive-widget` modes:

- The strip appears in **`resizes-visual`** (default) and **`overlays-content`**.
- It does **not** appear in **`resizes-content`** — but there the fixed `bottom: 0` bar instead sits
  about a navbar height too low, so the keyboard overlaps it by the same amount. The navbar-height
  error is present in every mode; only its outward form differs.

## Additional observations (possibly related)

- `VirtualKeyboard.boundingRect.height` is measured from the layout-viewport bottom and **omits the
  navigation bar**, so it is short by the navbar height versus the OS IME window.
- Outside fullscreen, with `resizes-visual`, focusing the field makes it pop up from under the keyboard
  with a visible delay — the same interaction-gated lag, without the phantom navbar on top.

## Workarounds found (not fixes)

- Size the container to **`100lvh`** (large viewport) instead of `100dvh`: it always covers the
  physical screen, so the stale shrink cannot expose the canvas. Pure CSS, so it applies everywhere the
  bug does.
- Never have fullscreen and the keyboard active at once: exit fullscreen when the field is focused and
  re-request it from the field’s `blur` (authorised by the dismiss gesture’s transient activation).
  **Only available when fullscreen was entered through the Fullscreen API** — see below.

The repro applies the second one only to browsers that fail the geometry test, so Chrome stays
fullscreen throughout. The verdict cannot wait for `blur`, since the keyboard covers the field for the
whole focused period; the page gives the browser 300 ms from focus to report a nonzero height, then
drops fullscreen mid-focus and latches for the session.

### The exit-fullscreen workaround does not exist in an installed `display: fullscreen` PWA

A web app installed from a manifest declaring `"display": "fullscreen"` is fullscreen as a *window
presentation mode*, not as Fullscreen-API state. Consequently:

- `document.fullscreenElement` is `null`, even though the app fills the screen and the bug reproduces.
- `document.exitFullscreen()` operates on the fullscreen element stack; with that stack empty it
  rejects and changes nothing.
- No API exists for leaving a manifest display mode.

So the app cannot drop out of fullscreen for the duration of the keyboard, and the workaround has
nothing to act on. The stale-viewport strip is then **unavoidable** except via `100lvh` — in precisely
the configuration a shipped fullscreen web app runs in. This raises the practical severity of the bug
beyond what the ordinary-tab reproduction suggests.

The reproduction linked above therefore ships `"display": "standalone"`, so that installing it and
tapping **Enter fullscreen** exercises the same Fullscreen-API path as a tab.

## Disclosure

This reproduction and write-up were **prepared with AI assistance** and **verified on-device by a
human** before filing. The minimal test case is public and self-contained at the links above.
