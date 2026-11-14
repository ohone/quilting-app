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

- **Note (duplicate sessions):** two other chats ("Quilting App Design Folders and Composite Canvas" 14d6bee9 and "Status Update Request" f0a556bf) have spawned duplicate workers in this same tree. The duplicate folders WIP was already stashed/discarded. A duplicate composites worker ("composites-build" 0c2d3db5, worker of f0a556bf) was told to STAND DOWN via steer message (2026-11-13); it cannot be stopped from this chat (403 — not our worker). Our own composites worker (198f7dea) was warned to check `git status`/`git diff` before coding and to work only on top of the committed b0b3fe0 state. If "composites-build" resurfaces and makes changes, its chat's orchestrator must stop it.

- **Composites: DONE & committed** — worker stalled mid-build with uncommitted work; orchestrator finished it: wired `drawNow()` to `drawCompositeNow()` (the composite editor's `draw()` was a no-op — `drawNow` early-returns for non-editor screens), reset pointer/pinch state on composite editor open, fixed `boot()`/import-reload to load **both** folder kinds (composites folders were lost on reload), updated WHATS_NEW. Commit `5ce292e`.

- **Verification (both features, final orchestrator pass):** 59 unit tests pass across 10 files; `shoot.mjs` + `shoot.mjs --se` exit 0, no console errors (needs `LD_LIBRARY_PATH=/tmp/pwlibs/usr/lib/x86_64-linux-gnu` on this host); `scripts/verify-composites.mjs` 11/11 PASS (full E2E composite flow); `scripts/verify-backup.mjs` 6/6 PASS (commit `8945e2d` — export → wipe IndexedDB → import → design/folder/composite and both folder memberships return exactly; note: the import sheet has a title "Import backup" that substring-matches `text=Import` — use `button:has-text("Import")` in selectors).

- **Pushed to `ohone/tide-stitch` main** (`4d2443a..5ce292e`). **DEPLOY FAILED:** GitHub Actions run 43496643229 — the repo has **zero Actions secrets**; `CLOUDFLARE_API_TOKEN`/`CLOUDFLARE_ACCOUNT_ID` are unset, so `npx wrangler deploy` exits 1. No Cloudflare credentials exist on this VM either. **Blocked on the owner** to add the two secrets to the repo (Settings → Secrets → Actions) and re-run the deploy workflow (or set them and tell us to re-run). Until then https://quilt.eoghan.dev serves the old version.

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
