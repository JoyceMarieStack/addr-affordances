# Job Stories & Activity Steps — Coach API

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