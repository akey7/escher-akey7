# Project Context

This is a personal, local fork/clone of Escher (opencobra/escher), used
solely to produce one static figure (a clean export) of a ~127-reaction
metabolic network for a manuscript. This is NOT a contribution back to
the upstream open-source project.

The only success criterion is: a correctly working local Builder that
lets the user drag-merge shared metabolite nodes between reactions, from
which one static export (SVG/PNG) can be produced. Do not optimize for
upstream compatibility, other users' use cases, or general robustness
beyond that.

# Non-goals (explicitly out of scope unless asked)

- Do not upgrade, downgrade, or otherwise change dependency versions
  (D3, webpack, Node, or anything else) without first stating exactly
  what you'd change and why, and waiting for confirmation.
- Do not refactor, restructure, rename, or "modernize" any code that
  isn't directly implicated in the bug currently being fixed.
- Do not migrate build tooling (webpack config, yarn/npm scripts, test
  framework, linting setup) unless explicitly requested.
- Do not file, draft, or reference upstream GitHub issues or PRs.
- Do not add configuration options, generalized abstractions, or
  defensive code for scenarios outside this specific model/map.
- Do not touch the Python `escher` package / Jupyter integration unless
  a bug is specifically traced there — assume frontend (JS/D3/webpack)
  scope unless told otherwise.

# Git — hard constraint

- Never run any git command that mutates repository or working-tree
  state: no commit, add, push, pull, checkout, restore, reset, stash,
  merge, rebase, branch, or worktree operations, and no equivalent
  actions through other tools (e.g. editing .git internals directly).
- Never create branches, PRs, tags, or worktrees.
- Read-only git commands (status, diff, log, show, blame) are fine for
  understanding current state.
- All commits and version-control decisions are made manually by the
  user. Leave changes unstaged in the working tree for manual review —
  do not stage or commit them "for convenience."

# Debugging workflow

- Root-cause before patching. For any bug, state a specific, falsifiable
  hypothesis about the cause and how you verified it (source inspection,
  resolved dependency versions, reproducing the error) before writing a
  fix. If you're still guessing, say so explicitly rather than
  presenting a guess as a diagnosis.
- Never suppress an error via try/catch, optional chaining, or a null
  check as a substitute for understanding why a value is unexpectedly
  undefined. If defensive code genuinely is the right fix, explain why,
  rather than silently wrapping the failure point to make it disappear.
- When working from a browser stack trace, resolve it to real source
  files via source maps before editing anything. Do not interpret or
  edit minified bundle output. If source maps aren't available or
  aren't resolving cleanly, say so and stop rather than guessing at
  bundled line numbers.
- After any change, state exactly what manual test the user should run
  in the browser to confirm it worked — what to click/drag/scroll, what
  to expect visually, and where relevant, what the exported map JSON
  should look like before vs. after. You cannot see the rendered canvas
  or a drag gesture; never declare a fix successful yourself.
- Keep diffs minimal and localized to the function/module implicated by
  the current hypothesis. Flag if a fix seems to require touching code
  outside that scope, rather than doing it silently.

# Data safety

- Treat the real model/map JSON file(s) as read-only unless explicitly
  asked to edit them. Do not regenerate, reformat, or "clean up" these
  files as a side effect of other work.
- If a small test model is useful for validating a fix in isolation
  (e.g. 3-4 reactions with one shared metabolite) before trying it on
  the full 127-reaction model, create a new file for that purpose —
  never repurpose or overwrite the real model file.

# Environment notes

- Node version: v26.5.0
- Package manager: Yarn (`yarn.lock` present at repo root; no
  `package-lock.json`). Do not run `npm install` in this repo — it can
  generate a competing `package-lock.json` and desync from
  `yarn.lock`. Use `yarn`, `yarn add`, `yarn why`, etc. for all
  dependency inspection and changes.
- Repo identity: `package.json` reports `escher@1.8.2`; git branch
  `develop-akey7`. This is a personal fork/clone, not upstream
  opencobra/escher — do not assume upstream issue trackers, release
  notes, or documentation describe this exact code state.
- Dev server: `yarn start` runs
  `webpack-dev-server --config webpack.dev.js`, serving on port 7621
  (confirmed against the original crash's `localhost:7621/bundle.js`).
- Production build: `yarn build` runs
  `webpack --config webpack.prod.js`.
- Entry points: dev entry is `./dev-server/index.js`
  (`webpack.dev.js`); production entry is an object with multiple
  named entries in `webpack.prod.js` (not fully enumerated this
  session — check that file directly if a build-only issue comes up).
- Bundler: webpack 5.94.0 — a current, well-maintained version. The
  earlier suspicion that stale tooling itself was the problem did not
  pan out and can be deprioritized as a hypothesis going forward.
  `webpack.common.js` sets `devtool: 'source-map'`, so source maps
  should be generated in dev builds — if devtools shows raw
  `bundle.js` line numbers instead of resolved source, check devtools'
  own source-map settings before assuming maps aren't being built.
- Test runner: Vitest (`yarn test`, `yarn coverage`), configured via
  `vite.config.js`. This repo has two independent toolchains: webpack
  builds/serves the actual app; Vite/Vitest exists solely for the test
  suite. Do not treat `vite.config.js` as the app's build config, and
  do not assume the project has migrated off webpack.
- JS transform: `babel-loader` with `@babel/preset-env`
  (`webpack.common.js`). Source files are `.js`/`.jsx` under `src/`.
- D3 dependencies: NOT a single `d3` package — modular submodules
  pinned individually (`d3-selection`, `d3-zoom`, `d3-drag`,
  `d3-brush`, `d3-axis`, `d3-dsv`, `d3-scale`, all at versions
  consistent with the modern D3 v7-era generation). Confirmed via
  `yarn.lock` and `yarn why` that `d3-selection`, `d3-zoom`, and
  `d3-drag` each resolve to a single consistent version — no
  duplicate-resolution issues. One outlier not yet investigated:
  `d3-request` is pinned at `^1.0.6`, a much older, long-discontinued
  package (superseded by `d3-fetch` in current D3) — worth treating as
  a separate landmine if a future bug traces back to data-loading
  code, but out of scope unless that happens.

# Communication style

- Keep status updates terse and factual: what you found, what you
  changed, what to test next, what's still unresolved.
- Flag assumptions explicitly instead of proceeding silently past an
  ambiguous point.
- Maintain a running plain-text log at DEBUG_LOG.md: append hypotheses
  tried, confirmed, and ruled out, with dates. Don't rewrite or delete
  prior entries — this is what preserves context across sessions on a
  codebase you didn't write.

# Known context / confirmed root cause

- Load-time crash: `TypeError: Cannot read properties of undefined
  (reading 'stopPropagation')` inside `wheelFn` in
  `src/ZoomContainer.js` (~line 185), when loading the model, before
  any drag interaction is attempted.
- CONFIRMED root cause (verified via source inspection, not a guess):
  `wheelFn` does `const ev = e.sourceEvent` and then calls
  `ev.stopPropagation()`. But this handler is bound directly to
  `this.container` via plain `d3-selection` `.on('wheel.escher', ...)`
  / `.on('mousewheel.escher', ...)` / `.on('DOMMouseScroll.escher', ...)`
  calls — NOT through a d3-zoom behavior. `.sourceEvent` is a property
  of d3-zoom's own event object; on a native event delivered through a
  plain selection `.on()` binding, `e` IS the native event already, so
  `e.sourceEvent` is undefined.
- RULED OUT: D3 dependency version mismatch. `d3-selection`,
  `d3-zoom`, `d3-drag` all resolve cleanly to a single version
  (3.0.0) with no duplicate/split resolution in yarn.lock. This is a
  source-level logic bug, not a dependency-version bug.
- CONFIRMED FIX (via git history, not a guess): `git log -p -L` on this
  function shows that prior to commit 4a9f4bc9 (Sep 2024, commit
  message "feat: update d3-related with rotate small problems"), this
  line read `const ev = event` (using a bare `event` reference — the
  old d3-selection pre-v3 pattern). That commit changed it to
  `const ev = e.sourceEvent`, incorrectly assuming these listeners
  receive a d3-zoom-wrapped event (where `.sourceEvent` holds the
  underlying native event). `.sourceEvent` only exists on events
  dispatched through an actual zoom behavior's callbacks
  (`'zoom'`/`'start'`/`'end'`) — these three listeners are bound
  directly to `this.container` via plain `d3-selection` `.on()` calls,
  not through a zoom behavior, so `e` is already the raw native event.
  THE FIX: use `e` directly in place of `e.sourceEvent` — i.e. restore
  the pre-2024 semantics (the event itself), just with the modern
  argument-passing convention instead of the old import.
- Scope constraint for this specific fix: change only this one
  assignment in `wheelFn`. Do not touch the three `.on(...)` binding
  calls, do not touch other handlers in this file, and do not "fix"
  other d3-zoom-related code elsewhere as a preventive measure without
  asking first — even though the commit that introduced this bug may
  have made similar mistakes elsewhere, each instance should be found
  and confirmed individually, not patched speculatively.
- After applying, manual test sequence: (1) confirm the model loads
  without the stopPropagation crash; (2) confirm mouse-wheel scroll
  over the canvas pans as expected (this is the behavior `wheelFn`
  implements); (3) only then proceed to testing the original
  motivating bug — dragging a metabolite node to merge it between two
  reactions in Builder mode — which has not yet been reached or tested
  at all.
