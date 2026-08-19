# Coach API

* Supports the training-volume coaching experience: monthly/weekly trend summaries and
  root-cause / anomaly explanations
* Built on top of WHOOP's real `GET /v2/activity/workout` endpoint — volume itself is a
  **derived** metric (see note below), not a native WHOOP field
* Scope: Public

| Operation Name | Description | Participants | Resource(s) | Emitted Events | Operation Details | Traits | Afforded Transitions |
|---|---|---|---|---|---|---|---|
| getMonthlyVolume() | Aggregate lifting volume by calendar month | Athlete, Coach | VolumeSummary, Workout | Volume.Summarized | __Request Parameters:__ athleteId, monthRange __Returns:__ VolumeSummary | safe / synchronous | `root-cause` → explainVolumeChange() *(if month-over-month change > 15%)* |
| explainVolumeChange() | Explain what drove a volume change between two months | Athlete, Coach | VolumeChangeExplanation, Workout | VolumeChange.Explained | __Request Parameters:__ athleteId, periodA, periodB __Returns:__ VolumeChangeExplanation | safe / synchronous | — |
| getWeeklyVolume() | Aggregate lifting volume by ISO week | Athlete, Coach | WeeklyVolume, Workout | WeeklyVolume.Viewed | __Request Parameters:__ athleteId, weekRange __Returns:__ WeeklyVolume | safe / synchronous | `anomaly-cause` → analyzeVolumeDip(weeks[]) *(if a week falls well below neighbors AND has ≥1 workout record — i.e. a real dip, not a gap)* |
| analyzeVolumeDip() | Explain why specific week(s) fell relative to their neighbors | Athlete, Coach | DipAnalysis, Workout | DipAnalysis.Viewed | __Request Parameters:__ athleteId, weeks[] __Returns:__ DipAnalysis | safe / synchronous | — |

## Note on derived data

`getMonthlyVolume()` and `getWeeklyVolume()` both aggregate over WHOOP's
`GET /v2/activity/workout` collection, filtered by `sport_id` to strength/weightlifting
activities. WHOOP's own record for each workout returns `score.strain`, duration, and heart-rate
data — it does **not** return per-exercise sets, reps, or weight. "Volume (kg)" is therefore
computed by the Coach API from a separate strength-logging source (or inferred from workout
metadata), not read directly from WHOOP. This is flagged here because every
`Afforded Transitions` condition above depends on that derived aggregation being correct — an
error in the aggregation layer produces a wrong chip, not just a wrong number.

## Note on the added column

`Afforded Transitions` is the Phase 2.5 (Afford) addition. Each entry names its rel type and, where
the transition isn't unconditional, the condition that gates it — sourced from
`coach-affordance-map.md`, not re-derived here.
