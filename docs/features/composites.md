# Feature: composites (new canvas mode + composite folders)

User's words: *"a new canvas mode / view, that allows me to composite different blocks
together. Those should have folders too."*

Build **after** `docs/features/design-folders.md` is complete and committed — this feature
reuses its `records` store, folder UI and patterns. Do not start it in the same working tree
before folders is done.

## What the user wants

A way to compose several different quilt blocks into one picture — a design wall / medallion
layout — where the blocks are the user's designs (any of them), placed on a canvas with
per-instance placement, scale and rotation, and the result is itself a first-class object
(own thumbnails, own name, filed into folders like designs are).

Think: mixed-block quilts, medallion quilts, samplers, wall hangings — anything that isn't
"N copies of one block".

## Design (keep it minimal and in-house style)

- New record type in the `records` store (created by the folders feature):
  `composite` = `{ id: 'cmp-…', kind: 'composite', name, created, updated, items: [...] }`
  where each item is `{ designId, x, y, sizeIn, rot }`:
  - `designId` — which design (its **block** is drawn; the composite references the design,
    it does not copy it — but snapshot the block at add-time? **Decide while coding**: copying
    the block into the item (self-contained, survives source-design deletion, matches backup
    semantics where designs travel together) is the safer choice; record `block` and `name`
    (the source design's, for the item list) on the item and drop the live link. Prefer the
    self-contained copy.)
  - `x, y` — top-left in **inches** on the composite canvas (floats).
  - `sizeIn` — the block's finished width in inches; height follows the block's aspect.
  - `rot` — 0/90/180/270 (quarter turns like `layoutXform`).
- Composite **folders**: same `folder` record with `kind: 'composites'`, same folder UI
  patterns, a separate "Composites" area (see UI).
- Storage: same `records` store; `composite.folderId` for membership (same single-membership
  model).

## UI

- Home gains a **"Composites" section** (between "Your designs" and "Explore examples"):
  its own grid of composite cards (thumbnail = drawn composite, see below) + "+ New
  composite" button, and the same folder chip row (kind: 'composites') + folder screens
  as designs get. Reuse the folders feature's components; do not fork them — parameterize
  by `kind` where the behaviors are identical, or accept small explicit branches.
- **Composite editor**: a new screen (not a tab in the block editor — the object is
  different, like the block editor vs. home). Structure it exactly like the existing editor
  chrome: topbar (back, name, menu) over a full-bleed canvas (`stage` + fixed canvas like
  `renderEditor` does; reuse `drawMat` for the cutting-mat backdrop so it feels native —
  pass per-inch scaling from the view transform).
  - Canvas: pan/zoom with the existing pinch pattern (S.czoom or a local {k,x,y}).
  - Empty state: centered muted hint on the mat ("Tap + to add a block").
  - **Add block**: "+" button → sheet listing the user's designs (with block thumbnails,
    reusing `thumb(d, 92, 'block')`) → pick one → it lands centered in the current view with
    a sensible default size (e.g. 12 in) → stays selected.
  - **Select / move**: tap an item to select (teal selection outline like the block
    editor's); drag to move (snap to ½ in or so — pick a snap that feels right on the mat,
    matching the block editor's half-grid snapping spirit). Drag with two fingers or pinch
    zooms/pans the canvas instead (follow the existing `cancelDrag` second-finger pattern).
  - **Selected-item controls** (panel below, like the block editor's `.panel`):
    - size stepper (finished width, inches, reusing `stepper`; label in the current unit
      via `U.unit` conversions like the quilt panel does),
    - rotate button (90° steps),
    - flip? (only if free — rot covers most needs; skip if it adds friction),
    - duplicate item,
    - delete item,
    - "bring forward / send back"? z-order = items array order; offer front/back buttons
      only if trivial; otherwise insertion order is fine for v1.
  - Undo/redo: reuse the `snapshot/commit/mutate` pattern over the composite record's
    `items` (same history-by-id map treatment; keep it simple: one history per open
    composite).
  - **Export image** in the composite menu (like `exportImage('block')`): draw at ~2400 px
    and `shareOrDownload`.
- Composites do **not** get cutting lists, quilt sashing, or AI in v1. The menu: Rename,
  Move to folder, Duplicate, Delete, Export image.

## Drawing

- `drawComposite(ctx, comp, x, y, sc, {selected})` — new pure function in `render.js`:
  for each item, compute its pixel rect from (x, y, sizeIn, block aspect, rot), then reuse
  the **tile approach** from `drawQuilt` (`layoutXform`-style quarter-turn transforms over a
  cached `drawBlock` tile) or simply `drawBlock` directly per item (composite item counts
  are small; direct draw with a per-item save/translate/rotate/drawBlock/restore is fine
  and simpler — measure `Fab.style` pattern alignment like drawBlock's ppi argument: pass
  `sc` as ppi so prints scale in real inches).
  - Selection overlay drawn by the editor on top (same stroke styles as the block
    selection: white + teal dashed).
- Thumbnail: same function with a fit-scale into `thumb()`-sized canvas. Home card shows
  composite extent (W×H in inches, `dims()`).

## Persistence & data safety

- Composite saves through the same debounce chain pattern (a parallel `persist`/`saveSoon`
  keyed on the composite record — or generalize the existing one; **prefer the lightest
  change**: a small `saveSoonRecord(record)` helper for records-store objects).
- Deleting a design does **not** affect composites (items are self-contained copies).
- Backup: extend `backup.js` — export includes records (folders + composites); import plans
  them like designs (id/content/newer-wins semantics, "already here — skipped"); bump
  backup VERSION to 3 (or fold with the folders feature's bump if this lands in the same
  release — coordinate: **the folders feature already bumped to 3; this one bumps to 4
  unless it ships together**). Validate composite items (block shape reuses `validDesign`'s
  block checks factored out — reuse `validPiece`/block grid checks; rot ∈ {0,90,180,270};
  sizeIn > 0).
- `validDesign` gains nothing; add `validComposite` + `validFolder` in backup.js.

## Out of scope

- Mixing arbitrary images/photos, text, or sashing on the composite canvas.
- Cutting lists for composites.
- Nested composites (a composite as an item inside another).
- Server/API changes.

## Done means

- All flows work on device-size screenshots (new composite → add two blocks → move/resize/
  rotate → export image; file into folder; folder view shows it).
- `npm test` passes; add pure-function tests where you factored any (e.g. composite item
  transforms/geometry helpers in a small module with node-test coverage).
- Backups round-trip composites and folders; an old app refuses version 4 backups cleanly
  (READS logic already handles that pattern).
- Service worker ASSETS + index.html updated for any new files (aim for one new file,
  `composite.js`, added to both; keep styles in styles.css).
- WHATS_NEW updated.
- Screenshot script gains a composite walkthrough section (extend `scripts/shoot.mjs`
  minimally — it's the release gate).
- Committed with a clear message. **Do not push.**
