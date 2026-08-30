# WWC API Changes Since 2026-08-28

This standalone change record covers public Bearer-token API changes made after the previous API changelog entry dated **28 August 2026**.

## Scope

The API base remains the same.

## 1. Claim reads are Admin-only

`GET /api/claims/` and `GET /api/claims/{id}` changed from ordinary `read` access to **`admin`** Bearer permission.

The permission hierarchy is still:

```text
read < write < admin
```

A normal `read` or `write` token can no longer retrieve claim records. Storm, codes, icons, authentication check and storm SSE continue to require `read`.

## 2. Explicit storm publication boundary

Introduced `storm.api_published`.

Public storm reads now have two required conditions:

```text
api_published = 1
AND status_code IN ('predicted', 'ongoing')
```

New storms default to private. WWC management controls the flag from Discord with `/storm publish` and `/storm private`.

This separates two concepts that were previously coupled:

- Discord/internal storm lifecycle;
- external API publication.

A storm may therefore be Predicted/Ongoing internally without being visible to API clients, this is to prevent storms without any data from needlessly appearing, or worse storms with temporary or incorrect triangulation data persisting until the next update of the client.

The public list and detail routes are explicitly wired to the publication-policy service:

```text
GET /api/storms/
GET /api/storms/{id}
```

A private or Concluded storm returns no public detail and is absent from the current list.

## 3. SSE publication semantics changed

Storm creation itself is no longer an external event. Because newly created storms are private by default, consumers are not told about creation until the storm enters the public publication boundary.

Current external storm event names are:

```text
storm.geometry_changed
storm.status_changed
storm.published
storm.unpublished
```

Publication transitions are explicit:

- `storm.published` for false → true;
- `storm.unpublished` for true → false.

Every event includes the resulting `api_published` flag and `active` value. `active` is true only when the resulting storm is both published and Predicted/Ongoing.

## 4. SSE delivery became event-driven internally

The durable `storm_event` SQLite outbox remains the authoritative replay source. Event IDs and `Last-Event-ID` behavior are unchanged.

The delivery loop no longer wakes on a short repeating SQLite polling interval during quiet periods. A local internal event broker wakes FastAPI after committed storm-domain mutations; FastAPI then reads the durable outbox and emits any new public events.

This is an implementation optimization, not a client-protocol replacement. The stream still provides:

- `stream.ready` on connect;
- durable numeric event IDs;
- replay after `Last-Event-ID`;
- reset-sequence recovery;
- approximately 15-second SSE keep-alive comments;
- Nginx anti-buffering headers/settings.

## 5. Graceful shutdown handles long-lived SSE connections

The FastAPI/Uvicorn entry point now explicitly closes WWC SSE transports during shutdown before completing normal graceful server shutdown. This avoids deployment/restart hangs caused by indefinitely open event streams.

No client API change is required; normal SSE reconnect logic is sufficient.

## 6. Current route surface is read-only

The actual public Bearer-token API currently exposes GET resources and the storm SSE stream only.

There are **no public POST, PUT, PATCH, or DELETE API routes** in the current application.

Current route set:

```text
GET /api/
GET /api/codes/
GET /api/storms/
GET /api/storms/{id}
GET /api/storms/events/
GET /api/claims/              # admin
GET /api/claims/{id}          # admin
GET /api/icons/rain/
GET /api/icons/snow/
```

## 7. Stable identity remains unchanged

These changes do not alter resource identity:

- `storm.id` remains the stable storm API ID;
- `claim.id` remains the stable claim API ID;
- Discord `thread_id` remains integration metadata;
- mutable claim `code` is not a stable API relationship key;
- contributor names in `APIStormObject` remain display values, not stable identifiers.

## Consumer action summary

Existing storm clients should:

1. treat `/api/storms/` as the authoritative public-current set;
2. handle `storm.published` and `storm.unpublished` events;
3. stop relying on a `storm.created` event;
4. reconcile or remove a storm whenever an event reports `active: false`;
5. continue reconnecting SSE with `Last-Event-ID` when available.
