---
name: Sync the DalCor English and French editions
description: Pull both language editions of dalcorpharma.com through the WPML wpml_language parameter, pair translations by slug, and detect where the French edition lags the English one.
api: openapi/dalcor-pharmaceuticals-content-openapi.yml
operations: [listPages, listPosts, getPage, getPost, listCategories]
generated: '2026-08-04'
method: generated
source: openapi/dalcor-pharmaceuticals-content-openapi.yml
---

# Sync the DalCor English and French editions

dalcorpharma.com runs WPML and publishes a French edition at `/fr/`. Every collection in the `wp/v2`
namespace accepts a `wpml_language` parameter, so both editions are retrievable from the same API.

**Base URL:** `https://dalcorpharma.com/wp-json`
**Auth:** none. **Send a browser `User-Agent`** or the edge returns 403.

## 1. The parameter

`wpml_language` takes `en` or `fr`. Omitting it returns the English (default) edition — so an
unparameterised crawl silently gets English only, which is the mistake to avoid.

```
GET /wp/v2/posts?wpml_language=en&per_page=100
GET /wp/v2/posts?wpml_language=fr&per_page=100
```

## 2. The editions are not the same size

Observed 2026-08-04 via `X-WP-Total`:

| collection | en | fr |
|---|---|---|
| posts | 24 | 6 |
| pages | 17 | 15 |

The French press-release stream carries a **quarter** of the English one. Treat the French edition as
a partial translation, not a mirror — do not assume a missing French record means the English one is
retired.

## 3. Pull both, pair by slug

`listPages` — `GET /wp/v2/pages?wpml_language={lang}&per_page=100&_fields=id,slug,title,link,parent,modified`
`listPosts` — `GET /wp/v2/posts?wpml_language={lang}&per_page=100&_fields=id,slug,date,modified,title,link`

Ids are **not** shared between translations. Pair on `slug` where WPML kept it, and fall back to
`date` + `featured_media` for posts whose French slug was localised. Where neither matches, record
the English record as untranslated rather than guessing a pairing.

## 4. Detect lag

For each paired record, compare `modified`. An English `modified` newer than its French counterpart
means the translation is stale. English records with no French pair are untranslated. Report both
counts — that is the deliverable.

`listCategories` — `GET /wp/v2/categories?wpml_language=fr` confirms whether the press-releases term
itself is translated before you filter on it; term ids may differ by language.

## 5. Bodies

`getPage` / `getPost` with `_fields=title,content,link,modified`. `content.rendered` is HTML in both
languages; strip tags. Keep each edition's own `link` — the French URLs sit under `/fr/`.

## Rules

- Never machine-translate an English record and present it as DalCor's French content. If a French
  record does not exist, say it does not exist.
- The accuracy rules in the *Assemble a DalCor clinical programme brief* skill apply to both
  languages — trial results must carry their qualifiers regardless of edition.
- Low-velocity surface with no ETag support; sync with `modified_after` and poll weekly at most.
