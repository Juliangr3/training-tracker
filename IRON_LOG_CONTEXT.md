# IRON LOG — Developer Context File
*Drop this file into a new Claude conversation to resume development without losing context.*

---

## What this app is

A single-file `iron-log.html` workout tracker for barbell strength training across multi-year macrocycles targeting:
- Bench Press: 130–135 kg
- Squat: 170–180 kg
- Deadlift: 215–225 kg

Built with React 18 + Babel Standalone (CDN), DM Mono + Bebas Neue fonts. All data in `localStorage`. No build step, opens in any browser.

---

## Current file

`iron-log.html` — ~88kb, ~1465 lines of JSX inside `<script type="text/babel">`.

---

## Architecture

```
iron-log.html
├── <style>          CSS classes + animations
└── <script type="text/babel">
    ├── LS{}                localStorage wrapper
    ├── KEYS{}              Storage key constants
    ├── EXERCISE_LIST[]     Grouped exercise dropdown (includes Pull-Ups, Dips)
    ├── PHASES[]            Phase definitions (1=FOUNDATION, 2=STRENGTH, 3=PEAK)
    ├── PHASE_RESET_PCT{}   Reset % by transition pair (3→1: 80%, 1→2/2→3: 95%)
    ├── PHASE_TEMPLATES{}   Workout templates for phases 1, 2, 3
    ├── DEFAULT_PROG[]      Default progression config
    ├── DEFAULT_PHASE{}     Default phase state (includes peakWeights:{})
    ├── Timezone helpers    todaySydney(), toDateStr() — always Australia/Sydney
    ├── Date/week helpers   fmtDate, isoWeek, isoWeekShort, nextIsoWeek, etc.
    ├── DeltaBadge          Inline +/- badge component
    ├── RestTimer           Floating countdown timer component
    ├── GuidedWorkout       Full-screen guided workout modal
    ├── DayCard             Collapsible day card in day picker modal
    └── App                 Main app component (all state, all views)
```

---

## localStorage key version history

| Key | Version | Notes |
|-----|---------|-------|
| `ironlog-entries-v4` | ✅ active | Added `setNumber`, `warmup`, `restAfterSec` vs v3 |
| `ironlog-progression-v1` | ✅ active | `startWeights[id].weekStart` is ISO week label e.g. `"2026-W09"` |
| `ironlog-phase-v1` | ✅ active | Added `peakWeights` and `isoWeek` fields to transitions in this session |
| `ironlog-cardio-v1` | ✅ active | date→bool map |

**Rule:** Bump entries key if entry shape changes. Other keys: only bump if old data would render incorrectly.

---

## localStorage schema

### `ironlog-entries-v4`
```json
{
  "id": "1709123456789",
  "date": "2026-03-05",
  "exercise": "Bench Press",
  "sets": "1",
  "reps": "5",
  "weight": "80",
  "notes": "",
  "pr": false,
  "warmup": false,
  "setNumber": 2,
  "restAfterSec": 142,
  "timestamp": "2026-03-05T08:32:00.000Z"
}
```

### `ironlog-progression-v1`
```json
{
  "startWeights": {
    "squat":    {"weight": 72.5, "weekStart": "2026-W10"},
    "bench":    {"weight": 57.0, "weekStart": "2026-W10"},
    "deadlift": {"weight": 88.0, "weekStart": "2026-W10"}
  },
  "config": [
    {"id": "squat",    "label": "Barbell Squat", "increment": 2.5},
    {"id": "bench",    "label": "Bench Press",   "increment": 1.25},
    {"id": "deadlift", "label": "Deadlift",      "increment": 2.5}
  ]
}
```

### `ironlog-phase-v1`
```json
{
  "currentPhase": 2,
  "phaseStartDate": "2026-03-05",
  "transitions": [
    {
      "from": 1, "to": 2,
      "date": "2026-03-05",
      "isoWeek": "2026-W10",
      "reason": "complete",
      "peaks": {"squat": 90, "bench": 75, "deadlift": 110},
      "resetPct": 0.95
    }
  ],
  "stallCounts": {"squat": 0, "bench": 0, "deadlift": 0},
  "peakWeights": {
    "1": {"squat": 90, "bench": 75, "deadlift": 110}
  }
}
```
- `reason` values: `"complete"` | `"manual"` | `"auto-stall"`
- `peakWeights` keyed by phase id string — preserved across phase cycles for macrocycle tracking

### `ironlog-cardio-v1`
```json
{"2026-03-03": true, "2026-03-05": true}
```

---

## Timezone

All "today" derivations use `Australia/Sydney` timezone explicitly:
```js
function todaySydney(){
  return new Date().toLocaleDateString("en-CA",{timeZone:"Australia/Sydney"});
}
function toDateStr(d=new Date()){
  return d.toLocaleDateString("en-CA",{timeZone:"Australia/Sydney"});
}
```
`en-CA` locale returns `YYYY-MM-DD` format. All date comparisons, `formDate` defaults, and `isoWeek()` calculations use Sydney date — never raw `new Date().toISOString()`.

---

## Phase system

### Three phases — one macrocycle
| Phase | Name | Days/wk | Structure | Role |
|-------|------|---------|-----------|------|
| 1 | FOUNDATION | 2 | 5×6 | Build base, establish movement |
| 2 | STRENGTH | 3 | 5×5 | Drive linear progression |
| 3 | PEAK | 2 | 5×6 | Peak output, max weights |

Cycling 1→2→3→1 is one **macrocycle**. Each new macrocycle starts heavier than the last.

### 5-week minimum lock
- Forward transitions (e.g. Phase 1→2) blocked until `weeksBetween(phaseStartDate, today) >= 5`
- Phase pills for locked phases rendered with 🔒 and weeks remaining
- Backwards transitions (e.g. manual demotion) unrestricted
- `canAdvancePhase(toId)` and `weeksUntilEligible()` are the gate functions

### COMPLETE PHASE flow
Triggered by the green **✓ COMPLETE PHASE N →** button (only shown when ≥5 weeks elapsed):
1. `capturePeakWeights(fromPhaseId)` — highest working set weight per lift since `phaseStartDate`
2. `completePhase()` — applies `PHASE_RESET_PCT` to peaks, writes to `startWeights`, saves transition with peaks/resetPct, sets `peakWeights[fromId]`
3. Confirm modal shows: lift name · phase peak · → · opening weight for new phase

### Reset percentages
```js
const PHASE_RESET_PCT = {
  "1-2": 0.95,  // carry momentum
  "2-3": 0.95,  // stay near peak
  "3-1": 0.80,  // full macrocycle reset
};
```
Opening weight rounded to nearest 0.5kg: `Math.round(peak * pct * 2) / 2`

---

## Phase templates

### Phase 1 — FOUNDATION (2 days/week, 5×6)
- **Day 1:** Barbell Squat, Bench Press, Lat Pulldown
- **Day 2:** Deadlift, Shrugs, Pull-Ups, Close Grip Bench Press

### Phase 2 — STRENGTH (3 days/week, 5×5)
- **Day 1:** Barbell Squat, Bench Press, Lat Pulldown, Barbell Shoulder Press, Crunches
- **Day 2:** Incline Bench Press, Barbell Rows, Shrugs, Lateral Raises, Calf Raises
- **Day 3:** Barbell Squat @80%, Close Grip Bench Press, Deadlift, Shoulder Flys, Crunches

### Phase 3 — PEAK (2 days/week, 5×6)
- **Day 1:** Barbell Squat, Dips, Pull-Ups, Shrugs, Calf Raises (4×15), Crunches (3×15)
- **Day 2:** Deadlift, Bench Press, Lat Pulldown, Barbell Shoulder Press, Calf Raises (4×15), Crunches (1×80)

Squat/Bench/Deadlift have `weightKey` linking to progression config for prescribed weight pre-fill.

---

## Prescribed weight calculation

```
prescribed = startWeight + (weeksElapsed × increment)
```
- `startWeight` — from `startWeights[lift.id].weight` (set by COMPLETE PHASE auto-calc or manual override)
- `weeksElapsed` — ISO weeks from `startWeekKey` to current ISO week, counted via `nextIsoWeek()` iterator
- `increment` — configurable per lift (default: squat/deadlift +2.5kg, bench +1.25kg)
- After COMPLETE PHASE: `startWeight` is automatically set to `peak × resetPct`, `startWeekKey` is set to current ISO week → progression restarts cleanly

---

## Load % column

Shown in Progression tab lift table. Calculated in `getProgData()`:
```js
loadPct = Math.round((thisWeekActual / prescribedWeight) * 100)
```
Colour coding:
- ≥100%: green (`#4a9a4a`) — on target
- 85–99%: amber (`#c87820`) — close
- <85%: red (`#9a4a4a`) — below

---

## Key functions

| Function | Location | Notes |
|----------|----------|-------|
| `todaySydney()` | top-level | Returns YYYY-MM-DD in Sydney timezone |
| `toDateStr()` | top-level | Same — used for all "today" derivations |
| `isoWeek(dateStr)` | top-level | Returns `"2026-W11"` (ISO 8601 Thursday rule) |
| `isoWeekShort(dateStr)` | top-level | Returns `"W11 2026"` for display |
| `nextIsoWeek(label)` | top-level | Advances one ISO week, handles year rollover |
| `getProgData()` | App | Rebuilds every render. Uses `nextIsoWeek()` for week counting. Excludes warmups. Calculates `loadPct`. |
| `capturePeakWeights(phaseId)` | App | Scans entries since `phaseStartDate`, returns max per lift |
| `completePhase()` | App | Full phase transition: peaks → reset weights → new start weights → transition record |
| `canAdvancePhase(toId)` | App | Returns bool — false if forward and <5 weeks elapsed |
| `weeksUntilEligible()` | App | Returns weeks remaining before phase can advance |
| `getPhaseOnDate(date)` | App | Reconstructs which phase was active on a given date (for CSV) |
| `switchPhase(toId,reason)` | App | Manual phase switch — checks lock, no peak capture |
| `DayCard` | component | Collapsible card; START button always visible |
| `GuidedWorkout` | component | One exercise at a time; LOG per set; rest timer; + ADD SET |
| `RestTimer` | component | SVG ring countdown; urgent pulse ≤10s |

---

## CSV columns (current)

`ISO Week, Date, Day of Week, Phase, Exercise, Set #, Warm-up, Reps, Weight (kg), Rest After (sec), PR, Cardio, Notes`

Phase column derived from `getPhaseOnDate()` using transition history.

---

## Bugs fixed (cumulative)

| Bug | Fix |
|-----|-----|
| Day picker START button unreachable on mobile | DayCard component: START always visible, exercises collapsible |
| All modals anchored top-left on mobile Safari | `position:fixed + transform:-50%` replaced with flex-centred backdrop wrapper |
| Progression week counting wrong across year boundaries | `nextIsoWeek()` iterator replacing manual week arithmetic |
| Phase history showed dates instead of week labels | `isoWeekShort()` used in history, `isoWeek` saved on transition |

---

## What's NOT yet built (future sessions)

- Macrocycle-over-macrocycle progress view (peak per phase across all macrocycles — data is already being preserved in `peakWeights`)
- Rest period tracking between phases (10 days, show in history)
- Phase column in workout History tab
- Body weight logging
- Any backend / sync

---

## Design tokens

```
Background:    #0d0d0d
Surface:       #111 / #161410
Border:        #1a1815 / #2a2820
Gold accent:   #e8c84a  (hover: #d4b83e)
Green (good):  #4a9a4a / #70e870
Red (stall):   #8b2222 / #e87070
Amber (warn):  #e8a040 / #c87820
Text primary:  #e8e0d5
Text muted:    #5a5550 / #4a4540
Text dim:      #3a3530
Fonts:         'Bebas Neue' (display), 'DM Mono' (body)
```

---

## How to resume development

1. Share this file + `iron-log.html` with Claude
2. State what you want to change
3. Claude reads the relevant sections, makes targeted edits, runs 40+ validation checks, delivers updated files

**Validation checklist Claude runs before delivering:**
- Brace `{}` and paren `()` balance both diff=0
- Sydney timezone present and used in all date derivations
- All phase templates present (1, 2, 3)
- Phase lock, completePhase, capturePeakWeights all present
- loadPct calculated and rendered
- isoWeekShort in history, isoWeek saved on transitions
- All 4 views present
- No extra localStorage keys introduced
- Stale copy removed
