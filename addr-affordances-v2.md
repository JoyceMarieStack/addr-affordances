# ADDR + Afford

| Phase | Focus | Artifact |
|---|---|---|
| Align | Job stories, activity steps | `job-stories.md` |
| Define | Resources, API profile | `api-profile.md` |
| Design | HTTP contract | `api-design.md` |
| **Afford** | Which responses suggest a next step, and what they point to | `affordance-map.md` |
| Refine | OpenAPI spec | `openapi.yaml` |

Afford sits after Design, before Refine — you need the endpoints to exist before you can point one
at another.

## Afford Phase

One artifact, one table. For each operation, decide whether its response should suggest a next
step, and if so, what condition triggers it and which operation it points to.

### `affordance-map.md`

| From operation | Chip label | Condition | Links to |
|---|---|---|---|
| `getWeeklyVolume()` | "Analyze why volume dipped in weeks 30 and 32" | A week falls well below the others nearby | `/volume/weekly/dip-analysis?weeks=` |

## Carrying into Refine

The table above becomes a `suggestion` field on the response schema — same decision, same place:

```json
{
  "weeks": [ ... ],
  "suggestion": {
    "label": "Analyze why volume dipped in weeks 30 and 32",
    "next": "/volume/weekly/dip-analysis?weeks=30,32"
  }
}
```

The server fills in `label` and `next` based on the Condition column; the client just renders
`label` as a chip and calls `next` if tapped.