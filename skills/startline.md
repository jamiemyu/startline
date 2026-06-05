---
name: startline
description: Full race readiness analysis — gap scoring, performance prediction, and HTML report
---

# /startline — Race Readiness Analysis

This skill runs a complete multi-step race readiness analysis. Follow every module in order.

---

## Module 1: Race Configuration
*(Implemented in issue #3 — placeholder)*

Read the race configuration from the conversation (sport, race type, date, A/B/C races). This will be filled in when issue #3 is merged.

---

## Module 2: Garmin Data Ingestion

This module fetches and processes the athlete's training history from Garmin. It produces a single `garminMetrics` object that all downstream modules consume. Follow every step in order.

### Step 1 — Determine the relevant activity types

Use the `sport` value from the race configuration (set in Module 1). Map it to Garmin activity type labels as follows:

| `sport` value | Garmin activity types to include |
|---|---|
| `running` | `running`, `trail_running`, `Run`, `Trail Run` |
| `road_cycling` | `cycling`, `road_cycling`, `Ride`, `Road` |
| `mtb` | `mountain_biking`, `MTB`, `Mountain Bike Ride` |

Keep this list of accepted types as `relevantTypes` — you will use it to filter in Step 1c below.

**Step 1a — Calculate the date window.**
Compute the start date as today minus 16 weeks (112 days). Format both dates as `YYYY-MM-DD`. Store as `windowStart` and `windowEnd` (today).

**Step 1b — Fetch activities from Garmin.**
Call `mcp__garmin__get_activities` with the following parameters:
- `startDate`: `windowStart`
- `endDate`: `windowEnd`
- `limit`: 200 (to capture a full 16-week window without truncation)

**Step 1c — Filter by sport.**
From the returned activity list, keep only activities whose `activityType` (or equivalent label field) matches one of the `relevantTypes` for the athlete's sport. Store the filtered list as `filteredActivities`.

---

### Step 2 — Compute volume metrics

Work from `filteredActivities` for all calculations in this step.

**Step 2a — Longest single effort.**
Find the activity with the highest `distance` value. Record:
- `distance`: the distance value in miles (convert from meters if necessary: divide by 1609.344)
- `duration`: the activity's elapsed time formatted as `HH:MM:SS`
- `date`: the activity start date in ISO format (`YYYY-MM-DD`)

Store as `longestEffort`.

**Step 2b — Top-3 long efforts average.**
Sort `filteredActivities` by `distance` descending. Take the top 3. Compute the arithmetic mean of their `distance` values (in miles). Store as `top3Average` (a single number, rounded to 2 decimal places).

If fewer than 3 activities exist, average however many are available. If zero activities, set `top3Average` to `null`.

**Step 2c — Weekly volume trend.**
Group `filteredActivities` by ISO week (format: `YYYY-Www`, e.g. `2025-W03`). For each week:
- Sum the `distance` values of all activities in that week (in miles)
- Count the number of activities

Produce an array of objects `{ week, totalDistance, activityCount }` sorted oldest-to-newest. Store as `weeklyTrend`.

Weeks with zero activities are omitted (sparse weeks produce gaps, not zero-rows).

---

### Step 3 — Compute elevation metrics

Continue working from `filteredActivities`.

**Step 3a — Average elevation per long effort.**
Take the same top-3 activities identified in Step 2b (by distance). Compute the arithmetic mean of their `elevationGain` values (in feet). Store as `avgElevationPerLongEffort`.

If elevation data is missing for an activity, treat it as 0 for averaging purposes. If zero activities, set to `null`.

**Step 3b — Cumulative block elevation gain.**
Sum `elevationGain` across all activities in `filteredActivities`. Store as `cumulativeBlockElevation` (feet).

---

### Step 4 — Fetch physiological metrics

Make the following four Garmin MCP calls independently (they do not depend on each other). Use today's date formatted as `YYYY-MM-DD` for all date parameters.

**Step 4a — Resting heart rate.**
Call `mcp__garmin__get_stats` with `date`: today.
Extract the `restingHeartRate` field (bpm). If unavailable or null, store `null`.

**Step 4b — HRV.**
Call `mcp__garmin__get_hrv_data` with `date`: today.
Extract the most recent HRV reading (milliseconds). Look for fields such as `lastNight`, `hrvValue`, or the most recent entry in a returned array. If unavailable, store `null`.

**Step 4c — Body battery.**
Call `mcp__garmin__get_body_battery` with `date`: today.
Extract the current or most recent body battery value (0–100). If unavailable, store `null`.

**Step 4d — VO2max estimate.**
Scan `filteredActivities` for a `vo2MaxValue` field (present on many Garmin run and ride activities). Find the most recent activity where `vo2MaxValue` is a non-null number. Store that value as `vo2maxEstimate`. If no activity carries this field, store `null`.

---

### Step 5 — Handle sparse history

Count the number of distinct ISO weeks present in `weeklyTrend`. Store as `weeksWithData`.

If `weeksWithData` is fewer than 4:
- Do **not** halt or ask the user for more data — continue to Step 6 with whatever data exists
- Set `sparseHistory: true`
- The HTML report (Module 8) will display: *"Limited training history — analysis based on [weeksWithData] weeks of data. Confidence intervals are wider."*

If `weeksWithData` is 4 or more, set `sparseHistory: false`.

---

### Step 6 — Assemble and store the metrics object

Assemble the following `garminMetrics` object and hold it in your working context for use by all downstream modules. Do not print the raw object to the user — it is an internal data structure.

```json
{
  "sport": "<value from race config>",
  "dataWindowWeeks": 16,
  "activitiesAnalyzed": <count of filteredActivities>,
  "sparseHistory": <true | false>,
  "volume": {
    "longestEffort": {
      "distance": <miles, number>,
      "duration": "<HH:MM:SS>",
      "date": "<YYYY-MM-DD>"
    },
    "top3Average": <miles, number | null>,
    "weeklyTrend": [
      { "week": "<YYYY-Www>", "totalDistance": <miles>, "activityCount": <n> }
    ]
  },
  "elevation": {
    "avgPerLongEffort": <feet, number | null>,
    "cumulativeBlockGain": <feet, number>
  },
  "physiological": {
    "vo2maxEstimate": <number | null>,
    "restingHR": <bpm, number | null>,
    "latestHRV": <ms, number | null>,
    "bodyBattery": <0-100, number | null>
  }
}
```

Rules for assembly:
- All distance values must be in **miles**. Convert if the Garmin API returns a different unit (divide meters by 1609.344).
- All elevation values must be in **feet**. Convert if the Garmin API returns meters (multiply by 3.28084).
- If any physiological field is unavailable, set it to `null` — never halt or ask the user to provide it manually.
- `activitiesAnalyzed` is the count of `filteredActivities` (after sport-type filtering), not the raw fetch count.

Once assembled, confirm to the user in one short sentence that training data has been loaded (e.g., "Loaded 47 activities across 14 weeks."), then proceed to Module 3.

---

## Module 3: Course Data Ingestion
*(Implemented in issue #4 — placeholder)*

---

## Module 4: Training Block Detection
*(Implemented in issue #5 — placeholder)*

---

## Module 5: Gap Analysis
*(Implemented in issues #6–#10 — placeholder)*

---

## Module 6: Performance Prediction
*(Implemented in issue #11 — placeholder)*

---

## Module 7: Readiness % Aggregation
*(Implemented in issue #12 — placeholder)*

---

## Module 8: HTML Report Generation
*(Implemented in issues #13–#16 — placeholder)*
