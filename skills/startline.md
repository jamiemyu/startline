---
name: startline
description: Full race readiness analysis — gap scoring, performance prediction, and HTML report
---

# /startline — Race Readiness Analysis

This skill runs a complete multi-step race readiness analysis. Follow every module in order.

---

## Module 1: Race Configuration

### Overview

This module collects the athlete's race calendar before any analysis begins. It runs exactly once per session. By the end, you will have a validated `raceConfig` object stored in working context for all downstream modules to reference.

---

### Step 1 — Collect primary race info (single combined prompt)

Ask the athlete this question in a single message:

> "What sport are you training for, what type of race is it, and when is the race date?"

Provide brief examples inline so the athlete understands the expected format:

> Examples: "Running, marathon, October 4" · "MTB, XCO, July 12" · "Road cycling, gran fondo, September 20"

Wait for the athlete's response before proceeding.

---

### Step 2 — Parse and validate the primary race response

Parse the athlete's freeform response into three structured fields: **sport**, **raceType**, and **date**.

#### 2a. Normalize Sport

Map the athlete's input to one of three canonical values:

| Athlete says | Normalized value |
|---|---|
| "running", "run", "trail running", "trail run" | `running` |
| "road cycling", "road bike", "cycling", "road" | `road_cycling` |
| "mtb", "mountain bike", "mountain biking", "mountain" | `mtb` |

If the sport cannot be mapped to one of these three values, respond:
> "I didn't recognize that sport. Supported sports are: **Running**, **Road Cycling**, and **MTB (Mountain Biking)**. Which are you training for?"

Then wait for a clarified response and re-parse.

#### 2b. Normalize Race Type

Once sport is known, validate the race type against the supported list for that sport:

| Sport | Valid race types (canonical) |
|---|---|
| `running` | `5k`, `10k`, `mile`, `half_marathon`, `marathon`, `ultra` |
| `road_cycling` | `criterium`, `gran_fondo`, `century` |
| `mtb` | `xco`, `enduro` |

Apply these aliases before validating:

| Athlete says | Canonical race type |
|---|---|
| "half", "half marathon", "HM", "half-marathon" | `half_marathon` |
| "full", "full marathon" | `marathon` |
| "gran fondo", "granfondo", "gf" | `gran_fondo` |
| "century ride", "century" | `gran_fondo` |
| "XCO", "cross country", "cross-country" | `xco` |

Match case-insensitively.

If the sport/race-type combination is not in the supported list, respond:
> "That sport/race-type combination isn't supported yet. Supported combinations are: Running (5K, 10K, Mile, Half Marathon, Marathon, Ultra), Road Cycling (Criterium, Gran Fondo/Century), MTB (XCO, Enduro). Which would you like to analyze?"

Wait for a clarified response and re-parse.

#### 2c. Parse Race Date

Parse the date from the athlete's response. Apply these rules:

**Explicit year provided** (e.g., "July 12, 2027", "2027-07-12", "12/7/2027"):
- Store the exact date as-is.

**Month and day only, no year** (e.g., "October 4", "Oct 4", "10/4"):
- Determine if that date has already passed in the current calendar year.
- If the date is still in the future this year, use the current year.
- If the date has already passed this year, ask for clarification:
  > "Did you mean [Month Day] this year ([current year]) or next year ([current year + 1])?"
- Wait for the athlete's response and store accordingly.

**Relative dates** (e.g., "in 12 weeks", "next October"):
- Convert to an approximate absolute date using today's date, then confirm with the athlete:
  > "I'll set that as [computed date] — does that look right?"

Store the final date in ISO 8601 format: `YYYY-MM-DD`.

#### 2d. Confirm the parsed primary race

Once all three fields are successfully parsed, confirm back to the athlete in a single message before proceeding. Example:

> "Got it — **Marathon** on **October 4, 2026**. I'll treat this as your A race."

If anything looks wrong, the athlete can correct it now.

---

### Step 3 — Collect additional B/C races

After the primary race is confirmed, ask:

> "Any other races on your calendar? You can add B races (important tune-ups) or C races (training races). These appear on your report timeline and C races feed into your finish time prediction. (Type 'none' or press Enter to skip.)"

**If the athlete provides a race:**

1. Parse the race using the same sport/race-type/date logic from Step 2.
2. Ask for the race name (optional):
   > "What's the name of this race? (Optional — press Enter to skip.)"
3. Ask for the designation:
   > "Is this a **B race** (important tune-up) or **C race** (training race)?"
4. Confirm the parsed entry:
   > "Added: [Race Name or race type] — [B/C race], [date]."
5. Then ask again:
   > "Any more races to add? (Type 'none' or press Enter when done.)"

**Termination conditions** — stop asking when the athlete:
- Types "none", "done", "no", "nope", "that's it", or any clear negative
- Provides an empty response (just presses Enter)
- Has added 10 or more races (safety cap — inform them if reached)

The A race from Step 1 never needs a name prompt or designation prompt — it is always the A race.

---

### Step 4 — Assemble and store raceConfig

Once all races are collected, assemble the `raceConfig` object and store it in working context. Structure:

```json
{
  "races": [
    {
      "designation": "A",
      "sport": "running",
      "raceType": "marathon",
      "date": "2026-10-04",
      "name": "Chicago Marathon"
    },
    {
      "designation": "B",
      "sport": "running",
      "raceType": "half_marathon",
      "date": "2026-08-15",
      "name": "Summer Half"
    },
    {
      "designation": "C",
      "sport": "running",
      "raceType": "10k",
      "date": "2026-07-04",
      "name": null
    }
  ],
  "primaryRace": {
    "designation": "A",
    "sport": "running",
    "raceType": "marathon",
    "date": "2026-10-04",
    "name": "Chicago Marathon"
  },
  "weeksToRace": 17,
  "daysToRace": 122
}
```

**Field rules:**

- `races` — array of all races in chronological order by date (ascending). The A race will appear wherever it falls chronologically.
- `primaryRace` — a copy of (or reference to) the A race object.
- `name` — use `null` if the athlete did not provide a name.
- `weeksToRace` — integer: `floor(daysToRace / 7)`.
- `daysToRace` — integer: number of calendar days from today's date to the A race date (inclusive of race day, exclusive of today). If the race is today, `daysToRace = 0`.

**Notify the athlete once the config is stored:**

> "Race calendar locked in. [N] races total — A race: [race type] on [date] ([X] weeks out). Moving on to training data..."

Then proceed immediately to Module 2.

---

### Error handling summary for Module 1

| Situation | Action |
|---|---|
| Unrecognized sport | Ask for clarification, list valid sports |
| Unsupported sport/race-type combo | Ask for clarification, list all valid combos |
| Ambiguous year in date | Ask: this year or next year? |
| Athlete provides no races in Step 3 | Accept immediately, set `races` to `[primaryRace]` |
| Athlete provides a race with an unsupported sport/type | Apply same validation as Step 2; ask for correction |

---

## Module 2: Garmin Data Ingestion
*(Implemented in issue #2 — placeholder)*
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
