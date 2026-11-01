# Presentation — speaker notes

Nine slides. Live page: [`../ui/present.html`](../ui/present.html) (EN / עב, arrow keys). Hebrew notes: [`present.he.md`](present.he.md).

| # | Slide | Say this |
|---|---|---|
| 1 | A catalog in front of a store | Storage is not the problem. They already have terabytes. The problem is getting diverse files in, then finding them. Bytes live in S3. Meaning lives in Postgres. |
| 2 | Four requirements | Reliable ingest, metadata at upload, enrich where possible, logical browse. Every choice maps to one of these. |
| 3 | Two planes | I signed a URL. I did not stream a file. Control plane = JSON. Data plane = parts to S3. S3 assembles; it does not run SQL or manage video. |
| 4 | Four stages | Intent → Transfer → Seal → Enrich. The file is usable from `uploaded`. A failed parser does not hide it. |
| 5 | Postgres vs Redis | Notebook vs speedometer. Ask Redis for the percent. Ask Postgres which parts are missing. Never the other way around. |
| 6 | Same moment, two writes | Part 3812: a new `upload_part` row in Postgres (resume), overwrite `progress:upl_44c` in Redis (the bar). |
| 7 | If one dies | Redis flush: bar vanishes, upload continues. Worker dies: file already uploaded. Postgres lost: that is the disaster. |
| 8 | Tools | S3 warehouse. Postgres notebook. Redis gauge. Spring catalog. Python formats. OpenSearch index. |
| 9 | Close | Bytes out of the API. Metadata first-class. Unknown is a type. Browse is a catalog. Then open ingest + browse. Deep questions → UML. |
