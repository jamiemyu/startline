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

### Overview

This module collects race course data from the athlete. Three collection tiers are tried in priority order: GPX/FIT file upload, Strava route URL, and manual stat entry. By the end, a `courseData` object is stored in working context for all downstream modules.

---

### Step 1 — Prompt the athlete

Ask the athlete in a single message:

> "Upload your race course GPX/FIT file, or share a Strava route URL. (If neither is available, provide the key stats: distance in miles and elevation gain in feet.)"

Wait for the athlete's response, then branch to the appropriate tier below.

---

### Tier 1 — GPX/FIT File Upload (primary)

**Trigger:** Athlete uploads a file attachment, or provides a local file path ending in `.gpx` or `.fit`.

**Parse the file to extract all of the following fields:**

**`totalDistance`** — total route distance in miles.
Convert from meters: divide by 1609.344.

**`totalElevationGain`** — total elevation gain in feet.
Convert from meters: multiply by 3.28084. Sum only positive elevation deltas between consecutive trackpoints.

**`totalElevationLoss`** — total elevation loss in feet.
Convert from meters: multiply by 3.28084. Sum only negative elevation deltas (as a positive number).

**`gradeVariabilityIndex` (GVI)** — standard deviation of grade across 50-meter GPS windows. Compute as follows:
1. Walk the GPS coordinate array in 50-meter segments (measure horizontal distance between consecutive points using the Haversine formula or equivalent)
2. For each 50-meter segment compute grade = (elevation change in meters) / (horizontal distance in meters) × 100
3. Compute the standard deviation of all segment grades
4. Round to 2 decimal places

GVI is used for MTB terrain analysis only; compute it regardless of sport so it is available if needed.

**`elevationProfileArray`** — a sampled array of `{ distanceMiles, elevationFeet }` objects representing the elevation profile.
- Walk the trackpoints cumulatively, recording distance (miles) and elevation (feet) at each point
- Downsample to at most 200 points: if the raw point count exceeds 200, take every Nth point where N = floor(rawCount / 200), ensuring the final point is always included

**Store result with `tier: "gpx_fit"`** and proceed to Step 2.

---

### Tier 2 — Strava Route URL (fallback)

**Trigger:** Athlete provides a URL containing `strava.com/routes/`.

**Prerequisite check — confirm Strava MCP is connected:**
Before attempting any Strava fetch, try calling `mcp__strava-mcp__health` or `mcp__strava-mcp__get_athlete_profile`. If no `mcp__strava*` tools respond (tool not found, connection error, or authentication error), skip directly to Tier 3 and inform the athlete:

> "Strava MCP isn't connected — I'll need the course stats manually instead."

**If Strava MCP is available:**
1. Extract the numeric route ID from the URL (the segment after `/routes/`)
2. Use available `mcp__strava-mcp__*` tools to fetch the route's GPS and elevation data
3. From the returned route data, extract the same five fields as Tier 1:
   - `totalDistance` in miles (convert from meters: ÷ 1609.344)
   - `totalElevationGain` in feet (convert from meters: × 3.28084)
   - `totalElevationLoss` in feet (convert from meters: × 3.28084)
   - `gradeVariabilityIndex` — compute from GPS stream data using the same 50-meter window method as Tier 1; set to `null` if no GPS stream is available
   - `elevationProfileArray` — build from the altitude stream (downsample to ≤200 points); set to `null` if no altitude stream is available

**Store result with `tier: "strava_url"`** and proceed to Step 2.

---

### Tier 3 — Manual Stat Entry (last resort)

**Trigger:** Athlete types stats directly, or neither Tier 1 nor Tier 2 was available or successful.

Ask the athlete:

> "No problem — what's the total race distance (in miles) and total elevation gain (in feet)?"

**Parse the response** to extract `totalDistance` and `totalElevationGain`. Handle varied formats, including:
- "26.2 miles, 1200 feet"
- "26.2 / 1200"
- "26.2 and 1200"
- Two bare numbers in sequence (first = distance, second = elevation gain)

If the input is ambiguous or the units are unclear, ask one clarifying follow-up.

**Store result with `tier: "manual"` and these flags:**
- `totalElevationLoss: null`
- `gradeVariabilityIndex: null`
- `elevationProfileArray: null`
- `elevationProfileAvailable: false` — suppresses the elevation overlay chart in the HTML report
- `terrainEstimated: true` — marks the terrain dimension as using race-type defaults (no GVI available)

Proceed to Step 2.

---

### Step 2 — Assemble and store courseData

Store the following `courseData` object in working context:

```json
{
  "tier": "gpx_fit" | "strava_url" | "manual",
  "totalDistance": <miles, number>,
  "totalElevationGain": <feet, number>,
  "totalElevationLoss": <feet | null>,
  "gradeVariabilityIndex": <number | null>,
  "elevationProfileArray": [{ "distanceMiles": <n>, "elevationFeet": <n> }] | null,
  "elevationProfileAvailable": true | false,
  "terrainEstimated": true | false,
  "source": "<description of source used>"
}
```

**Field rules:**
- `elevationProfileAvailable`: set to `true` for Tier 1 and Tier 2 (when an elevation stream was available), `false` for Tier 3 or when no altitude data was returned
- `terrainEstimated`: set to `false` for Tier 1 and Tier 2, `true` for Tier 3
- `source`: a short human-readable string, e.g. `"GPX file upload"`, `"Strava route 123456789"`, or `"Manual entry"`

**Confirm to the athlete in one line.** Format the confirmation as:

> "Course loaded: [X] mi, [Y,YYY] ft gain ([source shorthand])."

Examples:
- "Course loaded: 26.2 mi, 1,840 ft gain (GPX file)."
- "Course loaded: 50.0 mi, 5,200 ft gain (Strava route)."
- "Course loaded: 13.1 mi, 600 ft gain (manual entry)."

Then proceed immediately to Module 4.

---

### Report notation

When generating the HTML report in Module 8, always note which tier was used for course data:

| Tier | Report label |
|---|---|
| `gpx_fit` | "Course data: GPX/FIT file" |
| `strava_url` | "Course data: Strava route" |
| `manual` | "Course data: Manual entry — elevation profile chart not available" |

When `elevationProfileAvailable` is `false`, suppress the elevation overlay chart entirely — do not render an empty chart.

When `terrainEstimated` is `true`, add a note to the terrain dimension in the gap analysis: "Terrain scoring based on race-type defaults (no course file provided)."

---

### Error handling summary for Module 3

| Situation | Action |
|---|---|
| File upload fails to parse | Inform athlete, ask them to try re-uploading or provide manual stats |
| Strava MCP unavailable | Skip to Tier 3, inform athlete |
| Strava route fetch fails (e.g., private route, 404) | Inform athlete, fall through to Tier 3 |
| Manual entry is ambiguous | Ask one clarifying follow-up before proceeding |
| Athlete provides distance only (no elevation) | Ask for elevation gain specifically before assembling courseData |

---

## Module 4: Training Block Detection

This module auto-detects the athlete's current training block and phase — no manual date entry required. Follow every step in order.

---

### Step 1 — Attempt TrainingPeaks detection (higher confidence)

Call `mcp__trainingpeaks__tp_get_fitness`. If the call errors with "not connected", the tool does not exist, or any connection-related error occurs, skip immediately to Step 2.

**If TrainingPeaks is available:**

Call both of the following:
- `mcp__trainingpeaks__tp_get_fitness` — to get CTL/ATL/TSB history
- `mcp__trainingpeaks__tp_get_atp` — to get training plan structure and phase labels

**Detect the training block start date:**
Scan the CTL history for the last 20 weeks. Find the earliest date where CTL was at a local minimum before a consistent sustained rise — this is the block start date. A "local minimum" is a point where CTL stopped declining and began rising for at least 2 consecutive weeks thereafter.

**Detect phase transitions** by examining the CTL/ATL/TSB curves in order. Apply these rules:

| Transition | Signal |
|---|---|
| Base → Build | CTL rising AND ATL spikes relative to CTL (intensity increasing) |
| Build → Peak | CTL plateauing (rate of rise near zero) AND high-intensity work concentrated |
| Peak → Taper | CTL dropping intentionally AND ATL falling faster than CTL |

Walk the history chronologically and record the date of each transition that is observed. Do not fabricate transitions not supported by the data.

Set `confidence: "high"` and `source: "trainingpeaks"`.

Skip Step 2 and proceed directly to Step 3.

---

### Step 2 — Garmin-only detection (medium confidence)

Use the `garminMetrics.volume.weeklyTrend` array already computed in Module 2 (it is in working context).

**Detect block start date:**
Walk the `weeklyTrend` array from oldest to newest across the trailing 16 weeks. Find the earliest week where weekly volume began a sustained upward trend. "Sustained" means at least 3 consecutive weeks of non-decreasing volume. Use the start date of that first week as the block start date.

If no clear upward trend exists (no 3-consecutive-week non-decreasing run is found), default to 12 weeks before today as the block start date.

**Detect current phase** using these heuristics. Apply in order and use the first match:

| Phase | Condition |
|---|---|
| **Taper** | Last 2 weeks show volume drop ≥ 25% from the prior 4-week average |
| **Peak** | Weekly volume is within 10% of the 4-week max AND race is ≤ 6 weeks away |
| **Build** | Last 4 weeks average volume > prior 4 weeks average AND race is > 6 weeks away |
| **Base** | Default if none of the above match |

Use `raceConfig.weeksToRace` (from Module 1 working context) for the "weeks away" comparisons.

**Detect phase transitions:**
Walk the `weeklyTrend` array week by week from oldest to newest. At each week, apply the same four-rule heuristic above (using the data available up to that week). Record the week where each phase transition occurs — i.e., where the detected phase changes from the previously detected phase. Store only transitions that are actually observed.

**Set confidence:**
- If `garminMetrics.weeksWithData` ≥ 8: set `confidence: "medium"`
- If `garminMetrics.weeksWithData` < 8: set `confidence: "low"`

Set `source: "garmin"`.

---

### Step 3 — Surface to athlete for confirmation

Format the block start date as a human-readable date (e.g., "Apr 14"). Format today's date the same way. Present a single confirmation message:

> "Detected training block: **[start date] – [today]** (currently in **[current phase] phase**). Does this look right?"

If confidence is `"medium"` or `"low"`, append to the message:

> "(estimated from Garmin load trends)"

Wait for the athlete's response:

**If confirmed** (athlete says "yes", "looks right", "correct", "yep", "sure", or any clear affirmative): proceed to Step 4 with no changes.

**If adjustment requested**: accept any correction the athlete provides. Examples:
- "The block started in March" → update `startDate` to the athlete's stated date (parse as `YYYY-MM-DD`)
- "I'm in peak phase" → update `currentPhase` to the athlete's stated phase
- A combination of both → apply both corrections

After applying the corrections, re-confirm in a single message:

> "Updated: block start **[new start date]**, currently in **[new phase] phase**. Proceeding."

Then proceed to Step 4.

---

### Step 4 — Store trainingBlock

Assemble the following `trainingBlock` object and store it in working context:

```json
{
  "startDate": "YYYY-MM-DD",
  "currentPhase": "Base" | "Build" | "Peak" | "Taper",
  "phaseTransitions": [
    { "phase": "Base", "startDate": "YYYY-MM-DD" },
    { "phase": "Build", "startDate": "YYYY-MM-DD" }
  ],
  "confidence": "high" | "medium" | "low",
  "source": "trainingpeaks" | "garmin"
}
```

**Field rules:**
- `startDate` — the confirmed block start date (ISO 8601 format)
- `currentPhase` — the confirmed current phase: exactly one of `Base`, `Build`, `Peak`, `Taper`
- `phaseTransitions` — chronological array of phase-start events actually detected in the data. Include only phases that were observed. Do not fabricate transitions.
- `confidence` — as set in Step 1 or Step 2 (athlete corrections do not change the confidence level)
- `source` — `"trainingpeaks"` if Step 1 succeeded, `"garmin"` if Step 2 was used

**Notify the athlete:**

> "Training block confirmed: **[currentPhase]** phase, started **[startDate formatted as Month D, YYYY]**. Moving to gap analysis..."

Then proceed immediately to Module 5.

---

## Module 4b: TrainingPeaks Enrichment

This module runs after Module 4 (Training Block Detection) and before Module 5 (Gap Analysis). It is purely additive — if TrainingPeaks is not connected, all downstream modules continue on Garmin data only.

---

### Step 1 — Check TrainingPeaks availability

Attempt `mcp__trainingpeaks__tp_auth_status`.

- If it errors with "not connected" or the tool doesn't exist: set `tpEnrichment = null`, display "ℹ️ TrainingPeaks not connected — running on Garmin data only." and skip to Module 5 immediately.
- If it returns connected: proceed to Step 2.

---

### Step 2 — Pull CTL/ATL/TSB history

Call `mcp__trainingpeaks__tp_get_fitness` to retrieve fitness metrics history.

Extract and store:
- `ctlHistory` — array of `{ date, ctl }` for the trailing 16 weeks, sorted oldest→newest
- `atlHistory` — array of `{ date, atl }` for the same window
- `tsbHistory` — array of `{ date, tsb }` for the same window
- `currentCTL` — the most recent CTL value
- `currentATL` — the most recent ATL value
- `currentTSB` — the most recent TSB value (form: positive = fresh, negative = fatigued)

---

### Step 3 — Pull structured training plan

Call `mcp__trainingpeaks__tp_get_workouts` for the last 4 weeks and next 2 weeks (date range: today minus 28 days to today plus 14 days).

*(4 weeks = one standard mesocycle — long enough to capture a complete training block phase, short enough that coach notes remain contextually relevant to current fitness. Going further back risks surfacing notes from a different training phase that no longer apply.)*

Extract:
- `coachNotes` — array of any workout descriptions or coach notes found in the workouts
- `trainingPhase` — if any workout has a phase label (Base/Build/Peak/Taper), extract the most recent one; otherwise null
- `plannedWorkouts` — count of planned workouts in next 2 weeks (for freshness context)

---

### Step 4 — Pull subjective data

Call `mcp__trainingpeaks__tp_get_metrics` for the last 4 weeks. *(4-week window matches the workout plan window above — one mesocycle of subjective data provides enough signal for form trends without diluting with stale entries from a prior training phase.)*

Extract if available:
- Recent RPE values (1–10 scale) — store as `recentRPEValues` array
- Feel tags (strong / normal / weak) — store as `recentFeelTags` array

If `tp_get_metrics` is unavailable or returns no subjective data, set both to empty arrays — never halt.

---

### Step 5 — Assemble enrichment object

Store as `tpEnrichment` in working context:

```json
{
  "available": true,
  "currentCTL": <number>,
  "currentATL": <number>,
  "currentTSB": <number>,
  "ctlHistory": [{ "date": "YYYY-MM-DD", "ctl": <number> }],
  "atlHistory": [{ "date": "YYYY-MM-DD", "atl": <number> }],
  "tsbHistory": [{ "date": "YYYY-MM-DD", "tsb": <number> }],
  "coachNotes": ["..."],
  "trainingPhase": "Base" | "Build" | "Peak" | "Taper" | null,
  "plannedWorkouts": <number>,
  "recentRPEValues": [<1-10>],
  "recentFeelTags": ["strong" | "normal" | "weak"]
}
```

Notify the athlete: "✅ TrainingPeaks connected — CTL: [currentCTL], TSB: [currentTSB] ([fresh/fatigued]). Enriching analysis..." then proceed to Module 5.

(TSB > 0 = "fresh", TSB < -10 = "fatigued", between = "neutral")

---

### How downstream modules use tpEnrichment

All downstream modules check `if (tpEnrichment?.available)` before using TP data, and fall back to Garmin-only logic otherwise.

- **Gap analysis (Module 5):** Intensity gap scoring uses coach workout prescriptions as context for whether high-Z4/Z5 training was intentional.
- **Performance prediction (Module 6):** `currentCTL` used as fitness ceiling proxy; confidence interval narrowed when CTL history is available.
- **Training block detection (Module 4):** Already handled — Module 4 checks TrainingPeaks first; this module provides the richer structured data pass.
- **Taper phase:** `currentTSB` used to assess freshness; a very negative TSB close to race day is flagged.

---

## Module 5: Gap Analysis

This module scores the gap between the athlete's training and race demands across five dimensions. Each dimension produces a score: 🟢 On Track / 🟡 Gap / 🔴 Risk.

Work through each sub-section in order. Store all results in a `gapScores` object.

---

### 5a. Distance Gap

#### Inputs (already in working context)
- `garminMetrics.volume.longestEffort.distance` — longest single training effort in miles
- `garminMetrics.volume.longestEffort.date` — date of that longest effort (ISO YYYY-MM-DD)
- `garminMetrics.volume.top3Average` — average of 3 longest efforts in miles
- `courseData.totalDistance` — race distance in miles
- `raceConfig.weeksToRace` — integer weeks until race day
- `raceConfig.primaryRace.date` — race date (ISO YYYY-MM-DD)
- `raceConfig.primaryRace.sport` and `.raceType`

#### Step 1 — Look up race-type-specific thresholds

Distance gap thresholds vary significantly by race type — a marathon runner is expected to race 1.3× their longest run, but a 5K runner trains well beyond race distance. Use the following per-race-type table (derived from Pfitzinger, Daniels, Hal Higdon, CTS, and TrainingPeaks coaching literature):

| Sport | Race type | `distanceRatio` On Track | `distanceRatio` Gap | `distanceRatio` Risk | Notes |
|---|---|---|---|---|---|
| `running` | `mile` | Any ratio | — | — | Skip distance gap: athletes always train longer than 1 mi. Set `score: null, reason: "Distance not a limiting factor for mile races"` |
| `running` | `5k` | Any ratio | — | — | Skip distance gap: athletes train 3–5× race distance. Same null treatment. |
| `running` | `10k` | ≤ 2.5 | 2.5–3.5 | > 3.5 | Longest training run typically 12–16 mi (2–2.5× race dist) |
| `running` | `half_marathon` | ≤ 1.35 | 1.35–1.65 | > 1.65 | Peak long run 10–13 mi; racing up to ~1.3× longest run is normal (PMC7496388) |
| `running` | `marathon` | ≤ 1.35 | 1.35–1.55 | > 1.55 | Peak long run 20–22 mi (~75–85% of race dist); Pfitzinger, Hal Higdon consensus |
| `running` | `ultra` | ≤ 1.6 | 1.6–2.5 | > 2.5 | 50K: peak ~20–24 mi; longer ultras use back-to-back days; CTS/iRunFar norms |
| `road_cycling` | `criterium` | Any ratio | — | — | Skip: crit is intensity-limited (~45–90 min), not distance-limited. Null treatment. |
| `road_cycling` | `gran_fondo` | ≤ 1.4 | 1.4–1.75 | > 1.75 | Peak training ride 70–85 mi for 100-mi event; TrainingPeaks/EVOQ.BIKE consensus |
| `road_cycling` | `century` | ≤ 1.4 | 1.4–1.75 | > 1.75 | Same as gran fondo |
| `mtb` | `xco` | Any ratio | — | — | Skip: XCO (~1.5–2.5 hr race) is shorter than typical training rides. Null treatment. |
| `mtb` | `enduro` | ≤ 1.5 | 1.5–2.0 | > 2.0 | Technical terrain adds fatigue; peak training day ~4–5 hr; CTS/TrainingPeaks norms |

If the race type is one of the "skip" types (mile, 5k, criterium, xco): store `gapScores.distance = { score: null, skipped: true, reason: "<reason from table>" }` and display a one-line note to the athlete. Proceed to 5b.

Otherwise, compute:
`distanceRatio = courseData.totalDistance / garminMetrics.volume.longestEffort.distance`

#### Step 2 — Apply adaptive tightening

The closer to race day, the more urgently the same gap reads. Apply this tightening factor to the On Track / Gap threshold boundary:

| weeksToRace | Tightening factor |
|---|---|
| > 12 weeks | 1.0 (no tightening) |
| 8–12 weeks | 0.92 (thresholds shift 8% stricter) |
| 4–8 weeks | 0.85 |
| < 4 weeks | 0.75 |

Multiply the On Track ceiling and Gap ceiling by the tightening factor before comparing `distanceRatio`. A 🟡 Gap at 12 weeks may become 🔴 Risk at 3 weeks.

#### Step 3 — Check taper recency (longest effort too close to race day)

A long effort done too close to race day means the athlete may arrive at the start line fatigued — taper is essential for performance. Compute `daysFromLongestToRace = days between longestEffort.date and raceConfig.primaryRace.date`.

Use these race-type-specific risk windows (based on Pfitzinger, CTS, and TrainingPeaks taper literature):

| Race type | Ideal last long effort | Risk zone (too close) |
|---|---|---|
| `half_marathon` | ≥ 14 days before race | < 10 days |
| `marathon` | ≥ 21 days before race | < 14 days |
| `ultra` | ≥ 28 days before race | < 21 days |
| `gran_fondo` / `century` | ≥ 14 days before race | < 10 days |
| `enduro` | ≥ 14 days before race | < 10 days |

If `daysFromLongestToRace` is within the risk zone, set `taperRisk: true` and add this note to the report:
> ⚠️ **Taper flag:** Your longest training effort was [N] days before race day — less than the recommended [X]-day minimum. Racing on unrecovered legs can significantly impact performance. Prioritize rest.

If within the ideal window (≥ minimum but earlier than risk zone), set `taperRisk: false`.
If the longest effort was done very early (e.g., > 8 weeks ago), note this may indicate detraining rather than taper and flag it separately as `detrain: true` if `daysFromLongestToRace > 56`.

#### Step 4 — Store result

```json
{
  "score": "on_track" | "gap" | "risk" | null,
  "skipped": false,
  "longestEffortMiles": <number>,
  "longestEffortDate": "YYYY-MM-DD",
  "top3AverageMiles": <number>,
  "raceDistanceMiles": <number>,
  "distanceRatio": <number | null>,
  "weeksToRace": <number>,
  "tighteningFactor": <number>,
  "taperRisk": true | false,
  "detrain": true | false,
  "reason": null
}
```

Store as `gapScores.distance`.

Display one line for the distance score + a separate taper flag line if applicable:
- e.g. "📏 Distance: 🟡 Gap — longest run 18 mi, race is 26.2 mi (1.46× your longest effort)."
- e.g. "⚠️ Taper flag: longest run was 10 days before race day (minimum recommended: 14 days)."

---

### 5b. Elevation Gap

#### Inputs (already in working context)
- `garminMetrics.elevation.avgPerLongEffort` — average elevation gain per long effort in feet
- `garminMetrics.elevation.cumulativeBlockGain` — total elevation gain over block in feet
- `courseData.totalElevationGain` — race elevation gain in feet
- `raceConfig.weeksToRace`

#### Step 1 — Compute the gap ratio

`elevationRatio = courseData.totalElevationGain / garminMetrics.elevation.avgPerLongEffort`

If `avgPerLongEffort` is 0 or null:
- Use `cumulativeBlockGain` as a proxy divided by number of long efforts (estimate as `activitiesAnalyzed / 4`)
- If still null/0, set `score: null` with `reason: "No elevation data in training activities"` and skip scoring

#### Step 2 — Score the gap

Base thresholds:

| elevationRatio | Base score |
|---|---|
| ≤ 1.5 | 🟢 On Track |
| 1.5 – 2.5 | 🟡 Gap |
| > 2.5 | 🔴 Risk |

Apply the same adaptive tightening factor from 5a (use `raceConfig.weeksToRace` → same table).

#### Step 3 — Store result

```json
{
  "score": "on_track" | "gap" | "risk" | null,
  "avgElevationPerLongEffortFeet": <number | null>,
  "raceElevationGainFeet": <number>,
  "elevationRatio": <number | null>,
  "weeksToRace": <number>,
  "tighteningFactor": <number>,
  "reason": null
}
```

Store as `gapScores.elevation`.

Display one line: e.g. "⛰️ Elevation: 🔴 Risk — avg 800 ft/long run, race demands 4,200 ft (5.25× your training average)."

---

### 5c. Terrain Gap
*(Implemented in issue #7 — placeholder)*

---

### 5d. Intensity Gap

#### Inputs
- `garminMetrics` — full activity list from Module 2
- `raceConfig.primaryRace.sport`, `.raceType`
- `trainingBlock.currentPhase` — from Module 4
- `tpEnrichment` — from Module 4b (may be null)

---

#### Step 1 — Compute HR zone distribution

Take the top 10 most recent activities from `garminMetrics.filteredActivities` (sorted by date descending).

For each activity, call `mcp__garmin__get_activity_hr_in_timezones` with the activity ID. Extract the minutes spent in each zone (Z1–Z5).

Sum across all activities:
- `totalMinutesZ1`, `totalMinutesZ2`, `totalMinutesZ3`, `totalMinutesZ4`, `totalMinutesZ5`
- `totalMinutes` = sum of all zone minutes
- Compute percentages: `pctZ1` = `totalMinutesZ1 / totalMinutes × 100`, etc.

Store as `trainingDistribution = { pctZ1, pctZ2, pctZ3, pctZ4, pctZ5 }`.

**If HR zone data is unavailable** (tool errors for all activities or returns no zone data): attempt to estimate from pace/power data if available, or set `trainingDistribution = null` and skip intensity scoring:
```json
{ "score": null, "reason": "No HR zone data available in recent activities" }
```
Store as `gapScores.intensity` and skip to the next sub-section.

---

#### Step 2 — Look up expected race intensity profile

Match `raceConfig.primaryRace.sport` × `raceConfig.primaryRace.raceType` to the expected intensity profile:

| Sport | Race type | Expected profile (dominant zones) | Key descriptor |
|---|---|---|---|
| `running` | `mile` | Z5: >40%, Z4: >30% | VO2max dominant |
| `running` | `5k` | Z4–Z5: >60% combined | VO2max dominant |
| `running` | `10k` | Z4: >40%, Z3: >20% | Threshold + VO2max |
| `running` | `half_marathon` | Z3–Z4: >60% combined | Threshold dominant |
| `running` | `marathon` | Z2–Z3: >65% combined | Aerobic + marathon pace |
| `running` | `ultra` | Z1–Z2: >70% combined | Aerobic base, time-on-feet |
| `road_cycling` | `criterium` | Z5: >20%, Z4: >25% | Sprint + anaerobic capacity |
| `road_cycling` | `gran_fondo` | Z2–Z3: >60%, Z4: >15% | Sustained endurance + FTP |
| `road_cycling` | `century` | Z2–Z3: >65% combined | Sustained aerobic |
| `mtb` | `xco` | Z4–Z5: >50% combined | Punchy VO2max (5–30 sec efforts) |
| `mtb` | `enduro` | Z3–Z4: >40%, Z5 bursts: >10% | Threshold + punchy climbs |

Derive a per-zone numeric target from each profile descriptor (use the midpoint of any range as the target). For zones not explicitly listed, distribute the remaining percentage proportionally across them.

Store as `expectedProfile = { description: "<key descriptor>", dominantZones: "<zones>", zoneTargets: { pctZ1, pctZ2, pctZ3, pctZ4, pctZ5 } }`.

---

#### Step 3 — Score the intensity gap

**Compute match score:**

For each zone Z1–Z5, compute `|actual% − expected%|`. Sum all five absolute differences → `totalDeviation`.

| `totalDeviation` | Score |
|---|---|
| ≤ 20 percentage points | `on_track` → 🟢 On Track |
| 20–40 percentage points | `gap` → 🟡 Gap |
| > 40 percentage points | `risk` → 🔴 Risk |

**Phase-intensity alignment check:**

Inspect `trainingBlock.currentPhase` and compare against the race-type intensity demands:

- If phase is `Base` and `(pctZ4 + pctZ5) > 30` for a `marathon` runner → set `phaseAlignmentNote`: "Higher than typical intensity for base phase marathon training."
- If phase is `Peak` and `(pctZ4 + pctZ5) < 20` for a `5k` or `mile` runner → set `phaseAlignmentNote`: "Intensity may be too low for peak phase 5K/mile preparation."
- If phase is `Taper` → set `phaseAlignmentNote = null` (no intensity flag; taper naturally reduces all zone work).
- Otherwise → set `phaseAlignmentNote = null`.

**TP enrichment cross-reference:**

If `tpEnrichment?.available` is true, scan coach workout prescriptions for the recent training block. If coach notes indicate high-intensity sessions were planned and the athlete completed them, append to `phaseAlignmentNote` (or set it if null): "High Z4–Z5 training appears intentional per coach prescription."

---

#### Step 4 — Store result

Store as `gapScores.intensity`:

```json
{
  "score": "on_track" | "gap" | "risk" | null,
  "trainingDistribution": { "pctZ1": <n>, "pctZ2": <n>, "pctZ3": <n>, "pctZ4": <n>, "pctZ5": <n> } | null,
  "expectedProfile": { "description": "<key descriptor>", "dominantZones": "<zones>" },
  "totalDeviation": <number | null>,
  "phaseAlignmentNote": "<string>" | null,
  "reason": null
}
```

Display one line summarizing the result. Examples:
- "⚡ Intensity: 🟢 On Track — training distribution aligns well with half marathon threshold demands."
- "⚡ Intensity: 🟡 Gap — training is 45% Z2 aerobic; marathon pace requires more Z3 threshold work."
- "⚡ Intensity: 🔴 Risk — 5K preparation requires Z4–Z5 emphasis; current block is heavily aerobic (Z1–Z2: 72%)."
- "⚡ Intensity: *No HR zone data available in recent activities.*"

---

### 5e. Temperature Gap

#### Step 1 — Check race proximity

Compute `daysToRace` from `raceConfig.daysToRace` (already in working context).

**If `daysToRace > 10`:**
- Skip the temperature dimension entirely
- Store `gapScores.temperature = { score: null, skipped: true, reason: "Weather analysis available closer to race day (race is more than 10 days out)" }`
- Display to athlete: "🌡️ Temperature: *Weather analysis available closer to race day.*"
- Proceed to the next gap dimension.

**If `daysToRace <= 10`:** continue to Step 2.

#### Step 2 — Get race location

Check `raceConfig.primaryRace` for a location field. If none exists, ask:
> "What city or region is the race in? (Used for race day weather forecast.)"

Store the response as `raceLocation`.

#### Step 3 — Fetch race day forecast

Call the Open-Meteo API via a Bash command or WebFetch.

First, geocode the city using the Open-Meteo geocoding API:
```
https://geocoding-api.open-meteo.com/v1/search?name={CITY}&count=1&language=en&format=json
```

Extract `latitude` and `longitude` from the first result, then fetch the forecast:
```
https://api.open-meteo.com/v1/forecast?latitude={LAT}&longitude={LON}&daily=temperature_2m_max,temperature_2m_min&temperature_unit=fahrenheit&forecast_days=10&timezone=auto
```

Find the forecast entry matching the race date. Extract `temperature_2m_max` and `temperature_2m_min` for that day. Compute `raceDayTempF = (max + min) / 2`.

**If the API call fails or returns no data:**
- Store `gapScores.temperature = { score: null, skipped: true, reason: "Weather data unavailable" }`
- Display: "🌡️ Temperature: *Weather data unavailable — temperature dimension skipped.*"
- Proceed to next dimension.

#### Step 4 — Get training temperature

From `garminMetrics` (already in working context), the activities array contains weather data on many activities. Look for an `averageTemperature` or `temperature` field on each activity.

Average the temperature values across activities in the last 4 weeks (use only activities where temperature data is present). Convert to Fahrenheit if values appear to be in Celsius (values < 50 when racing in a warm climate are likely Celsius — multiply by 9/5 + 32).

Store as `trainingAvgTempF`.

**If no temperature data is available in Garmin activities:** set `trainingAvgTempF = null` and skip scoring (store `score: null, reason: "No training temperature data in Garmin activities"`).

#### Step 5 — Score the temperature gap

Compute `tempDeltaF = raceDayTempF - trainingAvgTempF` (positive = racing hotter than training).

Apply these thresholds:

| Condition | Score |
|---|---|
| `abs(tempDeltaF) <= 10°F` | 🟢 On Track |
| `tempDeltaF` between 10–20°F warmer OR 10–20°F cooler | 🟡 Gap |
| `tempDeltaF > 20°F` warmer OR `< -20°F` cooler | 🔴 Risk |

Heat exposure (racing significantly hotter than training) is more penalized than cold exposure — racing in the cold is generally less impactful on performance.

#### Step 6 — Store result

```json
{
  "score": "on_track" | "gap" | "risk" | null,
  "skipped": false,
  "trainingAvgTempF": "<number | null>",
  "raceDayTempF": "<number | null>",
  "tempDeltaF": "<number | null>",
  "raceLocation": "<city>",
  "reason": null
}
```

Display one line to athlete: e.g. "🌡️ Temperature: 🟡 Gap — training avg 52°F, race day forecast 74°F (+22°F)."

---

Once all gap dimensions are scored, assemble the final `gapScores` object and proceed to Module 6: Performance Prediction.

---

## Module 6: Performance Prediction
*(Implemented in issue #11 — placeholder)*

---

## Module 7: Readiness % Aggregation
*(Implemented in issue #12 — placeholder)*

---

## Module 8: HTML Report Generation
*(Implemented in issues #13–#16 — placeholder)*
