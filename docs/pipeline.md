# One-file pipeline — what each store holds

Interactive walkthrough: [`../ui/pipeline.html`](../ui/pipeline.html).

Example: one-hour UAV pass over Khan Younis, `khan-younis-uav.mp4`, 8 GB, **32 parts** of 256 MB, **3 in flight**. S3 is not told the part count at create time — only the ordered etag list at Complete. Same design as 100 parts; only the `upload_part` row count changes.

| Step | What happens | Postgres | Redis | S3 | OpenSearch / SQS |
|---|---|---|---|---|---|
| 0 Before | File chosen, Start not pressed | 0 rows for this file | empty | no object | empty |
| 1 Intent | JSON to API, MPU opened | 1 asset `draft` + 1 object + 1 user_metadata + 1 session. **0 parts**. session.part_count=32 | **1 key** `progress:upl_99` pct=0 of=32 | uploadId `mpu_X`, 0 parts stored. S3 does not know 32 yet | empty |
| 2 In flight | PUT parts 1–3 to S3 | unchanged | unchanged | 3 PUTs in the air | empty |
| 3 Part 2 lands first | Out of order is fine | **1** `upload_part` row (part 2). asset → `uploading` | same 1 key, pct=3 | part 2 stored | empty |
| 4 Parts 1 and 3 land | Start parts 4–6 | **3** part rows | same 1 key, pct=9 | 1–3 stored, 4–6 in flight | empty |
| 5 All 32 parts | Transfer done | **32** part rows. session still `open` | pct=100 | 32 staging parts. **GET file = missing** | empty |
| 6 Seal | CompleteMultipart — first time S3 sees the full list | asset → `uploaded`. session `complete`. 32 part rows stay | **key deleted** | **one 8 GB object** | 1 tiny SQS message. no search doc |
| 7 Detect | Range GET 64 KB | 1 `technical_metadata` (detect). state `enriching` | idle | 64 KB read, not 8 GB | message in flight |
| 8 Extract | ffprobe header | 2 technical rows. duration_s=3600. user_metadata untouched | idle | small range | no search doc |
| 9 Derive | One poster frame | 1 `derived_artifact` | idle | + 200 KB poster. raw 8 GB unchanged | no search doc |
| 10 Index | Browse works | asset → `ready`. ~41 rows total for this file | 0 keys | raw + poster | **1 search document**. SQS deleted |

Counts to say out loud:

- Redis never has 32 keys. It has **one** key, overwritten, then deleted.
- Postgres catalog is **4 rows** at Start, plus **one row per finished part**.
- Enrichment does **not** download 8 GB. Headers and one frame only.
- S3 learns the part list only on Complete. Until then it just holds numbered staging parts.
- The file is downloadable from **step 6**, before enrichment.

## Failures

| Failure | Product | Technical |
|---|---|---|
| Part 4 drops | Bar pauses, then moves | Retry only that PUT. Other `upload_part` rows stay. |
| Tab closed at 9% | Resume at 9% | IndexedDB + Postgres list missing parts. New URLs if expired. |
| Complete checksum mismatch | "Corrupt, try again" | Abort MPU. `asset=failed`. Copy metadata to a new session. |
| Redis flushed | Percent blinks | Recount Postgres parts, rewrite the one key. No re-upload. |
| Worker dies | File already downloadable | SQS retry. After N: `partial`, not missing. |
| Unknown type | `unknown` badge, tags remain | detect ends; extractors skipped. A type, not an error. |
| Forgot Complete | Looks stuck | 14-day S3 abort of orphan parts so we do not pay forever. |
