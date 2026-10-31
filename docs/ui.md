# UI

Two screens. Upload and browse. Designed as a working product surface, implemented here as high-fidelity HTML/CSS mockups so the review happens in a browser, not as a slide.

`ui/upload.html`  
`ui/browse.html`

## Why not a filesystem UI

A drive-style tree is the thing this estate already has, and it is the thing that does not work at tens of terabytes. The product is a catalog:

- **Collections** are the logical roots (a mission, a library, a campaign).
- **Facets** narrow by family, tag, time, sensor, enrichment state.
- **Inspector** shows one asset deeply — preview, user metadata, technical metadata — without leaving the grid.

Path, when we have it, is a chip. It is not the information architecture.

## Upload

The job of this page is to make a terabyte transfer feel controlled.

1. **Metadata is on the right, and it is first.** Collection, title, tags, custom fields. The primary action is “Start ingest”, not “pick files and hope”.
2. **The drop zone accepts files and folders.** A SIGMF pair is detected and grouped as one asset with a sidecar.
3. **The queue is honest.** Each row shows size, part plan, percent, throughput, checksum state, and pause/resume. A 1.5 TB row that says “uploading…” with no numbers is a failed design.
4. **Integrity is visible.** `sealed`, `checksum mismatch`, `waiting for parts` are states, not toast messages.
5. **In-flight assets already have a name.** Because metadata was committed in `draft`, a long transfer can be found in browse as “uploading”.

What I left out on purpose: a fake “cloud features” panel, a marketing hero, a wizard with five steps. Ingest is one screen with a queue.

## Browse

The job of this page is to make a large, mixed estate feel navigable in under three clicks.

1. **Left: collections and saved views.** “Orbit missions”, “Lab docs”, “Unsorted” — the last one is deliberate and should stay small.
2. **Top: search + facets.** Family, enrichment, time, size. Facets are how diversity becomes filterable instead of noisy.
3. **Center: grid or list.** Cards lead with family and a derivative (poster, page, spectrogram), not with a generic file icon.
4. **Right: inspector.** Opens on select. User metadata on top (intent). Technical metadata below (observation). Extractor rows can be `ok`, `running`, or `skipped`.
5. **Unknown files look finished, not broken.** An `unknown` badge is a type, not an error. The user metadata is still the way in.

## Visual language

Dark, dense, monospaced where numbers matter. This is closer to a control room than to a consumer drive.

| Token | Value | Use |
|---|---|---|
| Background | `#0C0E12` | Page |
| Surface | `#13171D` | Cards, rails |
| Line | `#2A3340` | Hairlines |
| Text | `#EEF2F6` | Primary |
| Muted | `#8D98A6` | Labels |
| Accent | `#F0A14A` | Ingest, focus, radar |
| Signal | `#3DDC97` | Sealed / ready |
| Type | Syne + IBM Plex Sans + IBM Plex Mono | Product / UI / numbers |

Amber is the ingest color on purpose. Green is reserved for integrity success. We do not paint the whole UI green and then have nothing left for “the checksum matched”.

## Interaction notes (mockup)

The HTML is not backed by an API. It does behave enough to read:

- Selecting a file in the queue highlights it and binds the metadata pane.
- Grid cards open the inspector.
- Facet chips toggle.
- List / grid switch changes the main view.

That is enough to walk the two flows in a design review. It is not a prototype of the upload protocol.