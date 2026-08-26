# WWC Weather API — Public Release Note: Storm SSE + Contributor Display Names

**Release date:** 26 August 2026

This is a fixed public release note covering two related API updates only:

1. the new real-time storm Server-Sent Events feed; and
2. the subsequent change from Discord contributor user IDs/mentions to readable Discord display names in storm responses.

It is **not** intended to become a continuously appended changelog.

## 1. Added — real-time storm Server-Sent Events

A new authenticated endpoint is available:

```text
GET /storms/events/
```

It uses `text/event-stream` and the same `Authorization: Bearer ...` authentication as the other read endpoints.

The stream emits these event names:

```text
storm.created
storm.status_changed
storm.geometry_changed
```

Clients are notified when:

- a new storm is created;
- a storm changes status, including a Concluded storm being restored to Predicted or Ongoing;
- an active Predicted/Ongoing storm changes `fhs_x`, `fhs_y`, or `radius`.

Geometry edits made while a storm is Concluded are not emitted.

The event payload is deliberately a change notification rather than a full storm object. It includes the stable `storm_id`, resulting status, `active` flag, changed fields, and event timestamp. When `active` is `true`, refetch `GET /storms/{storm_id}` or reconcile `GET /storms/`. When `active` is `false`, remove the storm from the current display.

The stream supports the standard `Last-Event-ID` header for reconnect/replay, sends a `stream.ready` control event on connection, and sends keep-alive comments during quiet periods.

For a race-resistant initial load: open SSE first, wait for `stream.ready`, fetch `GET /storms/`, then apply any numbered events received during that snapshot request in event-ID order.

## 2. Changed — storm contributor fields now use Discord display names

`GET /storms/` and `GET /storms/{id}` now return `APIStormObject` responses in which storm contributor fields contain readable Discord server display names rather than contributor snowflake IDs or `<@...>` mention strings.

Affected fields:

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

For the single-user fields, the value is one string. For the two analyst fields, multiple contributors are represented as a comma-separated display-name string.

Example:

```json
{
  "id": 3608617449183851,
  "designation": "Storm Alpha",
  "status_code": "ongoing",
  "named_by": "Kvass",
  "prediction_detected_by": "Weather Watcher",
  "analyst_ongoing": "Forecaster, Radar Lead",
  "tracking_plotted_by": "Map Tech"
}
```

The API prefers the member's server nickname. If no nickname exists it falls back to the Discord display name/username. If a historical contributor has left the server, an account-level display name/username is attempted. If Discord cannot resolve the user at all, that contributor field is omitted instead of returning the raw numeric ID.

### Integration impact

This is a response-type change for the six single-user storm contributor fields: clients that previously expected an integer Discord user ID must now expect a string display name. The two analyst fields remain strings, but their content is now readable comma-separated names instead of Discord mention syntax.

Contributor names are mutable presentation data and are **not stable identifiers**. Do not key application state by a nickname. Continue to use the stable storm `id` for resource identity.

This contributor-name change applies to storm resources only. Claim resources retain their existing numeric `user_id` owner field.

## 3. Unchanged

The following behavior remains unchanged across these updates:

- API authentication still uses the existing Bearer-token system;
- all current endpoints remain read-only;
- `/storms/` and `/storms/{id}` expose Predicted/Ongoing storms only;
- Concluded storms leave the current storm feed;
- SSE events continue to identify storms by stable `storm_id`;
- `thread_id` remains a Discord thread/channel identifier where present;
- claim API fields and claim ownership representation are unchanged.
