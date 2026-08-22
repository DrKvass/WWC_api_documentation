# WWC Weather API — User Documentation

The WWC Weather API provides authenticated, read-only access to current storm, weather-station claim, and reference-code information.

## Access

The API address and Bearer token are provided to authorized users.

For access or the current API base URL, contact **dr.kvass on Discord**.

The examples in this document use the non-production placeholder address:

```text
get the base URL from Dr.Kvass on discord.
```

Replace it with the API address supplied to you.

Send the issued token with every request:

```http
Authorization: Bearer YOUR_TOKEN
```

Treat the token like a password. Never publish it, place it in a URL, commit it to source control, or include it in screenshots/logs.

## Identity fields

Storms and claims have stable opaque integer `id` values used by detail routes.

- A storm's `thread_id` is a separate Discord identifier and may be absent.
- A claim's `code` is a separate user-facing in-game code and may change.

Use `id` for API resource URLs. Do not substitute `thread_id` or claim `code`.

## Available endpoints

```text
GET /
GET /codes/
GET /storms/
GET /storms/{id}
GET /claims/
GET /claims/{id}
```

All current endpoints require a token with read access or higher.

---

## Authentication check

```http
GET /
```

Confirms that the supplied token is valid and returns basic information about that token.

---

## Reference codes

```http
GET /codes/
```

Returns the current database-backed codes used by storm and claim data.

This endpoint is useful when building integrations because it provides the stable machine-readable `code`, numeric `id`, and current human-readable `name` for supported reference values.

The catalogue is read from the current database for every request. If a reference value is deliberately changed by the WWC maintainers, the next `/codes/` response reflects that change.

### Example request

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://api.example.com/codes/
```

### Response shape

```json
{
  "storm_codes": {
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
    },
    "type": {
      "unknown": {
        "id": 0,
        "name": "Unknown"
      },
      "rain": {
        "id": 1,
        "name": "Rain Storm"
      },
      "snow": {
        "id": 2,
        "name": "Snow Storm"
      }
    },
    "size": {
      "small": {
        "id": 1,
        "name": "Small"
      },
      "medium": {
        "id": 2,
        "name": "Medium"
      },
      "large": {
        "id": 3,
        "name": "Large"
      }
    },
    "intensity": {
      "weak": {
        "id": 1,
        "name": "Weak"
      },
      "medium": {
        "id": 2,
        "name": "Medium"
      },
      "strong": {
        "id": 3,
        "name": "Strong"
      },
      "severe": {
        "id": 4,
        "name": "Severe"
      },
      "low": {
        "id": 5,
        "name": "Low"
      },
      "high": {
        "id": 6,
        "name": "High"
      }
    },
    "analyst_type": {
      "named_by": {
        "id": 0,
        "name": "Named by"
      },
      "analysed_prediction": {
        "id": 1,
        "name": "Analyzed by (prediction)"
      },
      "analysed_ongoing": {
        "id": 2,
        "name": "Analyzed by (ongoing)"
      },
      "detected_concluded": {
        "id": 3,
        "name": "Detected by (concluded)"
      },
      "detected_prediction": {
        "id": 4,
        "name": "Detected by (prediction)"
      },
      "detected_ongoing": {
        "id": 5,
        "name": "Detected by (ongoing)"
      },
      "plotted_prediction": {
        "id": 6,
        "name": "Plotted by (prediction)"
      },
      "plotted_ongoing": {
        "id": 7,
        "name": "Plotted by (ongoing)"
      }
    },
    "time": {
      "started": {
        "id": 0,
        "name": "Started"
      },
      "concluded": {
        "id": 1,
        "name": "Concluded"
      },
      "reported": {
        "id": 2,
        "name": "Reported"
      },
      "detected": {
        "id": 3,
        "name": "Detected"
      }
    }
  },
  "claim_codes": {
    "state": {
      "online": {
        "id": 1,
        "name": "Online"
      },
      "claimed": {
        "id": 2,
        "name": "Claimed"
      },
      "offline": {
        "id": 3,
        "name": "Offline"
      }
    }
  }
}
```

The exact catalogue may evolve. Integrations should request `/codes/` when they need the current supported values rather than relying on copied display names or numeric IDs.

For application logic, prefer the dictionary key/code such as `rain`, `ongoing`, or `online`. Human-readable `name` values are intended for display.

---

## List storms

```http
GET /storms/
```

Example:

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://api.example.com/storms/
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

## Get one storm

```http
GET /storms/{id}
```

Example:

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://api.example.com/storms/3608617449183851
```

The path value is the storm's stable `id`, not its Discord `thread_id`.

---

## List claims

```http
GET /claims/
```

Example:

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://api.example.com/claims/
```

## Get one claim

```http
GET /claims/{id}
```

Example:

```bash
curl \
  -H "Authorization: Bearer $WWC_TOKEN" \
  https://api.example.com/claims/7303474700435961
```

The path value is the claim's stable `id`, not its public claim `code`.

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

For application logic, prefer stable machine-readable `*_code` fields over human-readable `*_name` fields. The `/codes/` endpoint provides the current code catalogue and display names.

## Common fields

### Storm

- `id`, optional `thread_id`, `designation`;
- status ID/code/name;
- optional type, size, origin, and intensity data;
- optional FHS coordinates and radius in metres;
- optional contributor, analyst, timeline, and weather-station information.

Timeline values are Unix timestamps in seconds.

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

base_url = "https://api.example.com"
token = os.environ["WWC_API_TOKEN"]

response = requests.get(
    f"{base_url}/storms/",
    headers={"Authorization": f"Bearer {token}"},
    timeout=10,
)
response.raise_for_status()

for storm in response.json():
    print(storm["id"], storm["designation"])
```

Replace the example base URL with the address supplied by **dr.kvass on Discord**.

## Security

- Use only HTTPS.
- Keep the token in an environment variable or secrets manager.
- Do not expose a private token in public/browser-side JavaScript.
- If a token may have been exposed, contact **dr.kvass on Discord** promptly.

## Support

For API access, the current API address, or integration help, contact **dr.kvass on Discord**.
