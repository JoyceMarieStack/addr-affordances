# Affordances — Coach API

Built from the Align-phase job stories and activity steps for the monthly and weekly volume
examples. Grounded against the real WHOOP v2 API (`developer.whoop.com/api`) — noted explicitly
where the underlying data isn't actually native to WHOOP.

---

## A note on the underlying data

WHOOP's real API doesn't expose lifting volume. The relevant real endpoint is:

```
GET /v2/activity/workout
```
Paginated, sorted by start time descending. Each record returns `sport_id`, `start`, `end`,
`score.strain`, and heart-rate/calorie data — **not** a per-exercise set/rep/weight breakdown.
"Volume (kg)" in the WHOOP coach screenshots is a **derived metric**, computed by the coach layer
on top of raw workout records (or a separate strength-logging data source), not something WHOOP's
API returns directly. Flagging this because the Afford phase's Condition column depends on that
derived aggregation existing and being reliable — it's not a free read against WHOOP's own data,
it's a build.

The operations below (`getMonthlyVolume`, `getWeeklyVolume`, etc.) sit in the **Coach API**, one
layer above WHOOP's `/v2/activity/workout`, which they call and aggregate.

---

## What a chip is

A **chip** is the rounded, tappable pill-button UI convention — the thing you see in the WHOOP
screenshots: `[ Can you summarize monthly volume trends? ]`. A **chip label** is just the text
written on that button — the exact words the consumer sees and taps.

It's the human-readable side of the affordance; the operation + parameters it resolves to is the
machine side. Same affordance, two views.

A chip label is a **specific claim about the data just shown** — not generic ("see more detail"),
but naming the actual week, month, or delta. The Condition column below is what has to be true
before you're allowed to make that claim. If the condition isn't met, there's no chip — not a
blank one, not a generic fallback, none at all.

## Two ways in

The consumer isn't limited to what the server offers — they can always type a free-form question.
The server *additionally* offers a small curated set of next steps. Both paths hit the same
operations; the suggestions are a shortcut, not the only door in.

- **Free text** → consumer types anything → resolved to whichever operation fits
- **Suggestion** → consumer taps a server-offered chip → resolved to a known operation with known parameters

---

## Set A — Monthly Volume Trend

### Job Stories

| ID | When... | I want to... | So I can... |
|---|---|---|---|
| JS1 | I've completed multiple lifting sessions this month | Ask for a summary of my training volume trend | See whether my training load is progressing appropriately |
| JS2 | I see a volume spike or drop in the monthly summary | Ask why it happened | Understand what drove the change (e.g. which muscle groups, which weeks) |

### Activity Steps

| Job Story | Activity | Activity Step | Description |
|---|---|---|---|
| JS1 | View monthly load | Requests monthly volume summary | Consumer asks for a trend summary across months |
| JS1 | View monthly load | Retrieves aggregated totals | System returns per-month totals + trend delta |
| JS2 | Investigate change | Requests root-cause explanation | Consumer asks why the summary moved, anchored to the period just shown |
| JS2 | Investigate change | Retrieves contributing detail | System returns the sessions/weeks responsible for the delta |

### Affordance Map

| From operation | Rel type | Condition | Chip label | Links to |
|---|---|---|---|---|
| `getMonthlyVolume()` — aggregates `GET /v2/activity/workout` by calendar month, `sport_id` filtered to strength/weightlifting | `root-cause` | Month-over-month change > 15% | "Why did my lifting volume peak in July?" | `explainVolumeChange(periodA=2026-06, periodB=2026-07)` |
| `getMonthlyVolume()` | — | No month-over-month change exceeds threshold | *(no chip offered)* | — |

---

## Set B — Weekly Volume Comparison

### Job Stories

| ID | When... | I want to... | So I can... |
|---|---|---|---|
| JS1 | I want to compare training load at a finer grain than monthly | See a weekly volume breakdown | Spot which weeks were heavy, moderate, or deload |
| JS2 | One or more weeks show volume well below surrounding weeks | Ask why those specific weeks dipped | Tell whether it was planned recovery, a logging gap, or something worth addressing |

### Activity Steps

| Job Story | Activity | Activity Step | Description |
|---|---|---|---|
| JS1 | View weekly load | Requests weekly breakdown | Consumer asks for volume by week rather than by month |
| JS1 | View weekly load | Retrieves per-week totals | System returns totals per ISO week over the requested range |
| JS2 | Investigate anomaly | Requests dip analysis | Consumer asks why named week(s) fell below baseline |
| JS2 | Investigate anomaly | Retrieves session-level detail | System returns contributing sessions for the flagged week(s), disambiguating a genuine low-volume week from a week with no logged session |

### Affordance Map

| From operation | Rel type | Condition | Chip label | Links to |
|---|---|---|---|---|
| `getWeeklyVolume()` — aggregates `GET /v2/activity/workout` by ISO week, `sport_id` filtered to strength/weightlifting | `anomaly-cause` | A week falls well below the others nearby, **and** at least one `workout` record exists in that week (i.e. it's a real dip, not a logging gap) | "Analyze why volume dipped in weeks 30 and 32" | `analyzeVolumeDip(weeks=[30,32])` |
| `getWeeklyVolume()` | — | A week has **zero** `workout` records at all in that week | *(no chip — this is a missing-data state, not a dip; shown as a gap in the chart, not offered as an explainable anomaly)* | — |

---

## Carrying into Refine

The map becomes a `suggestions` field on the Coach API's own response schema — a short,
server-picked list, separate from WHOOP's raw API shape entirely:

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

Free-text input doesn't go through this field — it goes through normal NL-to-operation
resolution. The `suggestions` field only covers the tap-a-chip path.