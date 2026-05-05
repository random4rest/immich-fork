# Fork Changelog

All custom features and patches developed in this detached fork of [immich-app/immich](https://github.com/immich-app/immich).

This fork detached from upstream at the `v2.7.5` release tag (preserved as the `forked-from-immich-v2.7.5` annotated tag). New work since then is independent — see [`AGENTS.md`](./AGENTS.md) for the workflow.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Each release section maps to a plain semver git tag (`vX.Y.Z`) and the deployed `IMMICH_VERSION` in `immich-app/.env`.

Conventions:
- Group entries under **Added / Changed / Fixed / Removed / Security / Docs**.
- Each bullet links to the relevant commit(s).
- For the rare upstream cherry-pick (CVE backport), note it under **Security** with the upstream sha.

---

## [Unreleased]

Changes on `main` that have not yet been tagged for deployment.

### Changed

- **Detached from upstream.** This fork no longer tracks `upstream/main` or future Immich release tags. Going forward all features and fixes are developed independently. The detachment point (Immich `v2.7.5`) is preserved as the annotated tag `forked-from-immich-v2.7.5`. Rationale + new workflow in `AGENTS.md` §1 and §9.1.
- **Branch model simplified.** `personal` renamed to `main`; the old upstream-mirror `main` branch deleted. There is now exactly one long-lived branch.
- **`upstream` git remote removed.** Re-add temporarily (`git remote add upstream https://github.com/immich-app/immich.git`) only when you need to look at a specific commit for a security backport — see `AGENTS.md` §4.
- **Tag scheme switched to plain semver** (`vX.Y.Z`). The legacy `personal-vUPSTREAM-N` scheme is retired. The `personal-v2.7.5-1-mainbase` tag is preserved for the pre-rebase history, and `personal-v2.7.5-1` remains as the deployed-image label until the next build.
- **`[fork]` commit prefix dropped.** Every commit in this repo is fork code by definition; the prefix added no information. Existing `[fork]` commits in history are unchanged.

### Earlier in this Unreleased cycle

- **`main` rebased from `upstream/main` onto release tag `v2.7.5`** (then renamed to `main`, see above). Old commits preserved at git tag `personal-v2.7.5-1-mainbase`.
- **Deployment caveat (still applies if you ever rebuild prod):** if your DB has migrations applied by the previous `upstream/main`-based build (e.g. `<ts>-DropAuditTable`), v2.7.5 will refuse to start with `corrupted migrations: previously executed migration <name> is missing`. The currently-deployed image `immich-server:personal-v2.7.5-1` was built from the pre-rebase code and runs against the migrated DB without issue; do NOT rebuild prod until you've either restored the DB from a pre-fork backup or accepted the migration-cleanup procedure in `RUNBOOK.md`.

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

### Detached from upstream

- Base codebase: Immich `v2.7.5` (preserved as `forked-from-immich-v2.7.5` tag). No further upstream merges planned.

---

## Template for the next release

When you tag `vX.Y.Z`, copy this block under a new heading and fill it in.

```markdown
## [vX.Y.Z] — YYYY-MM-DD

### Added
- ...

### Changed
- ...

### Fixed
- ...

### Removed
- ...

### Security
- (rare) Backported upstream `<sha>` — <CVE-id or description>.

### Docs
- ...
```

---

## How to find work since the last release

```bash
# All commits since tag vX.Y.Z
git log --oneline vX.Y.Z..HEAD

# Diff stat
git diff --stat vX.Y.Z..HEAD

# Find every commit since detachment from upstream
git log --oneline forked-from-immich-v2.7.5..HEAD
```
