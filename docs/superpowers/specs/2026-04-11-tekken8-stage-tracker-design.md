# Tekken 8 Stage Win Rate Tracker — Design Spec
*Date: 2026-04-11*

## Context

The user wants a personal tool to track their Tekken 8 stage win rates across seasons and play sessions. The goal is to identify strong/weak stages, monitor performance trends over time, and keep a longitudinal record of improvement. The app must be deployable to GitHub Pages with zero infrastructure — a single `index.html` file using only vanilla HTML, CSS, and JavaScript.

---

## Architecture

**Single file:** `index.html` — all markup, styles, and logic inline. No build step, no dependencies, no external CDN calls.

**Hosting:** GitHub Pages (static, no backend).

**Data layers (two sources merged at runtime):**

1. **Seed data** — hardcoded JS constant inside `<script>`. This is the authoritative baseline per season. Updated by editing the file and pushing a commit when new season data is available.

```js
const SEED_DATA = {
  S3: {
    "Arena": 100, "Baobab Horizon": 100, "Secluded Training Ground": 71,
    "Seaside Resort": 71, "Phoenix Gate": 71, "Descent into Subconscious": 66,
    "Coliseum of Fate": 60, "Yakushima": 58, "Midnight Siege": 56,
    "Arena Underground": 56, "Pac-Pixels": 55, "Fallen Destiny": 50,
    "Ortiz Farm": 50, "Elegant Palace": 50, "Genmaji Temple Daytime": 50,
    "Into the Stratosphere": 44, "Celebration on the Seine": 44,
    "Sanctum": 42, "Urban Square Evening": 40, "Urban Square": 40,
    "Rebel Hangar": 37, "Genmaji Temple": 33
  }
  // Future seasons added here
}
```

2. **localStorage** — stores a flat array of individual session entries under the key `tekken_entries`. Each entry:

```js
{ season: "S3", stage: "Arena", winRate: 88, date: "2026-04-11" }
```

**Merge logic (on load and after every save):**
- Start with seed data as baseline for each season.
- Apply localStorage entries: for each stage, the entry with the most recent date wins.
- "Current state" per season = merged result.

**Session definition:** All entries sharing the same calendar date (`YYYY-MM-DD`) for a given season form one implicit session. No explicit session creation step required.

---

## Data Computations

**Current state** for season X: seed baseline overridden by the most recent localStorage entry per stage.

**Overall win rate:** Mean of all stage win rates in current state for the active season.

**Best / worst stage:** Stage with highest / lowest win rate in current state.

**Delta — vs last season:**
- Find the previous season key (e.g. S2 for active S3).
- Current state of active season minus current state of previous season, per stage.
- `—` if the stage doesn't exist in the previous season.

**Delta — vs last session:**
- Find the most recent calendar date in localStorage that has entries for the active season and is before today (or before the latest date if viewing historical data).
- Current state minus that session's values per stage.
- `—` if the stage has no prior session entry.

**Trending up / down:** A stage is "trending up" if its delta (under the current toggle mode) is > 0; "trending down" if < 0. Stages with no delta (`—`) are excluded from both trend tabs.

---

## UI Layout

### Season Bar
- Pill buttons, one per season key found in seed data (S1, S2, S3…).
- Active season highlighted.
- `+ Add Stage Data` button (right-aligned) — scrolls to / focuses the entry form.

### Summary Cards (3-column row)
| Card | Value | Color |
|------|-------|-------|
| Overall win rate | Mean win rate % | White |
| Best stage | Stage name + (win rate%) | Green |
| Worst stage | Stage name + (win rate%) | Red/salmon |

### Chart Section

**Tab row:** `All stages` · `Trending up` · `Trending down`

**Filter row:** `All` · `Strong (70%+)` · `Weak (sub 45%)`

**Column header:** `Stage — S3 win rate — vs [last season | last session]`
- The `last season / last session` portion is a clickable inline toggle.

**Stage rows** (sorted high → low by win rate, filtered by active tab + filter):
- Stage name (left)
- Horizontal bar (proportional width, full container)
- Win rate % (right-aligned)
- Delta (right-aligned, colored)

**Bar / delta color coding:**
| Condition | Bar color | Delta color |
|-----------|-----------|-------------|
| ≥ 70% | Green `#4caf50` | — |
| 45–69% | Amber `#c8960c` | — |
| < 45% | Red/salmon `#e05a5a` | — |
| Delta positive | — | Green |
| Delta negative | — | Red/salmon |
| No comparison data | — | `—` (muted) |

### Entry Form

Section header: `Add / update data`

Fields:
- **Season** — dropdown populated from seed data keys
- **Stage** — dropdown populated from stage list for selected season
- **Win %** — number input (0–100)
- **Save** button — appends entry to `tekken_entries` in localStorage, triggers full re-render. Date is auto-set to today's date (`YYYY-MM-DD`) at save time — no date field in the form.

A confirmation toast ("Saved!") appears briefly after a successful save.

---

## State Management

No framework. All UI state lives in a single plain JS object:

```js
const state = {
  activeSeason: "S3",
  activeTab: "all",       // "all" | "trending-up" | "trending-down"
  activeFilter: "all",    // "all" | "strong" | "weak"
  deltaMode: "season",    // "season" | "session"
}
```

Every user interaction updates `state` and calls a single `render()` function that rebuilds the chart section. Summary cards and season bar also re-render on season change.

---

## Verification

1. Open `index.html` in a browser (local file or GitHub Pages URL).
2. Season S3 tab is active; 22 stages render with correct win rates and colors.
3. Best stage card shows "Arena (100%)" in green; worst shows "Genmaji Temple (33%)" in red.
4. Overall win rate card shows ~57%.
5. Toggle delta mode — column header label changes; delta values update.
6. Click "Trending up" tab — only stages with positive delta visible.
7. Click "Strong (70%+)" filter — only green-bar stages visible.
8. Enter a new win rate via the form (e.g. Arena → 80%) and save — bar updates immediately, delta reflects the change.
9. Reload page — manually entered value persists (localStorage).
10. Verify seed data is still correct baseline for stages not manually overridden.
