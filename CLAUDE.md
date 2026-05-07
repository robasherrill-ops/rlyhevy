# RlyHevy — Workout Tracker

## What This App Does

RlyHevy is a mobile-first workout tracking PWA (progressive web app). It lets users:
- Log workouts in real time, tracking sets, reps, weight, and TUT (Time Under Tension)
- Build a library of exercises (with notes), workout templates, and training splits
- Log and review mobility exercises separately from strength training
- Track progress over time (volume, PRs, workout history)
- Run multi-week training splits with per-day template assignments

The UI is designed for a 420px-wide phone viewport with a dark neon aesthetic. Navigation is a fixed bottom bar with five tabs: **Home, Library, Log, Mobility, Progress**.

---

## File Structure

```
RLYHEVY/
├── index.html      # The entire app — HTML, CSS, and React JS in one file
├── CLAUDE.md       # This file
└── README.md       # Minimal placeholder
```

This is a single-file app. Everything — styles, React components, business logic, Supabase calls — lives in `index.html`. There is no build step, no bundler, no node_modules.

---

## GitHub

Remote: `https://github.com/robasherrill-ops/rlyhevy.git` (branch: `main`)

To push changes:
```bash
cd "/Users/robsherrill/Desktop/CLAUDE/Code Base/RLYHEVY"
git add index.html
git commit -m "..."
git push origin main
```

---

## How Supabase Is Connected

Supabase credentials are **hardcoded** at the top of the `<script>` block:

```js
const SB = "https://grubhtqgbgxagqsiakal.supabase.co"
const KEY = "eyJhbGci..."  // anon/public key
```

The `db` object is a thin hand-rolled client that wraps `fetch` against Supabase's REST and Auth APIs. It handles:
- `db.signIn / signUp / signOut / updateUser` — Supabase Auth v1 endpoints
- `db.get / post / patch / del` — Supabase REST (PostgREST) CRUD
- Session persistence via `localStorage` key `rlyhevy_session`, with refresh-token restore on load
- `db.uid` — the current user's ID (set after sign-in)

**No Supabase JS SDK is used** — all calls are raw `fetch` to the REST API.

### Supabase Tables

| Table | Purpose |
|-------|---------|
| `exercises` | Exercise library (name, muscle_group, is_bodyweight, **notes**) — mobility exercises use `muscle_group: 'mobility'` |
| `templates` | Workout templates (name, is_global, user_id) |
| `template_exercises` | Exercises within a template (sort_order, default_sets/reps, TUT config) |
| `workouts` | Individual workout sessions (started_at, finished_at, notes) |
| `workout_exercises` | Exercises within a workout session |
| `sets` | Individual sets (weight, reps, completed, tut_type, tut_tension, tut_duration) |
| `splits` | Multi-week training programs (user_id) |
| `split_slots` | Days within a split (week_number, **day_order**, template_id) — `day_order` is the reorderable position within a week |
| `split_slot_exercises` | Per-slot exercise overrides |
| `split_runs` | Active run of a split for a user (tracks progress through weeks) |

User profile data (name, body_weight, weight_unit) is stored in **Supabase Auth user_metadata**, not a separate table.

**Applied migrations:**
- `ALTER TABLE exercises ADD COLUMN IF NOT EXISTS notes text;`
- `ALTER TABLE sets ADD COLUMN IF NOT EXISTS set_type text NOT NULL DEFAULT 'working';`
- `ALTER TABLE workout_exercises ADD COLUMN IF NOT EXISTS superset_group integer;`

> `split_slots.day_order` already exists and is used for per-week slot ordering. No migration needed for reorder feature.

---

## How to Run Locally

**Recommended — use the Claude Preview server:**
The app is configured to serve from `/tmp/rlyhevy/`. When editing, sync the file:
```bash
cp "/Users/robsherrill/Desktop/CLAUDE/Code Base/RLYHEVY/index.html" /tmp/rlyhevy/index.html
```
Then use `preview_start` (port 8081) or `python3 -m http.server 8081` from `/tmp/rlyhevy/`.

**Alternative:**
```bash
cd "/Users/robsherrill/Desktop/CLAUDE/Code Base/RLYHEVY"
python3 -m http.server 8080
# open http://localhost:8080
```

React 18 and all dependencies load from **unpkg CDN** at runtime — no install needed.

---

## App Architecture & Key Components

All components use `React.createElement` (no JSX). The file is transpile-free — it runs as-is in the browser.

### Bottom Nav Tabs (in order)
| Tab | Screen key | Component | What it does |
|-----|-----------|-----------|-------------|
| Home | `dashboard` | `Dashboard` | Recent workouts, effort score, active split progress |
| Library | `library` | `Library` | Tabbed: Exercises, Templates, Splits |
| Log | `log` | `Log` → `WorkoutStarter` → `ActiveWorkout` | Start and log a live workout |
| Mobility | `mobility` | `Mobility` | Mobility-specific exercise library with notes |
| Progress | `progress` | `Progress` | Workout history list + volume/PR tracking |

> **Note:** There is no History tab. History was removed and its content merged into the Progress tab. The `History` component still exists in the file but is no longer used.

### Other notable components
- `AuthScreen` — Login / signup with tab toggle + "Stay logged in" toggle
- `Settings` — Name, body weight, unit (lbs/kg), **accent color picker**, **change password**
- `TemplateDetail` — Edit a template's exercises, fork, delete; **Bulk Update** button (isMine + ≥1 exercise) applies sets/reps/TUT to all exercises at once
- `SplitDetail` — Configure split days, assign templates; **any slot (with or without a template) can be tapped to add/edit exercises** via `SlotEditor`; **▲▼ reorder buttons** on each slot row when week has >1 day
- `SlotEditor` — Exercises for a specific split slot; receives `dayLabel` prop for template-less slots; rows rendered via top-level `SlotExRow` component
- `SlotExRow` — **Top-level component** (not nested inside SlotEditor) so React never remounts it; holds local sets/reps state and saves only on blur
- `SplitTemplatePicker` — Pick a template for a slot; **includes inline "＋ New Template" creation**
- `ActiveWorkout` — Core workout logging UI (set entry, TUT, bodyweight %)
- `ExercisePicker` — Full-screen exercise selector with muscle group filter; **includes "Mobility" filter chip** — mobility exercises (muscle_group='mobility') are fully addable to templates
- `TUTSelector` — Picker for exercise Type + Duration (2s/3s when type ≠ none)
- `ExForm` — Reusable form for adding/editing exercises (name, muscle group, bodyweight toggle, **notes textarea**)
- `Toggle` — Reusable on/off toggle component

### Feature details

**Notes field on exercises:**
- Both the `Exercises` (Library) component and the `Mobility` component support a `notes` text field
- Shown as a textarea in add/edit modals (placeholder: "Cues, setup tips, video links...")
- Stored in `exercises.notes` (nullable text column)
- Query pattern: `select=*,notes&...`

**Stay logged in (AuthScreen):**
- Toggle on the login tab (defaults to `true`)
- Persisted to `localStorage.rlyhevy_stay`
- If off: session token is NOT saved to localStorage (ephemeral session)

**Accent color (Settings):**
- `ACCENT_COLORS` constant: 10 preset colors
- Saved to `localStorage.rlyhevy_accent`
- Applied via `applyAccent(c)` helper — sets `--acc`, `--acc4`, `--acc8`, `--acc12`, `--acc20`, `--acc25`, `--acc40`, `--acc50` all at once so derived colors (glows, tints, SVG filters) update with the selection
- Restored on app load via `applyAccent(saved)` in App's first useEffect
- All Neon* SVG icons use `stroke: "currentColor"` with `color: var(--acc)` on the parent

**Set Types:**
- `SET_TYPES = ['working','warmup','drop','failure']`, labels: W / WU / D / F
- Stored in `sets.set_type` (default `'working'`)
- Displayed as a tappable pill in each set row — tap to cycle through types
- `cycleSetType(weId, setId, cur)` patches DB and updates local state

**Previous Performance Display:**
- `prevSets` state in `ActiveWorkout`, keyed by `exercise_id`
- Loaded once on mount: fetches last 15 finished workouts, finds most recent session per exercise, stores completed sets
- Displayed as a "Last: 185×5 · 185×4" row between exercise header and column headers

**Exercise Progress Charts:**
- `ExerciseDetail` component — full screen with SVG line chart + session history list
- Chart shows max weight per session over the last 12 sessions
- Access via the wave-icon (📈) button on each exercise row in Library
- `selectedExercise` state in `App` controls routing; back button clears it

**Calendar Heatmap (Progress screen):**
- Replaced 30-day bar chart with a 16-week × 7-day GitHub-style SVG heatmap
- Cell colors: `--br` (0), `--acc20` (1), `--acc40` (2), `--acc` (3+)
- Month labels auto-generated from data

**Superset Support:**
- `supersets` state in `ActiveWorkout`: `{ [workoutExerciseId]: groupNumber }`
- `linkingId` state triggers linking mode — tap "SS" on exercise A, then tap exercise B to link them
- Linked exercises get a colored left border + "SS N" badge, and a dashed connector between them
- `unlinkSuperset(weId)` removes one exercise from a group (auto-cleans solo members)
- Group colors cycle through a palette (blue, green, yellow, orange, purple)

**Change password (Settings):**
- Collapsible "Security" section
- Calls `db.updateUser({ password: newPw })`

**Exercise Type (formerly "Time Under Tension"):**
- Column header in ActiveWorkout and SlotEditor is **"Type"** (not "TUT")
- `TUT_TYPE_LABEL`: `{ eccentric_hold:'Eccentric Hold', concentric_hold:'Concentric Hold', rep_release:'Rep Release', both:'Both', none:'None' }`
- Type array used in all modals: `['eccentric_hold','concentric_hold','rep_release','both','none']`
- Modal title is "Type"; shows 5 type buttons + Duration (2s/3s) sub-section when type ≠ none
- Default `tut_duration` for new exercises is `2` (matches the 2s/3s button set)
- `tut_tension` column still exists in DB but is no longer exposed in UI
- `tutLabel(type)` — simplified to just `TUT_TYPE_LABEL[type] || '—'`
- **Old DB values** (concentric, eccentric, hold, multiple) will display as '—' — users need to re-set type on existing exercises

**Reps input (ActiveWorkout):**
- `onChange` → `updateRepsLocal(weId, setId, val)` — updates state only (allows free typing, empty string permitted)
- `onBlur` → `saveReps(weId, setId, val)` — clamps to ≥1 and saves to DB
- Matches the same pattern as the weight field (`updateWeightLocal` + `saveWeight`)

**SlotEditor sets/reps inputs:**
- Handled by top-level `SlotExRow` component with local `useState` for interim values
- `onChange` → updates local state only (no DB write, keyboard stays open)
- `onBlur` → clamps to ≥1 and calls `saveRow()` to write to DB
- `SlotExRow` is **defined outside SlotEditor** — this is intentional; nesting it caused React to remount the component (and blur inputs) on every parent re-render

**Bulk Update (TemplateDetail):**
- "Bulk Update" button appears when `isMine && exercises.length > 0`
- Modal has Sets stepper, Reps stepper, TUT type grid, TUT duration (when type ≠ none)
- On confirm: `Promise.all(exercises.map(te => db.patch('template_exercises', payload, ...)))` — all rows updated in one parallel batch

**Dashboard effort score:**
- Scoped strictly to the current user's workouts (sequential queries: workouts → workout_exercises → sets filtered by user's IDs)
- Formula: `weight × reps × tut_duration` summed across completed sets

**Split slot editing:**
- Any slot (rest day or template day) opens `SlotEditor` on tap
- Template-less slots show `dayLabel` ("Day N") as the header
- Slots with templates show "Swap" button; without show "Template" button

---

## UI Conventions

- CSS custom properties (`--acc`, `--bg`, `--s1`, etc.) control the entire color theme
- `--acc` defaults to `#ff2d55` (red) but is user-configurable via Settings
- Derived accent variables: `--acc4`, `--acc8`, `--acc12`, `--acc20`, `--acc25`, `--acc40`, `--acc50` — all set together by `applyAccent(hex)`
- Fonts: `Bebas Neue` for display/headers, `DM Sans` for body
- Max width 420px, centered, phone-sized layout; `.wrap` has `overflow-x: hidden` to prevent horizontal scroll
- Class naming is abbreviated: `.btn-p` = primary button, `.btn-s` = secondary, `.mo` = modal overlay, `.mo-box` = modal content, `.bnav` = bottom nav, `.li` = list item, `.igrp` = input group, `.ilbl` = input label

---

## Things to Know Before Making Changes

1. **Single file** — all edits happen in `index.html`. Use `grep -n` to find line numbers before editing.
2. **No JSX** — all React is written as `React.createElement(...)` calls. Never introduce JSX.
3. **No build** — changes are live on browser refresh. Sync to `/tmp/rlyhevy/index.html` for the preview server.
4. **Supabase RLS** — the app relies on Row Level Security. If queries return empty unexpectedly, check RLS rules in the Supabase dashboard first.
5. **Type (formerly TUT)** — appears as `tut_type`, `tut_tension`, `tut_duration` in DB/code. UI calls it "Type". Current values: `eccentric_hold`, `concentric_hold`, `rep_release`, `both`, `none`. Duration is 2s or 3s. Default tut_duration is `2`.
6. **Bodyweight exercises** — `is_bodyweight` exercises use a `%` of bodyweight (`BW_PCTS = [25, 50, 75, 100]`) instead of absolute weight.
7. **Global exercises/templates** — records with `is_global: true` are shared across all users. User-created ones are private (`is_global: false`, `created_by: db.uid`).
8. **Always scope queries to the current user** — use `user_id=eq.${db.uid}` or `created_by=eq.${db.uid}` to avoid leaking other users' data.
9. **Paren counting** — `React.createElement` calls nest deeply. Count opening and closing parens carefully when editing; a missing `)` causes a blank app with a syntax error.
10. **Session storage** — auth tokens live in `localStorage` under `rlyhevy_session` and are refreshed on app load.
