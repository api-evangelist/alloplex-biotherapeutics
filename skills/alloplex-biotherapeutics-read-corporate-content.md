---
name: Read Alloplex Biotherapeutics corporate and scientific content
description: >-
  Enumerate and read the 19 published corporate pages behind alloplexbio.com, and fall back to HTML
  for the one page with an empty body and for the four content types the API does not expose.
api: openapi/alloplex-biotherapeutics-content-openapi.yml
operations: [listTypes, listPages, getPage, search]
generated: '2026-08-06'
method: generated
---

# Read Alloplex Biotherapeutics corporate content

Alloplex Biotherapeutics is a clinical-stage cellular immunotherapy company (founded 2016, Woburn
MA, with a subsidiary in Adelaide, Australia) developing SUPLEXA, produced by its ENLIST
cell-retraining platform. Its corporate story lives in 19 WordPress pages.

**Base URL:** `https://alloplexbio.com/wp-json`
**Auth:** none. This is the site's own content surface, not a developer product.

## 1. Confirm what is actually exposed

Call `listTypes` (`GET /wp/v2/types`) before assuming a collection exists. Twelve post types are
REST-registered; only `post`, `page` and `attachment` carry public content.

**Check this first, because the site's sitemaps lie about the API.** `sitemap_index.xml` advertises
`publication`, `team_member`, `conference` and `resource` sitemaps, but none of those post types is
REST-registered. `GET /wp/v2/publication` returns `404 rest_no_route`.

## 2. List the pages

```
GET /wp/v2/pages?per_page=100&_fields=id,slug,title,link,parent,menu_order,modified
```

19 pages at harvest time, **all top-level** (`parent` is 0 on every one), so there is no hierarchy
to walk. Read `X-WP-Total` for the count — the body is a bare array with no envelope.

The substantive ones:

| slug | id | page |
|---|---|---|
| `about` | 14 | About Alloplex — company, leadership, platform |
| `scientists` | 209 | the science of retrained immunity |
| `investors` | 210 | investment case |
| `information-for-patients` | 372 | patient-facing information |
| `newsroom` | 611 | Media and Press |
| `in-the-news` | 610 | third-party coverage index |
| `presskitoct24` | 642 | Alloplex Media Kit |
| `releases-and-updates` | 138 | news index |
| `faq` | 1019 | Frequently Asked Questions |
| `terms` | 106 | Terms of use |
| `privacy-policy` | 3 | Privacy Policy |
| `contact` | 9 | Contact |

Resolve ids by slug at runtime rather than hardcoding them.

## 3. Read a page

Call `getPage` (`GET /wp/v2/pages/{id}`). `content.rendered` is populated on most pages, so you get
real prose from the API.

**The one exception: the FAQ page (id 1019) returns an empty `content.rendered`.** Its body is
authored in page-builder post meta that WordPress does not project into REST. For that page, fetch
the HTML at its `link` instead. Apply the same fallback to any page where `content.rendered` comes
back as `""` — treat empty as "not available here", never as "no content".

## 4. Get the structured company facts

The REST API does **not** carry the site's schema.org graph. Fetch the HTML at a page's `link` and
parse the `<script type="application/ld+json">` block for the `Person`/`Organization` node. A
verbatim copy of the homepage graph is in this repo at
`json-ld/alloplex-biotherapeutics-organization.jsonld`.

Be aware it is thin: it carries `name` and `logo` but **no `sameAs`, no address, no founding date
and no legal name**. Richer company facts (founding year, headquarters, subsidiary, leadership
names and titles) exist only as prose on `/about/`.

## 5. Search

Call `search` (`GET /wp/v2/search?search=ENLIST&subtype=page`) to locate a page by keyword without
paging the whole collection.

## Traps

- **Author is unresolvable** — `/wp/v2/users` returns `401 rest_user_cannot_view`.
- **`/wp/v2/settings` returns 401**, so the site tagline, timezone and language are not readable
  there. Get them from the REST index at `GET /wp-json/` instead, which is anonymous and returns
  `name`, `url`, `home`, `gmt_offset` and `timezone_string`.
- **The shareholder portal at `/portal-login/` is a human web login for investors.** It is not an
  API credential path and issues nothing you can use here.
- **Do not treat this as a supported API.** There is no versioning policy, no deprecation policy, no
  SLA and no status page. The contract changes whenever WordPress or a plugin is upgraded.
