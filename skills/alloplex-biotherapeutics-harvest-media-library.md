---
name: Harvest the Alloplex Biotherapeutics media library
description: >-
  Page through the 225-item media library behind alloplexbio.com for logos, scientific figures,
  press imagery and PDF documents, with source URLs, MIME types, alt text and size variants.
api: openapi/alloplex-biotherapeutics-content-openapi.yml
operations: [listMedia, getMediaItem]
generated: '2026-08-06'
method: generated
---

# Harvest the Alloplex Biotherapeutics media library

The media library is the richest collection on this API — 225 attachments at harvest time, including
the company logo, scientific figures, press imagery and PDF documents such as the
`Alloplex_Executive_Summary_July_2026.pdf`.

**Base URL:** `https://alloplexbio.com/wp-json`
**Auth:** none.

## 1. Page the library

```
GET /wp/v2/media?per_page=100&page=1&_fields=id,slug,media_type,mime_type,source_url,alt_text,title,modified
```

- `per_page` maxes at **100**, so 225 items means three pages.
- Read `X-WP-Total` (225) and `X-WP-TotalPages` from the headers — the body is a bare array.
- Follow the `Link` header's `rel="next"` rather than incrementing `page` blindly.

## 2. Filter by kind

- `media_type=image` for figures and photography.
- `mime_type=application/pdf` for documents. This is how you find the executive summary and any
  press-kit collateral without walking the whole library.

## 3. Resolve the file

`source_url` is the direct, publicly fetchable file URL — no further resolution needed:

```
https://alloplexbio.com/wp-content/uploads/2026/07/Alloplex_Executive_Summary_July_2026.pdf
```

For images, `media_details.sizes` carries pre-rendered variants (thumbnail, medium, large and any
theme-registered sizes), each with its own `source_url`, `width` and `height`. **Use the variant
that fits your need rather than downloading the full-size original** — the originals on this site
include large scientific figures.

## 4. Attribute correctly

- `alt_text` and `caption.rendered` are the publisher's own descriptions — prefer them over
  generating your own.
- `post` links an attachment back to the post or page it was uploaded to (null for library-level
  uploads). Resolve it with `getPost` / `getPage` for context.
- These are **Alloplex Biotherapeutics' copyrighted assets**. The API being anonymous is not a
  licence. Check `/terms/` before redistributing, and do not treat scientific figures as free stock.

## 5. Re-sync incrementally

```
GET /wp/v2/media?modified_after=<ISO8601>&_fields=id,modified,source_url
```

Responses carry `Last-Modified` and `cache-control: max-age=600, must-revalidate` — honour the
freshness window rather than re-walking all three pages.

## Traps

- **Media ids are sparse.** They share the global post-id sequence, so low ids frequently do not
  exist (`GET /wp/v2/media/1` returns `404 rest_post_invalid_id`). Always enumerate via `listMedia`;
  never iterate ids.
- **`media_type` is `file`, not `document`, for PDFs.** Filter on `mime_type` when you want
  documents specifically.
- **No rate-limit headers.** `robots.txt` asks for `Crawl-delay: 10` — pace a 225-item harvest
  accordingly.
