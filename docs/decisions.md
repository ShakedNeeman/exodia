# Decisions

Each decision is the choice, the reason, and what I refused.

## ADR 1 — Bytes never go through the API

**Decision.** Presigned multipart upload directly to object storage. The API signs and records. It does not stream.

**Why.** A 2 TB PUT through Spring is a second copy, a memory profile I do not want, and a scaling axis tied to NIC capacity of the API tier. Signing a URL is a metadata operation. Metadata operations and byte operations have different failure modes and should not share a process.

**Rejected.**

- *Proxy via the API* — simple to write, fatal at this size.
- *tus on our servers* — great protocol, wrong hop. tus in front of S3 still makes us the pipe.
- *S3 Transfer Acceleration as the whole design* — a network optimization, not an architecture.

## ADR 2 — Asset ≠ object

**Decision.** `asset` is the catalog record people browse. `asset_object` is one blob in storage, with a role.

**Why.** Radar, geospatial, and “folder” ingest are sets. Forcing one HTTP file to be one catalog row either explodes the UI or hides sidecars. A SIGMF take is one thing. It is two objects.

**Rejected.**

- *One row per blob, let the UI group them* — grouping becomes every client’s problem.
- *Zip everything at ingest* — destroys range reads, doubles CPU, and makes enrichment worse.

## ADR 3 — User metadata is committed in `draft`

**Decision.** `POST /uploads` requires collection + user metadata. The asset exists before any part lands.

**Why.** A 12-hour transfer that dies at 90% still has a name, a mission, and tags. Operations can see what is in flight. Browse of “in progress” is possible. Metadata is also the only way an `unknown` file is more than a hash.

**Rejected.**

- *Upload first, form later* — the form is never later.
- *Store metadata only as S3 object tags* — 10-tag / 2 KB limits. Unusable as a catalog.

## ADR 4 — Postgres is the truth. OpenSearch is a lens.

**Decision.** All writes commit in Postgres. OpenSearch is updated from events and can be dropped and rebuilt.

**Why.** Faceted browse over a large estate needs inverted indexes. Asset state, ACLs, and billing-adjacent size totals need a database that does not lose a write during a reindex.

**Rejected.**

- *OpenSearch as the only store* — I have seen this go wrong. Rebuilds become outages.
- *Postgres full-text only* — fine for thousands of rows, not for the facet combinations this UI needs.

## ADR 5 — Enrichment is a plugin pipeline, and it cannot fail the file

**Decision.** Detect → extract → derive → index. Extractors register for families. A crash or an unknown type leaves the asset `uploaded` or `partial`.

**Why.** The brief says the data is highly diverse and that we enrich *where possible*. Those two sentences together forbid a closed parser list and forbid “processing failed → file missing”.

**Rejected.**

- *Synchronous enrichment on complete* — a 2 TB spectrogram would make upload appear broken.
- *One Python service with a giant `if mime`* — every new format is a release of the whole worker.
- *LLM-only metadata* — useful later as an optional extractor, not as the foundation. Radar headers are not a prompt.

## ADR 6 — Adaptive part size, durable part ledger

**Decision.** Part size grows with the object so we stay well under 10,000 parts. Completed parts are written to Postgres (and IndexedDB). Resume is a protocol.

**Why.** 8 MB parts on a 4 TB file is 524,288 parts — illegal. 512 MB parts on a 200 MB file is a worse retry. The ledger is what makes “reliable” a property of the system, not of a lucky network.

**Rejected.**

- *Single PUT* — S3 single PUT is 5 GB. The requirement is terabytes.
- *Browser memory as the only resume state* — a refresh would restart a night-long job.

## ADR 7 — Two checksums

**Decision.** Part-level checksums on the storage API (CRC64-NVME / CRC32C). Session-level SHA-256 recorded by the client. A worker may compute a full-object digest after seal when a family needs a stable content id.

**Why.** Storage checksums catch corruption on the wire. The session hash is what we show humans and what we use to detect “this is the same take uploaded twice”. SHA-256 over a 2 TB file in the browser is expensive; we compute it incrementally per part and combine, or we accept the storage full-object CRC as the seal and hash asynchronously. Default: seal on storage CRC, content id asynchronously.

**Rejected.**

- *MD5 / ETag only* — S3 multipart ETags are not MD5 of the file. Treating them as a content hash is a lie.
- *No checksum, trust TCP* — we are storing tens of TBs. Bit rot and truncated last parts happen.

## ADR 8 — Browse is collections + facets, not a filesystem

**Decision.** The left rail is collections (missions, projects, libraries). The main view is a faceted result. Path, if present, is a filter.

**Why.** The brief asks to browse “in a logical way”. A reconstructed POSIX tree of tens of TBs is not logical. It is how the estate became unusable in the first place. People look for “X-band, last quarter, orbit-12”, not `/mnt/storage3/newnew_final/`.

**Rejected.**

- *Mirror the original folders as the product* — we can ingest the path as metadata without making it the navigation model.
- *Tags only, no collections* — tags drift. Collections have owners and ACLs.

## ADR 9 — Spring for the API, Python for files

**Decision.** Catalog API in Java 21 / Spring Boot 3. Enrichment in Python 3.12.

**Why.** The API is transactional, authenticated, and boring on purpose. The workers need GDAL, Tika, ffprobe, SIGMF, and the next parser someone writes next month. Those ecosystems are Python. Forcing one language would make one of the two layers worse.

**Rejected.**

- *Node for everything* — fine for the UI and acceptable for an API; weak for this enrichment set.
- *Java for enrichment* — Tika is great. The rest of the scientific/sensor stack is not.
- *Go API* — I would take it for a greenfield byte-oriented service. This API is a catalog. Spring is the better fit for the team that has to own it.

## ADR 10 — Amazon S3 is the data plane

**Decision.** All raw objects and derivatives live in Amazon S3. The client uploads with presigned multipart URLs. The API only creates keys and signs.

**Why.** S3 already has the contract this problem needs: multipart, checksums, lifecycle, IAM, and objects large enough for terabyte takes. A filesystem on the API hosts, or a NAS, does not.

**Rejected.**

- *NFS / SMB as the data plane* — no multipart, no signed URL, painful consistency.
- *Proxying bytes through the API* — turns the control plane into a bandwidth bottleneck.

## ADR 11 — Soft delete, versioned objects, bounded derivatives

**Decision.** Catalog delete is a state change. Storage keeps the object until a lifecycle policy expires it. Derivatives are capped (one poster, one spectrogram strip, one PDF page, two image sizes).

**Why.** Accidental delete of a 4 TB take is not recoverable from a recycle bin in the UI unless the object is still there. Unbounded derivatives are how enrichment quietly doubles the bill.

**Rejected.**

- *Hard delete on button press* — too fast for this estate.
- *Keep every transcode “just in case”* — not just in case; always in the invoice.
