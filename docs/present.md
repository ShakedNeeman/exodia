# Presentation — speaker notes

Seven slides. Live page: [`../ui/present.html`](../ui/present.html). Hebrew: [`present.he.md`](present.he.md).

The story: problem from the brief → how I solved each requirement → the idea → the flow → tools → decisions → screens.

| # | Slide | Say this |
|---|---|---|
| 1 | The problem | Not another disk. Get large, diverse files in, then find them without `/mnt/storage3`. |
| 2 | The requirements | Each line in the brief became a rule. Parts to S3. Metadata before bytes. Enrichment in the background. Logical browse. |
| 3 | The idea | I split bytes from meaning. I signed a URL. I did not stream a file. |
| 4 | End to end | Intent → transfer → seal → enrich. From seal the file is already downloadable. |
| 5 | Tools | React, Spring+Postgres, S3, Python, OpenSearch. Each one for a requirement. |
| 6 | Decisions | Not through the server. Metadata first. Unknown is a type. Not a folder tree. |
| 7 | Screens | Ingest, then Browse. UML only if they go deep. |
