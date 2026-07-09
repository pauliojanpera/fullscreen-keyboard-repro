# Working in the hosted Claude Code environment

Notes for any future Claude (or human) session working on this repo from
Claude Code on the web. The repo is a single static page (plus a
no-custom-elements twin) served from GitHub Pages; there is no build step,
no package manager, and no server.

## Repo layout

- `index.html` — the repro, using a `<test-app>` custom element + shadow DOM.
- `no-custom-elements.html` — the same repro built from a plain light-DOM
  `<div class="test-app">`, to rule custom elements / shadow DOM in or out.
  Keep the two behaviourally identical; port any change to both.
- `README.md` — what the repro is and how to read it.
- `.github/workflows/deploy-pages.yml` — deploys the static site to Pages.

## The hosted environment

- The container is **ephemeral**: the repo is cloned fresh at session start
  and reclaimed after inactivity. Nothing survives unless committed **and
  pushed**.
- **No `gh` CLI and no raw GitHub API from the shell.** Use the GitHub MCP
  tools (`mcp__github__*`); find them with ToolSearch. Unauthenticated
  `curl` to the API returns nothing useful.
- Outbound HTTPS goes through a proxy; plain `curl` to public URLs (e.g. the
  live Pages site) works for verification.
- Chromium + Playwright are preinstalled. To smoke-test a page headless,
  require Playwright from the global modules and point at the pinned browser:
  `require('/opt/node22/lib/node_modules/playwright')` with
  `chromium.launch({ executablePath: '/opt/pw-browsers/chromium' })`.
  Do not run `playwright install`.
- `sleep`-then-do is blocked in Bash. To wait, use an `until <check>; do
  sleep 2; done` loop, or run the command in the background.

## Branch & deploy model

- Feature work happens on the session's designated `claude/*` branch. Do not
  push to `master` without explicit permission.
- **GitHub Pages deploys only from the default branch `master`.** The
  `github-pages` environment is locked to `master` (default GitHub behaviour
  when Pages is enabled via the UI). A Pages run triggered from a `claude/*`
  branch fails in ~4 s with **no step logs** — that empty-log fast-fail is
  the environment protection rejecting the branch, not a workflow bug.
- To publish: with the user's OK, fast-forward `master` to the feature
  branch (`git checkout master && git merge --ff-only <branch> && git push
  origin master`), then switch back. The push to `master` triggers the
  deploy. Alternatively, add the feature branch under **Settings →
  Environments → github-pages → Deployment branches** to deploy from it.

## One-time setup the token CANNOT do

The default `GITHUB_TOKEN` cannot enable Pages (`Resource not accessible by
integration`). These are owner-only UI steps — surface them, don't retry:

1. Repo must be **public**, or on a paid plan (Pages on a private repo needs
   GitHub Pro / Team / Enterprise).
2. **Settings → Pages → Build and deployment → Source → GitHub Actions.**

Once both are done, trigger the workflow (`mcp__github__actions_run_trigger`
`run_workflow`, or a push to `master`) and confirm success by fetching the
live URLs and checking for HTTP 200 + the expected `<title>`:

- https://pauliojanpera.github.io/fullscreen-keyboard-repro/
- https://pauliojanpera.github.io/fullscreen-keyboard-repro/no-custom-elements.html

## Verifying a deploy

Poll the run via `mcp__github__actions_get` (`get_workflow_run`) until
`status: completed`. If a run fails, `mcp__github__get_job_logs` returns the
step logs — **except** when the job was rejected at the environment gate, in
which case the logs 404 (no steps ran). That 404 is the tell-tale of the
`master`-only branch restriction described above.
