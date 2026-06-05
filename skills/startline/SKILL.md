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

**Inputs:**
- `raceConfig.primaryRace.sport` and `.raceType`
- `courseData.tier`, `courseData.gradeVariabilityIndex` (null if Tier 3 / manual)
- `courseData.terrainEstimated` — true if Tier 3 manual entry
- `garminMetrics` — full object from Module 2 (activity list with type labels)
- Strava MCP tools (optional — check availability)

---

**Road Cycling — skip terrain dimension**

If `sport === "road_cycling"`: terrain is handled entirely by the elevation dimension.

Store:
```json
{ "score": null, "skipped": true, "reason": "Terrain not applicable for road cycling — see elevation gap." }
```
Display: "🏔️ Terrain: *N/A — road cycling terrain is captured in the elevation dimension.*"
Proceed to 5d.

---

**MTB — Grade Variability Index method**

If `courseData.gradeVariabilityIndex` is available (Tier 1 or 2 course data):

1. Pull FIT data for the athlete's MTB activities from the last 16 weeks. For each activity where `activityType` is `mtb` / `mountain_biking` / `MTB` / `Mountain Bike Ride`, call `mcp__garmin__get_activity_fit_data` with `activityId` and note the GVI if already computed, OR compute GVI from the raw records:
   - Walk GPS coordinate records in 50-meter segments
   - For each segment: grade = elevation_change_meters / horizontal_distance_meters × 100
   - Standard deviation of all segment grades = GVI for that activity
2. Average the GVI across all MTB activities analyzed → `trainingGVI`
3. Compare to `courseData.gradeVariabilityIndex` → `raceGVI`

Score by `raceGVI / trainingGVI` ratio:
- ≤ 1.3 → 🟢 On Track
- 1.3–2.0 → 🟡 Gap
- > 2.0 → 🔴 Risk

If `courseData.gradeVariabilityIndex` is null (Tier 3 manual entry or no GPS data), fall back to race-type defaults:
- XCO: `raceGVI` default = 8.0 (high technical)
- Enduro: `raceGVI` default = 12.0 (very high technical)

If `trainingGVI` is also unavailable (no FIT data returned), use activity type labels as a proxy: if ≥ 50% of training rides are labeled MTB/Trail → assume `trainingGVI` = 6.0 (moderate technical); otherwise → 3.0 (low technical).

Apply same ratio thresholds. Set `method: "gvi_default"` in result to flag that defaults were used.

---

**Running — label + Strava surface method**

1. From `garminMetrics` activity list, count runs labeled `Trail Run` vs `Run` (or equivalent labels).
   - `trailRunPct = trailRunCount / totalRunCount`

2. If Strava MCP is available, attempt `mcp__strava-mcp__list_activities` for recent runs and look for surface tags (paved / unpaved / gravel / trail). Average surface type → `stravaSurfaceType`.

3. Determine race terrain type from `courseData`:
   - If course was GPX/Strava Tier 1/2: infer from GVI — GVI < 3 → road, GVI 3–6 → mixed, GVI > 6 → trail
   - If Tier 3 manual: use `raceType` as proxy — `marathon`/`10k`/`half_marathon` → road; `ultra` → trail (ask athlete to confirm if ambiguous)

4. Score terrain match:

| Race terrain | Athlete trail run % | Score |
|---|---|---|
| Road | Any | 🟢 On Track (road runners always fine on road) |
| Mixed / Trail | ≥ 60% trail runs | 🟢 On Track |
| Mixed / Trail | 30–60% trail runs | 🟡 Gap |
| Mixed / Trail | < 30% trail runs | 🔴 Risk |

Set `method: "label_based"` in result.

---

**Result object**

Store as `gapScores.terrain`:
```json
{
  "score": "on_track" | "gap" | "risk" | null,
  "skipped": false,
  "sport": "<sport>",
  "method": "gvi_measured" | "gvi_default" | "label_based" | null,
  "trainingGVI": <number | null>,
  "raceGVI": <number | null>,
  "trailRunPct": <0-1 | null>,
  "raceTerrainType": "road" | "mixed" | "trail" | null,
  "reason": null
}
```

Display one line: e.g. "🌲 Terrain: 🟡 Gap — 25% of your runs are trail/unpaved, race is trail terrain."

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

**Output:** a finish time range (e.g., "3:48–4:06") plus a confidence score (e.g., 61%).

---

### Step 1 — Collect prediction inputs (layered by weight)

Work through each input tier in order. Collect all available inputs — you will weight them in Step 3.

**Tier 1 — Recent full-effort race results (highest weight)**
From `garminMetrics.filteredActivities`, find activities flagged as races (look for `isRace: true`, activity name containing "race" / "marathon" / "5K" etc., or activity type = "RACE"). For each:
- Extract `distance`, `duration`, `date`
- Compute pace: `minutesPerMile = duration_minutes / distance_miles`
- Only include results from the last 18 months
- Label as `tier: "race_result"`

**Tier 2 — C race results (included at 60% weight)**
From `raceConfig.races`, find C-designation races whose dates are in the past. Fetch their Garmin activities (match by date ± 1 day). Extract same fields. Label as `tier: "c_race"`.

**Tier 3 — Key workout performance**
From `garminMetrics.filteredActivities`, find quality sessions:
- Tempo runs/rides: activities where >20% of time was in Z3-Z4
- Interval sessions: activities with high Z4-Z5 time AND short duration (<90 min)
- Extract best sustained pace/power over 20–60 min efforts
- Label as `tier: "key_workout"`

**Tier 4 — HR-based aerobic efficiency**
From recent activities, find efforts at a known HR. Compute pace or power at the athlete's aerobic threshold HR (approximately 75–80% of max HR; estimate max HR as 220 - age if not available from Garmin, or use the highest HR seen in Garmin data × 1.05).
Compute `aerobicEfficiency = pace_or_power_at_threshold_HR`.
Label as `tier: "aerobic_efficiency"`.

**Tier 5 — TrainingPeaks CTL (fitness ceiling proxy)**
If `tpEnrichment?.available`: use `tpEnrichment.currentCTL` as a fitness ceiling.
- For running: CTL > 80 → capable of strong marathon; CTL > 50 → capable of half marathon
- For cycling: CTL > 100 → strong gran fondo capability
Label as `tier: "ctl_ceiling"`.

---

### Step 2 — Apply Grade Adjusted Pace (GAP) elevation correction

If `courseData.elevationProfileArray` is available (Tier 1 or 2 course data):

1. Compute the athlete's flat-equivalent pace from training data (use aerobic efficiency or key workout paces on flat/rolling terrain — activities with < 500 ft elevation gain)
2. Apply GAP correction for the race course elevation profile:
   - Use standard GAP formula: for every 1% grade increase, add ~10 sec/mile to pace (Strava's published GAP approximation)
   - Walk `courseData.elevationProfileArray` segment by segment; compute grade per segment; apply pace adjustment
   - Sum adjusted time across all segments → `gapAdjustedPrediction`

If no elevation profile (Tier 3 manual entry): use total elevation gain heuristic — for every 1,000 ft of gain, add ~8–12 min for running, ~15–20 min for cycling.

---

### Step 3 — Compute weighted prediction

Assign weights to available inputs:

| Tier | Weight |
|---|---|
| Race result (< 6 months old) | 1.0 |
| Race result (6–18 months old) | 0.7 |
| C race result | 0.6 |
| Key workout performance | 0.5 |
| Aerobic efficiency | 0.4 |
| CTL ceiling (TP) | 0.3 |

For each input, derive an implied finish time for the target race distance using standard equivalency tables (e.g., for running: Daniels' VDOT equivalency — a 20-min 5K implies ~4:20 marathon pace; for cycling: FTP-based power-to-time estimates).

Compute the weighted average of all implied finish times → `predictedFinishTime`.

---

### Step 4 — Compute confidence score and widen range

Start with base confidence of 80%. Apply modifiers to widen the range (reduce confidence):

| Modifier | Confidence reduction |
|---|---|
| Sparse Garmin history (< 4 weeks) | −25% |
| No race results available (Tiers 1+2 both empty) | −20% |
| Only C race results (no full-effort A/B race results) | −10% |
| Training inconsistency (from `garminMetrics` CV > benchmark) | −10% |
| Large elevation mismatch (race elevation > 2× training avg) | −10% |
| No GPX/course file (Tier 3 manual entry) | −10% |
| CTL available from TrainingPeaks | +5% (increases confidence) |
| Multiple recent race results (≥ 2 within 12 months) | +5% |

Clamp final confidence between 20% and 90%.

**Range width**: the range is `predictedFinishTime ± margin`. Margin = `predictedFinishTime × (1 - confidence/100) × 0.15`. So a 60% confidence prediction has a wider range than an 85% confidence one.

Round the range to nearest minute. E.g., if predicted = 3:57 and margin = ±9 min → range is "3:48–4:06".

---

### Step 5 — Store result

Store the following as `prediction` in working context:

```json
{
  "predictedFinishTime": "H:MM:SS",
  "rangeLow": "H:MM:SS",
  "rangeHigh": "H:MM:SS",
  "confidencePct": "<20-90>",
  "inputsUsed": ["race_result", "key_workout", "..."],
  "modifiersApplied": ["No race results: -20%", "..."],
  "gapAdjusted": true
}
```

Notify the athlete: "🏁 Prediction: **[rangeLow]–[rangeHigh]** ([confidencePct]% confidence). Proceeding to readiness score..."

Then proceed to Module 7.

---

## Module 7: Readiness % Aggregation

### Step 1 — Collect gap scores

Retrieve from working context:
- `gapScores.distance` — score + skipped flag
- `gapScores.elevation` — score + skipped flag
- `gapScores.terrain` — score + skipped flag
- `gapScores.intensity` — score + skipped flag
- `gapScores.temperature` — score + skipped flag
- `prediction.confidencePct` — from Module 6

For each dimension, convert the score to a numeric value:
- `"on_track"` → 100
- `"gap"` → 60
- `"risk"` → 20
- `null` / `skipped: true` → excluded from aggregation (weight redistributed)

---

### Step 2 — Look up default dimension weights

Weights are sport × race-type specific. Use this table:

| Sport | Race type | Distance | Elevation | Terrain | Intensity | Temperature |
|---|---|---|---|---|---|---|
| `running` | `mile` | 0.05 | 0.05 | 0.10 | 0.60 | 0.20 |
| `running` | `5k` | 0.05 | 0.05 | 0.10 | 0.60 | 0.20 |
| `running` | `10k` | 0.15 | 0.10 | 0.10 | 0.50 | 0.15 |
| `running` | `half_marathon` | 0.25 | 0.15 | 0.10 | 0.35 | 0.15 |
| `running` | `marathon` | 0.35 | 0.15 | 0.05 | 0.35 | 0.10 |
| `running` | `ultra` | 0.30 | 0.25 | 0.20 | 0.15 | 0.10 |
| `road_cycling` | `criterium` | 0.05 | 0.05 | 0.00 | 0.70 | 0.20 |
| `road_cycling` | `gran_fondo` | 0.25 | 0.30 | 0.00 | 0.35 | 0.10 |
| `road_cycling` | `century` | 0.30 | 0.25 | 0.00 | 0.35 | 0.10 |
| `mtb` | `xco` | 0.10 | 0.15 | 0.25 | 0.40 | 0.10 |
| `mtb` | `enduro` | 0.15 | 0.20 | 0.30 | 0.25 | 0.10 |

Note: road cycling terrain weight is always 0.00 since that dimension is skipped for road cycling.

---

### Step 3 — Redistribute weights for missing/skipped dimensions

For any dimension where `score === null` or `skipped === true`, set its weight to 0 and redistribute that weight proportionally among the remaining scored dimensions.

Algorithm:
1. Start with the default weight table for this sport × race type
2. For each skipped/null dimension, remove its weight from the pool
3. Sum the remaining weights
4. Divide each remaining weight by the sum to renormalize (so all weights sum to 1.0)

Track which dimensions were included/excluded in `includedDimensions` and `excludedDimensions` arrays.

---

### Step 4 — Compute Readiness %

`readinessPct = sum(dimensionScore × normalizedWeight)` for all included dimensions.

Round to the nearest integer.

---

### Step 5 — Training consistency modifier

Compute the coefficient of variation (CV) of weekly training volume:
- Use `garminMetrics.volume.weeklyTrend` array from working context
- CV = (standard deviation of weekly distances) / (mean of weekly distances) × 100

Compare to sport × race-type benchmarks:

| Sport | Race type | Ideal CV threshold |
|---|---|---|
| `running` | `marathon` | < 20% |
| `running` | `5k` / `10k` / `half_marathon` | < 25% |
| `running` | `ultra` | < 30% |
| `road_cycling` | `gran_fondo` / `century` | < 25% |
| `road_cycling` | `criterium` | < 30% |
| `mtb` | `xco` / `enduro` | < 30% |

**High inconsistency behavior:**
- CV within threshold: no adjustment
- CV 1–1.5× threshold: add note "Training volume has been somewhat inconsistent — this widens the prediction range slightly." and reduce `prediction.confidencePct` by 5 percentage points.
- CV > 1.5× threshold: add note "Training volume has been highly inconsistent — prediction confidence interval is wider than usual." and reduce `prediction.confidencePct` by 10 percentage points.

**The Readiness % itself is NOT changed** by consistency. Consistency only widens the prediction confidence interval. After applying any reduction, re-clamp `prediction.confidencePct` to the range 20–90%.

---

### Step 6 — Store result

Store the following as `readiness` in working context:

```json
{
  "readinessPct": <0-100>,
  "dimensionWeights": {
    "distance": <0-1>,
    "elevation": <0-1>,
    "terrain": <0-1>,
    "intensity": <0-1>,
    "temperature": <0-1>
  },
  "includedDimensions": ["distance", "elevation", "intensity"],
  "excludedDimensions": ["terrain", "temperature"],
  "consistencyCV": <number>,
  "consistencyBenchmark": <number>,
  "consistencyNote": "<string>" | null
}
```

Also update `prediction.confidencePct` if the consistency modifier applies (re-clamp to 20–90%).

Notify the athlete:

> "📊 Readiness: **[readinessPct]%** (based on [N] of 5 dimensions — [excluded list] not available). [consistencyNote if any]"

Then proceed to Module 8.

---

## Module 8: HTML Report Generation

### Step 1 — Prepare report data

Collect all the values from working context that the report needs:
- From `raceConfig.primaryRace`: `sport`, `raceType`, `date`, `name`
- `raceConfig.weeksToRace`, `raceConfig.daysToRace`
- `readiness.readinessPct`
- `prediction.rangeLow`, `prediction.rangeHigh`, `prediction.confidencePct`
- `gapScores` object (all five dimensions)
- `readiness.includedDimensions`, `readiness.excludedDimensions`
- All races from `raceConfig.races` (for the race calendar strip)

Compute `reportFilename`:
- Format: `[sport]-[race-name-slugified]-[YYYY-MM-DD].html`
- Slugify race name: lowercase, replace spaces with hyphens, strip special characters
- Example: `running-chicago-marathon-2026-10-04.html`

### Step 2 — Create the reports directory

Run via Bash:
```bash
mkdir -p ~/startline-reports
```

### Step 3 — Generate the HTML file

Write a complete, self-contained HTML file to `~/startline-reports/[reportFilename]`.

The file must include all five sections below. Use Chart.js via CDN (`https://cdn.jsdelivr.net/npm/chart.js`). Load Google Fonts via CDN link tag.

**Design system (must match exactly):**
- Background: `#f0ebe4`
- Card background: `#faf7f2`
- Card border: `rgba(0,0,0,0.07)`
- Primary text: `#1c1814`
- Secondary text: `#a09080`
- Divider: `#e4ddd6`
- On Track: `#6aaa6e` / bg `#e2eedc`
- Gap: `#c09060` / bg `#f0e8da`
- Risk: `#c07058` / bg `#f0e0da`
- Fonts: Lora (serif, race title + metric values) + DM Sans (body/labels) via Google Fonts CDN

Use this HTML template structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>[Race Name] — Startline Report</title>
  <link href="https://fonts.googleapis.com/css2?family=Lora:wght@400;700&family=DM+Sans:wght@400;500;700&display=swap" rel="stylesheet">
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: 'DM Sans', sans-serif;
      background: #f0ebe4;
      color: #1c1814;
      padding: 32px 16px;
      max-width: 860px;
      margin: 0 auto;
    }
    .card {
      background: #faf7f2;
      border: 1px solid rgba(0,0,0,0.07);
      border-radius: 16px;
      padding: 28px 32px;
      margin-bottom: 20px;
    }
    .divider { border: none; border-top: 1px solid #e4ddd6; margin: 20px 0; }
    .secondary { color: #a09080; }
    .lora { font-family: 'Lora', serif; }

    /* Section 1 — Header */
    .header-race-name { font-family: 'Lora', serif; font-size: 2.2rem; font-weight: 700; line-height: 1.2; }
    .header-subtitle { color: #a09080; font-size: 1rem; margin-top: 6px; text-transform: capitalize; }
    .header-date { font-size: 0.95rem; color: #a09080; margin-top: 8px; }

    /* Section 2 — Race Calendar Strip */
    .calendar-title { font-weight: 700; font-size: 0.85rem; text-transform: uppercase; letter-spacing: 0.08em; color: #a09080; margin-bottom: 20px; }
    .calendar-strip { display: flex; align-items: flex-start; gap: 0; position: relative; }
    .calendar-strip::before {
      content: '';
      position: absolute;
      top: 10px;
      left: 0; right: 0;
      height: 2px;
      background: #e4ddd6;
      z-index: 0;
    }
    .calendar-race {
      display: flex;
      flex-direction: column;
      align-items: center;
      flex: 1;
      position: relative;
      z-index: 1;
    }
    .calendar-dot {
      width: 20px; height: 20px;
      border-radius: 50%;
      border: 3px solid #faf7f2;
      margin-bottom: 8px;
      flex-shrink: 0;
    }
    .calendar-dot.priority-a { background: #c07058; }
    .calendar-dot.priority-b { background: #c09060; }
    .calendar-dot.priority-c { background: #a09080; }
    .calendar-race-name { font-size: 0.75rem; font-weight: 500; text-align: center; line-height: 1.3; }
    .calendar-weeks { font-size: 0.7rem; color: #a09080; margin-top: 2px; }

    /* Section 3 — Hero Metrics */
    .metrics-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; }
    .metric-card {
      background: #faf7f2;
      border: 1px solid rgba(0,0,0,0.07);
      border-radius: 12px;
      padding: 20px;
      text-align: center;
    }
    .metric-label { font-size: 0.8rem; font-weight: 700; text-transform: uppercase; letter-spacing: 0.08em; color: #a09080; margin-bottom: 8px; }
    .metric-value { font-family: 'Lora', serif; font-size: 2rem; font-weight: 700; line-height: 1.1; }
    .metric-sub { font-size: 0.8rem; color: #a09080; margin-top: 4px; }

    /* Section 4 — Radar Chart */
    .radar-wrapper { display: flex; justify-content: center; }
    .radar-wrapper canvas { max-width: 400px; max-height: 400px; }
    .radar-na { font-size: 0.85rem; color: #a09080; text-align: center; margin-top: 12px; }

    /* Section 6 — Training Block Timeline */
    .timeline-header { margin-bottom: 6px; }
    .timeline-title { font-weight: 700; font-size: 0.85rem; text-transform: uppercase; letter-spacing: 0.08em; color: #a09080; }
    .timeline-subtitle { font-size: 0.82rem; color: #a09080; margin-top: 4px; margin-bottom: 18px; }
    .timeline-chart-wrapper { position: relative; height: 260px; }

    /* Section 7 — Elevation Profile Overlay */
    .elevation-header { margin-bottom: 6px; }
    .elevation-title { font-weight: 700; font-size: 0.85rem; text-transform: uppercase; letter-spacing: 0.08em; color: #a09080; }
    .elevation-subtitle { font-size: 0.82rem; color: #a09080; margin-top: 4px; margin-bottom: 18px; }
    .elevation-chart-wrapper { position: relative; height: 260px; }
    .elevation-unavailable { font-size: 0.9rem; color: #a09080; padding: 8px 0; }

    /* Section 5 — Gap Analysis */
    .gap-title { font-weight: 700; font-size: 0.85rem; text-transform: uppercase; letter-spacing: 0.08em; color: #a09080; margin-bottom: 16px; }
    .gap-row { display: flex; align-items: center; gap: 16px; padding: 14px 0; border-bottom: 1px solid #e4ddd6; }
    .gap-row:last-child { border-bottom: none; }
    .gap-icon { font-size: 1.3rem; width: 28px; text-align: center; flex-shrink: 0; }
    .gap-name { font-weight: 700; font-size: 0.95rem; min-width: 110px; }
    .status-pill {
      display: inline-flex; align-items: center; gap: 5px;
      font-size: 0.78rem; font-weight: 700;
      padding: 3px 10px; border-radius: 999px;
      white-space: nowrap; flex-shrink: 0;
    }
    .status-pill.on-track { background: #e2eedc; color: #6aaa6e; }
    .status-pill.gap      { background: #f0e8da; color: #c09060; }
    .status-pill.risk     { background: #f0e0da; color: #c07058; }
    .status-pill.na       { background: #ede9e4; color: #a09080; }
    .gap-detail { font-size: 0.85rem; color: #a09080; flex: 1; }
    .gap-detail strong { color: #1c1814; }
    .taper-banner {
      background: #f0e8da;
      border: 1px solid #c09060;
      border-radius: 8px;
      padding: 10px 14px;
      font-size: 0.85rem;
      color: #c09060;
      margin-top: -8px;
      margin-bottom: 8px;
    }
  </style>
</head>
<body>

  <!-- Section 1: Header -->
  <div class="card">
    <div class="header-race-name lora">[RACE_NAME]</div>
    <div class="header-subtitle">[SPORT] · [RACE_TYPE]</div>
    <div class="header-date">[RACE_DATE_FORMATTED] · [DAYS_TO_RACE] days out</div>
  </div>

  <!-- Section 2: Race Calendar Strip -->
  <div class="card">
    <div class="calendar-title">Race Calendar</div>
    <div class="calendar-strip">
      <!-- For each race in raceConfig.races, insert: -->
      <!--
      <div class="calendar-race">
        <div class="calendar-dot priority-[a|b|c]"></div>
        <div class="calendar-race-name">[race.name or race.raceType]</div>
        <div class="calendar-weeks">[weeksOut] wks</div>
      </div>
      -->
    </div>
  </div>

  <!-- Section 3: Hero Metrics -->
  <div class="metrics-grid" style="margin-bottom: 20px;">
    <div class="metric-card">
      <div class="metric-label">Readiness</div>
      <div class="metric-value lora">[READINESS_PCT]%</div>
    </div>
    <div class="metric-card">
      <div class="metric-label">Finish Window</div>
      <div class="metric-value lora" style="font-size:1.4rem;">[RANGE_LOW] – [RANGE_HIGH]</div>
    </div>
    <div class="metric-card">
      <div class="metric-label">Confidence</div>
      <div class="metric-value lora">[CONFIDENCE_PCT]%</div>
    </div>
  </div>

  <!-- Section 4: Radar Chart -->
  <div class="card">
    <div class="radar-wrapper">
      <canvas id="radarChart"></canvas>
    </div>
    <!-- If any dimensions were excluded/skipped: -->
    <div class="radar-na">N/A dimensions (excluded from chart): [EXCLUDED_LIST or "none"]</div>
  </div>

  <!-- Section 5: Gap Analysis -->
  <div class="card">
    <div class="gap-title">Gap Analysis</div>

    <!-- For each included dimension, render a .gap-row -->
    <!-- Example on-track row: -->
    <!--
    <div class="gap-row">
      <div class="gap-icon">📏</div>
      <div class="gap-name">Distance</div>
      <span class="status-pill on-track">● On Track</span>
      <div class="gap-detail">
        <strong>Longest run: 22 mi</strong> · Race distance: 26.2 mi
      </div>
    </div>
    -->

    <!-- Taper risk banner (if gapScores.distance.taperRisk === true): -->
    <!--
    <div class="taper-banner">⚠️ [taperMessage]</div>
    -->

    <!-- For excluded/skipped dimensions, render with na pill: -->
    <!--
    <div class="gap-row">
      <div class="gap-icon">🌡️</div>
      <div class="gap-name">Temperature</div>
      <span class="status-pill na">N/A</span>
      <div class="gap-detail">[reason from gapScores.temperature.reason]</div>
    </div>
    -->
  </div>

  <!-- Section 6: Training Block Timeline -->
  <div class="card">
    <div class="timeline-header">
      <div class="timeline-title">Training Block Timeline</div>
      <!-- subtitle: "Block start: [trainingBlock.startDate formatted as "Mon D, YYYY"] · Current phase: [trainingBlock.currentPhase]" -->
      <div class="timeline-subtitle">Block start: [BLOCK_START_DATE] · Current phase: [CURRENT_PHASE]</div>
    </div>
    <div class="timeline-chart-wrapper">
      <canvas id="timelineChart"></canvas>
    </div>
  </div>

  <!-- Section 7: Elevation Profile Overlay -->
  <div class="card">
    <div class="elevation-header">
      <div class="elevation-title">Course vs. Training Elevation Profile</div>
      <!--
        If courseData.elevationProfileAvailable === false:
          render only the .elevation-unavailable message below; omit canvas and subtitle entirely.
        Otherwise:
          render subtitle and canvas.
      -->
      <!-- subtitle (when available): "Race: [totalElevationGain] ft gain · Training effort: [longestEffort elevation] ft gain" -->
      <div class="elevation-subtitle">Race: [RACE_ELEV_GAIN] ft gain · Training effort: [EFFORT_ELEV_GAIN] ft gain</div>
    </div>
    <!-- Tier 3 / no elevation data: -->
    <!--
    <div class="elevation-unavailable">Elevation overlay not available — course data was entered manually.</div>
    -->
    <!-- When elevation profile is available: -->
    <div class="elevation-chart-wrapper">
      <canvas id="elevationChart"></canvas>
    </div>
  </div>

  <script>
    // Build radar chart from included dimensions only
    // Map status to numeric score: on_track=100, gap=60, risk=20
    // Omit dimensions where status is null/skipped
    const radarLabels = [/* dimension names for included dims */];
    const radarData   = [/* corresponding scores */];

    const ctx = document.getElementById('radarChart').getContext('2d');
    new Chart(ctx, {
      type: 'radar',
      data: {
        labels: radarLabels,
        datasets: [{
          label: 'Race Readiness',
          data: radarData,
          backgroundColor: 'rgba(192, 112, 88, 0.2)',
          borderColor: '#c07058',
          borderWidth: 2,
          pointBackgroundColor: '#c07058',
          pointRadius: 4,
        }]
      },
      options: {
        scales: {
          r: {
            min: 0, max: 100,
            ticks: { stepSize: 20, font: { family: 'DM Sans' }, color: '#a09080' },
            grid: { color: '#e4ddd6' },
            pointLabels: { font: { family: 'DM Sans', size: 13 }, color: '#1c1814' }
          }
        },
        plugins: { legend: { display: false } }
      }
    });

    // ── Section 6: Training Block Timeline ──────────────────────────────
    //
    // weeklyTrend: garminMetrics.volume.weeklyTrend  →  [{ week, totalDistance, activityCount }, ...]
    //   • week is an ISO date string for the Monday of that week (e.g. "2026-04-14")
    //   • totalDistance is in miles
    //
    // weekLabels: format each week ISO date as "Apr 14", "Apr 21", etc.
    const weekLabels = [/* one label per week in weeklyTrend */];
    const weeklyDistances = [/* totalDistance per week in miles */];

    // 4-week rolling average (null for first 3 weeks where window is incomplete)
    const rollingAvg = weeklyDistances.map((_, i) => {
      if (i < 3) return null;
      const window = weeklyDistances.slice(i - 3, i + 1);
      return window.reduce((a, b) => a + b, 0) / 4;
    });

    // Phase transition markers — simulate with a secondary dataset of points
    // For each entry in trainingBlock.phaseTransitions: { phase, startDate }
    // Find the index in weekLabels whose week contains startDate, then add a point at that index.
    // Point dataset: x = weekLabel index, y = weeklyDistances[index], label = phase name
    // Use pointStyle 'triangle', radius 8, color '#6b8e6b', no line (showLine: false)
    const phaseDataset = {
      type: 'scatter',
      label: 'Phase Transition',
      data: [/* { x: weekIndex, y: weeklyDistances[weekIndex] } for each phase transition */],
      pointStyle: 'triangle',
      pointRadius: 9,
      backgroundColor: '#6b8e6b',
      borderColor: '#6b8e6b',
      showLine: false,
    };

    // C race markers
    // For each race in raceConfig.races where designation === "C" and date is in the past:
    // find the matching week index in weeklyTrend, add a point at that index.
    const cRaceDataset = {
      type: 'scatter',
      label: 'C Race',
      data: [/* { x: weekIndex, y: weeklyDistances[weekIndex] } for each past C race */],
      pointStyle: 'star',
      pointRadius: 10,
      backgroundColor: '#c09060',
      borderColor: '#c09060',
      showLine: false,
    };

    // Key session markers — top 3 longest activities from garminMetrics.filteredActivities
    // Find the week index for each activity's start date, add a point.
    const keySessionDataset = {
      type: 'scatter',
      label: 'Key Session',
      data: [/* { x: weekIndex, y: weeklyDistances[weekIndex] } for each top-3 effort */],
      pointStyle: 'rectRot',
      pointRadius: 8,
      backgroundColor: '#c07058',
      borderColor: '#c07058',
      showLine: false,
    };

    const timelineCtx = document.getElementById('timelineChart').getContext('2d');
    new Chart(timelineCtx, {
      type: 'bar',
      data: {
        labels: weekLabels,
        datasets: [
          {
            type: 'bar',
            label: 'Weekly Volume (mi)',
            data: weeklyDistances,
            backgroundColor: 'rgba(107, 142, 107, 0.6)',
            borderColor: 'rgba(107, 142, 107, 0.8)',
            borderWidth: 1,
            order: 3,
          },
          {
            type: 'line',
            label: '4-Week Rolling Avg',
            data: rollingAvg,
            borderColor: '#c07058',
            borderWidth: 2,
            tension: 0.4,
            pointRadius: 3,
            pointBackgroundColor: '#c07058',
            fill: false,
            spanGaps: false,
            order: 2,
          },
          { ...phaseDataset, order: 1 },
          { ...cRaceDataset, order: 1 },
          { ...keySessionDataset, order: 1 },
        ]
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        scales: {
          x: {
            ticks: { font: { family: 'DM Sans', size: 11 }, color: '#a09080', maxRotation: 45 },
            grid: { color: '#e4ddd6' },
          },
          y: {
            title: { display: true, text: 'Miles', font: { family: 'DM Sans', size: 11 }, color: '#a09080' },
            ticks: { font: { family: 'DM Sans', size: 11 }, color: '#a09080' },
            grid: { color: '#e4ddd6' },
            beginAtZero: true,
          }
        },
        plugins: {
          legend: {
            display: true,
            labels: { font: { family: 'DM Sans', size: 11 }, color: '#1c1814', usePointStyle: true }
          },
          tooltip: {
            callbacks: {
              label: ctx => `${ctx.dataset.label}: ${typeof ctx.raw === 'object' ? ctx.raw.y?.toFixed(1) : ctx.raw?.toFixed(1)} mi`
            }
          }
        }
      }
    });

    // ── Section 7: Elevation Profile Overlay ────────────────────────────
    //
    // Only render the chart when courseData.elevationProfileAvailable !== false.
    // When false, show the .elevation-unavailable div and hide the canvas wrapper.
    //
    // Race course trace: courseData.elevationProfileArray → [{ distanceMiles, elevationFeet }, ...]
    // Training effort trace: built from mcp__garmin__get_activity_fit_data called in Module 8
    //   with garminMetrics.volume.longestEffort's activity ID.
    //   Extract GPS records → [{ distance_meters, altitude_meters }, ...]
    //   Convert: distanceMiles = cumulative distance in km / 1.60934
    //             elevationFeet = altitude_meters * 3.28084
    //   Normalize both traces to percentage of their own total distance,
    //   then re-scale x to miles using each trace's total distance.
    //   Downsample to ≤200 points each for render performance.
    //
    // Both datasets share the same X axis (miles). Align by re-interpolating the shorter
    // trace to match the longer trace's x range if needed, OR simply plot both raw and
    // let Chart.js handle different x-lengths via the scatter-line approach below.

    // courseTrace: array of { x: distanceMiles, y: elevationFeet } from courseData.elevationProfileArray
    const courseTrace = [/* { x, y } per point */];

    // effortTrace: array of { x: distanceMiles, y: elevationFeet } from Garmin fit data
    const effortTrace = [/* { x, y } per point */];

    const elevCtx = document.getElementById('elevationChart').getContext('2d');
    new Chart(elevCtx, {
      type: 'line',
      data: {
        datasets: [
          {
            label: 'Race Course',
            data: courseTrace,
            borderColor: '#c07058',
            borderWidth: 2,
            backgroundColor: 'rgba(192, 112, 88, 0.08)',
            fill: true,
            tension: 0.3,
            pointRadius: 0,
            parsing: { xAxisKey: 'x', yAxisKey: 'y' },
          },
          {
            label: 'Longest Training Effort',
            data: effortTrace,
            borderColor: 'rgba(107, 142, 107, 0.8)',
            borderWidth: 2,
            backgroundColor: 'rgba(107, 142, 107, 0.06)',
            fill: true,
            tension: 0.3,
            pointRadius: 0,
            parsing: { xAxisKey: 'x', yAxisKey: 'y' },
          }
        ]
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        scales: {
          x: {
            type: 'linear',
            title: { display: true, text: 'Distance (mi)', font: { family: 'DM Sans', size: 11 }, color: '#a09080' },
            ticks: { font: { family: 'DM Sans', size: 11 }, color: '#a09080' },
            grid: { color: '#e4ddd6' },
          },
          y: {
            title: { display: true, text: 'Elevation (ft)', font: { family: 'DM Sans', size: 11 }, color: '#a09080' },
            ticks: { font: { family: 'DM Sans', size: 11 }, color: '#a09080' },
            grid: { color: '#e4ddd6' },
          }
        },
        plugins: {
          legend: {
            display: true,
            labels: { font: { family: 'DM Sans', size: 11 }, color: '#1c1814', usePointStyle: true }
          },
          tooltip: {
            callbacks: {
              title: items => `${items[0].parsed.x.toFixed(2)} mi`,
              label: ctx => `${ctx.dataset.label}: ${Math.round(ctx.parsed.y)} ft`
            }
          }
        }
      }
    });
  </script>

</body>
</html>
```

Fill in all template placeholders with real values from working context before writing the file. Do not write the template literally — substitute every `[PLACEHOLDER]` with the computed value.

**Dimension icons and keys:**
| Dimension   | Icon | gapScores key   |
|-------------|------|-----------------|
| Distance    | 📏   | `distance`      |
| Elevation   | ⛰️   | `elevation`     |
| Terrain     | 🌲   | `terrain`       |
| Intensity   | ⚡   | `intensity`     |
| Temperature | 🌡️  | `temperature`   |

**Status → score mapping for radar:**
- `on_track` → 100
- `gap` → 60
- `risk` → 20
- `null` / skipped → omit axis entirely

**Race priority → dot class:**
- `A` → `priority-a` (coral `#c07058`)
- `B` → `priority-b` (amber `#c09060`)
- `C` → `priority-c` (grey `#a09080`)

**Weeks-to-race for calendar strip:** compute from today's date to each race's date. Round to nearest whole week. If the race is in the past, show "past".

**Number formatting:** format all numeric values for human readability — e.g., `4,200 ft`, `26.2 mi`, `3:45:00`.

### Step 4 — Note missing data inline

For any dimension where data was unavailable, render the gap analysis row with the grey `na` pill and include the `reason` field from the gap score object as the detail text. Do not omit the row.

### Step 5 — Append Section 8: Training Recommendation Cards

After Section 7, append a Section 8 block to the HTML report body with the heading **"🎯 Training Recommendations"** and subtitle **"Sorted by urgency (gap severity × weeks remaining)"**.

#### Card generation logic

For each gap dimension (`distance`, `elevation`, `terrain`, `intensity`, `temperature`), examine its `score` value from the gap scores object:

- `score === "gap"` or `score === "risk"` → generate at least one recommendation card for this dimension.
- `score === "on_track"` → generate a Low-priority maintenance card **only if** there are fewer than 5 gap/risk cards total; otherwise skip.
- `score === null` / dimension was skipped → no card.

Additionally, if `gapScores.distance.taperRisk === true`, generate a separate **Recovery & Taper** card regardless of the distance score.

**Card ordering:** Sort all cards by `gapSeverity × weeksRemaining` descending before rendering:
- `gapSeverity`: risk = 3, gap = 2, on_track = 1, taperRisk always treated as 3
- `weeksRemaining`: use `raceConfig.weeksToRace`

Higher product → card appears first.

#### Card content lookup by dimension

Use the following table to select card content based on the athlete's sport, race type, and gap dimension.

**Distance gap:**
- Half Marathon / Marathon / Ultra: category `"Long Efforts"`, gap context `"Closes your distance gap — build weekly volume and extend your long effort progressively"`, example session types: progressive long runs; back-to-back weekend runs (ultra only); race-pace long runs
- Gran Fondo / Century: category `"Endurance Rides"`, example session types: long steady rides; progressive centuries; back-to-back riding days

**Elevation gap:**
- Running sports: category `"Climbing / Vertical Work"`, gap context `"Race demands significantly more elevation than your training average"`, example session types: hill repeats; hilly long runs; sustained climb intervals
- Cycling sports: category `"Climbing Work"`, example session types: hill repeats on the bike; FTP climbing intervals; long climbs

**Terrain gap:**
- MTB: category `"Technical Terrain Exposure"`, gap context `"Race course is more technical than your training terrain"`, example session types: trail riding on technical singletracks; rock gardens; rooted descents
- Running (trail/technical race): category `"Trail Running"`, gap context `"Race is on trail/technical terrain — more trail runs needed"`, example session types: trail long runs; technical trail efforts; hiking poles practice (ultra)

**Intensity gap:**
- High-intensity race types (5K, 10K, mile, criterium, XCO): category `"Speed & Intensity Work"`, gap context `"Race demands more high-zone intensity than your current training"`, example session types: VO2max intervals; track repeats; short hard efforts
- Threshold race types (half marathon, marathon, gran fondo): category `"Threshold Training"`, gap context `"Build time at race-pace effort (Z3–Z4)"`, example session types: tempo runs/rides; lactate threshold intervals; race-pace miles
- Aerobic race types (ultra, century): category `"Aerobic Base"`, example session types: easy long efforts; Z2 base building; back-to-back days

**Temperature gap:**
- Heat gap (race day significantly warmer than training environment): category `"Heat Adaptation"`, gap context `"Race day is significantly warmer than your training environment"`, example session types: heat acclimation runs (midday or layered clothing); sauna sessions post-workout; early-morning heat exposure
- Cold gap (race day colder than training environment): category `"Cold Weather Prep"`, example session types: cold-weather long runs; layering practice; cold-start workouts

**Taper risk card** (when `gapScores.distance.taperRisk === true`):
- Category `"Recovery & Taper"`, priority always 🔴 High, gap context `"Your most recent long effort was too close to race day — prioritize rest and recovery"`, example session types: easy shakeout runs only; sleep and nutrition focus; no new hard efforts

#### Card HTML structure

Render each card as a `<div>` with the following structure and styles:

```html
<div style="background:#faf7f2; border:1px solid rgba(0,0,0,0.07); border-radius:12px; padding:16px;">
  <!-- Category name: bold, #1c1814, DM Sans -->
  <div style="font-family:'DM Sans',sans-serif; font-weight:700; color:#1c1814; margin-bottom:6px;">
    [Category Name]
  </div>
  <!-- Priority badge: pill shape -->
  <span style="display:inline-block; padding:2px 10px; border-radius:999px; font-size:12px; font-weight:600; margin-bottom:10px;
    background: [badge-bg]; color: [badge-color];">
    [🔴 High | 🟡 Medium | 🟢 Low]
  </span>
  <!-- Gap context sentence: muted text -->
  <p style="font-family:'DM Sans',sans-serif; color:#a09080; font-size:14px; margin:0 0 10px;">
    [Gap context sentence]
  </p>
  <!-- Example session types: 2–3 bullets, inspiration only -->
  <ul style="font-family:'DM Sans',sans-serif; color:#1c1814; font-size:14px; margin:0; padding-left:18px;">
    <li>[Example 1]</li>
    <li>[Example 2]</li>
    <li>[Example 3 if applicable]</li>
  </ul>
</div>
```

**Priority badge colour mapping:**
- 🔴 High (risk or taperRisk): background `#fde8e8`, color `#c0392b`
- 🟡 Medium (gap): background `#fef3cd`, color `#856404`
- 🟢 Low (on_track maintenance): background `#d4edda`, color `#155724`

Wrap all cards in a responsive CSS grid container:

```html
<div style="display:grid; grid-template-columns:repeat(auto-fill, minmax(300px, 1fr)); gap:16px;">
  <!-- cards here -->
</div>
```

Below the grid, render this footnote in small italic text:

> *Recommendations are category-level only — specific workout design belongs to your coach.*

#### Full section wrapper

Wrap the section heading, subtitle, card grid, and footnote in a `<section>` with consistent report styling (e.g., `max-width:860px; margin:40px auto; padding:0 24px`).

---

### Step 5b — Multi-race additions

This step appends B race summary cards and verifies C race wiring before the report is saved.

#### Race Calendar Strip verification

Confirm that the Section 2 Race Calendar Strip renders **all races** from `raceConfig.races`, sorted chronologically left to right. Each race entry must include:

- A dot with the correct CSS class: `priority-a` (coral `#c07058`), `priority-b` (amber `#c09060`), or `priority-c` (grey `#a09080`)
- A designation label (A / B / C) shown above the dot
- The race name (or race type if no name was given)
- Weeks-to-race below the dot: compute from today's date to each race's date, rounded to nearest whole week; if in the past, show "past"

Update the Section 2 HTML to include the designation label. Modify the per-race template to:

```html
<div class="calendar-race">
  <div class="calendar-designation" style="font-size:0.65rem; font-weight:700; text-transform:uppercase; letter-spacing:0.06em; color:[DESIGNATION_COLOR]; margin-bottom:3px;">[DESIGNATION]</div>
  <div class="calendar-dot priority-[a|b|c]"></div>
  <div class="calendar-race-name">[race.name or race.raceType]</div>
  <div class="calendar-weeks">[weeksOut] wks</div>
</div>
```

Where `[DESIGNATION_COLOR]` maps to: A → `#c07058`, B → `#c09060`, C → `#a09080`.

#### C race timeline wiring

C races should already appear as scatter points on the Section 6 Training Block Timeline (dataset "C Races", amber `#c09060`, star pointStyle). Verify that for each race in `raceConfig.races` where `designation === "C"` and `date` is before today, a scatter point exists.

If any past C races are present, update the Section 6 chart subtitle to append: `" · C races shown as ◆ markers"`.

#### B race summary cards

After the Section 8 Training Recommendation Cards, check `raceConfig.races` for any races with `designation === "B"`. For each B race (in chronological order by date), append a **B Race Summary Card** to the report body.

**Card structure:**

```html
<div class="b-race-card" style="background:#faf7f2; border-left:4px solid #c09060; border-radius:12px; padding:20px 24px; margin:16px 0; max-width:860px; margin-left:auto; margin-right:auto;">
  <!-- Header -->
  <div style="margin-bottom:14px;">
    <span style="font-size:0.7rem; font-weight:700; text-transform:uppercase; letter-spacing:0.08em; color:#c09060;">B Race</span>
    <div style="font-size:1.1rem; font-weight:700; color:#2d2d2d; margin-top:4px;">[B_RACE_NAME_OR_TYPE]</div>
    <div style="font-size:0.85rem; color:#a09080; margin-top:2px;">[B_RACE_DATE_FORMATTED] · [B_WEEKS_OUT] weeks out</div>
  </div>
  <!-- Key gaps -->
  <div style="margin-bottom:14px;">
    <div style="font-size:0.75rem; font-weight:700; text-transform:uppercase; letter-spacing:0.06em; color:#a09080; margin-bottom:8px;">Key Gaps</div>
    <!-- For each dimension scoring "gap" or "risk", render one line: -->
    <!-- <div style="font-size:0.85rem; color:#2d2d2d; margin-bottom:4px;">📏 Distance: 🟡 Gap</div> -->
    <!-- If no dimensions score gap or risk, render: -->
    <!-- <div style="font-size:0.85rem; color:#a09080;">No significant gaps flagged for this race.</div> -->
  </div>
  <!-- Finish time estimate -->
  <div style="margin-bottom:14px;">
    <div style="font-size:0.75rem; font-weight:700; text-transform:uppercase; letter-spacing:0.06em; color:#a09080; margin-bottom:6px;">Estimated Finish</div>
    <div style="font-size:1rem; color:#2d2d2d;">~[B_RACE_ESTIMATED_TIME] <span style="font-size:0.8rem; color:#a09080;">(based on A race analysis)</span></div>
  </div>
  <!-- Footer note -->
  <div style="font-size:0.78rem; color:#a09080; font-style:italic; border-top:1px solid #e8e0d4; padding-top:10px; margin-top:4px;">
    For full analysis of this race, run <code>/startline</code> with this race as your A race.
  </div>
</div>
```

**Header values:**
- `[B_RACE_NAME_OR_TYPE]` — `race.name` if provided, otherwise `race.raceType`
- `[B_RACE_DATE_FORMATTED]` — the B race date formatted as "Month D, YYYY" (e.g., "July 12, 2026")
- `[B_WEEKS_OUT]` — integer weeks from today to the B race date

**Key gaps (compact one-line summaries):**

Iterate over the five gap dimensions in `gapScores`: `distance`, `elevation`, `terrain`, `intensity`, `temperature`. For each dimension where `status === "gap"` or `status === "risk"`, render a single line using this format:

| Dimension | Icon | Status → label |
|---|---|---|
| distance | 📏 | gap → `🟡 Gap`, risk → `🔴 Risk` |
| elevation | ⛰️ | gap → `🟡 Gap`, risk → `🔴 Risk` |
| terrain | 🌲 | gap → `🟡 Gap`, risk → `🔴 Risk` |
| intensity | ⚡ | gap → `🟡 Gap`, risk → `🔴 Risk` |
| temperature | 🌡️ | gap → `🟡 Gap`, risk → `🔴 Risk` |

Capitalize the dimension name (e.g., "Distance", "Elevation"). Show only dimensions with `gap` or `risk` status — omit `on_track` and `null`/skipped dimensions.

**Finish time estimate:**

Derive a light finish time estimate for the B race by adjusting the A race predicted finish time from `finishPrediction.predicted` (the midpoint of the prediction range). Apply the following rules:

1. **B race is shorter than A race** — compute the ratio `bRaceDistance / aRaceDistance`. Apply Daniels VDOT-equivalent scaling for running, or power-to-time scaling for cycling. As a reasonable approximation for all sports:
   - Ratio < 0.5: multiply A race time by `ratio × 1.08` (shorter events are disproportionately faster)
   - Ratio 0.5–0.85: multiply A race time by `ratio × 1.05`
   - Ratio > 0.85: multiply A race time by `ratio × 1.02`
2. **B race is longer than A race** — compute `excessRatio = (bRaceDistance - aRaceDistance) / aRaceDistance`. Scale up: multiply A race time by `(bRaceDistance / aRaceDistance) × (1 + 0.05 × (excessRatio / 0.2))`.
3. Format the result as `H:MM:SS` (e.g., `1:52:30`). If the estimate is under 60 minutes, format as `M:SS`.

If `finishPrediction` is unavailable, display: "Estimate unavailable — insufficient A race data."

Use `raceConfig.primaryRace.distance` (in miles) as the A race distance and `race.distance` (in miles) as the B race distance.

#### Section order confirmation

The final HTML report body must contain sections in this order:

1. **Header** — A race name, sport, race type, date, days out
2. **Race Calendar Strip** — all races (A, B, C) with designation labels, sorted chronologically
3. **Hero Metrics** — readiness %, finish window, confidence (A race)
4. **Radar Chart** — gap dimensions (A race)
5. **Gap Analysis** — detailed gap rows (A race)
6. **Training Block Timeline** — weekly volume chart with phase transitions and C race markers
7. **Elevation Profile Overlay** — race course vs. longest training effort
8. **Training Recommendation Cards** — prioritized action cards (A race)
9. **B Race Summary Cards** — one card per B race, in chronological order by date

---

### Step 6 — Save and confirm

After writing the file, tell the athlete:

> "📄 Report saved to `~/startline-reports/[filename]`. Open it in your browser to view. (Google Fonts require an internet connection to render correctly.)"

### Step 6 — Append Section 6: Training Block Timeline

Append the Section 6 card to the report body, after Section 5.

**Data preparation:**

1. **Week labels** — format each entry in `garminMetrics.volume.weeklyTrend` as a human-readable week label. Parse the ISO date string in `week` and format as "Mon D" (e.g., "Apr 14"). Store as `weekLabels`.

2. **Weekly distances** — extract `totalDistance` (miles) from each entry in `weeklyTrend`. Store as `weeklyDistances`.

3. **Rolling average** — compute the 4-week rolling average over `weeklyDistances`. For weeks 0–2 (window incomplete), set the value to `null` so Chart.js skips the point.

4. **Phase transition markers** — for each `{ phase, startDate }` in `trainingBlock.phaseTransitions`:
   - Find the index in `weeklyTrend` whose `week` value is the nearest Monday on or before `startDate`.
   - Create a scatter point `{ x: weekIndex, y: weeklyDistances[weekIndex] }` with the phase name in the tooltip.

5. **C race markers** — for each race in `raceConfig.races` where `designation === "C"` and the race `date` is before today:
   - Find the matching week index in `weeklyTrend`.
   - Create a scatter point `{ x: weekIndex, y: weeklyDistances[weekIndex] }`.

6. **Key session markers** — sort `garminMetrics.filteredActivities` by `distance` descending, take the top 3. For each, find the week index by matching its `startTimeLocal` date to a week in `weeklyTrend`. Create scatter points.

**Substitute placeholders** in the HTML card before writing:
- `[BLOCK_START_DATE]` → `trainingBlock.startDate` formatted as "Mon D, YYYY"
- `[CURRENT_PHASE]` → `trainingBlock.currentPhase` (capitalize first letter)

**Chart datasets to render** (all on the same `<canvas id="timelineChart">`):

| Dataset | Type | Color |
|---|---|---|
| Weekly Volume (mi) | bar | `rgba(107, 142, 107, 0.6)` |
| 4-Week Rolling Avg | line overlay | `#c07058`, tension 0.4 |
| Phase Transitions | scatter | `#6b8e6b`, triangle pointStyle |
| C Races | scatter | `#c09060`, star pointStyle |
| Key Sessions | scatter | `#c07058`, rectRot pointStyle |

### Step 7 — Append Section 7: Elevation Profile Overlay

Append the Section 7 card after Section 6.

**Condition check first:**

If `courseData.elevationProfileAvailable === false` (Tier 3 manual entry):
- In the HTML card, remove the `<div class="elevation-subtitle">` and `<div class="elevation-chart-wrapper">` elements.
- Uncomment the `<div class="elevation-unavailable">` paragraph.
- Skip all chart JS for this section — do not emit the `elevationChart` script block.
- The card is still rendered (do not omit Section 7 entirely).

**If elevation data is available:**

1. **Race course trace** — from `courseData.elevationProfileArray`: array of `{ distanceMiles, elevationFeet }`. Convert to Chart.js scatter points `{ x: distanceMiles, y: elevationFeet }`. Downsample to ≤200 points if the array is longer (take every Nth point).

2. **Training effort trace** — call `mcp__garmin__get_activity_fit_data` with the activity ID from `garminMetrics.volume.longestEffort`. This returns GPS/FIT records. Extract cumulative distance and altitude:
   - Convert altitude from meters to feet (`× 3.28084`).
   - Convert cumulative distance from meters to miles (`÷ 1609.34`).
   - Output as `{ x: distanceMiles, y: elevationFeet }` scatter points.
   - Downsample to ≤200 points.

3. **Compute elevation gains** for the subtitle:
   - Race: sum positive altitude deltas from `courseData.elevationProfileArray` (or use `courseData.totalElevationGain` if available).
   - Training effort: sum positive altitude deltas from the FIT records.
   - Format both as integers with comma separator (e.g., `4,200`).

**Substitute placeholders** in the HTML card:
- `[RACE_ELEV_GAIN]` → computed race elevation gain, formatted
- `[EFFORT_ELEV_GAIN]` → computed training effort elevation gain, formatted

Both traces share the X axis (distance in miles). Plot using Chart.js `type: 'line'` with `parsing: { xAxisKey: 'x', yAxisKey: 'y' }` so each dataset uses its own x values rather than a shared label array.

Then ask:

> "Would you like to spotlight a recent key session and see how it maps to your race readiness?"
