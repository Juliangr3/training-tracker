# IRON LOG — Workout Tracker

A self-contained single-file HTML app for tracking barbell strength training. No install, no account, no internet needed after first load.

## How to use

1. **Save** `iron-log.html` to your device
2. **Open** it in any modern browser (Chrome, Safari, Firefox)
3. All data saves automatically to your browser's localStorage

## Add to phone home screen

**iPhone (Safari):** Open the file → Share → "Add to Home Screen"  
**Android (Chrome):** Open the file → Menu → "Add to Home Screen"

> Data is tied to the browser on that device. Use **↓ CSV** to back up.

## Tabs

| Tab | What it does |
|-----|--------------|
| **Log** | Log individual exercises manually, or use **▶ START WORKOUT** for guided mode |
| **History** | Browse past sessions by date, expand to see every set |
| **Dashboard** | Weekly volume, sets, PRs, exercise progression charts |
| **Progression** | Prescribed weights for Squat / Bench / Deadlift with ISO week tracking |

## Guided Workout Mode

1. Tap **▶ START WORKOUT** on the Log tab
2. Select a day (Day 1, 2 or 3)
3. Work through exercises one at a time
4. For each exercise: tap **W** on a row to flag it as a warm-up, fill in reps and kg, tap **LOG** to record the set
5. A rest timer starts automatically after each set (configurable: 60–240s)
6. Tap **NEXT EXERCISE** to advance, **FINISH & SAVE** on the last exercise

## Cardio

Tick the **🏃 CARDIO COMPLETED** checkbox on the Log tab for any day. Shows in history and exports as a column in CSV.

## CSV Export

Columns: `ISO Week, Date, Day of Week, Exercise, Set #, Warm-up, Reps, Weight (kg), Rest After (sec), PR, Cardio, Notes`

Use ISO Week column to group/pivot in Excel or Google Sheets.

## Data storage

| Key | Contents |
|-----|----------|
| `ironlog-entries-v4` | All workout entries |
| `ironlog-progression-v1` | Starting weights + increment config |
| `ironlog-phase-v1` | Current phase, transitions, stall counts |
| `ironlog-cardio-v1` | Cardio log by date |

## External dependencies (CDN only)
- React 18.2 + ReactDOM 18.2 (Cloudflare CDN)
- Babel Standalone 7.23 (Cloudflare CDN)
- DM Mono + Bebas Neue (Google Fonts)
