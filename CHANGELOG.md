# Fascia Places contract — changelog

Every published version of [places.v1.yaml](places.v1.yaml), newest first. A host vendors a
copy pinned to a tag and reads this to decide whether a diff needs its attention.
`scripts/publish-contract.sh` refuses to publish a version that has no entry here.

| Version | Date | Change | Does your client care? |
| --- | --- | --- | --- |
| `0.14.0` | 2026-10-01 | A path that does not exist answers `404 not_found` rather than `401`, with or without a host key. Switch on `error.code`, never on the status alone: `no_pack_for_country` is a settled answer about a country, `not_found` is a path this deployment does not have. | **Yes, if you map statuses.** A client that read `404` as "no pack" would tell a user their country is unserved when a developer typed a path wrong |
| `0.13.1` | 2026-10-01 | Editorial. Removed a key-shaped example string from the security scheme; listed the production server. First published version. | No |
| `0.13.0` | 2026-09-30 | Not published. `404 no_pack_for_country` on every read for a country with no pack, asserting no generation; `loaded` on `/v1/places/generation`; `country` and `generation` on `/v1/places/layers`. | n/a — superseded by 0.13.1 before publication |
