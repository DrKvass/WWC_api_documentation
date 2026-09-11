# WWC API Change Log

## 11 September 2026 — plot types and detector limits

Storm responses now include `plot_type`, a stable string: `triangulation`,
`partial_triangulation`, or `prediction_probability_map`. The latter two apply
only to Predicted storms. Every other lifecycle status uses `triangulation`.
Creating a triangulation or probability map sets this field automatically.
Authorized storm editors can set partial triangulation with `/storm edit plot_type`
or the website’s Edit → Classification → Plot type selector. Plot type is the
last Classification row in storm details. Existing storm-edit permissions apply.
Existing Predicted probability maps are classified during schema migration 26;
other existing storms default to triangulation. A plot-type change on a published
active storm emits `storm.geometry_changed` with `plot_type` in `changed_fields`.
Refetch the storm when this event arrives.

Recommended rendering for Predicted storms:

| `plot_type` | Display |
| --- | --- |
| `triangulation` | Pink outline. |
| `partial_triangulation` | Red dashed outline and red diagonal stripes at alpha 0.5 between the maximum radius and 1,092 m from the centre. At radius ≤1,092 m, use the outline only. |
| `prediction_probability_map` | Orange dashed outline around the candidate probability footprint. |

| `plot_type` | Meaning |
| --- | --- |
| `triangulation` | We know the exact centre and radius of the storm. |
| `partial_triangulation` | We know the exact centre of the storm, but the radius is a mystery. |
| `prediction_probability_map` | We know there will be a storm, but we dont know the centre or radius, this is just the chance a storm centre appears here. |

Clip fills, stripes and outlines to the map's hex footprint, including the
outline along a cut map edge. Distances are metres, using the existing FHS
metric conversion. Plot type does not change storm type, size or intensity.

Each of `prediction_detected_by`, `start_detected_by`, and `end_detected_by`
now represents zero, one or two people, independently. Public JSON retains its
existing optional string shape: omitted from public HTTP responses when no people are assigned, one server display name,
or two display names joined with `", "`. For example:

```json
{"plot_type":"partial_triangulation","prediction_detected_by":"Alice, Bob"}
```

Display names may contain commas and must not be parsed as stable identities.
Bot reports and website details include both names. A third assignment is
rejected; remove one person first. Named-by and plotted-by remain single-person,
and analysed-by remains unrestricted.