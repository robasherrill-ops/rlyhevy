# RlyHevy — Workout Tracker

## What This App Does

RlyHevy is a mobile-first workout tracking PWA (progressive web app). It lets users:
- Log workouts in real time, tracking sets, reps, weight, and exercise type (TUT)
- Build a library of exercises (with notes), workout templates, and training splits
- Log and review mobility exercises separately from strength training
- Track progress over time (volume, PRs, workout history)
- Run multi-week training splits with per-day template assignments

The UI is designed for a 420px-wide phone viewport with a dark matte aesthetic. Navigation is a fixed bottom bar with five tabs: **Home, Library, Log, Move, Stats**.

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
| `exercises` | Exercise library (name, muscle_group, is_bodyweight, notes) — mobility exercises use `muscle_group: 'mobility'` |
| `templates` | Workout templates (name, is_global, user_id) |
| `template_exercises` | Exercises within a template (sort_order, default_sets/reps, TUT config) |
| `workouts` | Individual workout sessions (started_at, finished_at, notes) |
| `workout_exercises` | Exercises within a workout session |
| `sets` | Individual sets (weight, reps, completed, set_type, tut_type, tut_tension, tut_duration) |
| `splits` | Multi-week training programs (user_id) |
| `split_slots` | Days within a split (week_number, day_order, template_id) — `day_order` is the reorderable position within a week |
| `split_slot_exercises` | Per-slot exercise overrides — **writable override layer** (never patch template_exercises directly) |
| `split_runs` | Active run of a split for a user (tracks progress through weeks) |

User profile data (name, body_weight, weight_unit) is stored in **Supabase Auth user_metadata**, not a separate table.

**Applied migrations:**
- `ALTER TABLE exercises ADD COLUMN IF NOT EXISTS notes text;`
- `ALTER TABLE sets ADD COLUMN IF NOT EXISTS set_type text NOT NULL DEFAULT 'working';`
- `ALTER TABLE workout_exercises ADD COLUMN IF NOT EXISTS superset_group integer;`
- `ALTER TABLE sets ALTER COLUMN tut_type TYPE text USING tut_type::text;`
- `ALTER TABLE sets ALTER COLUMN tut_tension TYPE text USING tut_tension::text;`
- `ALTER TABLE template_exercises ALTER COLUMN tut_type TYPE text USING tut_type::text;`
- `ALTER TABLE template_exercises ALTER COLUMN tut_tension TYPE text USING tut_tension::text;`
- `ALTER TABLE split_slot_exercises ALTER COLUMN tut_type TYPE text USING tut_type::text;`
- `ALTER TABLE split_slot_exercises ALTER COLUMN tut_tension TYPE text USING tut_tension::text;`
- `ALTER TABLE exercises ADD COLUMN IF NOT EXISTS default_bw_percent integer;`

> `split_slots.day_order` already exists and is used for per-week slot ordering.

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
| Home | `dashboard` | `Dashboard` | Effort score card, split dial, recent sessions |
| Library | `library` | `Library` | Tabbed: Exercises, Templates, Splits |
| Log | `log` | `Log` → `WorkoutStarter` → `ActiveWorkout` | Start and log a live workout |
| Move | `mobility` | `Mobility` | Mobility-specific exercise library with notes |
| Stats | `progress` | `Progress` | 16-week heatmap + exercise progress charts |

> **Note:** There is no History tab. The `History` component still exists in the file but is no longer rendered.

### Key Components
- `GuideSection` — Collapsible feature guide rendered at the bottom of Settings (6 categories)
- `AuthScreen` — Login / signup with tab toggle + "Stay logged in" toggle
- `Settings` — Name, body weight, unit (lbs/kg), accent color picker, change password, Guide & Features
- `Dashboard` — Home screen: effort 7D card, tactile split dial, recent sessions list
- `TemplateDetail` — Edit a template's exercises, fork, delete; **Bulk Update** applies sets/reps/type to all at once
- `SplitDetail` — Configure split days, assign templates; any slot opens `SlotEditor`; ▲▼ reorder buttons
- `SlotEditor` — Exercises for a specific split slot; uses override layer (`split_slot_exercises`)
- `SlotExRow` — **Top-level component** (not nested in SlotEditor) to prevent remount on parent re-render
- `SplitTemplatePicker` — Pick a template for a slot; includes inline "＋ New Template" creation
- `ActiveWorkout` — Core workout logging UI (sets, reps, weight, type, supersets)
- `ExercisePicker` — Full-screen exercise selector with muscle group filter; includes Mobility filter chip
- `TUTSelector` — Picker for exercise Type + Duration (2s/3s when type ≠ none)
- `Toggle` — Reusable on/off toggle component
- `ExerciseDetail` — Full-screen SVG progress chart + session history for a single exercise

---

## Feature Details

**Exercises (Library tab):**
- Tap any exercise row (including global/Default ones) to open an edit modal
- Edit: name, muscle group, bodyweight toggle, BW% preview, notes
- Delete uses 2-step inline confirmation (no `window.confirm`)
- **Select mode**: "Select" button in header enables checkboxes on all rows → "Delete N" with confirmation
- No hide/unhide system — exercises are either visible or deleted
- The "Default" badge is NOT shown on exercise rows (it still appears in template headers)
- Mobility exercises (`muscle_group: 'mobility'`) live in their own Move tab but can be added to templates via ExercisePicker

**Mobility (Move tab):**
- Separate library for stretches/flows/drills
- Same edit/delete pattern as Exercises (2-step confirm, no `window.confirm`)
- Supports bodyweight toggle with BW% preview and notes
- Query: `select=*,notes&muscle_group=eq.mobility&order=name.asc`

**Split slot override layer:**
- `split_slot_exercises` is the writable override table — NEVER patch `template_exercises` directly (RLS blocks non-owners)
- Load: fetch `template_exercises` as baseline + `split_slot_exercises` as overrides, merge by `exercise_id`
- Write: always post/patch `split_slot_exercises`; on first save for a row, POST to get an id, then PATCH thereafter

**Accent color system:**
- `ACCENT_COLORS`: 10 matte pastel swatches + `'multi'`
- Default: `#c47a88` (dusty rose)
- Saved to `localStorage.rlyhevy_accent`; migration clears old gold (`#c9a36a`) and old neon red (`#ff375f`) on load
- `applyAccent(c)` sets `--acc` and all derived vars (`--acc4` through `--acc50`)
- **Single color mode**: `--push` (lighter), `--pull` (accent), `--legs` (darker) all derived from accent hue via `hslToHex`
- **Multi mode**: `--push:#d4956a`, `--pull:#7ab0d4`, `--legs:#9abb6a` (distinct muted pastels); each muscle group gets its own color from `MULTI_MG_COLORS`
- `GLOW` object: `{ color: 'var(--acc)' }` — drop-shadow filter was removed; icon color only

**Color palette (neutral dark surfaces — no warm tint):**
- `--bg: #000000` — OLED canvas
- `--s1: #0d0d0d` — surface 1 (cards, inputs)
- `--s2: #141414` — surface 2 (nested surfaces)
- `--br: #1e1e1e` — border
- `--divider: #111111` — hairline dividers
- `--tx: #f3ece0` — primary text
- `--tm: #5a5550` — muted text
- `--ts: #8a847b` — subtle text
- Card/nav/modal gradients: `linear-gradient(180deg, #141414 0%, #0d0d0d 100%)`
- All warm brown values (`#18140f`, `#0e0c08`, `#221d16`, `#14110d`, `rgba(255,236,200,...)`) have been fully removed

**Set Types:**
- `SET_TYPES = ['working','warmup','drop','failure']`, labels: W / WU / D / F
- Stored in `sets.set_type` (default `'working'`)
- Tappable pill in each set row — cycles through types

**Exercise Type (formerly TUT):**
- Column header is **"Type"** (not "TUT")
- `TUT_TYPE_LABEL`: `{ eccentric_hold, concentric_hold, rep_release, both, none }`
- DB columns `tut_type` / `tut_tension` are now `text` type (migrated from enum)
- Default `tut_duration`: `2`; options are 2s or 3s
- Old DB enum values display as `'—'` — users re-set type on existing exercises

**Effort score (Dashboard):**
- Scoped to current user's workouts only
- Formula: `weight × reps × tut_duration` summed across completed sets
- Displayed as 7-day total with week-over-week % delta

**Previous performance (ActiveWorkout):**
- `prevSets` keyed by `exercise_id`; loaded on mount from last 15 finished workouts
- Shown as "Last: 185×5 · 185×4" above column headers

**Supersets (ActiveWorkout):**
- Tap SS on exercise A → tap exercise B to link; colored left border + "SS N" badge
- `unlinkSuperset(weId)` auto-cleans solo members
- Group colors cycle: blue, green, yellow, orange, purple

**Bodyweight exercises:**
- `is_bodyweight: true` → weight input shows % options (25/50/75/100% of `user.body_weight`)
- BW% preview shown in exercise edit modal when body weight is set

**Calendar heatmap (Stats tab):**
- 16-week × 7-day GitHub-style SVG; cell colors: `--br` (0 workouts), `--acc20` (1), `--acc40` (2), `--acc` (3+)

**Bulk Update (TemplateDetail):**
- Appears when `isMine && exercises.length > 0`
- Applies sets/reps/type/duration to all exercises in one parallel batch

**SlotEditor inputs:**
- `onChange` → local state only (keyboard stays open)
- `onBlur` → clamps to ≥1 and writes to DB
- `SlotExRow` defined at top level — intentional, prevents remount blur

**Guide & Features section (Settings):**
- `GuideSection` component rendered after the Security section
- 6 collapsible categories: Getting Started, Logging a Workout, Library, Mobility, Stats & Progress, Appearance
- Item labels use `var(--acc)` color; descriptions use `var(--ts)`

---

## UI Conventions

- CSS custom properties control the entire color theme — never hardcode colors in new code
- Default accent: `#c47a88` (dusty rose matte pastel)
- All accent-derived vars set together by `applyAccent(hex)`: `--acc`, `--acc4`, `--acc8`, `--acc12`, `--acc20`, `--acc25`, `--acc40`, `--acc50`
- No neon glows anywhere — `.btn-p`, `.auth-logo`, `.stat-v` have no `box-shadow`/`text-shadow` glow
- Font: `Geist` (fallback `system-ui, sans-serif`) — used throughout; `Bebas Neue` for legacy display elements
- Max width 420px, centered; `.wrap` has `overflow-x: hidden`
- Class naming is abbreviated: `.btn-p` = primary button, `.btn-s` = secondary, `.btn-g` = ghost, `.btn-d` = danger, `.mo` / `.mo-box` = modal, `.bnav` = bottom nav, `.li` = list item, `.igrp` / `.ilbl` = input group/label, `.srch` = search bar, `.fscroll` / `.fchip` = filter chip row

---

## Things to Know Before Making Changes

1. **Single file** — all edits happen in `index.html`. Use `grep -n` to find line numbers before editing.
2. **No JSX** — all React is `React.createElement(...)`. Never introduce JSX.
3. **No build** — changes are live on refresh. Sync to `/tmp/rlyhevy/index.html` for the preview server.
4. **Supabase RLS** — if queries return empty unexpectedly, check RLS rules in the Supabase dashboard first.
5. **Split saves go to `split_slot_exercises`** — never patch `template_exercises` for split slot overrides; RLS blocks non-owners silently.
6. **tut_type is now `text`** — not an enum. Accepted values: `eccentric_hold`, `concentric_hold`, `rep_release`, `both`, `none`.
7. **Bodyweight exercises** — `is_bodyweight: true` uses `%` of `user.body_weight`. Percentages: 25/50/75/100.
8. **Global exercises/templates** — `is_global: true` records are shared; `created_by: db.uid` for user-owned. Both are now fully editable/deletable from the UI.
9. **Paren counting** — `React.createElement` nests deeply. A missing `)` causes a blank screen with a JS syntax error.
10. **Accent color migration** — on app load, old gold (`#c9a36a`) and old neon red (`#ff375f`) are auto-replaced with the new default `#c47a88`.
11. **No warm tints** — surfaces use neutral dark grads. Do not reintroduce `#18140f`, `#0e0c08`, or `rgba(255,236,200,...)`.
12. **No `window.confirm`** — all destructive actions use 2-step inline confirmation state instead.
