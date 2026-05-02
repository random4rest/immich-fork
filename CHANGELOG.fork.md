# Fork Changelog

All custom features and patches added on top of upstream [immich-app/immich](https://github.com/immich-app/immich) live here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Each release section maps to a `personal-vUPSTREAM-N` git tag and the deployed `IMMICH_VERSION` in `immich-app/.env`.

Conventions:
- Group entries under **Added / Changed / Fixed / Removed / Docs**.
- Each bullet links to the relevant `[fork]` commit(s).
- When merging an upstream release, add a "Synced with upstream vX.Y.Z" line under the new release section.

---

## [Unreleased]

Changes on `personal` that have not yet been tagged for deployment.

### Changed

- **`personal` rebased from `upstream/main` onto release tag `v2.7.5`.** This branch was previously based on `upstream/main` (which is unreleased WIP). It is now based on the `v2.7.5` release tag. See `AGENTS.md` §9.1 for why this rule matters, and §9.2 for the spinner-of-death build bug that prompted the rebase.
  - Old `personal` HEAD preserved at git tag **`personal-v2.7.5-1-mainbase`** and at the deployed Docker image `immich-server:personal-v2.7.5-1`.
  - **Deployment caveat:** if your DB has migrations applied by a `main`-based build (e.g. `<ts>-DropAuditTable`), v2.7.5 will refuse to start with `corrupted migrations: previously executed migration <name> is missing`. To deploy this rebased `personal`, either restore the DB from a pre-fork backup, or stay on the existing `personal-v2.7.5-1` image until you do. Verified bit-equivalent to the rebuild from this commit, so the running container is safe to leave as-is.

- **Rotate adapted to `v2.7.5` API surface:**
  - `SyncAssetV2` → `SyncAssetV1` (V2 doesn't exist in v2.7.5; same fields we use)
  - `AssetEditReadyV2` websocket event → `AssetEditReadyV1` (V2 added in main only)
  - `RotateAction.svelte` import paths: `ButtonContextMenu.svelte` → `button-context-menu.svelte`, `MenuOption.svelte` → `menu-option.svelte` (PascalCase rename happened in main only)
  - `thumbnail.svelte` (lowercase) instead of `Thumbnail.svelte` (the rename happened in main); kept v2.7.5's string-typed `asset.duration` handling (`timeToSeconds(asset.duration)` instead of `asset.duration / 1000`).

### Added

- **Web: rotate action in photo viewer** — adds a `RotateAction` component on the asset detail page; rotation propagates through the timeline manager and websocket store so thumbnails and the grid stay in sync. New i18n string for the action label.
  - Files: `web/src/lib/components/timeline/actions/RotateAction.svelte` (new), `web/src/lib/components/assets/thumbnail/thumbnail.svelte`, `web/src/lib/managers/timeline-manager/types.ts`, `web/src/lib/stores/websocket.ts`, `web/src/lib/utils/asset-utils.ts`, `web/src/routes/(user)/photos/[[assetId=id]]/+page.svelte`, `i18n/en.json`
  - Commit: `d77f6335d` (post-rebase; original main-based commit was `ea40dd558`).

### Docs

- **`ARCHITECTURE.md`** — top-level codebase tour (server layers, schema, storage, ML service, web, end-to-end upload flow).
  - Commit: `48d9c0275` (post-rebase; was `f3ad10467`).
- **`docs/docs/developer/architecture.{mdx → md}`** — pure rename; the file contains no JSX so the `.mdx` extension was unnecessary.
  - Commit: `48d9c0275`.
- **`AGENTS.md`** — workflow reference (repo layout, branch model, dev/prod commands, upstream-sync process, build/tag/deploy loop, SSH gotchas).
  - Commit: `4ac7a383d` (was `0c430561c`).
- **`AGENTS.md` §9 "Lessons learned"** — added (a) the rule that `personal` must be based on release tags, (b) the SvelteKit `__sveltekit_<HASH>` mismatch / BUILD_ID gotcha, (c) when to use `--no-cache`. Includes a one-liner sanity check for verifying SSR + client hashes match after a build.
  - Commit: `8680d4d28`.
- **`CHANGELOG.fork.md`** — this file.

### Changed (other)

- **`README.md` — rewritten for the fork.** The upstream Immich README has been replaced with a slim fork-specific landing page (badges, why-this-fork, quick start, upstream-sync section, links to the other fork docs). Upstream README content is no longer carried in this repo; visitors are pointed at [immich-app/immich](https://github.com/immich-app/immich) for the product README. Done deliberately to make merge conflicts on `README.md` near-zero going forward.

### Synced with upstream

- Base: `v2.7.5` (immich release tag).

---

## Template for the next release

When you tag `personal-vX.Y.Z-N`, copy this block under a new heading and fill it in.

```markdown
## [personal-vX.Y.Z-N] — YYYY-MM-DD

Synced with upstream `vX.Y.Z`.

### Added
- ...

### Changed
- ...

### Fixed
- ...

### Removed
- ...

### Docs
- ...

### Upstream merge notes
- Conflicts resolved in: `path/to/file` — kept fork's <thing> because <reason>.
- New upstream feature affecting fork: <description>.
```

---

## How to find every fork commit

```bash
# All commits prefixed [fork] on the personal branch
git log --grep '^\[fork\]' --oneline personal

# Same, but only commits since the last upstream merge
git log --grep '^\[fork\]' --oneline upstream/main..personal

# Diff stat of fork patches vs upstream main
git diff --stat upstream/main..personal
```

Use those when writing the next release entry.
