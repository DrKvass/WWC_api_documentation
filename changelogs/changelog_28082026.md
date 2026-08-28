# WWC Weather API — Public Changelog

This changelog lists externally visible API changes that may matter to API consumers. It intentionally omits deployment, database-administration, and other backend-only details.

For the current API address and access details, contact **dr.kvass on Discord**.

## 2026-08-28 — API namespace moved to `/api/`

### Breaking route change

Every API endpoint is now namespaced under `/api/`. The site root `/` is no longer an API endpoint and is intentionally left available for the website.

Examples:

```text
GET /                  -> GET /api/
GET /codes/             -> GET /api/codes/
GET /storms/            -> GET /api/storms/
GET /storms/events/     -> GET /api/storms/events/
GET /storms/{id}        -> GET /api/storms/{id}
GET /claims/            -> GET /api/claims/
GET /claims/{id}        -> GET /api/claims/{id}
GET /icons/rain/        -> GET /api/icons/rain/
GET /icons/snow/        -> GET /api/icons/snow/
```

There is no compatibility alias for the old unprefixed API routes. API clients must update their base path. SSE clients must reconnect to `/api/storms/events/`; `Last-Event-ID` behavior and event payloads are otherwise unchanged.

The Nginx reverse proxy is likewise scoped to `/api/`, reserving `/` for the website.
