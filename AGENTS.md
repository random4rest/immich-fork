# AGENTS.md — Working in this Immich fork

This is a personal fork of [immich-app/immich](https://github.com/immich-app/immich), **detached from upstream as of `v2.7.5`**. New features and fixes are developed independently here; we do not pull updates from upstream. Read this file before making changes, then read `ARCHITECTURE.md` for the codebase tour.

The goals of this fork are:

1. Self-host Immich on a home PC (LAN + remote access via Tailscale).
2. Add custom features tailored to one user, without the constraints of an upstream contribution model.
3. Stay reproducible: every running container should map back to a specific git tag.

**Detached-fork policy** (decided 2026-05-02):

- We do not merge `upstream/main` or any future Immich release tag.
- Security-relevant CVE fixes in dependencies (npm packages, base Docker image) may be cherry-picked from upstream **on demand**, but never as a routine merge — see §4.
- The `upstream` git remote is not configured by default; re-add it only if you need to look at a specific upstream commit.

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
```

That's the only remote. The `upstream` remote was removed when we detached. To look at a specific upstream commit (e.g. for a CVE backport), re-add it temporarily:

```bash
git remote add upstream https://github.com/immich-app/immich.git
git fetch upstream
git log upstream/main -- path/to/file       # inspect what they did
git cherry-pick <upstream-sha>              # if you want to bring it in
git remote remove upstream
```

Branches:

| Branch | Purpose |
|---|---|
| `main` | The only long-lived branch. **Default working branch.** All custom features land here. |
| `feature/*` | Short-lived branches for non-trivial work; merge into `main` and delete. Trivial changes can go directly on `main`. |

The detachment point is preserved as the annotated tag **`forked-from-immich-v2.7.5`**. The pre-detachment fork history (when we still tracked upstream) is at the tag **`personal-v2.7.5-1-mainbase`**.

Commit messages: standard conventional-commit style is fine. There is no special prefix anymore — every commit in this repo is "fork code" by definition.

```text
feat(web): bulk-export selected assets as zip
fix(server): handle missing EXIF on rotate
docs: ...
chore(deps): bump sharp to 0.34.0
```

---

## 3. Implementing a new feature

### 3.1 Branch off `main`

For non-trivial work:

```bash
cd ~/Projects/immich/immich-src
git checkout main
git pull --ff-only origin main
git checkout -b feature/<short-name>
```

For small/trivial changes (typo fix, bumped string, log message), commit directly to `main`. Use your judgement.

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
git commit -m "feat(<area>): <what>"

git checkout main
git merge --no-ff feature/<short-name>
git branch -d feature/<short-name>
```

Then **add an entry to [`CHANGELOG.fork.md`](./CHANGELOG.fork.md)** under `[Unreleased]`, tag a build (see §5 below), and deploy.

---

## 4. Backporting a security fix from upstream (rare)

We do not routinely follow upstream. But if you become aware of a CVE or critical fix in upstream Immich code or its dependencies, you can cherry-pick a single commit:

```bash
cd ~/Projects/immich/immich-src

# Re-add upstream remote temporarily
git remote add upstream https://github.com/immich-app/immich.git
git fetch upstream

# Find the fix commit (search by keyword, file, or check upstream's release notes)
git log upstream/main --oneline -- server/src/path/to/file
git show <sha>          # inspect

# Cherry-pick onto main (or onto a branch first if it's risky)
git checkout main
git cherry-pick <sha>
# ... resolve conflicts if any (likely if our code diverged) ...

# Tear down the remote when done
git remote remove upstream
```

Add an entry under "Security" in `CHANGELOG.fork.md` describing the upstream commit you backported and why.

### Dependency security patches (the more common case)

CVEs in npm packages, sharp, exiftool, or the base Docker image are usually reported by `npm audit`, GitHub Dependabot alerts (enabled in repo settings), or the `pnpm audit` command. Bump the offending package in the relevant `package.json`, run `pnpm install` to update the lockfile, test, commit. No upstream involvement needed.

```bash
cd ~/Projects/immich/immich-src
pnpm -r audit                     # scan all workspaces
pnpm -F immich update <pkg>       # update specific package in server
pnpm install                      # refresh lockfile
```

---

## 5. Building & deploying

### 5.1 Tagging

After landing a release-worthy set of changes on `main`, tag a build. **Tag scheme: plain semver `vMAJOR.MINOR.PATCH`** — your project, your numbers. Bump major for breaking changes (DB migration that can't be auto-rolled back, new required env var, etc.), minor for new features, patch for bug fixes.

```bash
git tag v1.0.0
git push --follow-tags origin main
```

The `forked-from-immich-v2.7.5` annotated tag marks the point we detached — leave it alone as a permanent historical reference.

### 5.2 Building images

Build locally (no CI yet):

```bash
cd ~/Projects/immich/immich-src

docker build -t immich-server:v1.0.0 \
  --build-arg BUILD_ID=v1.0.0 \
  -f server/Dockerfile .
```

> **`--build-arg BUILD_ID=...` is mandatory.** See §9 for why. The `immich-app/docker-compose.yml` does this automatically via `args: { BUILD_ID: ${IMMICH_VERSION} }`; only standalone `docker build` invocations need it explicit.

ML uses the upstream `-cuda` image unchanged (we don't fork ML); pinned via `IMMICH_ML_VERSION` separately in `immich-app/.env`.

### 5.3 Deploying to production

`immich-app/.env` has an `IMMICH_VERSION` variable. Bump it and roll the stack:

```bash
cd ~/Projects/immich/immich-app

# edit IMMICH_VERSION=v1.0.0 in .env
docker compose build immich-server
docker compose up -d --force-recreate immich-server
docker image prune -f
```

**Always verify:**

```bash
docker compose ps                                  # everything running
docker compose logs --tail=50 immich-server        # no startup errors
curl -s http://localhost:2283/api/server/ping      # {"res":"pong"}

# SvelteKit hash sanity-check (see §9.2)
docker exec immich_server sh -c '
  echo "[index.html]"; grep -o "__sveltekit_[a-z0-9]*" /build/www/index.html | sort -u
  echo "[js chunks]";  grep -rho "globalThis.__sveltekit_[a-z0-9]*" /build/www/_app | sort -u
'
```

If something is wrong, roll back by changing `IMMICH_VERSION` back to the previous tag and rebuild. **DB caveat:** if the new version added a migration, rolling back the image won't roll back the schema; restore from `${UPLOAD_LOCATION}/backups/` instead.

---

## 6. Operational reminders

- **Backups**: Immich's built-in DB backup writes to `${UPLOAD_LOCATION}/backups/`. Make sure photos + that folder are backed up off-site (`restic`, `rclone`).
- **Remote access**: Tailscale on the server + every client; mobile app points at the Tailscale name.
- **Production stack ports**: `2283` (API + web). The dev stack uses `2283` too — stop one before starting the other.
- **Postgres data**: lives in the `immich_pgdata` named Docker volume. Never bind-mount it on WSL2 (race condition wipes the data dir on restart). See `immich-app/RUNBOOK.md`.

---

## 7. Push & SSH gotchas

- `origin` uses SSH; the SSH key is passphrase-protected. Run `eval "$(ssh-agent -s)" && ssh-add ~/.ssh/id_ed25519` once per shell session before pushing.
- Alternative: switch the remote to HTTPS + Personal Access Token if SSH is annoying:
  ```bash
  git remote set-url origin https://github.com/random4rest/immich-fork.git
  # next push will prompt for username + PAT
  ```

---

## 8. Quick reference

| Task | Command |
|---|---|
| Start dev stack | `cd immich-src/docker && docker compose -f docker-compose.dev.yml up --build` |
| Start prod stack | `cd immich-app && docker compose up -d` |
| Stop prod stack | `cd immich-app && docker compose down` |
| Tag a release | `git tag vX.Y.Z && git push --follow-tags origin main` |
| Build + deploy | edit `IMMICH_VERSION` in `immich-app/.env` → `docker compose build && docker compose up -d --force-recreate immich-server` |
| Roll back image | edit `IMMICH_VERSION` back → rebuild + recreate (note DB caveat in §5.3) |
| Backport a CVE fix from upstream | see §4 |
| Read the codebase | `ARCHITECTURE.md` |

---

## 9. Lessons learned

### 9.1 Why we detached from upstream

We initially tried to track `upstream/main` so we could pull in new features as they landed. That immediately bit us:

- `upstream/main` rolls every PR the moment it lands, including `chore!` and `refactor!` commits with breaking changes that haven't gone through any release validation. Building from `main` produced a server image that crashed the web client on load with `TypeError: Cannot read properties of undefined (reading 'env')` (just a spinner forever, see §9.2).
- It applied DB migrations (`<ts>-DropAuditTable`) that don't exist in any tagged release, making rollback to `ghcr.io/immich-app/immich-server:vX.Y.Z` impossible without restoring the DB from backup (`corrupted migrations: previously executed migration <ts>-... is missing`).

We then tried "rebase onto release tags" instead. That works, but in practice:

- Even release tags carry breaking changes (`chore!`, `refactor!`) that conflict with our customizations every cycle.
- Resolving conflicts requires understanding upstream's intent, which is non-trivial for a one-person fork.
- The upside (new features, security patches in Immich code) is small for a single-user home server already getting most of what we want.

So we chose the **detached fork** model: stop merging upstream entirely, pin to the v2.7.5 codebase, develop our own features. Security patches in dependencies are handled via `pnpm audit` / Dependabot (see §4). Critical CVEs in Immich code itself can be backported as one-off cherry-picks from upstream.

### 9.2 SvelteKit `__sveltekit_<HASH>` mismatch — the spinner-of-death bug

**Symptom:** web UI loads to the spinning logo and stays there. DevTools console shows `Uncaught (in promise) TypeError: Cannot read properties of undefined (reading 'env')` from a minified chunk. Pretty-printing the chunk reveals:

```javascript
var d = globalThis.__sveltekit_<HASH1>.env
```

…where `globalThis.__sveltekit_<HASH1>` is `undefined` because `index.html` initialised a *different* `globalThis.__sveltekit_<HASH2>`.

**Root cause:** `web/svelte.config.js` sets `kit.version.name` to `process.env.IMMICH_BUILD || Date.now().toString()`. SvelteKit hashes `version.name` into the `__sveltekit_<HASH>` global namespace. SvelteKit loads the config file more than once during a build (SSR/prerender pass and client pass), so `Date.now()` evaluates to two different values, producing two different hashes and a non-functional bundle.

**Fix:** always pass a stable `BUILD_ID` to the server image build:

```bash
docker build -t ... --build-arg BUILD_ID=personal-vX.Y.Z-N -f server/Dockerfile .
```

The `immich-app/docker-compose.yml` does this automatically via `build.args.BUILD_ID: ${IMMICH_VERSION}`. The Dockerfile's `web` stage exposes it as `ENV IMMICH_BUILD=${BUILD_ID}`, which `svelte.config.js` then prefers over the `Date.now()` fallback.

**Sanity check after every rebuild:**

```bash
docker exec immich_server sh -c '
  echo "[index.html]"; grep -o "__sveltekit_[a-z0-9]*" /build/www/index.html | sort -u
  echo "[js chunks]";  grep -rho "globalThis.__sveltekit_[a-z0-9]*" /build/www/_app | sort -u
'
```

Both lists must print **the same single hash**. If they differ, `BUILD_ID` is empty — check that `docker compose config | grep BUILD_ID` shows your version string.

### 9.3 Always `--no-cache` after a config change

Layer caching can pin a stale `web/build` directory even after you change source files or build args. After any change to `svelte.config.js`, `vite.config.ts`, the Dockerfile, or build args, do `docker compose build --no-cache immich-server` once. Routine code changes don't need it.
