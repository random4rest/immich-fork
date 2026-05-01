# AGENTS.md — Working in this Immich fork

This is a personal fork of [immich-app/immich](https://github.com/immich-app/immich). Read this file before making changes, then read `ARCHITECTURE.md` for the codebase tour.

The goals of this fork are:

1. Self-host Immich on a home PC (LAN + remote access via Tailscale).
2. Add custom features without losing the ability to pull in upstream releases.
3. Stay reproducible: every running container should map back to a specific git tag.

---

## 1. Repository layout

This file lives in the **source repo**. There are two sibling repos on the host:

```text
~/Projects/immich/
├── immich-src/   # this fork — code lives here
└── immich-app/   # production deployment — docker-compose.yml + .env + photos
```

Never put deployment-specific files (`docker-compose.yml`, `.env`, photos, Postgres data) in `immich-src`. Never put source code in `immich-app`.

Sibling docs in this repo:

- [`README.md`](./README.md) — the upstream Immich README, plus the fork header pointing at this file and the changelog.
- [`ARCHITECTURE.md`](./ARCHITECTURE.md) — full codebase tour.
- [`CHANGELOG.fork.md`](./CHANGELOG.fork.md) — running list of every customization in this fork.

---

## 2. Git remotes & branches

```bash
origin    git@github.com:random4rest/immich-fork.git   # this fork (push here)
upstream  git@github.com:immich-app/immich.git         # official repo (fetch only, never push)
```

Branches:

| Branch | Purpose |
|---|---|
| `main` | Mirror of `upstream/main`. Never commit to it directly. |
| `personal` | Long-lived branch with all custom features. **Default working branch.** |
| `feature/*` | Short-lived branches; merge into `personal` and delete. |

Commit message convention — **prefix every fork-only commit with `[fork]`**:

```text
[fork] feat(web): rotate action on photo viewer
[fork] fix(server): handle missing EXIF on rotate
[fork] docs: ...
```

Why: during upstream merges we use `git log --grep '^\[fork\]'` to audit our patches and see what needs reapplying.

---

## 3. Implementing a new feature

### 3.1 Branch off `personal`

```bash
cd ~/Projects/immich/immich-src
git checkout personal
git pull --ff-only origin personal
git checkout -b feature/<short-name>
```

### 3.2 Run the dev stack with hot-reload

The dev compose lives in `docker/docker-compose.dev.yml`. It bind-mounts the entire source tree into the container, so edits in `web/src/`, `server/src/`, and `machine-learning/immich_ml/` are reloaded live.

```bash
# Stop the production stack first to free port 2283
cd ~/Projects/immich/immich-app && docker compose down

cd ~/Projects/immich/immich-src/docker
cp example.env .env             # first time only; edit UPLOAD_LOCATION + DB_PASSWORD
docker compose -f docker-compose.dev.yml up --build
```

Endpoints during development:

| URL | What |
|---|---|
| `http://localhost:3000` | SvelteKit dev server with HMR |
| `http://localhost:2283` | NestJS API (TypeScript watch mode) |
| `http://localhost:3003` | Python ML service |
| `localhost:5432` | Postgres (separate from prod — bind-mounted at `${UPLOAD_LOCATION}/postgres`) |
| `localhost:9230 / 9231` | Node debug ports |

Notes:
- The dev DB is **not** the production DB. Don't expect to see your real library here.
- First boot runs `pnpm install` inside the `immich_init` container; subsequent boots are fast.
- For ML changes, set the `DEVICE` build arg in the compose file (`cpu`, `cuda`, etc.).

### 3.3 Where to make changes

| Want to change… | Edit |
|---|---|
| HTTP route, business logic, schema | `server/src/{controllers,services,repositories,schema}/` |
| Web UI | `web/src/{routes,lib}/` |
| Mobile UI | `mobile/lib/` |
| ML model / pipeline | `machine-learning/immich_ml/models/` |
| API contract (DTO) | `server/src/dtos/*.dto.ts` (Zod schemas → also generates the OpenAPI spec) |

After changing a DTO, regenerate the OpenAPI clients used by `web/`, `cli/`, and `mobile/`:

```bash
make open-api    # or: cd open-api && bash bin/generate-open-api.sh
```

See `ARCHITECTURE.md` §2 for the full controller → service → repository flow.

### 3.4 Commit, merge, tag

```bash
git add <files>
git commit -m "[fork] feat(<area>): <what>"

git checkout personal
git merge --no-ff feature/<short-name>
git branch -d feature/<short-name>
```

Then **add an entry to [`CHANGELOG.fork.md`](./CHANGELOG.fork.md)** under `[Unreleased]`, tag a build (see §5 below), and deploy.

---

## 4. Pulling new upstream releases

Run this whenever a new official release appears.

```bash
cd ~/Projects/immich/immich-src

# 1. Fetch upstream
git fetch upstream --tags

# 2. Update the mirror branch
git checkout main
git merge --ff-only upstream/main
git push origin main

# 3. Merge into personal (resolve conflicts here, NOT on main)
git checkout personal
git merge vX.Y.Z          # the upstream release tag
# ... resolve conflicts (most should be in fork-touched files only) ...
git commit                # if merge created a merge commit
```

Conflict-resolution tips:
- Run `git log --grep '^\[fork\]' upstream/main..personal` to remind yourself what your customizations are.
- For files where upstream made big rewrites, prefer `git checkout --theirs` then re-apply your `[fork]` changes manually rather than a messy 3-way merge.
- Run the full dev stack and smoke-test before tagging.

### Schema/DB migrations from upstream

Upstream often ships new migrations under `server/src/schema/migrations/`. They run automatically on next `docker compose up` (Kysely + Postgres advisory lock — see `server/src/main.ts`). **Always back up `${UPLOAD_LOCATION}/backups/` before deploying a release with new migrations.**

---

## 5. Building & deploying

### 5.1 Tagging

After merging a new feature or upstream release into `personal`, tag a build. Tag scheme: `personal-<upstream-version>-<bump>`.

```bash
git tag personal-v1.XYZ.0-1   # first build on top of upstream v1.XYZ.0
git push --follow-tags origin personal
```

### 5.2 Building images

If a CI workflow exists in `.github/workflows/` of the fork, the tag push triggers the build automatically. Otherwise build locally:

```bash
cd ~/Projects/immich/immich-src

docker build -t ghcr.io/random4rest/immich-server:personal-v1.XYZ.0-1 \
  -f server/Dockerfile .

docker build -t ghcr.io/random4rest/immich-machine-learning:personal-v1.XYZ.0-1-cuda \
  --build-arg DEVICE=cuda machine-learning

docker push ghcr.io/random4rest/immich-server:personal-v1.XYZ.0-1
docker push ghcr.io/random4rest/immich-machine-learning:personal-v1.XYZ.0-1-cuda
```

### 5.3 Deploying to production

`immich-app/.env` has a single `IMMICH_VERSION` variable. Bump it and roll the stack:

```bash
cd ~/Projects/immich/immich-app

# edit IMMICH_VERSION=personal-v1.XYZ.0-1 in .env
docker compose pull
docker compose up -d
docker image prune -f
```

**Always verify:**

```bash
docker compose ps                                  # everything running
docker logs immich_server --tail=50               # no startup errors
curl -s http://localhost:2283/api/server/ping     # {"res":"pong"}
```

If something is wrong, roll back instantly by changing `IMMICH_VERSION` back to the previous tag and `docker compose up -d` again.

---

## 6. Operational reminders

- **Backups**: Immich's built-in DB backup writes to `${UPLOAD_LOCATION}/backups/`. Make sure photos + that folder are backed up off-site (`restic`, `rclone`).
- **Remote access**: Tailscale on the server + every client; mobile app points at the Tailscale name.
- **Production stack ports**: `2283` (API + web). The dev stack uses `2283` too — stop one before starting the other.
- **Postgres data**: lives in the `immich_pgdata` named Docker volume. Never bind-mount it on WSL2 (race condition wipes the data dir on restart). See `immich-app/RUNBOOK.md`.

---

## 7. Push & SSH gotchas (lessons from setup)

- `origin` uses SSH; the SSH key is passphrase-protected. Run `eval "$(ssh-agent -s)" && ssh-add ~/.ssh/id_ed25519` once per shell session.
- This repo was originally cloned shallow (`--depth=1`). Pushing a new branch to `origin` will fail with `remote unpack failed` until the repo is unshallowed:
  ```bash
  git -c "url.https://github.com/.insteadOf=git@github.com:" fetch --unshallow upstream
  ```
  (Public repo over HTTPS, no auth needed.) This only has to be done once.
- Never push to `upstream` — that remote is configured with `pushurl=DISABLE` to make it impossible by accident.

---

## 8. Quick reference

| Task | Command |
|---|---|
| Start dev stack | `cd immich-src/docker && docker compose -f docker-compose.dev.yml up --build` |
| Start prod stack | `cd immich-app && docker compose up -d` |
| Stop prod stack | `cd immich-app && docker compose down` |
| Pull upstream release | `git fetch upstream --tags && git checkout personal && git merge vX.Y.Z` |
| Tag a build | `git tag personal-vX.Y.Z-N && git push --follow-tags origin personal` |
| Deploy a tag | edit `IMMICH_VERSION` in `immich-app/.env` → `docker compose pull && up -d` |
| Roll back | edit `IMMICH_VERSION` back → `docker compose up -d` |
| Read the codebase | `ARCHITECTURE.md` |
