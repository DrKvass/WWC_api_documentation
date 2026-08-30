# WWC Public API Guide

Current contract: **30 August 2026**

This document is intentionally self-contained. It describes the public WWC Bearer-token API as it is currently exposed at `https://example.url/api/`.

## Base URL

Contac dr.kvass on discord.

This doc will use

```
https://example.url/
```

All API-client routes are below `/api/`. The website login at `/`, browser OAuth routes under `/auth/`, and the protected interactive map under `/map/` are separate browser interfaces and do not use API Bearer tokens.

## Authentication

Every request requires an HTTP Bearer token:

```http
Authorization: Bearer <token>
```

Example:

```bash
curl -H "Authorization: Bearer $WWC_TOKEN" \
  https://example.url/api/
```

A successful authentication check returns an object containing the token display name and permission.

Bearer permissions are hierarchical:

```text
read < write < admin
```

A `write` token satisfies `read` checks. An `admin` token satisfies both `write` and `read` checks. The current public HTTP API exposes **read routes only**; there are no POST, PUT, PATCH, or DELETE API routes. Claim reads are deliberately restricted to `admin` tokens.

## Endpoint summary

| Method | Route                 | Required permission | Purpose                                                   |
| ------ | --------------------- | ------------------- | --------------------------------------------------------- |
| GET    | `/api/`               | read                | Verify the token and return its name/permission.          |
| GET    | `/api/codes/`         | read                | Current database-backed reference codes.                  |
| GET    | `/api/storms/`        | read                | List API-published active storms.                         |
| GET    | `/api/storms/{id}`    | read                | Return one API-published active storm by stable storm ID. |
| GET    | `/api/storms/events/` | read                | Publication-aware storm Server-Sent Events stream.        |
| GET    | `/api/claims/`        | **admin**           | List all active weather-station claims.                   |
| GET    | `/api/claims/{id}`    | **admin**           | Return one claim by stable claim ID.                      |
| GET    | `/api/icons/rain/`    | read                | Rain icon PNG.                                            |
| GET    | `/api/icons/snow/`    | read                | Snow icon PNG.                                            |

Trailing slashes are shown exactly as used by the route definitions. Storm and claim `{id}` values are opaque stable database IDs.

## Authentication check

### `GET /api/`

Example response:

```json
{
  "auth_req": "Successfully authorized with your token!",
  "name": "integration-name",
  "permission": "read"
}
```

Use this route to verify a token without fetching domain data.

## Reference codes

### `GET /api/codes/`

Returns the current database-backed code catalogue. The top-level keys are:

```json
{
  "storm_codes": {},
  "claim_codes": {}
}
```

Each category is keyed by stable code and contains database IDs and human-readable names. Clients should prefer the stable textual code for logic and use the name for presentation.

## Public storm visibility

Storm list/detail reads have an explicit publication boundary. A storm is returned only when **both** are true:

1. `api_published` is true; and
2. the lifecycle status code is `predicted` or `ongoing`.

New storms are private by default. WWC management explicitly publishes or hides a storm from Discord. Publication does not change the storm's lifecycle status.

Consequences:

- an unpublished Predicted/Ongoing storm is not returned;
- a published Concluded storm is not returned;
- a published Predicted/Ongoing storm is returned;
- a storm may disappear from the list because it was concluded **or** made private.

Treat `GET /api/storms/` as the authoritative current display set.

## Storm list

### `GET /api/storms/`

Returns an array of `APIStormObject` records for API-published Predicted/Ongoing storms.

Important identity fields:

- `id` — stable storm resource ID; use this for API detail requests;
- `thread_id` — Discord integration value when present; do not use it as the API resource ID;
- `designation` — current storm name;
- `api_published` — true for objects returned by the public storm routes.

Lifecycle/reference fields include the corresponding database ID, stable code, and display name where applicable:

- `status_id`, `status_code`, `status_name`;
- `type_id`, `type_code`, `type_name`;
- `size_id`, `size_code`, `size_name`;
- `origin_id`, `origin_name`;
- `intensity_id`, `intensity_code`, `intensity_name`.

Geometry fields:

- `fhs_x`;
- `fhs_y`;
- `radius` — outer radius in metres.

Timeline fields, when known:

- `report_time`;
- `detection_time`;
- `start_time`;
- `end_time`.

Weather-station relationship summaries, when known:

- `ws_prediction`;
- `ws_ongoing`.

Contributor presentation fields, when known:

- `named_by`;
- `prediction_detected_by`;
- `start_detected_by`;
- `end_detected_by`;
- `analyst_prediction`;
- `analyst_ongoing`;
- `prediction_plotted_by`;
- `tracking_plotted_by`.

Contributor values are current Discord server display names/nicknames, not Discord user IDs. The two analyst fields may contain comma-separated display names. Names are mutable presentation values and must not be treated as stable identity keys.

Fields whose value is `null` are omitted from the response.

## Storm detail

### `GET /api/storms/{id}`

Returns one API-published Predicted/Ongoing storm by stable `storm.id`.

If the storm does not exist **or is not currently public-active**, the route returns `404`. This intentionally avoids exposing private or historical storm state through the public detail route.

Example:

```bash
curl -H "Authorization: Bearer $WWC_TOKEN" \
  https://example.url/api/storms/123456789/
```

When a previously visible storm stops being public-active, clients should remove it from current displays instead of assuming the ID became invalid permanently.

## Storm Server-Sent Events

### `GET /api/storms/events/`

This route returns `text/event-stream` and is a **reconciliation signal**, not a complete storm snapshot. After receiving a relevant event, clients should reconcile with `GET /api/storms/` or fetch `GET /api/storms/{storm_id}` when appropriate.

### Connection behavior

On connection the stream emits a control event:

```text
event: stream.ready
retry: 3000
data: {"cursor":123,"latest_event_id":123,"replay_pending":false,"reset_recovery":false}
```

During quiet periods the server sends SSE comments approximately every 15 seconds:

```text
: keep-alive
```

A new connection without `Last-Event-ID` starts at the current durable cursor and receives future events. A reconnect may send the standard header:

```http
Last-Event-ID: 123
```

The server replays retained durable events with a larger ID. If the runtime database was reset and the client's cursor is ahead of the new outbox sequence, the server automatically recovers from the sequence rewind.

### Public event types

The external storm event types are:

```text
storm.geometry_changed
storm.status_changed
storm.published
storm.unpublished
```

**Storm creation itself is not a public SSE event.** A newly created storm is private by default and does not become externally visible until WWC management publishes it.

Publication transitions are explicit:

- `storm.published` — publication flag changed false → true;
- `storm.unpublished` — publication flag changed true → false.

Private storm lifecycle/geometry changes do not create public-relevant visibility by themselves.

### Event payload

A storm event JSON object contains:

```json
{
  "id": 124,
  "event_type": "storm.geometry_changed",
  "storm_id": 123456789,
  "status_id": 1,
  "status_code": "predicted",
  "api_published": true,
  "active": true,
  "changed_fields": ["fhs_x", "radius"],
  "created_at": 1788040000
}
```

Field meanings:

- `id` — durable SSE event cursor;
- `event_type` — one of the four public event types above;
- `storm_id` — stable storm resource ID;
- `status_id` / `status_code` — resulting lifecycle status;
- `api_published` — resulting publication flag;
- `active` — true only when the resulting storm is both published and Predicted/Ongoing;
- `changed_fields` — compact list of fields that caused the event;
- `created_at` — Unix timestamp.

Recommended consumer behavior:

- if `active` is true, refetch the storm or reconcile the list;
- if `active` is false, remove the storm from a current-storm display;
- treat `storm.published` as a reason to fetch/reconcile;
- treat `storm.unpublished` as a reason to remove/reconcile.

The delivery loop is event-driven internally; quiet connections do not poll SQLite repeatedly. This internal optimization does not change the public SSE wire format or replay semantics.

## Claims

Claim routes require an **admin** Bearer token.

### `GET /api/claims/`

Returns all active weather-station claims.

### `GET /api/claims/{id}`

Returns one claim by stable `claim.id`.

Claim identity fields:

- `id` — stable opaque database ID used by relationships/API detail;
- `code` — mutable user-facing/in-game claim code;
- `claim_hex_name` — display name;
- `user_id` — current Discord owner ID;
- `hex_id`, `hex_name` — map hex identity/presentation;
- `state`, `state_id`, `state_code` — current state;
- `fhs_x`, `fhs_y` — optional location;
- `linked_id` — stable ID of a paired station when present.

Do not use `code` as a stable relationship key; it may change while `id` remains the same.

## Icons

### `GET /api/icons/rain/`

### `GET /api/icons/snow/`

Return authenticated PNG files for the current WWC rain/snow icons.

## FHS and storm-radius conventions

Stored FHS coordinates use:

```text
FHS_x: 0.0 through 256.0
FHS_y: -205.7 through -50.3
```

Storm `radius` is a positive whole number of metres and represents the **outer** coverage radius. WWC map rendering uses the inner 60% of radial distance as the solid inner zone and the 60%–100% interval as the transition/gradient band.

## HTTP errors

Typical responses:

- `401 Unauthorized` — Bearer token missing/invalid;
- `403 Forbidden` — token authenticates but lacks the required permission, such as a non-admin token requesting claims;
- `404 Not Found` — stable resource ID is absent or a requested storm is outside the public-active publication boundary;
- `400 Bad Request` — malformed `Last-Event-ID` or other request validation failure.

Do not infer private resource existence from a public storm `404`.
