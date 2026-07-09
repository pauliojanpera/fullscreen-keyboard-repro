# Bug report — draft for the Chromium issue tracker

Paste-ready for <https://issues.chromium.org/new> (shortcut `crbug.com/new`). Fill in the
**`<…>`** placeholders (device, versions) before filing. This is a **functional** viewport bug — file
it on the normal tracker, **not** the security/VRP pipeline.

Before filing: search the tracker and, if a matching issue exists, star + comment instead of opening a
duplicate. The closest existing issues are [40391415 — *Top Controls change layout viewport height*](https://issues.chromium.org/issues/40391415)
and [40924170 — *Popup soft keyboard sizing bug in Chrome on Android*](https://issues.chromium.org/issues/40924170);
this report is deliberately framed as the **navigation-bar** counterpart to those (see “Relationship”).

---

## Title

Fullscreen + `100dvh`: dismissing the on-screen keyboard leaves a stale, navbar-height viewport shrink that persists until the next touch

## Summary

In element-fullscreen on Android, dismissing the virtual keyboard shrinks the reported viewport by
exactly one navigation-bar height and **keeps it shrunk until the next touch**. A top-level container
sized to `100dvh` follows the stale value, exposing a persistent strip of the page canvas along the
bottom edge. This is the same interaction-gated “viewport is stale until you tap” behaviour already
reported for the **omnibox / top controls**, but here the omnibox is absent (fullscreen), so the
**bottom OS navigation bar** is the moving part.

## Environment

- Device: `<e.g. Pixel 7>`
- OS: `<e.g. Android 16>`
- Browser + version: `<e.g. Chrome 141.0.xxxx.xx>` (Chromium-based)
- Navigation mode: **3-button navigation bar** (`<confirm; gesture nav may differ>`)
- Not a PWA install — reproduced in an ordinary tab via the Fullscreen API.

## Live reproduction

<https://pauliojanpera.github.io/fullscreen-keyboard-repro/>

Minimal, self-contained page (single HTML file, source:
<https://github.com/pauliojanpera/fullscreen-keyboard-repro>). In fullscreen — the repro condition —
the root canvas turns dark magenta, so any uncovered strip is unmistakable (while windowed it matches
the bottom bar and an ordinary strip blends into the UI); a one-line readout shows `window.innerHeight`
sitting frozen at the stale value until the next touch. A **Workaround** checkbox on the page toggles the mitigation
described below.

## Steps to reproduce

1. Open the live URL on the device above.
2. Tap **“Enter fullscreen (API)”**.
3. Tap the text field at the bottom → the on-screen keyboard opens.
4. Dismiss the keyboard with a **short tap** on the card area.
5. Do **not** touch the screen afterwards; watch the bottom edge and the HUD.

## Expected

After the keyboard is dismissed, the reported viewport returns to the full fullscreen height and the
`100dvh` container covers the screen — no canvas strip.

## Actual

- A **magenta strip one navigation-bar tall** appears along the bottom and **stays there
  indefinitely**. It does not recover on its own.
- `innerHeight`, `visualViewport.height` and `VirtualKeyboard.boundingRect` remain frozen at the
  stale (shrunk) value in the HUD for as long as you wait.
- The very next real touch anywhere snaps all of them to the correct values and the strip disappears.

So the update is **interaction-gated, not time-based** — no amount of waiting lets the page catch up.
Dismissing with a **long press** instead of a quick tap usually avoids the strip.

## Root cause, as observed

The single defect is: **the reported viewport metrics stay stale after the keyboard hides and only
update on the next touch.** Everything visible is a consequence of that one thing — `100dvh` faithfully
renders the stale height, the “phantom navbar” is the missing navbar-height difference, and fullscreen
merely makes it visible as a canvas leak.

## Relationship to existing reports

Public write-ups of “`innerHeight` is wrong until you tap” attribute it to the **omnibox / top
controls** (the URL bar / toolbar that hide and show on scroll) — e.g. issue 40391415 and the
community write-ups. This report is the **same interaction-gated update with the omnibox removed**:
fullscreen has no top controls, yet the identical stale shrink still occurs, of exactly navbar height.
Fullscreen therefore isolates the **navigation bar** as an independent trigger of the same class of
bug — the part that does not appear to be separately tracked.

| | Known reports (ordinary tab) | This report (fullscreen) |
|---|---|---|
| Moving chrome | Omnibox / top controls (top) | OS navigation bar (bottom) |
| Stale `innerHeight`/`dvh` until next touch | Yes | Yes |
| Surface symptom | Misplaced fixed elements, wrong `100vh` | `100dvh` leaves a persistent canvas strip |

## `interactive-widget` mode dependence

Observed across `interactive-widget` modes (a fuller earlier build of the repro flipped these live;
see git history):

- The strip appears in **`resizes-visual`** (default) and **`overlays-content`**.
- It does **not** appear in **`resizes-content`** — but there the fixed `bottom: 0` bar instead sits
  about a navbar height too low, so the keyboard overlaps it by the same amount. The navbar-height
  error is present in every mode; only its outward form differs.

## Additional observations (possibly related)

- In VirtualKeyboard overlay mode, `geometrychange` appears to fire **one interaction late** — on open
  it still reports the closed geometry; the real height arrives at the next focus change.
- `VirtualKeyboard.boundingRect.height` is measured from the layout-viewport bottom and **omits the
  navigation bar**, so it is short by the navbar height versus the OS IME window.

## Workarounds found (not fixes)

- Size the container to **`100lvh`** (large viewport) instead of `100dvh`: it always covers the
  physical screen, so the stale shrink cannot expose the canvas.
- Never have fullscreen and the keyboard active at once: exit fullscreen when the field is focused and
  re-request it from the field’s `blur` (authorised by the dismiss gesture’s transient activation).

## Disclosure

This reproduction and write-up were **prepared with AI assistance** and **verified on-device by a
human** before filing. The minimal test case is public and self-contained at the links above.
