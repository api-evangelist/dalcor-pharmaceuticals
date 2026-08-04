---
name: Track DalCor press releases
description: Incrementally pull DalCor Pharmaceuticals' press-release stream from the site's anonymously readable WordPress REST content API, with category filtering, modified_after sync, pagination limits and featured-image resolution.
api: openapi/dalcor-pharmaceuticals-content-openapi.yml
operations: [listCategories, listPosts, getPost, listMedia, getMediaItem]
generated: '2026-08-04'
method: generated
source: openapi/dalcor-pharmaceuticals-content-openapi.yml
---

# Track DalCor press releases

DalCor's entire news stream is reachable as JSON from `https://dalcorpharma.com/wp-json` — no
scraping, no credentials.

**Base URL:** `https://dalcorpharma.com/wp-json`
**Auth:** none.
**Required:** a normal browser `User-Agent`. The Cloudflare edge answers an absent or default
User-Agent with **HTTP 403**, including on the JSON routes. This is the failure you will hit first.

## 1. Find the category

`listCategories` — `GET /wp/v2/categories`

Two terms exist. Take the one with slug `press-releases` — id **13**, 24 posts as of 2026-08-04.
`uncategorized` (id 1) is empty. Do not hardcode 13 without checking; read the slug.

## 2. List the stream

`listPosts` — `GET /wp/v2/posts?categories=13&per_page=100&orderby=date&order=desc&_fields=id,slug,date,modified,title,excerpt,link,featured_media`

- `per_page` is capped at **100**; asking for more returns `400 rest_invalid_param`.
- Read `X-WP-Total` and `X-WP-TotalPages` from the response headers, or follow the RFC 8288 `Link`
  header's `rel="next"`. Paging past the last page returns `400 rest_post_invalid_page_number` —
  stop at `X-WP-TotalPages`, do not probe for the end.
- With 24 records the whole stream fits in one call.

## 3. Sync incrementally

On every run after the first, pass the highest `modified` timestamp you have already stored:

`GET /wp/v2/posts?categories=13&modified_after=2026-01-01T00:00:00&orderby=modified&order=asc`

There is **no** `ETag` or `Last-Modified` on collections, so conditional requests are not available —
`modified_after` is the sync mechanism. Also note this is a low-velocity surface: the most recent
post observed on 2026-08-04 was dated **2024-12-11**. Poll daily at most; weekly is plenty.

## 4. Get the body

`getPost` — `GET /wp/v2/posts/{id}?_fields=title,content,date,modified,link`

`title.rendered`, `content.rendered` and `excerpt.rendered` are **HTML strings inside objects**, not
plain text. Strip tags before indexing. Keep `link` — it is the citable public URL.

A bad id returns `404 rest_post_invalid_id`.

## 5. Images

`featured_media` is a media id (0 when unset). Resolve it either with

- `getMediaItem` — `GET /wp/v2/media/{id}?_fields=source_url,alt_text,mime_type`, or
- `_embed` on the post request, which inlines it under `_embedded['wp:featuredmedia']` and saves a
  round trip.

`listMedia` — `GET /wp/v2/media?per_page=100&_fields=id,slug,title,source_url,mime_type` returns all
54 attachments if you want the whole library, including scientific-congress poster assets published
as both JPEG and PDF.

## 6. Do not try to attribute posts to authors

Every post carries an `author` id, but `/wp/v2/users` is **blocked at the edge** and returns an HTML
`403` — not the JSON error envelope. The author reference is permanently dangling on this
deployment. Attribute to "DalCor Pharmaceuticals", not to a person.

## Rules

- Cite the `link`, not the API URL.
- These are pharmaceutical press releases about an **investigational** compound. Dalcetrapib is not
  approved. Never restate a trial result without its qualifiers, and never present a post-hoc or
  subgroup finding as a primary endpoint result — the dal-GenE primary composite endpoint was **not**
  statistically significant (HR 0.88, p=0.12); the 21% MI reduction was a secondary finding.
- No rate-limit headers are published. Back off on any 403 or 429 and keep concurrency at 1.
