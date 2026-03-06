# IRON LOG — Developer Context File
*Drop this file into a new Claude conversation to resume development without losing context.*

---

## What this app is

A single-file `iron-log.html` workout tracker for barbell strength training. Built with React 18 + Babel Standalone (CDN), styled with DM Mono + Bebas Neue fonts. All data persists in `localStorage`. No build step, no server, opens directly in any browser.

---

## Current file

`iron-log.html` — ~80kb, ~1228 lines of JSX inside a `<script type="text/babel">` tag.

---

## Architecture

```
iron-log.html
├── <style>          CSS classes + animations
└── <script type="text/babel">
    ├── LS{}                localStorage wrapper (get/set with JSON parse/stringify)
    ├── KEYS{}              Storage key constants
    ├── EXERCISE_LIST[]     Grouped exercise dropdown data
    ├── PHASES[]            Phase definitions (id, name, days, sets, reps)
    ├── PHASE_TEMPLATES{}   Workout templates keyed by phase id
    ├── DEFAULT_PROG[]      Default progression config
    ├── DEFAULT_PHASE{}     Default phase state shape
    ├── Date/week helpers   toDateStr, fmtDate, monStart, weekEnd, isoWeek, etc.
    ├── DeltaBadge          Inline component for +/- badges
    ├── RestTimer           Floating countdown timer component
    ├── DayCard             Collapsible day card used inside the day picker modal
    ├── GuidedWorkout       Full-screen guided workout modal component
    └── App                 Main app component (all state, all views)
```

---

## localStorage key version history

Bumping a key name means old data under the previous key is **not deleted** — it stays in the browser under the old name but the app no longer reads it. Always export a CSV backup before overwriting the HTML file if you have real data to preserve.

| Key | Current version | Previous versions | Why bumped |
|-----|----------------|-------------------|------------|
| `ironlog-entries-v4` | ✅ active | `ironlog-entries-v3` (original export session) | Added `setNumber`, `warmup`, `restAfterSec` fields to entry shape |
| `ironlog-progression-v1` | ✅ active | — | Never bumped; `startWeights.weekStart` changed from a date string to ISO week label (`"2026-W09"`) in this session — not backwards compatible if start weights were previously set |
| `ironlog-phase-v1` | ✅ active | — | Never bumped |
| `ironlog-cardio-v1` | ✅ active | — | New in this session, no prior version |

**Rule for future sessions:** Bump the entries key version any time the shape of an entry object changes (new required fields, renamed fields, type changes). Leave the key unchanged only if the change is purely additive and old entries will still render correctly without the new fields.

---

## localStorage schema

### `ironlog-entries-v4` — Array of entry objects
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
**Key points:**
- Guided workout saves **one entry per set** (`sets:"1"`, `setNumber:1..N`)
- Manual log saves one entry per exercise (sets/reps as entered, `setNumber:null`)
- `warmup:true` entries are excluded from: progression calcs, stall detection, volume stats, dashboard charts
- `restAfterSec` = actual elapsed time between tapping LOG on consecutive sets (null if first set or manual log)

### `ironlog-progression-v1`
```json
{
  "startWeights": {
    "squat":    {"weight": 100, "weekStart": "2026-W09"},
    "bench":    {"weight": 80,  "weekStart": "2026-W09"},
    "deadlift": {"weight": 120, "weekStart": "2026-W09"}
  },
  "config": [
    {"id": "squat",    "label": "Barbell Squat", "increment": 2.5},
    {"id": "bench",    "label": "Bench Press",   "increment": 1.25},
    {"id": "deadlift", "label": "Deadlift",      "increment": 2.5}
  ]
}
```
- `weekStart` is an ISO week label e.g. `"2026-W09"` (changed from date string in this session)
- Prescribed weight = `startWeight + (weeksElapsed × increment)`
- Weeks elapsed counted by iterating ISO week labels from `weekStart` to current ISO week

### `ironlog-phase-v1`
```json
{
  "currentPhase": 2,
  "phaseStartDate": "2026-02-03",
  "transitions": [
    {"from": 1, "to": 2, "date": "2026-02-03", "reason": "manual"}
  ],
  "stallCounts": {"squat": 0, "bench": 1, "deadlift": 0}
}
```
- `stallCounts` reset to 0 on every phase switch
- `transitions` reason is `"manual"` or `"auto-stall"`

### `ironlog-cardio-v1`
```json
{
  "2026-03-03": true,
  "2026-03-05": true
}
```
Simple date → boolean map. Only `true` entries are ever written.

---

## Key functions

| Function | Location | Notes |
|----------|----------|-------|
| `isoWeek(dateStr)` | top-level helper | Returns `"2026-W11"` format. Uses ISO 8601 Thursday-rule. |
| `getProgData()` | App component | Rebuilds from live entries every render. Uses `isoWeek()` keys. Excludes warmups. |
| `weekStats(ws)` | App component | Excludes warmup entries from volume/sets counts |
| `GuidedWorkout` | separate component | Receives `progData` as prop for prescribed weight lookup |
| `tmplWeight(key,mult)` | App component | Used in day picker modal to show preview weights |
| `RestTimer` | separate component | SVG ring countdown, auto-dismisses at 0, urgent pulse <10s |
| `DayCard` | separate component | Collapsible card in day picker; START button always visible; exercise list toggles open/closed |

---

## Progression logic (FIXED in this session)

**Previous bug:** `weekMap` used calendar date strings as keys → deleted entries could still influence "last logged" because the logic didn't re-derive from scratch each render cleanly (it did, but display labels were date-based and confusing).

**Fix applied:**
1. `weekMap` now keyed by ISO week label (`isoWeek(e.date)`) not calendar date
2. "Last Logged" column in Progression tab shows ISO week label e.g. `2026-W08`, not a date
3. `startWeekKey` in `startWeights` is now stored as an ISO week label
4. Progression table shows "This Week" as `{todayIsoWk} · pending` when not yet logged
5. All warmup sets (`e.warmup === true`) filtered out before building `weekMap`

---

## Phase templates (Phase 2 only, others TBD)

```
Phase 2, Day 1: Barbell Squat (5×5), Bench Press (5×5), Lat Pulldown (5×5),
                Barbell Shoulder Press (5×5), Crunches (3×15)

Phase 2, Day 2: Incline Bench Press (5×5), Barbell Rows (5×5),
                Shrugs (4×10), Lateral Raises (4×12), Calf Raises (4×15)

Phase 2, Day 3: Barbell Squat @ 80% (5×5), Close Grip Bench Press (5×5),
                Deadlift (5×5), Shoulder Flys (4×12), Crunches (3×15)
```

Template exercises with `weightKey` get prescribed weights auto-filled from `getProgData()`.  
`weightMult: 0.8` on Day 3 squat applies the 80% calculation dynamically.

---

## Guided workout flow

1. User taps **▶ START WORKOUT** → Day Picker modal opens
2. Modal shows one `DayCard` per day. Each card shows: day number, label, exercise count, and a gold **START DAY N →** button that is always visible (never hidden behind a scroll). Tapping the card header expands/collapses an exercise preview list (name, sets×reps, prescribed weight). Tapping START launches `GuidedWorkout`.
3. One exercise at a time — nav pills at top show progress
4. Each exercise has N set rows pre-populated:
   - From last session's set data if available (matches by `exercise + setNumber`)
   - Otherwise from template defaults (prescribed weight, template reps)
   - Prescribed weight cascades down to blank weight fields automatically
5. Per set row: **W** toggles warmup, reps/weight inputs, **LOG** button records the set
6. After LOG: rest timer starts (configurable 60–240s), can be dismissed
7. **NEXT EXERCISE →** / **FINISH & SAVE** commits all set rows as individual entries

**Weight drop detection:** If a working set's weight is lower than the previous working set, the row highlights amber and a warning banner appears. This is an overload indicator.

---

## CSV columns (current)

`ISO Week, Date, Day of Week, Exercise, Set #, Warm-up, Reps, Weight (kg), Rest After (sec), PR, Cardio, Notes`

---

## Bugs fixed

| Bug | Root cause | Fix applied |
|-----|-----------|-------------|
| Day picker START button unreachable on mobile | Scrollable container had no `flex:1` or constrained height — content overflowed the modal with no way to scroll | Rebuilt day picker with `DayCard` component: START button always visible, exercise list collapsible; scroll container given `flex:1` |
| All modals anchored to top-left on mobile Safari | `position:fixed` + `transform:translate(-50%,-50%)` breaks when a parent has a CSS transform or scroll context | Replaced with single flex-centred wrapper: `position:fixed; inset:0; display:flex; align-items:center; justify-content:center` — modal is a child, `e.stopPropagation()` prevents backdrop tap from closing |

---

## What's NOT yet built (future sessions)

- Phase 1 (Deload) and Phase 3 (Peak) workout templates — structure is ready, just needs `PHASE_TEMPLATES[1]` and `PHASE_TEMPLATES[3]` populated
- Volume-per-set live calculation in guided mode
- Body weight logging
- Notes per workout session (not per exercise)
- Any backend / sync — currently browser-local only

---

## Design tokens

```
Background:    #0d0d0d
Surface:       #111 / #161410
Border:        #1a1815 / #2a2820
Gold accent:   #e8c84a
Gold hover:    #d4b83e
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
3. Claude will read the current HTML file, make targeted edits, validate balance/checks, then present the updated file

**Validation checklist Claude runs before delivering:**
- Brace balance `{}` diff = 0
- Paren balance `()` diff = 0
- All 4 views present in source
- localStorage keys present
- Key feature strings present (guided workout, cardio, ISO week, etc.)
