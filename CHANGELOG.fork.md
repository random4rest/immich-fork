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

### Added

- **Web: rotate action in photo viewer** — adds a `RotateAction` component on the asset detail page; rotation propagates through the timeline manager and websocket store so thumbnails and the grid stay in sync. New i18n string for the action label.
  - Files: `web/src/lib/components/timeline/actions/RotateAction.svelte` (new), `web/src/lib/components/assets/thumbnail/Thumbnail.svelte`, `web/src/lib/managers/timeline-manager/types.ts`, `web/src/lib/stores/websocket.ts`, `web/src/lib/utils/asset-utils.ts`, `web/src/routes/(user)/photos/[[assetId=id]]/+page.svelte`, `i18n/en.json`
  - Commit: `ea40dd558`

### Docs

- **`ARCHITECTURE.md`** — top-level codebase tour (server layers, schema, storage, ML service, web, end-to-end upload flow).
  - Commit: `f3ad10467`
- **`docs/docs/developer/architecture.{mdx → md}`** — pure rename; the file contains no JSX so the `.mdx` extension was unnecessary.
  - Commit: `f3ad10467`
- **`AGENTS.md`** — workflow reference (repo layout, branch model, dev/prod commands, upstream-sync process, build/tag/deploy loop, SSH gotchas).
  - Commit: `0c430561c`
- **`CHANGELOG.fork.md`** — this file.

### Changed

- **`README.md` — rewritten for the fork.** The upstream Immich README has been replaced with a slim fork-specific landing page (badges, why-this-fork, quick start, upstream-sync section, links to the other fork docs). Upstream README content is no longer carried in this repo; visitors are pointed at [immich-app/immich](https://github.com/immich-app/immich) for the product README. Done deliberately to make merge conflicts on `README.md` near-zero going forward.

### Synced with upstream

- `b55466479` — `chore!: duration in milliseconds (#28003)` (initial baseline; no merge required yet).

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
