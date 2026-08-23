# WWC Weather API — Public Changelog

This changelog lists externally visible API changes that may matter to API consumers. It intentionally omits deployment, database-administration, and other backend-only details.

For the current API address and access details, contact **dr.kvass on Discord**.

## 2026-08-23 — Active storm feed and weather icons

### Changed — `/storms/`

`GET /storms/` now returns only storms that are still active in the public feed: **Predicted** and **Ongoing** storms.

Concluded storms are no longer returned.

**Integration impact:** polling clients should treat each `/storms/` response as authoritative. If a storm ID that was previously displayed disappears from the response, remove it from the client display.

### Changed — `/storms/{id}`

`GET /storms/{id}` now follows the same active-feed rule.

A storm that has become Concluded is no longer returned from this endpoint and should be treated as unavailable to the current-storm display.

### Added — weather icon endpoints

```text
GET /icons/rain/
GET /icons/snow/
```

Both endpoints require normal API authentication and return `image/png` content.

### Documented — storm gradient convention

For clients rendering WWC-style storm coverage from FHS centre coordinates and the stored radius:

- the stored radius is the outer storm radius;
- the inner zone extends from `0%` to `60%` of the radius;
- the outer transition/gradient band extends from `60%` to `100%` of the radius.

The 60% value refers to radial distance from the storm centre.

---
