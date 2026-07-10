# Working in this repo

Commit and push to **`master`**. GitHub Pages deploys only from `master`, so
that is the only branch whose results the user can actually see on the live
site. Work done on a `claude/*` branch is invisible until it reaches `master`.

Live URL:

- https://pauliojanpera.github.io/fullscreen-keyboard-repro/

## Change only what was asked

Do exactly what was requested — nothing adjacent, nothing "while I'm here". No unrequested refactors,
reorderings, extra event listeners, added links, or "improvements" bundled into an unrelated edit.

Existing code here is deliberate: this page is a bug repro, and details that look like oversights are
often the point. Before touching anything that was not mentioned, check `git log` — e.g. the Workaround
checkbox's text is intentionally *not* a `<label>`, because the page body doubles as the
keyboard-dismiss tap target and a tappable label would toggle the option by accident.

If something else looks wrong, say so in prose and let the user decide.
