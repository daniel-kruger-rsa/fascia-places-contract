# Fascia Places API v1

Host-facing geography rail. Hosts consume this; they do not store polygons.

A **host** is a platform that builds Places into its own product. Its members use the host's app, and the host's backend calls this API on their behalf with a host key, never from a client. The host owns its members, its communities and which cells they cover; Places answers questions about geography and knows nothing of members or communities.


- **Base:** `https://places.fascia.network` (production), `http://localhost:8080` (local compose)
- **Version prefix:** `/v1/places`
- **Pack generation (ZA):** `mdb-zaf-2020`
- **Every response** carries the header `x-places-generation: <generation>`; polygon rows and FeatureCollections also carry `generation` in the body.
- **Machine contract:** [`places.v1.yaml`](places.v1.yaml) — where this prose and the contract disagree, the contract wins. Published at [fascia-places-contract](https://github.com/daniel-kruger-rsa/fascia-places-contract), tagged per version so a vendored copy can pin one

## Auth

| Path | Auth |
|---|---|
| `/health`, `/v1/places/generation`, `/v1/places/generations`, `/v1/places/coverage`, `/v1/places/layers` | none |
| Everything else under `/v1/places` | `Authorization: Bearer <host-key>` |

The host key is issued per host and is **server-to-server only**: call Places from your backend. It must never ship in a mobile or web bundle.

Unknown or missing key → `401 { "error": { "code": "unauthorized" } }`. Keys are compared in constant time.

`/` returns 404 in production. Locally it serves a Leaflet tester when `SERVE_UI=1`.

Every read accepts an optional **`generation`** parameter, resolved **within the country asked for**. Omit it and you get that country's default (`GET /v1/places/generation?country=ZA`); name one that country's pack does not hold and the answer is `400 bad_request` — including when the generation exists for another country, because a generation of one pack is not a generation of another. `x-places-generation` reports the generation the response was served from. This is what makes a cutover possible: load the new generation beside the old, compare, then move `PACK_GENERATION`.

Common validation, all `400 bad_request`: `country` must be a two-letter code (default: the deployment's default country, `ZA` in production); `kind` must be one of the layer codes or names; a bbox must have all four edges, `west < east`, `south < north`, within WGS84 range.

### A cell may have no geometry, and `within` says so rather than going quiet

A pack can hold a cell's **name, codes and parent chain without a polygon**. The lawful boundary source for a country may not publish a cell at the level a host needs — India is the case, where the official boundary database carries no urban cell below the municipality and its conditions forbid altering the dataset — so the names exist and the geometry does not.

Two consequences for a reader:

- **`bbox` is omitted** on an `/areas/search` hit for such a cell, rather than returned as four nulls. A client that indexes it unconditionally will read `undefined`.
- **`within` resolves by ancestry** when its target has no polygon: is the target anywhere in the candidate's parent chain. For a pack built from one hierarchy that is containment, not an approximation of it. Where the target *has* a polygon, geometry still decides — South African sub-places and wards come from different sources and genuinely do not nest, so containment remains the truthful test.

**`400 within_unsupported`** when the target has neither geometry nor anything claiming it as a parent:

```json
{ "error": { "code": "within_unsupported",
             "message": "this pack cannot constrain a search to that area…",
             "within": "…" } }
```

Neither mechanism can say what is inside it, and an empty list would be indistinguishable from *no matches* — a host following the right rule would read it as the name being absent from the region. Treat it as *this pack cannot answer that question*.

### A pack may withhold its geometry

Some sources permit their boundaries to be **used** and not **redistributed**. Survey of India is the case: no prior permission is needed to use the Administrative Boundary Database, and altering the dataset is forbidden — so the India pack answers which cell contains a point and never hands over the polygon that decided it.

For such a country, the three endpoints that would return a polygon answer **`403 geometry_not_served`**:

```json
{ "error": { "code": "geometry_not_served",
             "message": "this pack may not serve geometry for IN…",
             "country": "IN",
             "attribution": [ { "source": "Survey of India", "…": "…" } ] } }
```

`/v1/places/geojson`, `/v1/places/area/{area_id}/geojson`, `/v1/places/street/{street_id}/geojson`.

**Everything that is not a polygon still works.** `/point` resolves a coordinate to a cell, `/areas/search` returns cells with their parent chains, `/area/{area_id}` returns the cell. A host that draws boundaries needs a fallback for such a country; a host that picks a cell from a list does not notice.

**And the credit rides along.** Where a source requires acknowledgement, the credit joins `attribution` on every response with room to carry one, beside the layer credits rather than instead of them: `/point`, `/area/{area_id}`, `/area/{area_id}/neighbours`, `/area/{area_id}/geojson`, `/v1/places/geojson`, `/harvest?summary=1`, `/building/{building_id}`, and the `403` itself.

**South Africa carries one as of 0.21.0.** Statistics South Africa supplied the acknowledgement they accept on 20 October 2026, so the South African pack records it and the API hands it over. See "Attribution is not optional" below for what a host must do with it.

### Switch on `error.code`, never on the status alone

Two different `404`s, and a host that reads only the status will conflate them:

| Status | `error.code` | Means |
| --- | --- | --- |
| `404` | `no_pack_for_country` | A settled answer about a country. No pack covers it; it will not start to between two screens |
| `404` | `not_found` | This deployment has no such path — a client bug or a stale client, not a fact about geography |
| `401` | `unauthorized` | The path exists; the key does not, or is wrong |

**A path that does not exist answers `404` with or without a key**, because existence is not a function of who is asking. It used to answer `401`, which hid the route table from unauthenticated callers — an argument that ended when the route table was published. A host read a `401` on a path Places has never served as a route that existed and was protected, and spent effort on a door that was never there. The change shipped in 0.14.0, on 1 October 2026.

Both `404`s assert no generation: `x-places-generation` is absent, and `no_pack_for_country` carries `generation: null` in the body.

### A country with no pack is refused by name

**`404 no_pack_for_country`**, on every read and on `/coverage` and `/generation`:

```json
{ "error": { "code": "no_pack_for_country",
             "message": "no pack for country IN; this deployment serves ZA",
             "country": "IN", "countries": ["ZA"] },
  "generation": null }
```

This response **asserts no generation** — `generation` is null in the body and `x-places-generation` is absent from the headers. That is the point of it. The alternative, a `200` holding an empty result, cannot be told apart from a coordinate in the sea, and whatever generation it carried would belong to another country's pack: a host obeying *"never an `area_id` without its generation"* would persist a South African generation against an Indian community, using a value we supplied.

A country that is **configured but not yet ingested** is a different answer: reads return `200` and `/coverage` returns `measured: false`, both carrying that country's own generation. "We have no measures yet" and "we do not serve this country" are answers a host acts on differently.

## Layers

Polygons (harvest cells and admin stack) and named points (labels/search only).

**`adm` is the level; `kind` is the word the source uses for it.** Compare levels with `adm`, which is the same everywhere. `kind` differs by country at the same level — South Africa's ADM3 is a `municipality`, India's is a `subdistrict`, which is a taluk and not a municipality in any sense — and the list grows as countries are added.

| Code | `kind` (ZA) | `kind` (IN) | Source | Role |
|---|---|---|---|---|
| ADM1 | `province` | `province` | geoBoundaries / MDB · SOI ABDB | Admin |
| ADM2 | `district` | `district` | geoBoundaries / MDB · SOI ABDB | Admin |
| ADM3 | `municipality` | `subdistrict` | geoBoundaries / MDB · SOI ABDB | Admin — **the same level, a different thing** |
| ADM4 | `ward` | — | geoBoundaries / MDB | Admin |
| MP | `mainplace` | — | Stats SA Census 2011 | Place |
| SP | `subplace` | `subplace` (a village) | Stats SA Census 2011 · SOI ABDB | **Harvest cell** |
| GN | `geoname` | — | GeoNames (feature class P) | Label / search — **not** a harvest cell. Hits carry `capital_of`: `national`, `province`, `district`, `municipality` or `null` |
| ST | `street` | — | OpenStreetMap (ODbL) | Label / search — **not** a harvest cell. One row per street per SP |

`kind` query params accept either the code (`SP`) or the name (`subplace`). **An `adm` code resolves to whichever word that country's pack uses**, so `kind=ADM3` finds municipalities in South Africa and sub-districts in India without the caller knowing which. A literal word that the country does not use returns nothing rather than an error, the same as a bbox over the sea.

**NU** on MP/SP names = Stats SA **Non-Urban** (farm/rural remainder), not a suburb.

Always persist `area_id` **with** `generation`. An id without a generation is not a stable key.

---

## Public

### `GET /health`

```json
{ "ok": true }
```

### `GET /v1/places/coverage`

What this pack can and cannot answer for a country, as **data a host reads** rather than a country list it ships — one that would be wrong inside a single country anyway, since sparse Farmall AH identifies buildings better than dense Faerie Glen.

| Query | Required | Notes |
|---|---|---|
| `country` | no | Default `ZA` |
| `generation` | no | Default the pack's |

```json
{
  "country": "ZA", "generation": "mdb-zaf-2020", "measured": true,
  "measures": {
    "buildings_any":    { "value": 0.998, "detail": { "cells": 22151, "cells_with_any": 22117, "structures": 29766029 } },
    "crowding_10m":     { "value": 1.2,   "detail": { "p50": 1.2, "p90": 2.6, "measured_cells": 22117 } },
    "osm_housenumber":  { "value": 0.04,  "detail": { "buildings": 1200000, "with": 48000 } },
    "osm_entrance_ref": { "value": 0.001, "detail": { "entrances": 4200, "with_ref": 5 } }
  },
  "guidance": { "building_candidates": 3, "entrance_picker": false }
}
```

- **`buildings_any`** is the structural number: can a building be identified *at all*. **`osm_*`** are the quality-of-life numbers: entrances, floors, house numbers. They are reported separately on purpose — conflating them builds a feature that works in one country and is absent in another.
- **`crowding_10m`** is the mean number of **other** structures whose centroid lies within 10 m of a structure's centroid, averaged over a sample per cell. It sizes a picker. It is **measured, not derived from density** — settlements cluster, so a density estimate understates a township threefold. Measured for ZA: median 0.47 across cells, 2.25 in Khayelitsha, 0.43 in Faerie Glen, 0.23 in the Johannesburg CBD.
  Note it is not the same number as the `within 10 m` column in the Stage 0 table, which counted structures around a *simulated fix* and included the structure itself, and so runs about one higher.
- **`guidance`** is convenience, not gospel: `building_candidates` is the country-typical list length, and the per-cell figure on `/area/{id}` is the one to use. `entrance_picker` is false where almost no entrance carries a `ref`.
- `measured: false` means no ingest has measured that country yet. It is not a 404 — the pack simply does not know.
- **`layers` says what the pack holds, and `false` means absent rather than empty.** Check it before rendering a zero as "none here":

| A response | With the layer absent | Indistinguishable from |
| --- | --- | --- |
| `/v1/places/streets?q=…` | `[]` | no street of that name |
| `/v1/places/area/{id}/neighbours` | `count: 0` | a cell with no neighbours |
| `/v1/places/area/{id}` | no `stats` | an unmeasured cell |
| `/point?include=building` | no candidates | no structures nearby |

  Those four cases are the same bytes either way, so a host that says "nothing here" without consulting `layers` will eventually say it about a layer that was never ingested. `/coverage` itself is the exception — `measured: false` already says it does not know. Every key is present either way, including for a country configured but never ingested, where all of them read `false`. Read from the data, not from what an ingest recorded, because a fixture pack records one source for everything and a half-run ingest can record a source whose rows were never written.

### `GET /v1/places/generations`

**A generation belongs to a country.** Census years differ, so cadence does: South Africa can sit on 2011 cells while another country's pack is already on a newer vintage. Each country has its own default and its own cutovers.

Everything this pack holds. During a geography change the old and the new generation are both loaded, and a host can read either one.

```json
{
  "default": "mdb-zaf-2020",
  "generations": [
    { "generation": "mdb-zaf-2020", "loaded_at": "…", "layers": ["polygons_adm","places_census","points_geonames","lines_streets"], "default": true }
  ]
}
```

```json
{
  "default_country": "ZA",
  "default": "mdb-zaf-2020",
  "generations": [ { "generation": "mdb-zaf-2020", "loaded_at": "…", "layers": ["…"], "default": true } ],
  "countries": [
    { "country": "ZA", "default": "mdb-zaf-2020", "loaded": true,
      "generations": [ { "generation": "mdb-zaf-2020", "…": "…" } ] }
  ]
}
```

`countries` is the whole picture; the top-level `default` and `generations` describe the **default country**, so a host written against the single-country response keeps working. `loaded: false` means a country is configured but its pack has not been ingested yet.

A generation appears here once an ingest has written its `pack_meta` row.

### `GET /v1/places/generation`

Takes `country` (default: the deployment's default country).

```json
{ "country": "ZA", "generation": "mdb-zaf-2020", "loaded": true, "contract": "0.21.0",
  "attribution": [ { "source": "Statistics South Africa", "statement": "…", "…": "…" } ] }
```

Pin this on the host, with the country. When it changes, refresh cached `area_id`s. A country with no pack gets `404 no_pack_for_country`, not the default country's generation.

**`contract` is the version of [the published contract](https://github.com/daniel-kruger-rsa/fascia-places-contract) this deployment implements.** Compare it at boot against the version you vendored. Below your floor deserves its own failure reason — *the server is older than this app expects* — and never a sentence about geography: a rollback past your floor would otherwise serve you a contract you did not pin, with every call looking fine. It is also on every response that carries `x-places-generation`, as `x-places-contract`.

**`attribution` is this country's display duty** — the credits whose sources require acknowledgement, each with the wording that source has approved, and empty where no source asks for one. Build the licence screen from it at boot; see [Attribution is not optional](#attribution-is-not-optional).

**This is the capability probe, so read `loaded` with the generation.** A country that is configured but not yet ingested answers with its own generation — correctly — and holds nothing. A probe that treated a generation coming back as "ready" would un-hide every Places surface while every read returned empty: a flag that reports ready while nothing works, which is worse than no flag. `loaded: false` is the state to wait on. `GET /v1/places/generations` carries the same field per country, for a host that wants the whole picture.

### `GET /v1/places/layers`

Takes `country` (default: the deployment's default country), because the response carries a generation.

```json
{
  "country": "ZA",
  "generation": "mdb-zaf-2020",
  "layers": [
    { "adm": "ADM1", "kind": "province",      "vintage": "2020" },
    { "adm": "ADM2", "kind": "district",      "vintage": "2020" },
    { "adm": "ADM3", "kind": "municipality",  "vintage": "2020" },
    { "adm": "ADM4", "kind": "ward",          "vintage": "2020",
      "superseded_by": "MDB ward delimitation 2024/2026",
      "superseded_from": "2026-11-04", "current": true },
    { "adm": "MP",   "kind": "mainplace",     "vintage": "2011",
      "superseded_by": "Stats SA Census 2022 place names", "current": true },
    { "adm": "SP",   "kind": "subplace",      "vintage": "2011",
      "superseded_by": "Stats SA Census 2022 place names", "current": true },
    { "adm": "GN",   "kind": "geoname" }
  ]
}
```

**Every layer says which vintage it is of.** `vintage` is the year the boundaries represent;
`effective_from` is the date it starts describing what is in force; `superseded_by` names a
newer vintage that exists; `superseded_from` is the date ours stops; and **`current` is the one
to branch on**. All are omitted on a layer whose pack records no vintage.

Three states, and `current` is true only in the middle one:

| | `current` |
| --- | --- |
| Before `effective_from` — published, not yet in force | `false` |
| In force | `true` |
| From `superseded_from` | `false` |

**A boundary that has been published is not yet a boundary that exists.** South Africa's ward
layer is the MDB's 2026 delimitation — 4,485 wards, downloadable since September 2026 and
applying from **4 November**, election day. Until that morning `current` is false, and on it the
flag flips with no deployment. The cells are real throughout: `/point` resolves them and
`/areas/search` finds them. A layer that is not in force is **labelled, never withheld**.

A generation name cannot carry this, because one generation holds layers on different cadences
from different custodians: in `mdb-zaf-2020`, the wards are a 2020 demarcation and the
sub-places a 2011 census. And the two kinds of staleness are not the same — a newer census
existing does not make the older snapshot wrong, while a ward whose delimitation has been
replaced simply no longer exists. `current` is false only for the second.

**South Africa's wards go stale on 4 November 2026**, election day, when 4,468 wards become
4,488. `current` flips on the day, computed per request, with no deployment. If you filter or
label by ward, read it; the sub-place harvest cell is unaffected.


---

## `GET /v1/places/point`

Resolve a click / GPS fix. **The primary verb for a host.**

| Query | Required | Notes |
|---|---|---|
| `lng` | yes | WGS84 |
| `lat` | yes | WGS84 |
| `country` | no | default `ZA` |
| `kind` | no | One layer. Omit for full stack |

**Full stack (no `kind`):** smallest containing polygon (usually SP) plus every layer that contains the point, plus nearest GeoNames place.

```json
{
  "lat": -26.2041,
  "lng": 28.0473,
  "area_id": "…",
  "name": "Edenburg",
  "kind": "subplace",
  "country": "ZA",
  "generation": "mdb-zaf-2020",
  "place": { "geoname_id": 958724, "name": "Sandton", "meters": 180, "lat": -26.107, "lng": 28.057 },
  "nearby": [],
  "layers": {
    "ADM1": { "area_id": "…", "name": "Gauteng", "kind": "province" },
    "SP": { "area_id": "…", "name": "Edenburg", "kind": "subplace" }
  },
  "stack": [{ "adm": "ADM1", "name": "Gauteng", "kind": "province", "area_id": "…" }]
}
```

- `stack` is large → small: ADM1 … SP.
- `place` / `nearby` are GeoNames **P** within 8 km.
- **`include=building`** adds `building`: a **ranked candidate list**, nearest first, of the structures within 60 m — never a single answer. A 10 m fix picks the right structure 90.7% of the time in a Moscow panel district and 25.6% in Khayelitsha, so the response carries `confirm_required: true` and the member picks. How many candidates comes from the cell's `crowding_10m`. Omit the parameter and no building data is returned: a structure is close to a home.
- `street` is the nearest named street within 120 m, or `null`: `{ street_id, name, highway, sp_area_id, meters }`. A real street in range beats a nearer named `service` way (OSM puts shop and complex names on those). Villages and many townships have no street names in OSM at all, so hosts must treat it as optional. When it is present the response carries `attribution`.
- **Miss:** `{ "lat", "lng", "generation", "place", "nearby" }` — no `name` / `area_id`.
- **`?kind=SP`:** that one area only. No `stack`.
- `/point` never assigns membership. The host may *propose* a community; the member confirms. Persist the cell (`area_id` + `generation`), not the coordinate.

`400` if `lng`/`lat` missing or out of range, or `kind` unknown.

### `GET /v1/places/building/{building_id}`

One structure: what a host stores after a member confirms it, and what it compares when asking whether two people are in the same building.

```json
{ "building_id": "…", "generation": "mdb-zaf-2020", "country": "ZA",
  "source": "google_open_buildings", "source_ref": "5G5C+14",
  "area_m2": 300, "lng": 28.0477, "lat": -26.2044,
  "area": { "area_id": "…", "name": "Johannesburg SP", "kind": "subplace", "adm": "SP" },
  "attribution": [ { "source": "Google Open Buildings", "…": "…" } ] }
```

`source_ref` is the source's own key, kept because **no ML source publishes a stable id**: Google and Microsoft re-detect from imagery each release. `building_id` is minted here and is stable **within a generation** — persist it with the generation, as with `area_id`. Unknown id → `{}`. Non-UUID → `400`.

---

## `GET /v1/places/search`

GeoNames name search (labels, not harvest cells). Host then calls `/point` on the chosen hit.

| Query | Required |
|---|---|
| `q` | yes, min 2 chars |
| `country` | no, default `ZA` |

---

## `GET /v1/places/areas/search`

Name search over the polygon layers: ADM1–4, MP and SP. Use it when the member or mapping team knows the area's name ("Parktown", "City of Tshwane", "Ga-Rankuwa Unit 16"); use `/search` for GeoNames labels.

| Query | Required | Notes |
|---|---|---|
| `q` | yes, min 2 chars | Case-insensitive; matched literally (`%` and `_` are not wildcards) |
| `kind` | no | One layer or a comma-separated list, codes or names (`SP`, `ADM3,MP`, `subplace`). Default: all six. `GN` is `400` — use `/search` |
| `within` | no | An `area_id` of any layer. Resolved by **geometry** where the target has a polygon (point-on-surface inside it), and by **ancestry** where it does not — for a pack built from one hierarchy, ancestry is containment rather than an approximation. Unknown id still gives `[]`; a target with neither geometry nor anything claiming it as a parent is refused, see below |
| `limit` | no | Default 20, max 50 |
| `country` | no | Default `ZA` |

Order: exact name, then prefix, then contains; within each, larger layers first (ADM1 … SP), then shorter names.

Response (values illustrative):

```json
[
  {
    "area_id": "…",
    "name": "Parktown",
    "kind": "subplace",
    "adm": "SP",
    "country": "ZA",
    "generation": "mdb-zaf-2020",
    "parents": [
      { "adm": "ADM1", "area_id": "…", "name": "Gauteng", "kind": "province" },
      { "adm": "ADM2", "area_id": "…", "name": "City of Johannesburg", "kind": "district" },
      { "adm": "ADM3", "area_id": "…", "name": "City of Johannesburg", "kind": "municipality" },
      { "adm": "MP", "area_id": "…", "name": "Johannesburg", "kind": "mainplace" }
    ],
    "bbox": [28.03, -26.18, 28.05, -26.17]
  }
]
```

- `parents` is the pack's parent chain, large → small: SP → MP → ADM3 → ADM2 → ADM1; a ward's chain is ADM3 → ADM2 → ADM1. Show it next to the name — names repeat across the country.
- `bbox` is `[west, south, east, north]` for fitBounds; `/area/{id}/geojson` gives the outline.
- Persist `area_id` **with** `generation`, as everywhere.

---

## `GET /v1/places/streets`

Street name search. Streets are **labels**, like GeoNames: never harvest cells, never a membership key.

| Query | Required | Notes |
|---|---|---|
| `q` | yes, min 2 chars | Normalised the same way the ingest normalised street names, so "Main Rd" finds "Main Road" |
| `within` | no | An `area_id` of any layer: only streets in that cell, or in cells inside it. Same two mechanisms and the same refusal as `/areas/search`, since a street hangs off a cell |
| `limit` | no | Default 20, max 50 |
| `country` | no | Default `ZA` |

Order: exact normalised name, then prefix, then contains; real streets before named `service`/`track` ways; longer first.

```json
{
  "generation": "mdb-zaf-2020",
  "attribution": [
    { "source": "OpenStreetMap", "text": "© OpenStreetMap contributors",
      "url": "https://www.openstreetmap.org/copyright", "license": "ODbL 1.0" }
  ],
  "count": 1,
  "streets": [
    {
      "street_id": "…",
      "name": "Atterbury Road",
      "highway": "primary",
      "length_m": 6039,
      "subplace": { "area_id": "…", "name": "Faerie Glen", "kind": "subplace", "adm": "SP" },
      "parents": [ { "adm": "ADM1", "name": "Gauteng", "…": "…" } ],
      "bbox": [28.27, -25.79, 28.31, -25.78]
    }
  ]
}
```

One row per street **per sub-place**: a road crossing three sub-places is three rows. "Main Road" exists hundreds of times, so show `subplace` and `parents` beside the name.

### `GET /v1/places/street/{street_id}/geojson`

One street's line (`MultiLineString`) for drawing or fitBounds, with `attribution`. Unknown id → empty FeatureCollection. Non-UUID → `400`.

### Attribution is not optional

Street data is OpenStreetMap under ODbL. Every response carrying streets carries `attribution`, and the host **must** show "© OpenStreetMap contributors" wherever a street name is displayed, linked to openstreetmap.org/copyright. The method producing this layer is published at [fascia-places-streets](https://github.com/daniel-kruger-rsa/fascia-places-streets).

**Boundaries carry their own duty, and it differs by country.** A pack whose source requires acknowledgement says so, and its credit arrives in the same `attribution` array:

| Field | What to do with it |
| --- | --- |
| `text` | The short line. Show it wherever the data is shown — under a map, beside a cell name |
| `statement` | The full acknowledgement, in the rights holder's own words, present only where one was supplied. Show it on a licence or about screen |
| `url`, `license` | Link and terms, for that screen |

**Read the duty once, at boot, from `GET /v1/places/generation?country=XX`.** Its `attribution` array is the whole display duty for that country and generation. Three reasons that is the right place rather than collecting credits from responses: the duty is static per pack, the endpoint needs no host key, and three read endpoints — `/search`, `/areas/search`, and `/harvest` without `summary=1` — answer with a bare JSON array and so cannot carry a credit at all. Those shapes are not changing inside v1.

**South Africa, from 0.21.0.** Boundaries derive from Statistics South Africa's Census 2011 place-name framework. Stats SA supplied the wording they accept on 20 October 2026; the pack carries it as `statement` and it is the text to show. It covers two things at once — that Stats SA is the source and rights holder, and that the boundaries are not sold or charged for as a standalone product, any fee attaching to work built on top of them.

---

## `GET /v1/places/crosswalk`

What became of a cell when the geography changed. For re-linking stored `area_id`s after a cutover, and for nothing else.

```
GET /v1/places/crosswalk?country=ZA&from=mdb-zaf-2020&to=stats-sa-2022&area_id=<uuid>,<uuid>
```

```json
{ "country": "ZA", "from": "mdb-zaf-2020", "to": "stats-sa-2022",
  "results": [
    { "area_id": "…", "verdict": "same",
      "to": [ { "area_id": "…", "name": "Rietpoort SP", "kind": "subplace", "adm": "SP", "share": 0.997 } ] },
    { "area_id": "…", "verdict": "split",
      "to": [ { "area_id": "…", "name": "Ennerdale Ext 14", "share": 0.41, "…": "…" },
              { "area_id": "…", "name": "Ennerdale Ext 12", "share": 0.33, "…": "…" },
              { "area_id": "…", "name": "Ennerdale Ext 13", "share": 0.25, "…": "…" } ] },
    { "area_id": "…", "verdict": "unresolved", "to": [] },
    { "area_id": "…", "verdict": "unknown",    "to": [] } ] }
```

**It returns a list, not one id, and that is the point.** One id is always available and often wrong. Measured on Census 2011 against Census 2022 sub-places: 86.2% of old cells have exactly one successor, 13.8% split into two or more, and **every single old centre falls inside some new cell**. A single-id answer would be well formed and wrong for one cell in seven, and a host would store it without ever learning.

**Branch on `verdict`.** Not on the shares — the thresholds are ours to tune and yours to ignore.

| `verdict` | Means | What to do |
| --- | --- | --- |
| `same` | One successor holding ≥ 98% of the old cell | Re-point the stored id. No human needed |
| `moved` | One successor, 50–98% | Re-point it; the boundary moved materially |
| `split` | Two or more successors | **Ask a person.** Which successor a member belongs to is not a geometry question |
| `unresolved` | No successor holds half the old cell | Do not guess. Keep the old id, keep reading the old generation for that row, or send someone to look. `to` still lists any partial overlaps — the verdict says do not re-point automatically, not that nothing is known |
| `unknown` | This generation does not hold that `area_id` | Your id is wrong — wrong country, wrong layer, or a typo. Deliberately not `unresolved` |

Up to **200** `area_id`s per call; re-link in batches. Results come back in the order asked.

**A generation pair with no crosswalk computed answers `404 crosswalk_not_loaded`,** naming the pair. That is not a claim that the cells are unchanged — it means the mapping does not exist yet. A crosswalk is built per layer pair and takes minutes of compute, so it exists for the pairs someone has needed. A misspelled generation lands here too, and the refusal names the pair so you can see it.

**Neither generation has to still be loaded.** A crosswalk outlives the generations it maps between, and that is the point: once we drop the old generation, a host still holding its ids needs this answer more than it did before. A successor's `name` and `kind` are omitted if the target generation is gone; `area_id` and `share` are always there.

**Names change too, and the crosswalk does not carry them.** 35% of 2011 sub-places have a differently-spelled or differently-named 2022 successor. If you stored a name string rather than reading it from us, re-read it.

---

## `GET /v1/places/geonames`

Viewport GeoNames dots. Bbox required. `limit` default 250, max 500. Optional `country` (default `ZA`).

Returns a point FeatureCollection with `generation`, `kind: "geoname"`, `adm: "GN"`, `count`.

---

## `GET /v1/places/harvest`

### Covered-set contract (decided)

| Role | Owner |
|---|---|
| Which cells are **covered** (linked to one of the host's groups) | **Host** |
| Which polygons exist in a bbox | **Fascia** |

Fascia does not know about communities. The harvest *field* is a host-owned list of SP `area_id`s + `generation`.

`GET /harvest?covered=` is an optional **small-set convenience** (mapping-team picker, a few dozen ids). It is **not** the production map path.

**Map paint:** call `GET /geojson` for “SP (or ADM*) cells in bbox”. The host's backend subtracts its own covered set and paints two colours. Do **not** send the national covered set to Fascia on every pan.

Do not put thousands of UUIDs on a GET query string. If a server-side remainder of a large set is ever needed, that would be a future `POST /harvest` with a JSON body — not this GET.

| Query | Required | Notes |
|---|---|---|
| `west` `south` `east` `north` | yes | bbox |
| `kind` | no | default `subplace` |
| `covered` | no | comma-separated UUID `area_id`s, **at most 200** — more is `400`; non-UUIDs are `400` |
| `summary` | no | `1` / `true` → counts only |
| `country` | no | default `ZA` |

**List:** `[{ "area_id", "name", "kind", "country", "generation" }]` (max 5000, ordered by name). Persist `area_id` **with** `generation`.

**Summary:** `{ "generation", "kind", "uncovered", "covered", "uncovered_totals": { "road_km", "named_road_km", "buildings" } }` — `uncovered_totals` sums `area_stats` over the uncovered cells in the viewport, so a host can report "18 km of road and 2,400 structures still uncovered" rather than a cell count. Values are `null` until the measuring ingest has run; `attribution` accompanies them.

---

## `GET /v1/places/geojson`

Viewport polygons for **one** layer. This is the map overlay.

Zoom → kind ladder (host-enforced): ADM1/ADM2 at country zoom; SP only when close. Never request SP polygons at country zoom.

| Query | Default |
|---|---|
| `kind` | `subplace` |
| `west` `south` `east` `north` | **required** for `ward`, `mainplace`, `subplace` (`400` without). Optional for `province`, `district`, `municipality`, which may be fetched pack-wide. |
| `limit` | 2500 (max 8000) |
| `simplify` | 0.0006 degrees |
| `country` | `ZA` |

Feature `properties` carry `area_id`, `name`, `kind`, `adm`, `generation`.

`count` = features returned; `total` = matches in viewport. If `count < total`, zoom in. Quantise bbox to a tile grid on the host so two members on the same city share a cache key.

---

## `GET /v1/places/area/{area_id}`

Properties only, including `parent_ids` and `generation`. Unknown id → `{}`. Non-UUID → `400`.

When an ingest has measured the cell, the response also carries `stats` and the credits those measures require:

```json
{
  "stats": { "road_km": 30.0, "named_road_km": 5.0, "buildings": 2500,
             "area_km2": 8.3, "land_class": "urban",
             "centroid": { "lng": 28.318136, "lat": -25.786582 } },
  "attribution": [ { "source": "OpenStreetMap", "…": "…" }, { "source": "Google Open Buildings", "…": "…" } ]
}
```

- `road_km` counts **all** street-like OSM ways in the cell, named or not; `named_road_km` only the named ones. In much of the country the two differ sharply — the Eastern Cape has named streets in 12% of sub-places, Gauteng in 96% — so **coverage should be measured against `road_km` or `buildings`, never against named streets**.
- `buildings` is a count of Google Open Buildings structures whose centroid falls in the cell. No footprints are stored or served.
- `area_km2` and `centroid` are derived from the polygon; the centroid is a point **on** the cell, so it can pin a label even on a crescent-shaped one.
- `land_class` is `urban`, `non_urban`, `agricultural_holdings` or `smallholdings` for MP and SP, and `null` for admin layers. It is inferred from Stats SA's naming (` NU`, ` AH`, ` SH` suffixes), which is the same inference behind "do not harvest an NU cell as though it were a suburb".
- `crowding_10m` and `building_candidates` size a building picker for **this cell**, which is better than the country typical value from `/v1/places/coverage`.
- Any column is `null` when its ingest has not run for that generation.

### `GET /v1/places/area/{area_id}/neighbours`

The cells touching this one — the answer to "what should this group take next", which a bounding box cannot give. Sub-places only; any other layer returns an empty list.

```json
{
  "area_id": "…", "generation": "mdb-zaf-2020", "count": 8,
  "neighbours": [
    { "area_id": "…", "name": "Garsfontein", "kind": "subplace", "adm": "SP",
      "country": "ZA", "area_km2": 7.3, "land_class": "urban", "buildings": 11850 }
  ]
}
```

Each neighbour carries its own measures, so a picker can rank candidates without a second call. Adjacency is by intersection rather than strict touching: census polygons share slivers and rounding, and a strict test drops neighbours a person would point at.

### `GET /v1/places/area/{area_id}/geojson`

One simplified polygon (highlight / fitBounds). FeatureCollection carries `generation`. Unknown id → empty FeatureCollection.

Parent chain in the pack: subplace → mainplace → municipality → district → province; ward → municipality. Census and MDB layers come from different sources, so cross-source links are by point-on-surface containment.

---

## Errors

| HTTP | `error.code` |
|---|---|
| 400 | `bad_request` |
| 401 | `unauthorized` |
| 404 | `not_found`, or `no_pack_for_country` — see above |
| 500 | `unavailable` — body never includes database detail |
