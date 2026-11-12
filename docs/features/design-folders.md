# Feature: design folders

User's words: *"I want users to be able to create folders of created designs. Should be able
to add to folder, and a new folders view."*

This document is **the build order for the folders feature and for all shared infrastructure
the composites feature also needs** (records store, folder UI, migrate-blob pattern). Build it
first and completely; `docs/features/composites.md` builds on top.

## What exists (read these first)

The app is at `app/` (GitHub `ohone/tide-stitch`, branch `main`). Everything about the dev
loop — test command, local server, Playwright screenshot script, release checklist — is in
`app/docs/dev-flow.md`. **Read it before writing any code.** Key facts:

- Vanilla ES modules in `app/public/`, no build step; `scripts/build.sh` stamps the version
  into `sw.js`/`index.html` for deploy. The service worker precaches a hardcoded `ASSETS`
  list — **any new public/ file must be added to it**.
- `app/public/db.js` opens IndexedDB `tide-stitch` at VERSION **1**; `onupgradeneeded` creates
  stores unconditionally (no upgrade path — this feature must add one; see Data model).
- `app/public/app.js` is the orchestrator (state object `S`, `renderHome()`, `renderEditor()`,
  `designMenu()`, save system: `persistNow`/`saveSoon`/`saveChains`, `S.designs` array).
- `app/public/ui.js`: `h()` hyperscript, `openSheet`, `actionSheet`, `inputSheet`, `toast`,
  `stepper`. `app/public/icons.js`: `icon(name, size)` — add new icons to its `PATHS` dict.
- Backup format: `app/public/backup.js` (VERSION 2, `READS = [1, 2]`, strict `validDesign`).
- Theme: cutting-mat glass in the second half of `app/public/styles.css` (`.card`, `.btn`,
  `.sheet`, `.seg`, `--cond` Barlow Semi Condensed, ruler yellow `#F2CF3D` selected accent).
  `app/public/home-storage.css` has Home layout rules. Match existing class conventions
  exactly; reuse existing classes wherever possible.
- `app/public/sw.js` `ASSETS` — add every new file.
- Home structure today: "Your designs" section (search + sort + `.cards` grid), then
  "Explore examples" rows. `renderHome()` in app.js.
- UX house rules visible in the code: confirm-before-destructive, toasts for quick feedback,
  sheets for anything with choices, 44px touch targets, action sheets list actions as
  buttons, designs persist via the debounced save system (never `db.put` directly from UI
  code — but folders are small records; a plain `db.put('records', …)` like the settings/
  meta writes is fine for them).

## Goal

Users can group their designs into named folders:

1. **Create folders** — from the Home screen ("+ New folder" affordance) and while filing a
   design.
2. **Add designs to folders** — from a design's "⋯" menu (Move to folder…), and when viewing
   a folder, an "Add designs" action to file more designs into it. A design can be in **no
   folder or exactly one folder** (single-membership model — simplest mental model; like
   photos albums on iOS: recent picks one).
3. **A Folders view** — a section/screen on Home listing folders, each opening a grid of its
   designs (same cards as "Your designs"), with rename/delete, and the same search/sort as
   the main grid. Folder cards show a stack-style thumbnail (up to 4 designs' thumbnails,
   tilted stack) — build it as a pure function reusing `homeThumb`/`thumb`, clipped with
   CSS transform in a wrapper, consistent with the app's canvas-thumbnail approach.
4. **Unfiling** — "Remove from folder" in the design menu (the design returns to the
   unfiled pool; it is never deleted).

## Data model (and the IndexedDB upgrade)

- New record type: `folder` = `{ id: 'fld-…', kind: 'designs' | 'composites', name, created,
  updated }`. Composites use the **same** store and `kind` field (the composites feature
  adds `kind: 'composites'` folders; this feature only ever creates `kind: 'designs'`).
- **New IndexedDB store `records`** (keyPath `id`) holding folders now (composites records
  later). Because `db.js` has no upgrade path: bump `DB_NAME`? No — bump **VERSION to 2** and
  in `onupgradeneeded` create `records` only when `oldVersion < 2` (guard the existing three
  stores with `oldVersion < 1` so fresh installs still get them). Add `onversionchange` →
  close + banner is out of scope (backlog), but **do** branch on `oldVersion` (the backlog
  explicitly flags unconditional creation as the bug).
- A design's folder membership: **store `folderId` on the design record** (`d.folderId =
  'fld-…'` or absent). Simplest, travels with the design in backups, one source of truth.
- Migration: on boot, `S.folders = await db.getAll('records')` filtered to `kind ===
  'designs'` (plus a `migrate()` that future-proofs: none needed now — `folderId` on designs
  is additive and old code ignores unknown fields safely; backups of designs with `folderId`
  round-trip since import keeps unknown fields? **Check**: `backup.js` `validDesign` doesn't
  strip fields and `plan()` copies the record wholesale — confirm while coding, and extend
  `validDesign` to accept an optional string `folderId` only if it round-trips already; do
  NOT add it to the strict validators if unknown fields already pass through.)
- Deleting a folder: designs inside are unfiled (their `folderId` cleared), never deleted;
  confirm dialog states this.

## UI (all inside Home)

- Home header keeps "Your designs". Add a **"Folders" heading row** in the "Your designs"
  section header area or directly under it: a horizontally-scrolling chip row of folder
  names (reuse `.cards-row` horizontal scroll pattern + a compact chip style) **plus** a
  "+ Folder" chip that creates one (name sheet via `inputSheet`). Empty state: no chips
  row at all until the first folder exists (keep Home uncluttered — the folder row is
  hidden when there are no folders and no folders are in use).
- Tapping a folder chip opens a **folder screen**: same layout language as Home ("Your
  designs in X"), with its own search + sort, cards grid, an "Add designs" button (sheet
  listing unfiled + other-folder designs with checkboxes → file selected into this
  folder), back button to Home (topbar-style header like the editor's, glass), folder
  rename/delete via a "⋯" icon-btn in that header (reuse `designMenu`-style action sheet).
  Cards inside a folder open the editor exactly as on Home.
- Design menu ("⋯" on a card, and in-editor menu): new action **"Move to folder"** →
  action-sheet-style sheet listing folders + "New folder…" + "Remove from folder" (only
  when filed). Selecting files it (sets `folderId`, bumps `updated`, save via existing
  design save system, re-render).
- "Your designs" grid on Home continues to show **all** designs (filed ones can carry a
  small folder-name sub-line in `card-sub` — optional, do it if it fits cleanly: append
  "· In <folder>" is enough).

## Out of scope

- Nested folders, multiple membership, folder reordering, folder colors/icons.
- Any composite-feature UI (next feature doc).
- Server changes, AI changes, cutting list changes.
- The backlog's accessibility overhaul — but do not regress: use `aria-label`s like the
  existing code does.

## Done means

- `npm test` passes (add a small test only if you add pure helpers; UI logic stays in
  app.js conventions).
- `bash scripts/build.sh dev && python3 -m http.server -d dist 8765` + `node
  scripts/shoot.mjs` and `--se`: no page errors, Home looks unchanged with no folders, all
  new flows reachable, screenshots look styled (not unstyled).
- Create folder → file 2 designs → folder view shows them → remove from folder → delete
  folder (designs survive) all work in the browser (verify via screenshots or a quick
  Playwright REPL run).
- Backups: export includes folder records and folderId on designs; import restores both
  (add `folders` to backup export/import; keep VERSION bump to 3 with `READS = [1, 2, 3]` —
  validate folders like designs are validated, best-effort: unknown folder ids on designs
  are tolerated).
- Service worker ASSETS updated if files were added (likely none needed — folders live in
  app.js + a small `folders.js`? Prefer **no new files**: keep everything in app.js +
  styles.css additions, so sw.js and index.html need no changes. If you must add a file,
  add it to ASSETS and index.html).
- WHATS_NEW line in app.js updated.
- Committed with a clear message. **Do not push.**
