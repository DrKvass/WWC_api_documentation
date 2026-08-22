# WWC Weather API — User Documentation

The WWC Weather API provides authenticated, read-only access to current storm and weather-station claim information.

**Base URL**

```text
get the base URL from Dr.Kvass on discord.
```

## Access

To request API access, contact **dr.kvass on Discord**.

Send the issued token with every request:

```http
Authorization: Bearer YOUR_TOKEN
```

Treat the token like a password. Never publish it, place it in a URL, commit it to source control, or include it in screenshots/logs.

## Identity fields

Storms and claims have stable opaque integer `id` values used by detail routes.

- A storm's `thread_id` is a separate Discord identifier and may be absent.
- A claim's `code` is a separate user-facing in-game code and may change.

Use `id` for API resource URLs. Do not substitute `thread_id` or `code`.

## Endpoints

### Authentication check

```http
GET /
```

Returns information about the authenticated token. This is useful for confirming that a token works.

### List storms

```http
GET /storms/
```

Example:

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://k.w4.si/storms/
```

Successful response:

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

If no storms exist, the response is `[]`.

### Get one storm

```http
GET /storms/{id}
```

Example:

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://k.w4.si/storms/3608617449183851
```

The path value is the storm's `id`, not its Discord `thread_id`.

### List claims

```http
GET /claims/
```

Example:

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://k.w4.si/claims/
```

### Get one claim

```http
GET /claims/{id}
```

Example:

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://k.w4.si/claims/7303474700435961
```

The path value is the claim's stable `id`, not its public `code`.

Example claim response:

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

## Response behavior

Unknown optional values are omitted instead of being returned as `null`. Clients must not assume every optional field is present.

For application logic, prefer stable machine-readable `*_code` fields over human-readable `*_name` fields, for example `status_code = "ongoing"` rather than comparing the display label.

In future create and Update implementations only `*_code` will be passable back to the API and not `*_name` fields. Please consider this when designing your software.

## Common fields

### Storm

- `id`, optional `thread_id`, `designation`;
- status ID/code/name;
- optional type, size, origin, and intensity data;
- optional FHS coordinates and radius in metres;
- optional contributor, analyst, timeline, and weather-station information.

Timeline values are Unix timestamps in seconds, FHS coordinates refers to foxholestats coordinates.

### Claim

- `id` — stable API identifier;
- `code` — current public claim code;
- claim name, owner, hex, and state data;
- optional FHS coordinates;
- optional `linked_id`, containing the stable API `id` of the paired station.

### Linked claims

A claim may be paired with one other weather station. When a pair exists, the response contains:

```json
{
  "linked_id": 8123456789012345
}
```

Use that value with `GET /claims/{id}` to retrieve the linked station. The field is omitted when the claim is not linked. A claim can belong to only one pair at a time.

## HTTP responses

| Status | Meaning                         |
| -----: | ------------------------------- |
|  `200` | Request succeeded               |
|  `401` | Token missing or invalid        |
|  `403` | Token lacks required permission |
|  `404` | Resource ID does not exist      |
|  `422` | Invalid path/request value      |
|  `500` | Unexpected server error         |
|  `503` | API temporarily unavailable     |

## Python example

```python
import os
import requests

token = os.environ["WWC_API_TOKEN"]
response = requests.get(
    "https://k.w4.si/storms/",
    headers={"Authorization": f"Bearer {token}"},
    timeout=10,
)
response.raise_for_status()

for storm in response.json():
    print(storm["id"], storm["designation"])
```

## Security

- Use only HTTPS.
- Keep the token in an environment variable or secrets manager.
- Do not expose a private token in public/browser-side JavaScript.
- If a token may have been exposed, contact **dr.kvass on Discord** promptly.

## Support

For access or integration help, contact **dr.kvass on Discord**.
