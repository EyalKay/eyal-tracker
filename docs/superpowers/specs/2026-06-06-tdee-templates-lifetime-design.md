# Tracker Options: TDEE, Meal Templates, Lifetime Stats — Design Spec

**Date:** 2026-06-06
**App:** Eyal Tracker (`index.html`, single-file vanilla JS + Firebase Firestore)
**Status:** Approved for implementation

## Motivation

After 30 days of use, three friction points emerged:

1. **Old data is invisible.** Stats page caps at 30 days. The user can no longer see longer-term trends or all-time progress.
2. **Daily logging feels like a chore.** Tapping `+`/`−` per macro per meal adds up. Sustained adherence drops as friction rises.
3. **The 1960 kcal target is the dietitian's generic plan, not personalized.** The user has no way to check whether the chosen 5P/3C/2F target produces a realistic deficit for *their* body.

This spec adds three features in one bundle: lifetime data view, editable meal templates for one-tap logging, and a TDEE calculator with deficit readout.

## Scope

### In scope
- Lifetime data range toggle on Stats page (30d / 90d / All)
- All-time summary card on Stats page
- Editable meal templates (add / edit / delete from Settings; one-tap apply from Today)
- TDEE calculator card in Settings, using Mifflin–St Jeor + activity multiplier
- New Settings page accessed via a gear icon in the header
- Config doc schema extension (additive — does not break existing fields)

### Explicitly out of scope
- Reminders / push notifications
- Meal photos or image attachments
- Trendline overlays on the weight chart
- Editing the hardcoded macro targets (`TARGETS = {p:5, c:3, f:2}`) or CAL constants
- Sharing templates across users
- Importing food databases (USDA, etc.)
- Macro suggestions based on TDEE (TDEE is read-only advice; it does not change targets automatically)

## Feature 1 — Lifetime Stats

### Range toggle
Three buttons at the top of the Stats page: **30d** (default) / **90d** / **All**. Clicking re-runs the existing stats computations against the selected range.

### Affected computations on Stats page
All of the following recompute when range changes:
- Weakest trigger
- Average steps
- Weight trend (first → last in range)
- Best streak in range
- Weight chart points

### All-time summary card
A new card pinned to the top of the Stats page, **always shown regardless of range**:
- **Days logged** (count of date docs in Firestore with any tracked data)
- **Lifetime weight Δ** (earliest weight → most recent weight, kg, with sign)
- **Best streak ever** (longest run of consecutive days hitting all 5P/3C/2F)

### Chart behavior for "All"
Reuse the existing weight chart component. Smaller dot radius when range > 90 days. Y-axis auto-scales. No aggregation, no smoothing — just plot every weight.

### Data
No schema change. Stats fetches all date docs already (just filters to 30 days); broaden the fetch and filter.

## Feature 2 — Meal Templates

### Goal
Make logging a repeating meal take a single tap instead of 4–6 `+` taps.

### Apply (Today page)
A horizontal scrollable chip row sits **directly under the macro card** on Today. Each chip shows the template name. Tapping a chip **adds** its `p`/`c`/`f` portion values to today's running totals (additive, half-portion safe). The save flow is the same as `+`/`−` taps.

If the user has zero templates defined, the chip row is replaced by a small placeholder: "No meal templates yet — add some in Settings →" (clickable, navigates to Settings).

### Manage (Settings page)
A "Meal Templates" section listing all templates with inline edit, plus an "Add template" button. Each template has:
- **Name** (string, required, max 30 chars)
- **P portions** (0–10 in 0.5 increments)
- **C portions** (0–10 in 0.5 increments)
- **F portions** (0–10 in 0.5 increments)

Edit and delete are inline per row. No reorder in v1 (display order = creation order).

### Data shape
```js
templates: [
  { id: "tpl_1717689412345", name: "Usual breakfast", p: 1, c: 1, f: 0 },
  { id: "tpl_1717689500111", name: "Coffee + oats",   p: 0, c: 1, f: 0 },
  ...
]
```
- `id` is a stable string generated at creation (`'tpl_' + Date.now()`)
- Stored on `eyal/config`, alongside existing `habits`, `goal`, `startWeight`

### Edge cases
- Tapping a template when today already has 5/3/2 logged: portions still add (no cap). User can `−` if they overshot.
- Deleting a template while it's mid-tap: race acceptable — the add still applies because the apply reads the chip's data at click time.

## Feature 3 — TDEE & Goal Check

### Inputs (Settings page)
A "TDEE & Goal" card with:
- **Height** (cm, integer, 100–250)
- **Age** (years, integer, 10–100)
- **Sex** (`M` / `F` toggle)
- **Activity level** (dropdown):
  - Sedentary (× 1.2)
  - Light (× 1.375)
  - Moderate (× 1.55)
  - Active (× 1.725)
  - Very active (× 1.9)

### Formula
**Mifflin–St Jeor BMR:**
- Male: `BMR = 10·weight + 6.25·height − 5·age + 5`
- Female: `BMR = 10·weight + 6.25·height − 5·age − 161`

**TDEE:** `BMR × activity multiplier`

**Weight source:** Most recent logged weight in Firestore (search backward from today until a weight is found). If no weight ever logged, the card shows inputs but the readout displays "Log a weight first to see TDEE."

### Output card (auto-recalculates on any input change)
Shown immediately below the inputs:
- **BMR:** XXXX kcal
- **TDEE:** XXXX kcal
- **Deficit vs 1960 target:** ±XXX kcal/day
- **Projected:** ±X.X kg/week (deficit × 7 ÷ 7700)

If deficit is negative (target above TDEE), label it as **"Surplus"** and project weight gain. If TDEE − target > 1000, show an amber warning: "Deficit > 1000 kcal/day is aggressive — consider reviewing with your dietitian."

### Data shape
```js
tdee: {
  height: 175,    // cm
  age: 38,
  sex: "M",       // "M" | "F"
  activity: "moderate"  // "sedentary"|"light"|"moderate"|"active"|"very"
}
```

Stored on `eyal/config` alongside existing fields. Absence means TDEE has never been configured — show inputs in empty state, no readout.

## Navigation & Pages

### New: gear icon in header
A small gear icon is added to the **right side of the page header** on the Today page (where the sync dot currently lives — gear sits next to it). Tapping opens the Settings page.

### New: Settings page
- Reachable via the gear icon. Not in the bottom nav (would crowd it).
- Header: "Settings" title, back arrow → Today.
- Sections (in order):
  1. **TDEE & Goal** (Feature 3)
  2. **Meal Templates** (Feature 2)

### No other navigation changes
Today / Week / Stats / Reset cards remain as-is.

## Data model — Firestore `eyal/config`

**Before:**
```js
{ habits: [...], goal: 80, startWeight: 87 }
```

**After:**
```js
{
  habits: [...],
  goal: 80,
  startWeight: 87,
  templates: [ { id, name, p, c, f }, ... ],   // new, default []
  tdee: { height, age, sex, activity }         // new, optional
}
```

### Critical: merge, do not overwrite
The current `saveConfig()` at `index.html:606` calls `setDoc()` with `{habits, goal, startWeight}`. This will **wipe `templates` and `tdee`** as soon as any existing write fires after the new fields exist. **All config mutations must go through a single `saveConfig()` that writes the full in-memory config object** (habits, goal, startWeight, templates, tdee). No partial config writes anywhere in the codebase. Adding templates / editing TDEE updates in-memory state, then calls the same `saveConfig()`.

## UI / Style notes

- Match existing card style (`var(--card)` background, 16px radius, 1px border `var(--border)`)
- Chip row uses pill-shaped buttons, `var(--card2)` background, white text, horizontal scroll-snap
- Settings inputs use the same number-stepper pattern as macros and weight (`+` / `−` buttons) where appropriate, native `<input type=number>` where stepping doesn't fit (height, age)
- Sex toggle: two-button segmented control, P-green active state
- Activity dropdown: native `<select>` (consistent with other dropdowns in app, if any) styled to match cards

## Error handling

- All Firestore writes already wrap in `try/catch` and set sync-dot state — keep that pattern.
- TDEE inputs validate client-side (range bounds above); invalid values disable the readout and show "Enter valid values" in the output card.
- Template name empty/whitespace → "Add" button disabled.

## Testing

Manual smoke test after implementation (no automated tests in this repo):
- [ ] Add 3 templates, verify chips appear, tap each, verify macros update and save
- [ ] Edit a template's portions, verify chip behavior updates
- [ ] Delete a template, verify it disappears immediately
- [ ] Sign out / sign in — templates persist
- [ ] Set TDEE inputs, verify BMR/TDEE/deficit/projection update live
- [ ] Change activity level, verify TDEE recomputes
- [ ] Log a new weight, return to Settings, verify TDEE uses new weight
- [ ] Stats page: toggle 30 / 90 / All — every stat and the chart updates
- [ ] All-time card shows correct days-logged and lifetime Δ
- [ ] Hard refresh — all new data round-trips through Firestore correctly
- [ ] Confirm `habits`, `goal`, `startWeight` are NOT wiped after saving templates or TDEE (config merge correctness)

## Risk

The single biggest risk is the config-merge bug: if any new write path overwrites the config doc with a partial, existing user data (habits, goal, startWeight) gets wiped. Mitigation: route all config writes through one function that always writes the full object.
