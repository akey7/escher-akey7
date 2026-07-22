# Debug Log

## 2026-07-21

- CONFIRMED root cause: load-time crash `TypeError: Cannot read properties
  of undefined (reading 'stopPropagation')` in `wheelFn`,
  `src/ZoomContainer.js` (~line 185). `const ev = e.sourceEvent` assumed a
  d3-zoom-wrapped event, but the three listeners (`mousewheel.escher`,
  `DOMMouseScroll.escher`, `wheel.escher`) are bound directly to
  `this.container` via plain d3-selection `.on()` calls, not through a zoom
  behavior — so `e` is already the raw native event and `.sourceEvent` is
  undefined. Verified via `git log -p -L` on the function: prior to commit
  4a9f4bc9 (Sep 2024, "feat: update d3-related with rotate small
  problems") the line read `const ev = event` (pre-v3 d3-selection bare
  `event` pattern); that commit incorrectly changed it to
  `e.sourceEvent`.
- FIX APPLIED: changed `const ev = e.sourceEvent` to `const ev = e` in
  `src/ZoomContainer.js` wheelFn. Surrounding code
  (`ev.stopPropagation()`, `ev.preventDefault()`, `ev.returnValue`,
  `ev.wheelDeltaX`/`ev.deltaX`/`ev.wheelDeltaY`/`ev.deltaY`) all standard
  native WheelEvent properties — consistent with `ev` being the raw
  native event. No other lines in this block touched.
- USER CONFIRMED (manual test): page loads, model JSON loads without the
  stopPropagation crash, mouse-wheel panning works in all directions.
  Scope of this fix considered closed.
- Drag-merge bug: user reports dragging a metabolite node onto another
  same-`bigg_id` node "silently does nothing" — no console output, no
  merge.
- CONFIRMED root cause (verified via source inspection, git history, AND
  live reproduction against the running dev server — not a guess):
  in `getSelectableDrag` (`src/Behavior.js`, ~lines 505-769), there are
  TWO separate `behavior.on('start', ...)` registrations on the same
  d3-drag behavior instance:
    - line ~524: `function (e) {...}` — sets `dragging = true`, arms the
      200ms z-order timeout, AND attaches the `mouseover.combine` /
      `mouseout.combine` listeners to all `.metabolite-circle` elements
      (this is the ENTIRE mechanism that flags a same-`bigg_id` node as
      a valid merge target via the `node-to-combine` class).
    - line ~561: `(e) => { lastX = e.x; lastY = e.y }` — records drag-
      start coordinates for manual displacement tracking used by the
      `drag` handler.
  Both use the bare (un-namespaced) event name `'start'`. d3's event
  dispatcher (underlying d3-drag) keeps only ONE callback per
  un-namespaced type name, so the second `.on('start', ...)` call
  silently REPLACES the first — the combine-setup handler never runs on
  any drag. Result: `.node-to-combine` can never be applied, so
  `nodeToCombineArray.length` is always 0 in the `end` handler, so
  `combineNodesAndDraw` is NEVER invoked. Plain node movement still
  works because the surviving `start` handler + `drag`/`end` handlers
  are unaffected — this matches the user's exact symptom (drag "does
  nothing" specifically for merging; no crash, no console output).
- Introduced by the SAME commit as the wheelFn bug: `git log -p -L` on
  these lines shows commit 4a9f4bc9 (Sep 2024, "feat: update d3-related
  with rotate small problems") added the second `.on('start', ...)`
  call (for lastX/lastY tracking) right next to the pre-existing
  combine-setup `start` handler, clobbering it. Prior to that commit
  there was only one `start` handler.
- Verified live: loaded the built-in E. coli core test map fixture
  (`src/tests/helpers/get_map.js`) in the running dev server via
  `window.builder.load_map(...)`, repositioned two `h_c` metabolite
  node instances near each other, and drove a properly-targeted native
  mousedown -> mouseover -> mousemove -> mouseup sequence directly on
  the DOM circles (bypassing an unrelated screenshot/viewport
  coordinate-scaling quirk in the automated browser tool used for
  testing). Confirmed: (a) plain drag movement updates `map.nodes[...].x/y`
  correctly, but (b) `.node-to-combine` is never applied and
  `combineNodesAndDraw` never runs, even while hovering directly over
  the target node mid-drag.
- FIX APPLIED (user confirmed go-ahead): namespaced the two handlers —
  `.on('start', function (e) {...})` -> `.on('start.combine', ...)`
  (line ~524) and `.on('start', (e) => {...})` ->
  `.on('start.track', ...)` (line ~561) — so neither clobbers the
  other. No other lines touched.
- Re-verified live after the fix (full page reload to pick up the
  change, since webpack HMR could not hot-apply this module and
  required a reload): reloaded the same E. coli core test fixture,
  repositioned the two `h_c` node instances near each other, and
  re-ran the identical mousedown -> mouseover -> mousemove -> mouseup
  sequence directly on the DOM circles. Result this time: dragged node
  `1576625` no longer exists in `map.nodes` (deleted by
  `combineNodesAndDraw`), and the fixed node `1576574`'s
  `connected_segments` now includes BOTH the original segment (410)
  AND the segment reassigned from the dragged node (103) — confirms
  the merge ran end-to-end. No new console errors from the drag/merge
  itself (pre-existing unrelated "Bad scale value" errors appear on
  every page load regardless, not investigated, out of scope).
- USER CONFIRMED (manual test): drag-merge works, undo works, PNG and
  SVG export both work. User has manually committed the fix. Both the
  wheelFn crash and the drag-merge bug are considered CLOSED.
