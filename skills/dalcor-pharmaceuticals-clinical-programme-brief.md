---
name: Assemble a DalCor clinical programme brief
description: Pull DalCor Pharmaceuticals' science, dalcetrapib history, precision-medicine rationale and the dal-GenE / Dal-GenE-2 trial pages into one structured brief, plus the leadership and board roster, using only the site's anonymously readable content API.
api: openapi/dalcor-pharmaceuticals-content-openapi.yml
operations: [getNamespaceIndex, listTypes, listPages, getPage, search, listPosts]
generated: '2026-08-04'
method: generated
source: openapi/dalcor-pharmaceuticals-content-openapi.yml
---

# Assemble a DalCor clinical programme brief

Everything DalCor publishes about dalcetrapib, the ADCY9 precision-medicine thesis and its trial
programme is reachable as JSON from `https://dalcorpharma.com/wp-json`.

**Base URL:** `https://dalcorpharma.com/wp-json`
**Auth:** none. **Send a browser `User-Agent`** or the edge returns 403.

## 0. Confirm the surface

`getNamespaceIndex` — `GET /wp/v2` returns all 118 registered routes with their arguments.

Use this, **not** `GET /` (`getSiteIndex`). On this deployment the root index returns empty
`namespaces` and `routes` arrays — the standard WordPress discovery entry point is suppressed here.

`listTypes` — `GET /wp/v2/types` confirms the content types. Important negative result: DalCor
registers **no custom content types**. There is no `team`, `publication`, `trial` or `pipeline` type.
Everything below lives in `page`.

## 1. The page map

`listPages` — `GET /wp/v2/pages?per_page=100&_fields=id,slug,title,link,parent,menu_order`

17 English pages as of 2026-08-04, in a two-level hierarchy:

| id | slug | parent | what it holds |
|---|---|---|---|
| 60 | `about` | — | company overview, licence position, ADCY9 genotype prevalence |
| 75 | `management` | 60 | executive team bios |
| 110 | `board-of-directors` | 60 | board bios |
| 135 | `partners` | 60 | partners (currently near-empty — expect no text) |
| 403 | `science` | — | science landing |
| 149 | `dalcetrapib` | 403 | Dalcetrapib History — dal-OUTCOMES, the 2012 Montreal Heart Institute ADCY9 finding |
| 144 | `cardiovascular-diseases` | 403 | disease burden framing |
| 154 | `precision-medicine-for-cardiovascular-diseases` | 403 | dal-PLAQUE-2, IMT, cholesterol efflux, hs-CRP rationale |
| 159 | `publications` | 403 | publication list |
| 293 | `dal-gene-trial` | — | dal-GenE (DAL-301) design and results |
| 594 | `dal-gene-2-trial` | — | Dal-GenE-2 (DAL-302) design and timeline |
| 1260 | `dal-302-study` | — | information for clinical trial sites |
| 660 | `press-releases` | — | news index page |
| 298 | `contact-us` | — | four offices, phones, info@DalCorpharma.com |
| 1196 | `privacy-policy` | — | privacy policy |
| 11 | `home` | — | homepage |
| 1337 | `walgreens-referral-program` | — | referral programme page |

Ids drift when pages are rebuilt — resolve by `slug` (`GET /wp/v2/pages?slug=dal-gene-2-trial`)
rather than by id.

## 2. Pull the bodies

`getPage` — `GET /wp/v2/pages/{id}?_fields=title,content,link,modified`

`content.rendered` is HTML built by Elementor. Strip tags. The whitespace is noisy; normalise runs of
whitespace before extraction.

## 3. People come out of HTML, not out of records

There is no people content type. The management and board rosters are prose inside
`pages/75` and `pages/110` — name, then an ALL-CAPS title line, then a paragraph. Parse the heading
structure of `content.rendered`; do not expect structured fields, and do not invent a person's title
if the parse is ambiguous.

## 4. Fill gaps by search

`search` — `GET /wp/v2/search?search=ADCY9&per_page=20&subtype=page,post`

Returns lightweight `{id, title, url, type, subtype}` records across pages and posts. Use it to find
mentions you did not anticipate, then `getPage`/`getPost` for the body.

## 5. Add recent news

`listPosts` — `GET /wp/v2/posts?categories=13&per_page=10&_fields=title,date,link,excerpt` for the
"what changed" section. See the *Track DalCor press releases* skill for the full sync pattern.

## Accuracy rules — read before writing anything

This is an investigational drug. The brief must survive a clinician reading it.

- **Dalcetrapib is not approved anywhere.** Always say "investigational".
- **dal-GenE (DAL-301) missed its primary endpoint.** The prespecified primary composite (CV death,
  resuscitated cardiac arrest, non-fatal MI, non-fatal stroke) was not statistically significant:
  9.5% vs 10.6%, HR 0.88, 95% CI 0.75–1.03, **p=0.12**. The 21% relative risk reduction in fatal and
  non-fatal MI (5.9% vs 7.3%, HR 0.79, p=0.02) is a **secondary** result. Reporting the second
  without the first misrepresents the trial.
- **dal-PLAQUE-2 findings on this page are post-hoc.** Label them as such.
- **Dal-GenE-2 (DAL-302) is ongoing**, not complete — 2,000 post-ACS patients with the ADCY9 AA
  genotype, coordinated by the Montreal Health Innovations Coordinating Centre; the site states first
  patient expected Q3 2023, interim efficacy analysis 2026, completion anticipated 2027. Those are
  the company's stated projections; check the press-release stream and a trial registry before
  treating any of them as current.
- Cite the page `link`, carry the `modified` date, and say when you fetched it.
