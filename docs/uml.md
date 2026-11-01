# High-level UML

Architecture overview. Interactive page: [`../ui/uml.html`](../ui/uml.html).

The assignment asked for an architecture overview — components, how they talk, where data lives at each stage — plus tools and key decisions. That is this file.

## What the system must do

- **Reliable TB uploads** — presigned multipart to Amazon S3. The API never carries file bytes.
- **Metadata at upload** — user fields are committed in `draft`, before the first part lands.
- **Enrich where possible** — async plugin workers. Unknown types stay first-class.
- **Logical browse** — collections and facets, not a reconstructed filesystem.

---

## 1. Component diagram

Control plane (client + API) talks metadata. Data plane (S3) talks bytes. Workers read S3 and write meaning back to Postgres / OpenSearch.

```mermaid
flowchart LR
  subgraph Control["«component» Control plane"]
    C["«component» Client<br/>React + TypeScript"]
    API["«component» Catalog API<br/>Spring Boot 3"]
  end

  subgraph Data["«component» Data plane"]
    S3["«component» Amazon S3<br/>raw / derived"]
  end

  subgraph Work["«component» Enrichment"]
    W["«component» Workers<br/>Python"]
  end

  C -->|"JSON: sessions, metadata, search"| API
  C -->|"presigned PUT / GET"| S3
  API -->|"CreateMPU · sign · Complete"| S3
  API --> PG[(PostgreSQL<br/>truth)]
  API --> Redis[(Redis<br/>live progress)]
  API --> SQS[SQS]
  API --> OS[OpenSearch<br/>projection]
  SQS --> W
  W -->|"range-read raw · write derived"| S3
  W --> PG
  W --> OS
```

| Component | Talks to | Data that lives there |
|---|---|---|
| **Client** | API (JSON), S3 (bytes) | Resume journal in IndexedDB, metadata being edited, parts in flight |
| **Catalog API** | Postgres, Redis, SQS, OpenSearch, S3 (control only) | Nothing durable except what it writes to stores |
| **Amazon S3** | Client, API, workers | Raw objects, multipart parts, derived previews |
| **PostgreSQL** | API, workers | Assets, sessions, ACL, user metadata, technical metadata |
| **Redis** | API | Live progress. Not the resume ledger |
| **SQS** | API, workers | `asset.uploaded` messages |
| **Workers** | S3, Postgres, OpenSearch, SQS | Temporary decode buffers |
| **OpenSearch** | API, workers | Flattened search documents. Rebuildable |

---

## 2. Sequence diagram — client, server, processing, storage

Four stages, as the brief asked. Amber path in the HTML is metadata. Green path is bytes.

```mermaid
sequenceDiagram
  actor Client
  participant API as Catalog API
  participant PG as PostgreSQL
  participant Redis as Redis
  participant S3 as Amazon S3
  participant SQS as SQS
  participant W as Workers
  participant OS as OpenSearch

  Note over Client,OS: 1. Intent — metadata first
  Client->>API: POST /uploads + user metadata
  API->>PG: INSERT asset draft, object, user_metadata, session
  API->>S3: CreateMultipartUpload
  Note right of S3: S3 is not told how many parts
  S3-->>API: uploadId
  API->>Redis: SET progress:session pct=0
  API-->>Client: session + presigned part URLs

  Note over Client,OS: 2. Transfer — bytes go to S3
  loop 3 parts in flight
    Client->>S3: PUT part N
    S3-->>Client: ETag
    Client->>API: part N landed (etag, crc)
    API->>PG: INSERT upload_part
    API->>Redis: SET progress:session (overwrite)
  end

  Note over Client,OS: 3. Seal — S3 assembles
  Client->>API: POST complete + checksums
  API->>S3: CompleteMultipartUpload
  S3-->>API: object assembled
  API->>PG: asset=uploaded, session=complete
  API->>Redis: DEL progress:session
  API->>SQS: asset.uploaded
  API-->>Client: uploaded — download allowed

  Note over Client,OS: 4. Enrich — after the file exists
  SQS->>W: asset.uploaded
  W->>S3: range GET header
  W->>PG: INSERT technical_metadata
  W->>S3: PUT derived/poster
  W->>PG: INSERT derived_artifact
  W->>OS: PUT search document
  W->>PG: asset=ready

  Note over Client,OS: Browse
  Client->>API: GET /search
  API->>OS: query
  API-->>Client: hits + signed GET
  Client->>S3: GET derived artifact
```

S3 stores parts and concatenates them on Complete. The client and API drive the session. S3 does not run SQL transactions and does not manage video.

---

## 3. Class diagram — where data lives

```mermaid
classDiagram
    class Collection{
        +id
        +name
        +acl
    }
    class Asset{
        +id
        +title
        +lifecycle
        +family
        +size_bytes
    }
    class AssetObject{
        +role
        +storage_key
        +checksum
    }
    class UploadSession{
        +multipart_upload_id
        +part_size
        +part_count
    }
    class UserMetadata{
        +tags
        +fields
        +source_path
    }
    class TechnicalMetadata{
        +extractor
        +schema_ver
        +data
    }
    class DerivedArtifact{
        +kind
        +storage_key
    }
    class EnrichmentJob{
        +job_state
    }
    Collection --> Asset
    Asset --> AssetObject
    Asset --> UserMetadata
    Asset --> TechnicalMetadata
    Asset --> DerivedArtifact
    Asset --> EnrichmentJob
    Asset --> UploadSession
```

- **Asset** is logical — what people browse.
- **AssetObject** is physical — what S3 holds.
- **UserMetadata** is intent — written at `POST /uploads`.
- **TechnicalMetadata** is observation — written after seal.

---

## 4. State machine

```mermaid
stateDiagram-v2
    [*] --> Draft
    Draft --> Uploading : first_part
    Uploading --> Uploaded : complete
    Uploading --> Failed : abort
    Draft --> Failed : abort
    Uploaded --> Enriching : worker
    Enriching --> Ready : ok
    Enriching --> Partial : unknown
    Partial --> Ready : rerun
    Failed --> Draft : retry
```

Download is allowed from `uploaded` onward. Enrichment never hides the file. `failed` is integrity / session only.

---

## 5. Tools

| Layer | Choice | Why |
|---|---|---|
| Client | React + TypeScript | Long-lived upload queue and inspector |
| API | Java 21, Spring Boot 3 | Transactional catalog. Metadata, not bytes |
| Workers | Python 3.12 | exif, ffprobe, Tika, SIGMF, GDAL |
| Storage | Amazon S3 | Native multipart, checksums, lifecycle, 50 TB objects |
| Truth | PostgreSQL 16 | Relations and a real state machine |
| Search | OpenSearch | Facets. Rebuildable. Never truth |
| Queue | Amazon SQS | Seal must survive a worker crash |
| Progress | Redis | Whiteboard only. Ledger is Postgres |

---

## 6. Decisions I would not reverse

1. **Control plane ≠ data plane.** Bytes never go through the API.
2. **Asset ≠ object.** A SIGMF take is one catalog row and two blobs.
3. **Enrichment is best-effort.** The brief says “where possible”. Unknown is a type.
4. **Browse is a catalog.** Collections + facets. The old path is a filter, not the product.
