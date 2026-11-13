# quilting-app

Kept current by the project's orchestrator: each new orchestrator session knows only what
is written here and in docs/.

## Goal

Add two features to the owner's quilting app **Tide Stitch** (GitHub `ohone/tide-stitch`,
clone at `app/`, live at https://quilt.eoghan.dev):

1. **Design folders** — users can create folders, file designs into them, and browse a
   Folders view.
2. **Composites** — a new canvas mode/view for compositing different blocks together on one
   canvas; composites get folders too.

Maintain current styling and best practice (tests, screenshots, backup compatibility,
service-worker asset list, WHATS_NEW). Commit and deploy when done.

## State

- Feature docs written (commit `de600c6`):
  - `docs/features/design-folders.md` (built first — includes all shared infrastructure)
  - `docs/features/composites.md` (second — builds on the folders infrastructure)

- **Folders: DONE & committed** on `app/` main — `d0f5a85` (folder CRUD, Folders view, backup v3, DB v2 `records` store), `a704426` (helper cleanup), `840f64f` (hoist `cardOf`, Home folder cards, add-designs button fix, folder-screen layout). Verified: unit tests pass per-file (48–49 tests), `shoot.mjs` screenshot suite clean, full e2e folder flow walked in headless Chromium with no console errors. WHATS_NEW updated. Remaining optional: backup round-trip browser check, a Folders shot in `shots/`.

- **Note:** a duplicate folders worker was started by a second project chat ("Quilting App Design Folders and Composite Canvas"); its leftover WIP (broken `setFolder` call in `app.js`) was stashed and discarded by the composites worker. That session may still exist — stop it from that chat if it resurfaces.

- **Composites: IN PROGRESS** — worker "composites-build" building per `docs/features/composites.md` (started after folders landed; found+stashed the stale WIP above, verified baseline 48 tests pass).

## Decisions

- Work is split into two **sequential** worker builds (folders first, then composites):
  both touch the same shared files (`app.js`, `db.js`, `backup.js`, `sw.js`, `styles.css`,
  `icons.js`), so parallel workers would collide.
- Deploy happens only after both features are built and verified (per `app/docs/dev-flow.md`),
  by the orchestrator, via push to `main` (GitHub Action builds + `wrangler deploy`). Owner
  confirmed: push when done, deploy via Actions.
- Pre-push verification pass (orchestrator): full test suite, build, backup round-trip browser
  check (export → wipe → import → folders/filings return) — the one e2e gap in the folders work.
- The outer repo (`quilting-app/`) is the project folder; `app/` inside it is the cloned
  upstream repo with its own git. Deploy = push `app/` to `origin main`.
