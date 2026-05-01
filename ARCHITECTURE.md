# Immich — Codebase Map

Immich is a self-hosted photo/video manager. The repo is a pnpm monorepo (`pnpm-workspace.yaml`) of independent apps that share an OpenAPI-generated SDK.

## 1. Top-level layout

```text
immich-src/
├── server/             # NestJS backend (REST API + workers + SSR)
├── web/                # SvelteKit web client
├── mobile/             # Flutter mobile app
├── machine-learning/   # Python FastAPI ML service
├── cli/                # Node CLI for bulk uploads
├── open-api/           # Generated OpenAPI spec + TS SDK
├── e2e/                # End-to-end tests
├── i18n/               # Translations
├── docs/               # Docusaurus site
├── deployment/ docker/ # Deploy + compose files
└── plugins/            # First-party plugin examples
```

The four runtime services that run together (compose):

| Service | Lang | Role |
|---|---|---|
| `immich-server` | TypeScript / NestJS | REST API + WebSocket + serves web app + runs background workers |
| `immich-machine-learning` | Python / FastAPI | CLIP, face detection, OCR inference |
| Postgres (`vectorchord` image) | SQL | Single source of truth, vector search |
| Redis | KV | BullMQ job queues, pub/sub |

Storage on disk is rooted at `IMMICH_MEDIA_LOCATION` and split into folders defined in `enum.ts → StorageFolder`: `library/`, `upload/`, `thumbs/`, `encoded-video/`, `profile/`, `backups/`.

---

## 2. Server (`server/src/`)

NestJS app; entry point `main.ts` forks/threads three kinds of **workers** declared in `enum.ts → ImmichWorker`:

```ts
class Workers {
  workers: Partial<Record<ImmichWorker, ...>> = {};
  ...
  async bootstrap() {
    const isMaintenanceMode = await this.isMaintenanceMode();
    const { workers } = new ConfigRepository().getEnv();
    if (isMaintenanceMode) {
      this.startWorker(ImmichWorker.Maintenance);
    } else {
      await this.waitForFreeLock();
      for (const worker of workers) this.startWorker(worker);
    }
  }
}
```

| Worker file | Purpose |
|---|---|
| `workers/api.ts` | Forked Node process. Runs `ApiModule` → HTTP REST + WebSocket + SSR. |
| `workers/microservices.ts` | `worker_thread`. Runs `MicroservicesModule` → BullMQ consumers (thumbnails, metadata, ML, etc). |
| `workers/maintenance.ts` | Boots `MaintenanceModule` when DB migrations are running. |

There's also a CLI app (`ImmichAdminModule`, in `commands/`) launched as `immich-admin`.

### Architectural layers (NestJS, layered)

```text
HTTP request
  ↓
controllers/*.controller.ts   (route + DTO + AuthGuard)
  ↓
services/*.service.ts          (business logic, all extend BaseService)
  ↓
repositories/*.repository.ts   (Kysely SQL or external I/O)
  ↓
schema/tables/*.table.ts       (Postgres tables, decorators from @immich/sql-tools)
```

**Key cross-cutting modules:**

- `app.module.ts` — wires three modules: `ApiModule`, `MicroservicesModule`, `MaintenanceModule`. All share `repositories`, `services`, `commonImports` (Kysely, BullMQ, OpenTelemetry, nestjs-cls).
- `middleware/`
  - `auth.guard.ts` — global `AuthGuard`; routes opt in via `@Authenticated({ permission })`.
  - `error.interceptor.ts`, `global-exception.filter.ts`, `logging.interceptor.ts`.
  - `file-upload.interceptor.ts`, `asset-upload.interceptor.ts` — multipart asset uploads.
  - `websocket.adapter.ts` — Socket.IO adapter for real-time events.
- `cores/storage.core.ts` — singleton that resolves on-disk paths for assets, thumbnails, person faces, encoded video; handles atomic moves via the `move_history` table.
- `decorators.ts` — `@Endpoint`, `@Authenticated`, `@OnJob`, `@OnEvent`, `@UpdatedAtTrigger`, etc.
- `dtos/*.dto.ts` — Zod schemas wrapped by `nestjs-zod` → also produce the OpenAPI spec.
- `enum.ts` — every enum used across server (worker names, queues, jobs, permissions, asset types, …).
- `schema/` — declarative Postgres schema (tables, enums, functions, migrations) using `@immich/sql-tools` decorators, compiled by Kysely.
- `validation.ts`, `config.ts`, `database.ts`, `constants.ts` — bootstrapping helpers.

### How a REST request flows

Example: `GET /api/assets/:id`

1. `controllers/asset.controller.ts` — declares the route:

```ts
@Get(':id')
@Authenticated({ permission: Permission.AssetRead, sharedLink: true })
@Endpoint({ summary: 'Retrieve an asset', ... })
getAssetInfo(@Auth() auth: AuthDto, @Param() { id }: UUIDParamDto): Promise<AssetResponseDto> {
  return this.service.get(auth, id) as Promise<AssetResponseDto>;
}
```

2. `AuthGuard` resolves `auth: AuthDto` from `Bearer`, cookie, API key, or shared link.
3. `@Authenticated({ permission })` records required `Permission` (enum from `enum.ts`); `requireAccess` checks ACL via `AccessRepository`.
4. `services/asset.service.ts` calls one or more repositories, returns a DTO.
5. `ZodSerializerInterceptor` validates the response shape against the Zod-generated DTO.
6. `dtos/*.dto.ts` is the same source of truth used by Swagger; the spec is dumped to `open-api/immich-openapi-specs.json`, which generates the TS SDK in `open-api/typescript-sdk/` (used by `web/`, `cli/`) and the Dart SDK (used by `mobile/`).

### Background jobs

- BullMQ (Redis) queues defined in `enum.ts → QueueName`: `thumbnailGeneration`, `metadataExtraction`, `videoConversion`, `faceDetection`, `facialRecognition`, `smartSearch`, `duplicateDetection`, `backgroundTask`, `storageTemplateMigration`, `migration`, …
- Job names in `enum.ts → JobName` (e.g. `AssetDetectFaces`, `AssetGenerateThumbnails`, `AssetEncodeVideo`).
- `repositories/job.repository.ts` discovers handlers tagged with `@OnJob({ name, queue })` on services and registers BullMQ workers.
- After upload, `AssetMediaService` enqueues a chain of jobs that build sidecars/thumbnails, extract EXIF, run ML, etc.

### Auth model (`AuthDto`)

A request's identity can be one of:
- session cookie (`session.table.ts`)
- bearer JWT
- API key (`api_key.table.ts` + `Permission` scopes)
- shared-link key/slug (`shared_link.table.ts`, partial read-only access)
- OAuth/OIDC (`oauth.controller.ts`)

Permissions are fine-grained (~120 entries in `Permission` enum) and checked by `AccessRepository` for every protected entity.

### Web/SSR

The same NestJS process serves the prebuilt SvelteKit `web/` app via `sirv` in `app.common.ts` and falls back to SSR through `ApiService.ssr`.

---

## 3. Database schema (`server/src/schema/tables/`)

Single Postgres database `immich` with extensions: `uuid-ossp`, `unaccent`, `cube`, `earthdistance`, `pg_trgm`, `plpgsql`, plus `vectorchord`/`vchord` for embeddings (via the custom Postgres image).

Tables are declared as TypeScript classes; `schema/index.ts` lists them all and exports the `DB` interface used by Kysely.

### Core entities

```text
user ── 1:N ── asset ──┬── asset_exif      (1:1 EXIF + GPS)
                       ├── asset_file      (N: original/preview/thumbnail/sidecar/encoded_video paths)
                       ├── asset_metadata  (N: free-form key/value)
                       ├── asset_face ──── person   (face crops + person grouping)
                       ├── asset_ocr ───── ocr_search   (per-asset OCR text + tsvector)
                       ├── asset_edit      (crop/rotate/mirror history)
                       ├── asset_job_status (per-asset job state)
                       ├── tag_asset ───── tag (+ tag_closure for hierarchy)
                       ├── album_asset ─── album ── album_user (sharing roles)
                       ├── memory_asset ── memory
                       ├── shared_link_asset ── shared_link
                       ├── stack          (group raw+jpg, bursts, etc.)
                       └── smart_search   (CLIP embeddings, IVF/vector index)
face_search    ─ ML embeddings used by facial recognition
geodata_places, naturalearth_countries  ─ reverse geocoding
library        ─ external on-disk libraries (vs upload)
api_key, session, partner, notification  ─ identity / sharing / events
plugin, plugin_filter, plugin_action     ─ first-party plugin runtime
workflow, workflow_filter, workflow_action ─ rule-based automations
video_stream_session/variant/segment      ─ HLS adaptive streaming state
move_history    ─ tracks in-progress filesystem moves so they survive restarts
system_metadata ─ singleton key/value (config snapshots, ML state, etc.)
version_history ─ schema/version checkpoints
*_audit         ─ shadow tables filled by AFTER DELETE triggers, used by /sync API
```

### Key columns of `asset` (`schema/tables/asset.table.ts`)

- `id` (`uuidv7`), `ownerId → user`, `libraryId → library`
- `type` (`AssetType`: IMAGE/VIDEO/AUDIO/OTHER), `originalPath`, `originalFileName`
- `checksum` (sha1, unique per `(ownerId, libraryId)`), `thumbhash`
- `fileCreatedAt`, `fileModifiedAt`, `localDateTime` (+ trigram + month indexes for the timeline)
- `livePhotoVideoId → asset` (paired motion-photo)
- `stackId → stack`, `duplicateId`
- `status` (Active/Trashed/Deleted), `visibility` (Timeline/Archive/Hidden/Locked)
- `isFavorite`, `isOffline`, `isExternal`, `isEdited`
- `width`, `height`, `duration`
- Audit + soft-delete (`deletedAt`, `updatedAt`, `updateId`) for incremental sync

The `user` table holds `email`, hashed `password`, `pinCode`, `oauthId`, `quotaSizeInBytes`, `quotaUsageInBytes`, `storageLabel`, `isAdmin`, `status`, etc.

### Migrations

`schema/migrations/` contains generated SQL migrations applied at boot via Kysely + a Postgres advisory lock (see `main.ts → waitForFreeLock`).

---

## 4. Storage layout on disk

Rooted at `IMMICH_MEDIA_LOCATION`, organized per-user and selected via `StorageCore`:

```text
<media>/library/<storageLabel|userId>/...   # original uploads (storage template)
<media>/upload/<userId>/<chunked>           # in-progress uploads
<media>/thumbs/<userId>/<assetId>_<type>[_edited].<ext>
<media>/encoded-video/<userId>/<assetId>.mp4
<media>/profile/<userId>.<ext>
<media>/backups/                            # pg_dumpall output
```

The "storage template" is a user-configurable path pattern; `services/storage-template.service.ts` rewrites paths in batches and `move_history` makes the rename crash-safe.

---

## 5. Machine-learning service (`machine-learning/immich_ml/`)

Standalone FastAPI app that the server calls over HTTP.

```text
immich_ml/
├── main.py        # FastAPI app: GET /, GET /ping, POST /predict
├── config.py      # MACHINE_LEARNING_* env vars, preload settings
├── schemas.py     # ModelTask / ModelType / pipeline payload typings
├── models/
│   ├── base.py            # InferenceModel ABC
│   ├── cache.py           # in-memory ModelCache with TTL idle unload
│   ├── transforms.py      # PIL / numpy / ONNX I/O helpers
│   ├── clip/              # CLIP visual + textual (ONNX/Armnn)
│   ├── facial_recognition/ # detection + recognition (insightface)
│   └── ocr/                # text detection + recognition
└── sessions/      # ONNX Runtime / Armnn / OpenVINO providers
```

Single endpoint:

```python
@app.post("/predict", dependencies=[Depends(update_state)])
async def predict(
    entries: InferenceEntries = Depends(get_entries),
    image: bytes | None = File(default=None),
    text: str | None = Form(default=None),
) -> Any:
    ...
    response = await run_inference(inputs, entries)
    return ORJSONResponse(response)
```

The server's `MachineLearningRepository` (`server/src/repositories/machine-learning.repository.ts`) wraps it: `detectFaces`, `encodeImage`, `encodeText`, `ocr`. It supports a list of ML URLs with health checks and failover.

Resulting embeddings are written to `smart_search` (CLIP) and `face_search` (face) tables, which are queried via vectorchord similarity search inside `SearchRepository`.

---

## 6. Web app (`web/src/`)

SvelteKit (Svelte 5). Route groups:

```text
src/routes/
├── (user)/        # main app (timeline, albums, search, sharing, settings…)
├── admin/         # /admin/* (users, jobs, queues, system settings, …)
├── auth/          # login, register, change-password, pin-prompt, oauth
├── link/          # public shared-link viewer
└── maintenance/   # shown when server is in maintenance mode
src/lib/
├── components/ elements/ modals/ attachments/ actions/   # UI building blocks
├── stores/ managers/ services/                           # state + API wrappers
├── i18n/ utils/ workers/ constants.ts
```

All API calls go through the generated `@immich/sdk` (output of `open-api/typescript-sdk/`) so the contract is the same as Swagger/OpenAPI.

`hooks.server.ts` proxies SvelteKit's SSR through Nest; `hooks.client.ts` wires up real-time events from the server's Socket.IO namespace.

---

## 7. Other clients

- `mobile/` — Flutter app with parallel layered architecture (`domain/`, `infrastructure/`, `repositories/`, `services/`, `providers/`, `pages/`, `widgets/`). Talks to the server via the Dart SDK generated from the same OpenAPI spec.
- `cli/` (`@immich/cli`) — Node CLI used for bulk imports; uses `@immich/sdk`.
- `e2e/` — black-box tests against a running server + DB + ML stack.

---

## 8. How "everything fits together" (one concrete flow)

Uploading an image:

1. Client (web/mobile/cli) `POST /api/assets` (multipart) → `AssetMediaController.uploadAsset` (`asset-media.controller.ts`).
2. `AssetUploadInterceptor` streams the file to `<media>/upload/<userId>/`, computes sha1.
3. `AssetMediaService` checks for duplicates by `checksum`, inserts an `asset` row + an `asset_file` row of type `Original`, and (if no template) moves the file into the user's `library/` folder via `StorageCore`/`move_history`.
4. Service enqueues a BullMQ job chain: `AssetExtractMetadata` → `AssetGenerateThumbnails` → `AssetDetectFaces`/`AssetSmartSearch`/`AssetDetectDuplicates`/`AssetEncodeVideo`/`AssetOcr`.
5. Microservices worker picks each job up, calls `MachineLearningRepository.encodeImage`/`detectFaces`/`ocr` against the Python service, persists results into `asset_exif`, `asset_file`, `asset_face`, `face_search`, `smart_search`, `asset_ocr`, etc.
6. `EventRepository` emits `WebSocket` events; clients refresh the timeline via `/api/sync` (audit tables + `updateId` columns make it incremental).
7. Auth, ACL, sharing, partners, albums, memories, tags, smart-search are all just additional joins/queries against this same schema.

That's the full mental model: **NestJS server with controller → service → Kysely repository → typed Postgres schema, BullMQ workers for everything async, a separate Python ML microservice for inference, on-disk files organized by `StorageCore`, and an OpenAPI spec that generates the SDKs used by every client.**
