<p align="center">
  <img src="design/immich-logo-stacked-light.svg" width="220" alt="Immich logo" />
</p>

<h1 align="center">Immich · <code>random4rest</code> fork</h1>

<p align="center">
  My personal homelab build of <a href="https://github.com/immich-app/immich">Immich</a> — the high-performance, self-hosted photo &amp; video manager.
  <br/>
  Custom features welded on top, upstream releases merged in regularly.
</p>

<p align="center">
  <a href="https://github.com/immich-app/immich"><img alt="Forked from immich-app/immich" src="https://img.shields.io/badge/forked%20from-immich--app%2Fimmich-1f6feb?style=for-the-badge&logo=github&logoColor=white"></a>
  <a href="https://opensource.org/license/agpl-v3"><img alt="License AGPL v3" src="https://img.shields.io/badge/license-AGPL%20v3-3F51B5?style=for-the-badge"></a>
  <a href="https://github.com/random4rest/immich-fork/commits/personal"><img alt="Last commit" src="https://img.shields.io/github/last-commit/random4rest/immich-fork/personal?style=for-the-badge&color=8a3ffc"></a>
  <a href="./CHANGELOG.fork.md"><img alt="Fork changelog" src="https://img.shields.io/badge/fork-changelog-ff6b6b?style=for-the-badge"></a>
</p>

<p align="center">
  <img alt="TypeScript" src="https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white">
  <img alt="NestJS" src="https://img.shields.io/badge/NestJS-E0234E?style=flat-square&logo=nestjs&logoColor=white">
  <img alt="SvelteKit" src="https://img.shields.io/badge/SvelteKit-FF3E00?style=flat-square&logo=svelte&logoColor=white">
  <img alt="Flutter" src="https://img.shields.io/badge/Flutter-02569B?style=flat-square&logo=flutter&logoColor=white">
  <img alt="Python" src="https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=python&logoColor=white">
  <img alt="FastAPI" src="https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white">
  <img alt="Postgres" src="https://img.shields.io/badge/Postgres%20%2B%20vectorchord-336791?style=flat-square&logo=postgresql&logoColor=white">
  <img alt="Redis" src="https://img.shields.io/badge/Redis-DC382D?style=flat-square&logo=redis&logoColor=white">
  <img alt="Docker" src="https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white">
  <img alt="NVIDIA CUDA" src="https://img.shields.io/badge/CUDA-RTX%204070-76B900?style=flat-square&logo=nvidia&logoColor=white">
  <img alt="Tailscale" src="https://img.shields.io/badge/access-Tailscale-242424?style=flat-square&logo=tailscale&logoColor=white">
</p>

---

## Why this fork exists

I run Immich on my own hardware as my full photo library. Vanilla Immich is excellent, but a few personal itches needed scratching:

- I want **custom UI tweaks** that aren't likely to land upstream (or aren't worth the review cycle).
- I want a **reproducible deploy** where every running container traces back to one git tag.
- I want to **stay current with official releases** instead of stagnating on a snowflake build.

This repo is the result. The `personal` branch is what's deployed; everything fork-specific is prefixed `[fork]` in the commit log so it stays trivially auditable when merging upstream releases.

> **Looking for the actual product?** Head to [**immich-app/immich**](https://github.com/immich-app/immich) · [**docs.immich.app**](https://docs.immich.app/) · [**immich.app**](https://immich.app). All credit for the platform goes to the Immich team and contributors. ❤️

---

## Quick start

> **Full workflow lives in [`AGENTS.md`](./AGENTS.md).** This is the one-screen version.

```bash
# 1. Clone the fork (full history; do NOT use --depth=1 — see AGENTS.md §7)
git clone git@github.com:random4rest/immich-fork.git
cd immich-fork
git checkout personal

# 2. Hack on it with hot-reload
cd docker
cp example.env .env       # set UPLOAD_LOCATION + DB_PASSWORD
docker compose -f docker-compose.dev.yml up --build
# web    -> http://localhost:3000
# api    -> http://localhost:2283
# ml     -> http://localhost:3003

# 3. Ship a tagged build
git tag personal-v1.XYZ.0-1
git push --follow-tags origin personal
# CI (or your local docker build) produces:
#   ghcr.io/random4rest/immich-server:personal-v1.XYZ.0-1
#   ghcr.io/random4rest/immich-machine-learning:personal-v1.XYZ.0-1-cuda

# 4. Roll prod (in the sibling immich-app repo)
cd ../../immich-app
sed -i 's/^IMMICH_VERSION=.*/IMMICH_VERSION=personal-v1.XYZ.0-1/' .env
docker compose pull && docker compose up -d
```

---

## Staying in sync with upstream

Every week-ish:

```bash
git fetch upstream --tags
git checkout personal
git merge vX.Y.Z          # the latest official release tag
# resolve conflicts, run the dev stack, tag a new build, deploy
```

Detailed merge procedure (with conflict-resolution strategy) in [`AGENTS.md` §4](./AGENTS.md#4-pulling-new-upstream-releases).
