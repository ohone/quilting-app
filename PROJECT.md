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

- App examined thoroughly (see `docs/features/*` for the architecture notes each feature
  needs). Nothing built yet.
- Feature docs written:
  - `docs/features/design-folders.md` (built first — includes all shared infrastructure)
  - `docs/features/composites.md` (second — builds on the folders infrastructure)

## Decisions

- Work is split into two **sequential** worker builds (folders first, then composites):
  both touch the same shared files (`app.js`, `db.js`, `backup.js`, `sw.js`, `styles.css`,
  `icons.js`), so parallel workers would collide.
- Deploy happens only after both features are built and verified (per `app/docs/dev-flow.md`),
  by the orchestrator, via push to `main` (GitHub Action builds + `wrangler deploy`).
- The outer repo (`quilting-app/`) is the project folder; `app/` inside it is the cloned
  upstream repo with its own git. Deploy = push `app/` to `origin main`.
