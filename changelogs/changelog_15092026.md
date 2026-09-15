# WWC API Change Log - 15 September 2026

## predicted plot hatching

Partial triangulation now uses the size-specific minimum radius (Small 1,092 m,
Medium 2,191 m, Large 2,846 m). Regular predicted triangulations add pink
crosshatching across the plot; ongoing storms remain unhatched. Bot images,
website layers, editor previews and legend samples follow these rules.
Public API fields and event names are unchanged. Rendering recommendations below
have been updated.

## analysts and stations consolidated (breaking change)

Storm list/detail responses now expose one `analysed_by` display-name string
and one `storm_claims` weather-station-name string for each storm. Both apply
throughout the storm lifecycle. Schema 28 merges existing assignments and
removes duplicate people/stations within each group.

| Removed fields | Replacement |
| --- | --- |
| `analyst_prediction`, `analyst_ongoing` | `analysed_by` |
| `ws_prediction`, `ws_ongoing` | `storm_claims` |

The replacements retain the optional, comma-separated string shape; null fields
are omitted from public HTTP responses. Names are presentation values and can
contain commas, so do not parse them as identifiers. Detector and plotted-by
fields remain unchanged. Storm lifecycle, publication rules and SSE event
names remain unchanged. Refresh cached storm responses after deployment.

```json
{"analysed_by":"Alice, Bob","storm_claims":"Station Alpha, Station Bravo"}
```