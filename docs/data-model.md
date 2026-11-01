# Data model

Postgres is the source of truth. JSONB is used where the shape changes by file family. Relations are used where integrity matters.

## Core tables

```sql
collection (
  id            text primary key,          -- col_…
  name          text not null,
  parent_id     text references collection(id),
  owner_id      text not null,
  acl           jsonb not null,            -- { roles: { editor: [], viewer: [] } }
  created_at    timestamptz not null
);

asset (
  id              text primary key,        -- ast_…
  collection_id   text not null references collection(id),
  title           text not null,
  state           text not null,           -- draft|uploading|uploaded|enriching|ready|partial|failed
  family          text,                    -- image|video|document|radar.sigmf|radar.iq|geotiff|unknown
  size_bytes      bigint not null,
  created_by      text not null,
  created_at      timestamptz not null,
  uploaded_at     timestamptz,
  content_sha256  text                     -- async, after seal
);

asset_object (
  id            text primary key,
  asset_id      text not null references asset(id),
  role          text not null,             -- primary|sidecar|attachment
  filename      text not null,
  storage_key   text not null unique,
  size_bytes    bigint not null,
  mime          text,
  checksum      text
);

upload_session (
  id                    text primary key,  -- upl_…
  asset_id              text not null references asset(id),
  multipart_upload_id   text not null,
  part_size             int not null,
  part_count            int not null,
  state                 text not null,     -- open|completing|complete|aborted
  created_at            timestamptz not null
);

upload_part (
  session_id    text not null references upload_session(id),
  part_number   int not null,
  etag          text not null,
  checksum      text,
  size_bytes    bigint not null,
  completed_at  timestamptz not null,
  primary key (session_id, part_number)
);

user_metadata (
  asset_id      text primary key references asset(id),
  description   text,
  tags          text[] not null default '{}',
  fields        jsonb not null default '{}',   -- { platform, sensor, mission, ... }
  source_path   text                           -- original path, if any; a facet, not a tree
);

technical_metadata (
  asset_id      text not null references asset(id),
  extractor     text not null,             -- exif|ffprobe|tika|sigmf|gdal|detect
  schema_ver    int not null,
  data          jsonb not null,
  extracted_at  timestamptz not null,
  primary key (asset_id, extractor)
);

derived_artifact (
  id            text primary key,
  asset_id      text not null references asset(id),
  kind          text not null,             -- thumb_512|thumb_2048|poster|pdf_page|spectrogram
  storage_key   text not null,
  width         int,
  height        int
);

enrichment_job (
  id            text primary key,
  asset_id      text not null references asset(id),
  extractor     text not null,
  state         text not null,             -- queued|running|ok|failed|skipped
  attempt       int not null default 0,
  error         text,
  started_at    timestamptz,
  finished_at   timestamptz
);
```

## User metadata vs technical metadata

They are not the same document.

| | User metadata | Technical metadata |
|---|---|---|
| Who writes it | The person uploading | An extractor |
| When | `POST /uploads` | After seal |
| Trusted as intent | Yes | No — it is observed |
| Schema | Small and stable + `fields` JSON | Per extractor, versioned |
| Example | `sensor: X-SAR`, tags `night` | `{ "center_frequency_hz": 9.6e9, "sample_rate": 2.0e8 }` |

A user can override a display title. A user cannot overwrite `ffprobe` output. If they disagree with the extractor, they add a user field. The two stay auditable.

## OpenSearch document

Flattened, rebuildable, not authoritative.

```json
{
  "asset_id": "ast_8K1",
  "collection_id": "col_mission_orbit",
  "title": "Orbit 12 — X-band take",
  "family": "radar.sigmf",
  "state": "ready",
  "tags": ["x-band", "orbit-12", "night"],
  "size_bytes": 1649267441664,
  "created_by": "shaked",
  "uploaded_at": "2026-10-31T10:12:00Z",
  "platform": "unmanned-aerial-vehicle-7",
  "sensor": "X-SAR",
  "duration_s": 412.4,
  "width": null,
  "height": null,
  "text": "Orbit 12 — X-band take  unmanned-aerial-vehicle-7  X-SAR  night"
}
```

ACL is not trusted from this document. The API hydrates allowed `collection_id`s from Postgres for the current principal and passes them as a filter.

## Custom fields

`user_metadata.fields` is open, but not untyped at the product level. A collection can declare a schema:

```json
{
  "collection_id": "col_mission_orbit",
  "fields": [
    { "key": "platform", "type": "enum", "values": ["unmanned-aerial-vehicle-7", "unmanned-aerial-vehicle-2", "ground"] },
    { "key": "sensor",   "type": "string", "required": true },
    { "key": "heading",  "type": "number" }
  ]
}
```

Unknown keys are still stored. Required keys block `POST /uploads`. That is how we stay structured without pretending every future file family is known.

## Why not a document database for the catalog

Collections, ACLs, sessions, and parts are relational. JSONB already gives extractors freedom. Introducing MongoDB would split the truth across two write paths for no gain.
