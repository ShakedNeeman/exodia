# Architecture

Sentinel is a catalog in front of an object store. The catalog answers “what is this and who can see it”. The object store answers “give me the bytes”.

Nothing in the design assumes the files are small, uniform, or already labeled.

## Component map

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Browser                                                                 │
│  React + TypeScript                                                      │
│  · metadata form          · part planner + checksums                     │
│  · upload queue           · IndexedDB resume journal                     │
│  · browse + inspector     · collection / facet state                     │
└────────────┬─────────────────────────────────────┬───────────────────────┘
             │ HTTPS JSON                          │ HTTPS PUT/GET
             │ sessions, search, ACL               │ presigned, multipart
             ▼                                     ▼
┌────────────────────────────┐          ┌────────────────────────────┐
│  Catalog API               │          │  Object storage            │
│  Spring Boot 3 / Java 21   │   keys   │  Amazon S3                 │
│                            │ ───────► │                            │
│  /uploads  /assets         │          │  raw/{yyyy}/{mm}/{id}      │
│  /search   /collections    │          │  derived/{id}/...          │
└──┬─────────┬─────────┬─────┘          └─────────────▲──────────────┘
   │         │         │                              │
   ▼         ▼         ▼                              │ range read
Postgres   Redis      SQS ──► Enrichment workers ─────┘
catalog    progress   jobs    Python 3.12
                              detect / extract / derive / index
   │
   └──► OpenSearch  (search projection, rebuildable)
```

## Where data lives

| Data | Store | Why |
|---|---|---|
| Asset identity, state machine, ACLs | PostgreSQL | Needs transactions and a real state machine |
| User metadata | PostgreSQL | Written at session start, queried with the asset |
| Technical metadata | PostgreSQL JSONB | Shape depends on the file family |
| Upload session + completed parts | PostgreSQL | Resume must survive a Redis flush |
| Live progress % | Redis | High write rate, disposable |
| Raw file | Object storage | The only place tens of TBs belong |
| Thumbnails, posters, spectrograms | Object storage, `derived/` | Regenerable, must not sit on API disks |
| Search document | OpenSearch | Facets and free text |
| In-flight work | SQS | Survives worker death |
| Resume journal in the browser | IndexedDB | Refresh / laptop sleep |

## Control plane vs data plane

```
   metadata QPS (small)                    byte throughput (enormous)

   user --> Catalog API --> Postgres       Client --> S3
                 |                         Client <-- S3
                 +--> SQS --> workers --> S3
```

The API creates keys and signs URLs. It does not stream the object. Workers are the only servers that read file bytes, and they do it from storage, not from the client.

## Asset state machine

```
draft ──► uploading ──► uploaded ──► enriching ──► ready
   │          │             │            │
   │          ▼             ▼            ▼
   └─────►  failed       failed      partial ──► ready
              │                        │
              └── retry upload         └── re-run extractor
```

- `draft` — session created, metadata saved, no complete object.
- `uploading` — at least one part accepted.
- `uploaded` — object sealed, checksums match. Download is allowed.
- `enriching` — workers are running.
- `ready` — extractors that apply have finished.
- `partial` — at least one extractor failed or the type is unknown. The file is still first-class.
- `failed` — integrity or session failure. Object is not published.

Browse includes `uploaded`, `enriching`, `ready`, and `partial`. A file does not hide until a parser succeeds.

## Deployment shape

One AWS region to start. Three logical services, independently scalable.

| Service | Runtime | Scale trigger |
|---|---|---|
| Catalog API | ECS / EKS, Spring Boot | Request rate, not bytes |
| Enrichment workers | ECS, Python, per-family queues | Queue depth and family (video ≠ exif) |
| Storage / search / queue | Managed | Capacity and retention |

## Networking

- Browser to API: HTTPS, OIDC bearer token.
- Browser to storage: HTTPS to the presigned URL. CORS is locked to the app origin, methods `PUT` / `GET` / `HEAD`, headers limited to checksum and content-type.
- API to storage: IAM role, no long-lived keys on disk.
- Workers to storage: same role, read `raw/`, write `derived/`.
- No public buckets. Ever.

## Multi-object assets

S3 can now hold a single object up to ~50 TB. That covers a single take. It does not cover a *set*.

Examples that must not be flattened into one blob:

- SIGMF: `capture.sigmf-meta` + `capture.sigmf-data`
- A mission folder of frames + a sidecar CSV
- A document and its attachments uploaded together

`asset` is what the user sees. `asset_object` is each blob, with a role (`primary`, `sidecar`, `attachment`). Enrichment can run on the set (the SIGMF extractor reads both) or on each object.

## Capacity sketch

These are planning numbers, not a promise.

| Item | Order of magnitude |
|---|---|
| Estate | 20–200 TB, growing |
| Objects | 10⁷–10⁸ over a few years if the set is mixed small files + large takes |
| Concurrent uploads | tens of users, a handful of TB-class transfers |
| Part parallelism per transfer | 4–8 (browser and NIC limited) |
| API | small: metadata and URL signing |
| Workers | scale out; a 2 TB SIGMF spectrogram is hours, not seconds |
| Search documents | one per asset, not per byte |

The thing that actually grows without bound is storage. Everything else is sized for metadata and for the enrichment backlog.

## Observability

One trace id from `POST /uploads` through complete, through the SQS message, through each extractor.

Metrics that matter:

- `upload.part.failures` and `upload.complete.checksum_mismatch`
- `upload.bytes_per_second` by client
- `enrichment.duration` by extractor
- `enrichment.unknown_type` (this is a product signal, not just an error)
- OpenSearch lag vs Postgres `uploaded_at`

Logs never include presigned URLs or raw checksums of classified objects beyond what the audit table already stores.
