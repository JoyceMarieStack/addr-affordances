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

## Two ways in

The consumer isn't limited to what the server offers — they can always type a free-form
question. The server *additionally* offers a small curated set of next steps. Both paths hit the
same operations; the suggestions are a shortcut, not the only door in.

- **Free text** → consumer types anything → resolved to whichever operation fits
- **Suggestion** → consumer taps a server-offered chip → resolved to a known operation with known parameters

## What a chip label actually is

A **chip** is the rounded, tappable pill-button UI convention — the thing you see in the WHOOP
screenshots: `[ Can you summarize monthly volume trends? ]`. A **chip label** is just the text
written on that button — the exact words the consumer sees and taps.

It's the human-readable side of the affordance; `next` (the URL/operation it calls) is the
machine side. Same affordance, two views.

A chip label is a **specific claim about the data just shown** — not generic ("see more detail"),
but naming the actual week, month, or delta. The Condition column is what has to be true before
you're allowed to make that claim. If the condition isn't met, there's no chip — not a blank one,
not a generic fallback, none at all.

### More examples

| From operation | Condition | Chip label | Links to |
|---|---|---|---|
| `getWeeklyVolume()` | A week falls well below the others nearby | "Analyze why volume dipped in weeks 30 and 32" | `/volume/weekly/dip-analysis?weeks=30,32` |
| `getWeeklyVolume()` | A week is well above the others nearby | "What made week 28 such a big week?" | `/volume/weekly/dip-analysis?weeks=28` |
| `summarizeVolumeTrend()` | Month-over-month change > 15% | "Why did my lifting volume peak in July?" | `/volume/monthly/change-analysis?periodA=2026-06&periodB=2026-07` |
| `summarizeVolumeTrend()` | No dip and no spike this period | *(no chip offered)* | — |
| `getWeeklyVolume()` | always | "Break this down by muscle group" | `/volume/muscle-group-breakdown?range=` |

## Afford Phase

One artifact, one table. For each operation, decide whether its response should offer a curated
next step, what condition triggers it, and which operation it points to.

### `affordance-map.md`

| From operation | Chip label | Condition | Links to |
|---|---|---|---|
| `getWeeklyVolume()` | "Analyze why volume dipped in weeks 30 and 32" | A week falls well below the others nearby | `/volume/weekly/dip-analysis?weeks=` |

The server picks 2–4 of these per response, not all of them — that's the curation. Which ones get
picked is a ranking decision (relevance to what just happened), separate from whether they're
valid at all (the Condition column).

## Carrying into Refine

The table becomes a `suggestions` field on the response schema — a short, server-picked list:

```json
{
  "weeks": [ ... ],
  "suggestions": [
    {
      "label": "Analyze why volume dipped in weeks 30 and 32",
      "next": "/volume/weekly/dip-analysis?weeks=30,32"
    }
  ]
}
```

Free-text input doesn't go through this field at all — it goes through normal NL-to-operation
resolution, same as any conversational query. The `suggestions` field only covers the second,
tap-a-chip path.