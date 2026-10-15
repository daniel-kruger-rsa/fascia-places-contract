# Fascia Places contract — changelog

Every published version of [places.v1.yaml](places.v1.yaml), newest first. A host vendors a
copy pinned to a tag and reads this to decide whether a diff needs its attention.
`scripts/publish-contract.sh` refuses to publish a version that has no entry here.

| Version | Date | Change | Does your client care? |
| --- | --- | --- | --- |
| `0.15.0` | 2026-10-15 | `/v1/places/coverage` gains `layers`: per-layer presence, read from the data. `false` means a layer is **absent**, not empty. Every key is present either way, including for a configured country with no ingest, where all read false. | **Yes, if you render a zero as "none here".** `[]` from `/streets`, `count: 0` from `/neighbours`, a missing `stats`, no building candidates — each of those is identical whether the layer is absent or empty. Advance OS asked for this as a gate before an India pack, which will be partially ingested by design |
| `0.14.1` | 2026-10-15 | Editorial. The production server is `https://places.fascia.network`. The old name, `places.fascia.asone.net.za`, serves the same deployment and the same host keys until hosts have moved, then retires. | **Yes, once.** Move your base URL. **If your vendored copy is older than 0.14.0, read that entry first:** it changed how a `404` must be handled. Advance OS, coming from 0.13.1, found a client that mapped every `404` to "country not served" |
| `0.14.0` | 2026-10-01 | A path that does not exist answers `404 not_found` rather than `401`, with or without a host key. Switch on `error.code`, never on the status alone: `no_pack_for_country` is a settled answer about a country, `not_found` is a path this deployment does not have. | **Yes, if you map statuses.** A client that read `404` as "no pack" would tell a user their country is unserved when a developer typed a path wrong |
| `0.13.1` | 2026-10-01 | Editorial. Removed a key-shaped example string from the security scheme; listed the production server. First published version. | No |
| `0.13.0` | 2026-09-30 | Not published. `404 no_pack_for_country` on every read for a country with no pack, asserting no generation; `loaded` on `/v1/places/generation`; `country` and `generation` on `/v1/places/layers`. | n/a — superseded by 0.13.1 before publication |
