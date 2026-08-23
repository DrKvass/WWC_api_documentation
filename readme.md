# WWC Weather API — Public User Documentation

The WWC Weather API provides authenticated, read-only access to current storm information, weather-station claims, reference codes, and public weather icons.

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
    "radius": 2000
  }
]
```
StormObject:
```py
class StormObject(WWCBaseModel):
    """Complete mutable storm domain object shared by Discord and FastAPI.

    ``id`` is the stable internal primary key. ``thread_id`` is the optional,
    unique Discord forum-thread ID. Discord-created storms always have a
    thread ID; the nullable schema supports future API creation/orchestration.
    Reference-table-backed properties expose their integer database ID, stable
    code, and human-readable name.
    """
 
    id: int # unique ID/sqlite3 PK
    thread_id: int | None = None # discord thread ID where the storm is managed in WWC
    designation: str # name of the storm

    status_id: int # status (ongoing/predicted/concluded) id/code/name
    status_code: str
    status_name: str

    type_id: int | None = None # type (rain/snow) id/code/name
    type_code: str | None = None
    type_name: str | None = None 

    size_id: int | None = None # size id/code/name
    size_code: str | None = None
    size_name: str | None = None

    origin_id: int | None = None # origin hex id/name
    origin_name: str | None = None

    intensity_id: int | None = None # Max intensity id/code/name
    intensity_code: str | None = None
    intensity_name: str | None = None

    fhs_x: float | None = None # Foxholestats x coord
    fhs_y: float | None = None # Foxholestats y coord
    radius: int | None = None # radius in meters

    named_by: int | None = None # discord user who named the storm
    prediction_detected_by: int | None = None # discord user who detected the prediction of the storm
    start_detected_by: int | None = None # discord user who detected the start of the storm
    end_detected_by: int | None = None # discord user who detected the end of the storm

    analyst_prediction: str | None = None # analysts of a predicted storm listed in a pre-formatted discord users string
    analyst_ongoing: str | None = None # analysts of an ongoing storm listed in a pre-formatted discord users string

    prediction_plotted_by: int | None = None # discord user who plotted the prediction map
    tracking_plotted_by: int | None = None # discord user who plotted the tracking map

    report_time: int | None = None # When the storm report was created
    detection_time: int | None = None # The time of the storm being found
    start_time: int | None = None # When the marked start time of the storm is/was
    end_time: int | None = None # When the marked end time of the storm is/was

    ws_prediction: str | None = None # returns a semi formatted list of all WS claim names used in a given storm to predict a storm
    ws_ongoing: str | None = None # returns a semi formatted list of all WS claim names used in a given storm to track a storm
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
ClaimObject:
```py
class ClaimObject(WWCBaseModel):
    """One weather-station claim from the active database.

    ``id`` is the stable internal primary key. ``code`` is the unique,
    user-facing in-game claim code and can change without changing ``id``.
    """

    id: int # Unique id / sqlite3 PK
    code: int # code of the WS used to connect between WSs
    claim_hex_name: str # name of the WS
    user_id: int # discord user owner of the WS
    hex_id: int # id of the hex that the claim is in
    hex_name: str | None = None # name of the hex the claim is in
    state: str # Online/Offline/Claimed -> describes if the WS exists (offline or online) / is planned (claimed) / is usable (online only)
    state_id: int | None = None # id of state
    state_code: str | None = None # code of state
    fhs_x: float | None = None # FSH x,y coords
    fhs_y: float | None = None
    linked_id: int | None = None # id of a weather station that is connected to this weather station. 
    # Note: links are always one-to-one relations, so if the other WS returns a different linked_id, then the ws link was deleted/switch between API calls!
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

| Status | Meaning |
|---:|---|
| `200` | Request succeeded |
| `401` | Token missing or invalid |
| `403` | Token lacks required permission |
| `404` | Resource is unavailable in the current endpoint/feed |
| `422` | Invalid path/request value |
| `500` | Unexpected server error |
| `503` | API temporarily unavailable |

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

## Changelog

See [`WWC_API_CHANGELOG.md`](WWC_API_CHANGELOG.md) for public integration-impact changes.
