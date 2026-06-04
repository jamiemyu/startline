---
name: startline-setup
description: First-time configuration — checks MCP connectivity for Garmin, Strava, and TrainingPeaks
---

# /startline-setup — MCP Connectivity Setup

You are running the startline setup wizard. Follow every step below in order.

---

## Step 1: Read existing setup state

Use the Read tool to check if `~/.startline/setup.json` exists.

- If it exists, parse the JSON and note which MCPs were previously `connected: true`.
- If it does not exist, proceed as a fresh setup (all MCPs are unverified).

---

## Step 2: Check Garmin MCP (REQUIRED)

Attempt to call the Garmin MCP tool `mcp__garmin__get_user_profile` with no arguments.

- **If the tool succeeds** (returns any data without an "MCP not available" or connection error): Garmin is ✅ connected. Record `garmin.connected = true` and the current ISO timestamp in `garmin.checkedAt`.

- **If the tool is unavailable or returns a connectivity error**: Garmin is disconnected.
  - Display this message to the user:

    ```
    ❌ Garmin MCP is not connected — this is required for startline to work.

    To install the Garmin MCP, run:
        claude mcp add garmin
    Then follow the prompts at: https://github.com/NuloShipDev/garmin-mcp

    Re-run /startline-setup after connecting Garmin.
    ```

  - Record `garmin.connected = false` and the current ISO timestamp in `garmin.checkedAt`.
  - Save setup state to `~/.startline/setup.json` (see Step 5) and HALT — do not proceed to Steps 3 or 4.

---

## Step 3: Check Strava MCP (RECOMMENDED)

Attempt to call any available Strava MCP tool (e.g., `mcp__strava-mcp__eligibility` or any tool whose name starts with `mcp__strava`).

- **If a Strava tool succeeds**: Strava is ✅ connected. Record `strava.connected = true` and the current ISO timestamp in `strava.checkedAt`.

- **If no Strava tool is available or all return connectivity errors**: Strava is disconnected.
  - Display this message (but DO NOT halt):

    ```
    ⚠️  Strava MCP is not connected (recommended but not required).
    Connect Strava for Strava route URL support.
    To install: claude mcp add strava
    ```

  - Record `strava.connected = false` and the current ISO timestamp in `strava.checkedAt`.
  - Continue to Step 4.

**Re-check optimization**: If `~/.startline/setup.json` already shows `strava.connected = true` from a prior run, and the Strava tool call above succeeded again, you may note "Strava still connected — skipping prompt." and continue silently.

---

## Step 4: Check TrainingPeaks MCP (OPTIONAL)

Attempt to call the TrainingPeaks MCP tool `mcp__trainingpeaks__tp_auth_status` with no arguments.

- **If the tool succeeds**: TrainingPeaks is ✅ connected. Record `trainingpeaks.connected = true` and the current ISO timestamp in `trainingpeaks.checkedAt`.

- **If the tool is unavailable or returns a connectivity error**: TrainingPeaks is disconnected.
  - Display this message (but DO NOT halt):

    ```
    ℹ️  TrainingPeaks MCP is not connected (optional).
    Connect TrainingPeaks for richer analysis with CTL/ATL/TSB, coach notes, and structured training data.
    To install: claude mcp add trainingpeaks
    ```

  - Record `trainingpeaks.connected = false` and the current ISO timestamp in `trainingpeaks.checkedAt`.
  - Continue to Step 5.

**Re-check optimization**: If `~/.startline/setup.json` already shows `trainingpeaks.connected = true` from a prior run, and the TrainingPeaks tool call above succeeded again, you may note "TrainingPeaks still connected — skipping prompt." and continue silently.

---

## Step 5: Save setup state

Use Bash to create the `~/.startline/` directory if it does not exist:

```bash
mkdir -p ~/.startline
```

Then write the following JSON to `~/.startline/setup.json` using the Write tool (replace placeholder values with actual results from Steps 2–4):

```json
{
  "lastChecked": "<current ISO timestamp>",
  "mcps": {
    "garmin": { "connected": <true|false>, "checkedAt": "<ISO timestamp>" },
    "strava": { "connected": <true|false>, "checkedAt": "<ISO timestamp>" },
    "trainingpeaks": { "connected": <true|false>, "checkedAt": "<ISO timestamp>" }
  }
}
```

---

## Step 6: Re-check optimization (subsequent runs)

If you read a valid `~/.startline/setup.json` in Step 1 and, after re-verifying connectivity in Steps 2–4, ALL previously-connected MCPs are still connected and NO previously-disconnected MCPs have changed status, display:

```
✅ Setup already complete — all previously-connected MCPs are still connected. No changes detected.
```

and skip repeating any install prompts for MCPs that were already disconnected in the prior run (the user has already been informed).

---

## Step 7: Display summary

After saving state, display a summary like this (adjust based on actual connection results):

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  startline setup — MCP status
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ✅  Garmin          (required)
  ✅  Strava          (recommended)
  ℹ️   TrainingPeaks  (optional — not connected)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Setup saved to ~/.startline/setup.json
  Run /startline to begin your race readiness analysis.
```

Use ✅ for connected MCPs, ⚠️ for recommended-but-missing, and ℹ️ for optional-and-missing.

If Garmin was missing, this step is skipped (setup halted in Step 2).
