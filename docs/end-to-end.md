# End-to-end flow

Client, server, processing, and storage at every stage — and the contracts between them.

## Sequence: one file, from drop to browse

```
Client                 Catalog API              Amazon S3               SQS                 Worker
  │                        │                        │                    │                    │
  │  POST /uploads         │                        │                    │                    │
  │  metadata + size       │  INSERT asset draft    │                    │                    │
  │  + checksum plan       │  CreateMultipartUpload │                    │                    │
  │                        │───────────────────────►│                    │                    │
  │  session + part URLs   │                        │                    │                    │
  │◄───────────────────────│                        │                    │                    │
  │                        │                        │                    │                    │
  │  PUT part N (parallel) │                        │                    │                    │
  │────────────────────────────────────────────────►│                    │                    │
  │                        │  POST /uploads/id/parts│                    │                    │
  │                        │  (part etag, crc)      │                    │                    │
  │                        │                        │                    │                    │
  │  POST /uploads/id/complete                      │                    │                    │
  │───────────────────────►│  CompleteMultipart     │                    │                    │
  │                        │───────────────────────►│                    │                    │
  │                        │  verify checksums      │                    │                    │
  │                        │  asset = uploaded      │                    │                    │
  │                        │  publish ────────────────►                  │                    │
  │                        │                        │                    │  detect            │
  │                        │                        │◄───────────────────│  extract           │
  │                        │  PATCH technical meta  │  PUT derived/      │  derive            │
  │                        │◄───────────────────────│◄───────────────────│  index             │
  │  GET /search           │                        │                    │                    │
  │───────────────────────►│  OpenSearch + Postgres │                    │                    │
  │  GET presigned preview │                        │                    │                    │
  │────────────────────────────────────────────────►│                    │                    │
```

## Stage 1 — Intent

The user has not sent a single file byte yet.

### Client

- Accepts files and folders. A folder becomes one or many assets, depending on whether the user marks it as a set (SIGMF pair, mission pack) or as independent files.
- Collects user metadata: title, collection, tags, description, custom fields.
- Plans parts from file size. Computes the checksum algorithm it will use (`CRC64NVME` preferred, `SHA-256` kept in the session).
- Refuses to start if the collection is missing. Collection is how browse stays logical; it is required, not optional.

### Server

```
POST /v1/uploads
{
  "filename": "orbit-12.sigmf-data",
  "size_bytes": 1649267441664,
  "content_type": "application/octet-stream",
  "collection_id": "col_mission_orbit",
  "user_metadata": {
    "title": "Orbit 12 — X-band take",
    "tags": ["x-band", "orbit-12", "night"],
    "fields": { "platform": "unmanned-aerial-vehicle-7", "sensor": "X-SAR" }
  },
  "siblings": [
    { "filename": "orbit-12.sigmf-meta", "role": "sidecar", "size_bytes": 18432 }
  ]
}
```

Response:

```
{
  "asset_id": "ast_8K1",
  "session_id": "upl_44c",
  "part_size": 268435456,
  "part_count": 6144,
  "objects": [
    {
      "role": "primary",
      "key": "raw/2026/10/ast_8K1/orbit-12.sigmf-data",
      "multipart_upload_id": "upload-X",
      "parts": { "from": 1, "to": 32, "url_batch": ["https://…"] }
    }
  ]
}
```

Presigned URLs are issued in batches (32 at a time), not 6,000 at once.

### Storage

`CreateMultipartUpload` has run. No object is readable.

### Processing

Nothing. There is no object.

## Stage 2 — Transfer

### Client

- Uploads parts with `PUT`, concurrency 4–8.
- On 5xx / network error: retry that part with backoff. Do not restart the file.
- On 403: ask the API for a fresh URL batch (expiry, not corruption).
- After each successful part: write `{session, part, etag, crc}` to IndexedDB and `POST /v1/uploads/{id}/parts`.
- Pause / resume is cooperative. Pause stops issuing new PUTs. Resume lists completed parts from the API and continues.

### Server

- `POST /uploads/{id}/parts` is idempotent. The same part number with the same etag is a no-op.
- State moves `draft → uploading` on the first part.
- Redis holds `progress:{session}` for the UI. Postgres holds the part ledger.

### Storage

Parts exist under the multipart upload id. They incur storage cost. A lifecycle rule aborts uploads older than 14 days so abandoned terabyte jobs do not leak money.

### Processing

Still nothing. Workers do not touch incomplete objects.

## Stage 3 — Seal

### Client

```
POST /v1/uploads/upl_44c/complete
{
  "parts": [
    { "n": 1, "etag": "…", "crc64": "…" },
    { "n": 2, "etag": "…", "crc64": "…" }
  ]
}
```

### Server

1. Load the session. Reject if the part set is incomplete.
2. `CompleteMultipartUpload` with the part etags and checksums.
3. Compare the object checksum storage returns with the session plan.
4. On match: `asset.state = uploaded`, `uploaded_at = now()`, write the content etag, publish `asset.uploaded`.
5. On mismatch: abort the multipart upload, `asset.state = failed`, return a typed error. The user retries from part 1 of a new session; user metadata is copied forward.

### Storage

The object is now a first-class key under `raw/`. It can be read with a presigned GET. Versioning is on. Deletes are soft in the catalog and become lifecycle transitions in storage.

### Processing

A message exists. No worker has necessarily started. The file is already a catalog citizen.

## Stage 4 — Enrich and browse

### Processing

Workers are dumb about HTTP and strict about files.

```
message: { asset_id, objects[], hint_mime, requested_extractors? }

1. detect     libmagic + extension + user hint → family
2. extract    only extractors that registered for that family
3. derive     thumbnail / poster / first PDF page / spectrogram
4. index      flatten into an OpenSearch document
```

Each extractor is a process with a timeout and a memory cap. A 4K video probe and a 2 TB IQ file do not share a queue. Families:

| Family | Detector | Extractor | Derivative |
|---|---|---|---|
| `image` | MIME / magic | exiftool, Pillow | 512px + 2048px |
| `video` | MIME / magic | ffprobe | poster frame |
| `document` | MIME / magic | Apache Tika | first page |
| `radar.sigmf` | sidecar pair | SIGMF parser | spectrogram strip |
| `radar.iq` | extension + user type | generic IQ (rate from metadata) | optional spectrogram |
| `geotiff` / `nitf` | magic | GDAL | overview tile |
| `unknown` | everything else | hashes + size + magic | none |

Unknown is a successful outcome. It is how a catalog stays honest about diversity.

### Server

- Extractors `PATCH /internal/assets/{id}/technical` with their JSON and a schema version.
- The indexer consumes `asset.enriched` and upserts OpenSearch.
- `GET /v1/assets/{id}` is always Postgres. It is never “sorry, search is lagging”.
- `GET /v1/search?q=&type=&tag=&collection=&from=&size=` is OpenSearch, with a Postgres hydrate for the current ACL.

### Storage

Workers use range reads when the format has a header (SIGMF, TIFF, MP4 moov). They never download a 2 TB object to a local disk to read twenty bytes of header. Derivatives are written to `derived/{asset_id}/` and are disposable.

### Client

Browse is a catalog:

- Collections on the left (the logical structure).
- Facets on top: family, tag, time, enrichment state, size.
- Grid or list in the center.
- Inspector on the right: preview, user metadata, technical metadata as it lands.

A file in `enriching` is visible. The inspector shows a live list of extractors, not a spinner that blocks the page.

## Resume after failure

| Failure | Recovery |
|---|---|
| Browser refresh | IndexedDB + `GET /uploads/{id}` → continue remaining parts |
| Laptop sleep | same |
| Presigned URL expired | `POST /uploads/{id}/urls` → new batch |
| One part 500 | retry part |
| Complete checksum mismatch | new session, metadata copied, old multipart upload aborted |
| Worker OOM on a huge video | message returns to SQS; video queue is isolated; asset stays `uploaded` |
| OpenSearch down | detail pages work; facet search degrades; indexer catches up |

## API surface (control plane only)

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/uploads` | Create asset + session + multipart upload |
| `POST` | `/v1/uploads/{id}/urls` | Next presigned batch |
| `POST` | `/v1/uploads/{id}/parts` | Record a completed part |
| `POST` | `/v1/uploads/{id}/complete` | Seal |
| `POST` | `/v1/uploads/{id}/abort` | User cancelled |
| `GET` | `/v1/uploads/{id}` | Resume snapshot |
| `GET` | `/v1/assets/{id}` | Catalog record |
| `PATCH` | `/v1/assets/{id}` | Edit user metadata after ingest |
| `GET` | `/v1/assets/{id}/content` | Redirect / return presigned GET |
| `GET` | `/v1/search` | Faceted search |
| `GET/POST` | `/v1/collections` | Logical browse roots |
| `POST` | `/v1/assets/{id}/reprocess` | Re-run extractors |

There is no `POST /v1/files` that accepts a multipart body. That endpoint is how these systems die.
