# Fascia Places — API contract

The OpenAPI 3.0 document for the Fascia Places v1 API, published so that a host can read
it, vendor a copy, and diff against it when it changes. Nothing else lives here.

- **[places.v1.yaml](places.v1.yaml)** — the contract. Current version: **0.14.0**.
- Every published version is tagged (`v0.14.0`), so a vendored copy can pin one:
  `https://raw.githubusercontent.com/daniel-kruger-rsa/fascia-places-contract/v0.14.0/places.v1.yaml`

## Why this repo exists

A host writes its client against this document. If the only copy is the one that was read
at the time, a drift between client and contract surfaces as a defect in somebody's screen,
found by a user rather than by a diff. Advance OS asked for something to diff against on
1 October 2026; this is it.

## Versioning

`info.version` moves on every change to the document.

| Change | Version |
| --- | --- |
| A new field, endpoint or response | minor (`0.13` → `0.14`) |
| Editorial only — wording, examples, descriptions, no interface change | patch (`0.13.0` → `0.13.1`) |

A patch release never changes what the API accepts or returns. `0.13.1` is one: it removed a
key-shaped example string from the security scheme and listed the production server.

Breaking changes do not happen inside v1. The path carries the major version, so an
incompatible contract would be `/v2/places` and a new document.

## What the API serves, and what that costs you

Reads need a **host key**, issued per host and held server-side — never in a client bundle.
Four endpoints are open, because they answer *what can this pack do* before a host has
decided to call anything: `/health`, `/v1/places/generation`, `/v1/places/generations`,
`/v1/places/coverage` and `/v1/places/layers`.

Two things in the contract carry obligations rather than conveniences:

- **`attribution`** is returned beside street and building data and is not decorative.
  Streets derive from OpenStreetMap (ODbL 1.0) and must credit it wherever a street name is
  shown; building counts and centroids derive from Google Open Buildings (CC BY 4.0).
- **Boundaries** derive from Statistics South Africa and the Municipal Demarcation Board.
  Stats SA's conditions of use apply to anything built from them, including a requirement
  that derivative products be vetted before publication.

The data is not open because this document is. Ask before redistributing anything the API
returns.

## Licence

The document is MIT (see [LICENSE](LICENSE)) — vendor it, diff it, generate clients from it.
The licence covers this file, not the data the API serves, and not access to the API.

## Contact

AsOne Fascia. Issues on this repo are read.
