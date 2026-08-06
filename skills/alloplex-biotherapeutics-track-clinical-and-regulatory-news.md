---
name: Track Alloplex Biotherapeutics clinical, regulatory and scientific news
description: >-
  Build and incrementally poll an index of Alloplex Biotherapeutics' 61 published posts — FDA
  designations, SUPLEXA-101 Phase 1 milestones, scientific publications, conference appearances and
  media coverage — using category filters and modified_after for cheap re-sync.
api: openapi/alloplex-biotherapeutics-content-openapi.yml
operations: [listCategories, listPosts, getPost, search]
generated: '2026-08-06'
method: generated
---

# Track Alloplex Biotherapeutics clinical and regulatory news

Alloplex Biotherapeutics is a clinical-stage cellular immunotherapy company. Its news archive is the
public record of SUPLEXA's regulatory and clinical progress. This surface is the company's own
WordPress REST content API — **there is no developer program, no API key, and no support channel for
it.** Read-only, anonymous, and it can change without notice when the site is upgraded.

**Base URL:** `https://alloplexbio.com/wp-json`
**Auth:** none. Send no `Authorization` header.

## 1. Resolve the categories first

All editorial grouping lives in categories. **The `tags` taxonomy is registered but empty (0
terms)** — never filter on it.

Call `listCategories` (`GET /wp/v2/categories?per_page=100&_fields=id,slug,name,count`). Eleven
terms existed at harvest time:

| slug | count | what it holds |
|---|---|---|
| `conferences` | 19 | conference appearances and presentations |
| `scientific-news` | 17 | science and platform news |
| `company-news` | 16 | corporate announcements |
| `opinion` | 13 | opinion and commentary pieces |
| `diary-marker` | 13 | dated diary entries |
| `media-coverage` | 12 | third-party press coverage |
| `clinical-news` | 11 | trial and regulatory milestones |
| `research-news` | 5 | research updates |
| `backgrounder` | 4 | background explainers |
| `audio-video` | 2 | audio and video assets |
| `uncategorized` | 0 | empty |

Resolve slugs to ids at runtime — **do not hardcode the ids**, they are site-local and the term set
can change.

## 2. Page the archive

Call `listPosts`. Use sparse fieldsets to keep the index cheap, then fetch bodies only for the posts
you care about:

```
GET /wp/v2/posts?per_page=100&page=1&_fields=id,slug,date,modified,title,link,categories
```

- `per_page` maxes at **100**. Outside 1–100 you get `400 rest_invalid_param` with
  `data.details.per_page.code = rest_out_of_bounds`.
- **The body is a bare JSON array with no envelope.** Totals live only in headers: read
  `X-WP-Total` (61 at harvest) and `X-WP-TotalPages`, and follow the `Link` header's `rel="next"`.
  A client reading only the body cannot tell whether more pages exist.
- Filter to regulatory and trial news with `categories=<clinical-news id>`; combine ids for a
  broader sweep.
- Bound by date with `after` / `before` (ISO 8601).

## 3. Read a post

Call `getPost` (`GET /wp/v2/posts/{id}`). Unlike many corporate WordPress sites,
**`content.rendered` and `excerpt.rendered` are fully populated here**, so you get real prose
directly from the API — no HTML scraping needed for posts.

## 4. Keyword search

Call `search` (`GET /wp/v2/search?search=SUPLEXA&subtype=any&per_page=100`) for a fast lookup. It
returns lightweight results — note `title` is a **plain string** here, not a `{rendered: ...}`
object, unlike every other collection. Follow `_links.self` for the full object.

## 5. Re-sync incrementally

Two mechanisms, use both:

- `GET /wp/v2/posts?modified_after=<ISO8601>&_fields=id,modified` returns only changed posts.
- Responses carry `Last-Modified` and `cache-control: max-age=600, must-revalidate`. Honour the
  600-second freshness window; this deployment sits behind Cloudflare and WP Engine, so polling
  faster buys you nothing but a cached copy.

## Traps

- **Author is unresolvable.** Every post has a populated `author` integer, but `/wp/v2/users`
  returns `401 rest_user_cannot_view`. There is no API route to the author's name — it appears only
  in the JSON-LD graph in the page HTML.
- **`tags` is always empty.** Use `categories`.
- **Publications and conferences are not in this API.** The site registers `publication`,
  `team_member`, `conference` and `resource` post types and lists them in its sitemaps, but none is
  REST-registered. `GET /wp/v2/publication` returns `404 rest_no_route`. For those, parse
  `https://alloplexbio.com/publication-sitemap.xml` and
  `https://alloplexbio.com/conference-sitemap.xml` and fetch the HTML.
- **No rate-limit headers exist.** Nothing tells you when you are throttled. `robots.txt` asks for
  `Crawl-delay: 10`; respect it as the only published throughput expectation.
- **Errors are not RFC 9457.** Branch on the `code` string in the bespoke WordPress envelope
  `{code, message, data.status}` — never on `message`, which is localized and unstable.
