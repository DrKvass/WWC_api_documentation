# WWC Weather API — Public User Documentation

The WWC Weather API provides authenticated, read-only access to current storm information, real-time storm-change Server-Sent Events (SSE), weather-station claims, reference codes, and public weather icons.

For the current API address and a Bearer token, contact **dr.kvass on Discord**.

All examples below use the placeholder base URL:

```text
https://api.example.com
```

Replace it with the address supplied to you.

## Authentication

Send your token with every request:

```http
Authorization: Bearer YOUR_TOKEN
```

Treat the token like a password. Use HTTPS only, never put the token in a URL, and do not publish it in source control, screenshots, logs, or public client-side code.

## Current endpoints

```text
GET /
GET /codes/
GET /storms/
GET /storms/events/
GET /storms/{id}
GET /claims/
GET /claims/{id}
GET /icons/rain/
GET /icons/snow/
```

All current endpoints require `read` access or higher.

## Important storm-feed behavior

`GET /storms/` and `GET /storms/{id}` expose only **current Predicted and Ongoing storms**.

Storms whose status becomes **Concluded** are intentionally no longer returned by the storm API, even though the API code catalogue may still include the `concluded` status as a valid lifecycle value.

### Integration requirement

Treat `GET /storms/` as the authoritative set of storms that should currently be displayed.

If your application previously received a storm and that storm is no longer present in a later `/storms/` response, **stop displaying it**. Do not keep a stale storm visible simply because it exists in a client cache.

Likewise, `GET /storms/{id}` returns `404 Not Found` when that storm is no longer part of the Predicted/Ongoing feed.

## Storm contributor names

Storm list/detail responses now return contributor **Discord server display names** instead of contributor user IDs or Discord mention strings. A server nickname is used when the member has one; otherwise the API falls back to the member's Discord display name/username.

The affected storm fields are:

```text
named_by
prediction_detected_by
start_detected_by
end_detected_by
prediction_plotted_by
tracking_plotted_by
analyst_prediction
analyst_ongoing
```

The first six fields contain one display-name string when present. `analyst_prediction` and `analyst_ongoing` may represent several users and are returned as comma-separated display-name strings.

Example:

```json
{
  "named_by": "Kvass",
  "prediction_detected_by": "Weather Watcher",
  "analyst_prediction": "Kvass, Forecaster",
  "prediction_plotted_by": "Map Tech"
}
```

These names are **display data, not stable identifiers**. Members can change their nickname, so integrations must not use a contributor name as a database key or identity. Continue to identify the storm itself with the stable storm `id`. Nickname changes can take up to about five minutes to appear because the API briefly caches Discord member names.

If a historical contributor has left the server, the API attempts to use the Discord account display name/username. If Discord cannot resolve the account at all, that contributor field is omitted rather than returning the raw numeric user ID.

This change applies to storm contributor fields only. Claim resources still expose their existing `user_id` owner field.

## Real-time storm updates with SSE

`GET /storms/events/` is a long-lived authenticated Server-Sent Events stream. It is intended to reduce frequent polling while still keeping `/storms/` and `/storms/{id}` as the authoritative current-state resources.

The stream notifies clients when:

- a new storm is created;
- any storm changes lifecycle status, including a Concluded storm being restored to Predicted or Ongoing;
- `fhs_x`, `fhs_y`, or `radius` changes while the storm is Predicted or Ongoing.

Geometry edits made while a storm is Concluded are not emitted.

Connect with a normal Bearer header and disable client-side response buffering. For example:

```bash
curl -N \
  -H "Accept: text/event-stream" \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://api.example.com/storms/events/
```

A connection begins with a control event similar to:

```text
retry: 3000
event: stream.ready
data: {"cursor":42,"latest_event_id":42,"replay_pending":false,"reset_recovery":false}
```

Normal storm events look like:

```text
id: 43
event: storm.geometry_changed
data: {"id":43,"event_type":"storm.geometry_changed","storm_id":3608617449183851,"status_id":2,"status_code":"ongoing","active":true,"changed_fields":["fhs_x","fhs_y","radius"],"created_at":1787774400}
```

The possible `event` values are:

```text
storm.created
storm.status_changed
storm.geometry_changed
```

The payload is a **change notification**, not a full storm snapshot. Use `storm_id` with `GET /storms/{id}` or reconcile the full `GET /storms/` list after receiving an event.

For a race-resistant initial load, open the SSE connection first, wait for `stream.ready`, then fetch `GET /storms/` while keeping the stream open. Queue any numbered storm events that arrive during the snapshot request, apply the snapshot, and then apply those queued events in event-ID order. This avoids a gap between the initial snapshot and the live stream.

- `active: true` means the storm is currently Predicted/Ongoing and can be refetched from the current storm API.
- `active: false` means it is outside the current feed (normally Concluded), so remove it from a current-storm display.
- `changed_fields` identifies the relevant fields that triggered the event. Creation events use an empty list.

### Reconnecting without losing events

SSE clients should preserve the last numbered event ID. On reconnect, send the standard header:

```http
Last-Event-ID: 43
```

The server replays newer retained events and then continues waiting for live changes. Many SSE libraries handle `Last-Event-ID` automatically.

A fresh connection with no `Last-Event-ID` starts at the current event cursor and receives future changes only. A database reset clears the runtime event outbox; the server detects the sequence rewind so a pre-reset cursor does not block later storm events.

During quiet periods the server sends `: keep-alive` comments. These are not application events and can be ignored.

> **Browser note:** the native browser `EventSource` API does not provide a portable way to set an `Authorization` header. Use a backend connection or an SSE/fetch client that supports custom headers. Do not put the Bearer token in the URL.

---

## `GET /`

Checks that the supplied Bearer token authenticates successfully.

Example:

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://api.example.com/
```

---

## `GET /codes/`

Returns current database-backed reference codes used by storm and claim data.

The response is keyed by stable machine-readable codes. Each code includes its current numeric ID and human-readable name.

Example shape:

```json
{
  "storm_codes": {
    "type": {
      "rain": {
        "id": 1,
        "name": "Rain Storm"
      },
      "snow": {
        "id": 2,
        "name": "Snow Storm"
      }
    },
    "status": {
      "predicted": {
        "id": 1,
        "name": "Predicted"
      },
      "ongoing": {
        "id": 2,
        "name": "Ongoing"
      },
      "concluded": {
        "id": 3,
        "name": "Concluded"
      }
    }
  },
  "claim_codes": {
    "state": {
      "online": {
        "id": 1,
        "name": "Online"
      }
    }
  }
}
```

The exact catalogue may evolve. Clients should request `/codes/` rather than hard-coding copied display names or numeric IDs.

For application logic, prefer stable dictionary keys/codes such as `rain`, `ongoing`, and `online` over human-readable `name` values.

---

## `GET /storms/`

Returns all currently exposed storms.

Only storms whose lifecycle has **not** reached Concluded are returned. In the current lifecycle this means Predicted and Ongoing storms.

Example:

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://api.example.com/storms/
```

Example response:

```json
[
  {
    "id": 3608617449183851,
    "thread_id": 1538620956556402688,
    "designation": "Storm Alpha",
    "status_id": 2,
    "status_code": "ongoing",
    "status_name": "Ongoing",
    "type_id": 1,
    "type_code": "rain",
    "type_name": "Rain Storm",
    "fhs_x": 128.0,
    "fhs_y": -128.0,
    "radius": 2000,
    "named_by": "Kvass",
    "prediction_detected_by": "Weather Watcher",
    "analyst_ongoing": "Forecaster, Radar Lead",
    "tracking_plotted_by": "Map Tech"
  }
]
```

If no Predicted/Ongoing storms are available, the response is:

```json
[]
```

### Polling clients

If you poll this endpoint, reconcile your local display against the entire returned list. Any previously displayed storm ID that disappears from the response should be removed from the client display.

---

## `GET /storms/{id}`

Returns one currently exposed storm by its stable API `id`.

The path value is **not** the Discord `thread_id`.

Example:

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://api.example.com/storms/3608617449183851
```

The endpoint returns `404 Not Found` if:

- the ID does not exist; or
- the storm exists historically but is no longer in the Predicted/Ongoing API feed.

---

## Storm rendering guidance

Storm geometry may include:

```text
fhs_x
fhs_y
radius
```

`radius` is expressed in metres.

If your integration renders the WWC-style storm coverage gradient, treat the stored radius as the **outer radius** of the storm area.

The inner zone extends from the centre to:

```text
0.60 × radius
```

The outer transition/gradient band extends from:

```text
0.60 × radius  →  1.00 × radius
```

This is a **60% radial fraction**, not 60% of the circle's total area.

This convention applies when rendering either rain or snow storm coverage.

---

## `GET /claims/`

Returns all weather-station claims.

Example:

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://api.example.com/claims/
```

---

## `GET /claims/{id}`

Returns one weather-station claim by its stable API `id`.

The path value is **not** the public/in-game claim `code`.

Example:

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://api.example.com/claims/7303474700435961
```

Example response:

```json
{
  "id": 7303474700435961,
  "code": 8764,
  "claim_hex_name": "Deadlands",
  "user_id": 123456789012345678,
  "hex_id": 27,
  "hex_name": "Deadlands",
  "state": "Online",
  "state_id": 1,
  "state_code": "online",
  "fhs_x": 45.6,
  "fhs_y": -105.3,
  "linked_id": 8123456789012345
}
```

A claim may be paired with one other station. If present, `linked_id` is the stable API ID of the linked claim and can be followed with `GET /claims/{id}`.

---

## `GET /icons/rain/`

Returns the current rain icon as a PNG image.

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://api.example.com/icons/rain/ \
  --output rain.png
```

Successful responses use:

```http
Content-Type: image/png
```

---

## `GET /icons/snow/`

Returns the current snow icon as a PNG image.

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://api.example.com/icons/snow/ \
  --output snow.png
```

Successful responses use:

```http
Content-Type: image/png
```

---

## Response behavior

Optional values that are currently unknown are omitted rather than returned as `null`. Clients must not assume every optional field is present.

Storm and claim detail routes use stable opaque integer `id` values:

- `storm.id` is separate from its optional Discord `thread_id`;
- `claim.id` is separate from its mutable public `code`.

For logic, prefer `*_code` fields over `*_name` display fields.

## Common HTTP responses

| Status | Meaning                                                       |
| -----: | ------------------------------------------------------------- |
|  `200` | Request succeeded                                             |
|  `400` | Invalid request metadata, such as a malformed `Last-Event-ID` |
|  `401` | Token missing or invalid                                      |
|  `403` | Token lacks required permission                               |
|  `404` | Resource is unavailable in the current endpoint/feed          |
|  `422` | Invalid path/request value                                    |
|  `500` | Unexpected server error                                       |
|  `503` | API temporarily unavailable                                   |

## Python polling example

```python
import os
import requests

base_url = "https://api.example.com"
token = os.environ["WWC_API_TOKEN"]
headers = {"Authorization": f"Bearer {token}"}

response = requests.get(
    f"{base_url}/storms/",
    headers=headers,
    timeout=10,
)
response.raise_for_status()

# Treat this complete set as authoritative for the current display.
current_storms = {
    storm["id"]: storm
    for storm in response.json()
}
```

For the real API address, access details, token problems, or integration questions, contact **dr.kvass on Discord**.
