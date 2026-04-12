# Tekken 8 Stage Win Rate Tracker — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single `index.html` static web app for GitHub Pages that tracks Tekken 8 stage win rates across seasons and sessions with bar charts, trend filtering, and localStorage persistence.

**Architecture:** All HTML, CSS, and JavaScript lives in one `index.html` file with no dependencies. Seed data is a hardcoded JS constant; manual entries are persisted to localStorage as timestamped records. The app merges both sources at runtime — localStorage entries (most recent per stage) override seed baselines. A single `render()` call rebuilds all dynamic UI from a central `state` object. Stage names in innerHTML are escaped via `escapeHtml()` to prevent XSS from tampered localStorage data.

**Tech Stack:** Vanilla HTML5, CSS3, JavaScript (ES6+). No frameworks, no build tools, no CDN.

**Spec:** `docs/superpowers/specs/2026-04-11-tekken8-stage-tracker-design.md`

---

## Chunk 1: Foundation — HTML skeleton, CSS, seed data, data layer

### Task 1: Create index.html with HTML skeleton and complete CSS

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create index.html with full page structure**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Tekken 8 Stage Tracker</title>
  <style>
    /* ── Reset & base ── */
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      background: #141414;
      color: #e0e0e0;
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      font-size: 14px;
      min-height: 100vh;
    }

    /* ── Layout ── */
    .container { max-width: 860px; margin: 0 auto; padding: 24px 20px 60px; }

    /* ── Season bar ── */
    .season-bar {
      display: flex;
      align-items: center;
      justify-content: space-between;
      margin-bottom: 24px;
    }
    .season-pills { display: flex; gap: 8px; flex-wrap: wrap; }
    .season-btn {
      background: transparent;
      border: 1px solid #444;
      color: #bbb;
      border-radius: 20px;
      padding: 6px 18px;
      cursor: pointer;
      font-size: 13px;
      transition: background 0.15s, color 0.15s, border-color 0.15s;
    }
    .season-btn:hover { border-color: #888; color: #eee; }
    .season-btn.active { background: #2e2e2e; border-color: #888; color: #fff; }
    .add-data-btn {
      background: transparent;
      border: 1px solid #444;
      color: #bbb;
      border-radius: 8px;
      padding: 6px 14px;
      cursor: pointer;
      font-size: 13px;
      white-space: nowrap;
      transition: border-color 0.15s, color 0.15s;
    }
    .add-data-btn:hover { border-color: #888; color: #eee; }

    /* ── Summary cards ── */
    .summary-cards {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      gap: 12px;
      margin-bottom: 24px;
    }
    .card {
      background: #1e1e1e;
      border: 1px solid #2a2a2a;
      border-radius: 10px;
      padding: 18px 20px;
    }
    .card-value {
      font-size: 28px;
      font-weight: 700;
      line-height: 1.1;
      margin-bottom: 4px;
    }
    .card-label { font-size: 12px; color: #777; }
    .card-value.green { color: #4caf50; }
    .card-value.red   { color: #e05a5a; }
    .card-value.white { color: #fff; }

    /* ── Chart panel ── */
    .chart-panel {
      background: #1e1e1e;
      border: 1px solid #2a2a2a;
      border-radius: 10px;
      padding: 0 0 8px;
      margin-bottom: 24px;
    }

    /* Tabs */
    .tabs {
      display: flex;
      border-bottom: 1px solid #2a2a2a;
      padding: 0 20px;
    }
    .tab-btn {
      background: transparent;
      border: none;
      color: #777;
      padding: 14px 14px 12px;
      cursor: pointer;
      font-size: 13px;
      border-bottom: 2px solid transparent;
      margin-bottom: -1px;
      transition: color 0.15s;
    }
    .tab-btn:hover { color: #ccc; }
    .tab-btn.active { color: #fff; border-bottom-color: #fff; }

    /* Filters */
    .filters { display: flex; gap: 8px; padding: 12px 20px 0; }
    .filter-btn {
      background: transparent;
      border: 1px solid #333;
      color: #888;
      border-radius: 6px;
      padding: 4px 12px;
      cursor: pointer;
      font-size: 12px;
      transition: background 0.15s, color 0.15s, border-color 0.15s;
    }
    .filter-btn:hover { border-color: #666; color: #ccc; }
    .filter-btn.active { background: #2e2e2e; border-color: #888; color: #fff; }

    /* Column header */
    .chart-header {
      display: flex;
      align-items: center;
      padding: 10px 20px 6px;
      font-size: 11px;
      color: #555;
      gap: 4px;
    }
    .delta-toggle {
      background: transparent;
      border: none;
      color: #666;
      cursor: pointer;
      font-size: 11px;
      padding: 0 2px;
      text-decoration: underline;
      transition: color 0.15s;
    }
    .delta-toggle:hover { color: #aaa; }
    .delta-toggle.active { color: #ccc; }

    /* Stage rows */
    .stage-list { padding: 4px 20px 8px; }
    .stage-row {
      display: grid;
      grid-template-columns: 200px 1fr 52px 52px;
      align-items: center;
      gap: 12px;
      padding: 5px 0;
    }
    .stage-name { font-size: 13px; color: #ccc; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
    .bar-track { background: #2a2a2a; border-radius: 3px; height: 8px; overflow: hidden; }
    .bar-fill { height: 100%; border-radius: 3px; transition: width 0.3s ease; }
    .bar-fill.green { background: #4caf50; }
    .bar-fill.amber { background: #c8960c; }
    .bar-fill.red   { background: #e05a5a; }
    .win-rate { font-size: 13px; color: #ccc; text-align: right; }
    .delta { font-size: 12px; text-align: right; white-space: nowrap; }
    .delta.pos  { color: #4caf50; }
    .delta.neg  { color: #e05a5a; }
    .delta.none { color: #444; }
    .no-stages  { padding: 20px; color: #555; text-align: center; font-size: 13px; }

    /* ── Entry form ── */
    .entry-panel {
      background: #1e1e1e;
      border: 1px solid #2a2a2a;
      border-radius: 10px;
      padding: 20px;
    }
    .entry-panel h3 { font-size: 12px; color: #666; text-transform: uppercase; letter-spacing: 0.05em; margin-bottom: 14px; }
    .entry-form { display: flex; gap: 10px; align-items: flex-end; flex-wrap: wrap; }
    .entry-form select,
    .entry-form input[type="number"] {
      background: #141414;
      border: 1px solid #333;
      color: #ccc;
      border-radius: 6px;
      padding: 8px 10px;
      font-size: 13px;
      outline: none;
      transition: border-color 0.15s;
    }
    .entry-form select:focus,
    .entry-form input[type="number"]:focus { border-color: #666; }
    #form-season  { width: 80px; }
    #form-stage   { flex: 1; min-width: 180px; }
    #form-winrate { width: 80px; }
    .save-btn {
      background: #2e2e2e;
      border: 1px solid #555;
      color: #ddd;
      border-radius: 6px;
      padding: 8px 20px;
      cursor: pointer;
      font-size: 13px;
      transition: background 0.15s, border-color 0.15s;
    }
    .save-btn:hover { background: #3a3a3a; border-color: #888; }

    /* ── Toast ── */
    .toast {
      position: fixed;
      bottom: 24px;
      left: 50%;
      transform: translateX(-50%) translateY(20px);
      background: #333;
      color: #fff;
      padding: 10px 22px;
      border-radius: 8px;
      font-size: 13px;
      opacity: 0;
      pointer-events: none;
      transition: opacity 0.2s, transform 0.2s;
    }
    .toast.show { opacity: 1; transform: translateX(-50%) translateY(0); }
  </style>
</head>
<body>
  <div class="container">

    <!-- Season bar -->
    <div class="season-bar">
      <div class="season-pills" id="season-pills"></div>
      <button class="add-data-btn" id="add-data-btn">+ Add Stage Data</button>
    </div>

    <!-- Summary cards -->
    <div class="summary-cards" id="summary-cards"></div>

    <!-- Chart panel -->
    <div class="chart-panel">
      <div class="tabs" id="tabs"></div>
      <div class="filters" id="filters"></div>
      <div class="chart-header" id="chart-header"></div>
      <div class="stage-list" id="stage-list"></div>
    </div>

    <!-- Entry form -->
    <div class="entry-panel" id="entry-panel">
      <h3>Add / update data</h3>
      <div class="entry-form">
        <select id="form-season"></select>
        <select id="form-stage"></select>
        <input type="number" id="form-winrate" min="0" max="100" placeholder="Win %" />
        <button class="save-btn" id="save-btn">Save</button>
      </div>
    </div>

  </div>
  <div class="toast" id="toast">Saved!</div>

  <script>
    // JS goes here in subsequent tasks
  </script>
</body>
</html>
```

- [ ] **Step 2: Open index.html in a browser — verify dark background and layout structure with no console errors**

Open `index.html` via `file://` URL. Expected: dark page, empty season bar area, empty card slots, empty chart panel, entry form with unstyled dropdowns. Zero console errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add HTML skeleton and dark theme CSS"
```

---

### Task 2: Seed data, XSS helper, and data layer pure functions

**Files:**
- Modify: `index.html` — replace `// JS goes here in subsequent tasks`

- [ ] **Step 1: Add escapeHtml(), seed data, localStorage helpers, and data layer functions**

Replace `// JS goes here in subsequent tasks` with:

```js
// ── XSS guard ───────────────────────────────────────────────────────────────
// Use escapeHtml() on any string from localStorage before inserting into innerHTML.
function escapeHtml(str) {
  return String(str)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;')
}

// ── Seed data ────────────────────────────────────────────────────────────────
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
  // Add future seasons here: S4: { "Stage Name": winRate, ... }
}

// ── localStorage helpers ─────────────────────────────────────────────────────
const LS_KEY = 'tekken_entries'

function getEntries() {
  try { return JSON.parse(localStorage.getItem(LS_KEY) || '[]') }
  catch { return [] }
}

function saveEntry(season, stage, winRate) {
  const entries = getEntries()
  entries.push({ season, stage, winRate, date: getToday() })
  localStorage.setItem(LS_KEY, JSON.stringify(entries))
}

function getToday() {
  return new Date().toISOString().slice(0, 10) // "YYYY-MM-DD"
}

// ── Data computations ────────────────────────────────────────────────────────

/**
 * Merges seed data with localStorage entries for a season.
 * Most recent localStorage entry per stage overrides the seed baseline.
 * Returns { stageName: winRate, ... }
 */
function buildCurrentState(season) {
  const base = { ...(SEED_DATA[season] || {}) }
  const entries = getEntries()
    .filter(e => e.season === season)
    .sort((a, b) => b.date.localeCompare(a.date)) // newest first
  const seen = new Set()
  for (const entry of entries) {
    if (!seen.has(entry.stage)) {
      base[entry.stage] = entry.winRate
      seen.add(entry.stage)
    }
  }
  return base
}

/** Returns the season key immediately before the given one (by sort order), or null. */
function getPreviousSeasonKey(season) {
  const keys = Object.keys(SEED_DATA).sort()
  const idx = keys.indexOf(season)
  return idx > 0 ? keys[idx - 1] : null
}

/**
 * Returns the most recent date (YYYY-MM-DD) with localStorage entries
 * for the given season that is strictly before today. Returns null if none.
 */
function getLastSessionDate(season) {
  const today = getToday()
  const dates = getEntries()
    .filter(e => e.season === season && e.date < today)
    .map(e => e.date)
  if (!dates.length) return null
  return dates.sort().at(-1)
}

/**
 * Computes per-stage deltas for the given season and mode.
 * mode: "season" — current vs previous season's current state
 * mode: "session" — current vs last session (prior calendar day)
 * Returns { stageName: number | null }  (null → render as "—")
 */
function computeDeltas(season, mode) {
  const current = buildCurrentState(season)
  let reference = {}

  if (mode === 'season') {
    const prevKey = getPreviousSeasonKey(season)
    reference = prevKey ? buildCurrentState(prevKey) : {}
  } else {
    const lastDate = getLastSessionDate(season)
    if (lastDate) {
      const base = { ...(SEED_DATA[season] || {}) }
      const entries = getEntries()
        .filter(e => e.season === season && e.date <= lastDate)
        .sort((a, b) => b.date.localeCompare(a.date))
      const seen = new Set()
      for (const entry of entries) {
        if (!seen.has(entry.stage)) {
          base[entry.stage] = entry.winRate
          seen.add(entry.stage)
        }
      }
      reference = base
    }
  }

  const deltas = {}
  for (const stage of Object.keys(current)) {
    deltas[stage] = stage in reference ? current[stage] - reference[stage] : null
  }
  return deltas
}

/** Mean win rate (rounded) across all stages in the given state object. */
function getOverallWinRate(stateData) {
  const values = Object.values(stateData)
  if (!values.length) return 0
  return Math.round(values.reduce((a, b) => a + b, 0) / values.length)
}

/** Returns { name, winRate } for the highest win rate stage. */
function getBestStage(stateData) {
  return Object.entries(stateData).reduce(
    (best, [name, rate]) => rate > best.winRate ? { name, winRate: rate } : best,
    { name: '—', winRate: -Infinity }
  )
}

/** Returns { name, winRate } for the lowest win rate stage. */
function getWorstStage(stateData) {
  return Object.entries(stateData).reduce(
    (worst, [name, rate]) => rate < worst.winRate ? { name, winRate: rate } : worst,
    { name: '—', winRate: Infinity }
  )
}

/** Returns the CSS color class for a win rate value. */
function barColor(rate) {
  if (rate >= 70) return 'green'
  if (rate >= 45) return 'amber'
  return 'red'
}

// ── Data layer tests (verify in console on load) ─────────────────────────────
function runDataTests() {
  const s3 = buildCurrentState('S3')
  console.assert(s3['Arena'] === 100, 'Arena S3 seed = 100')
  console.assert(s3['Genmaji Temple'] === 33, 'Genmaji Temple S3 seed = 33')
  console.assert(Object.keys(s3).length === 22, 'S3 has 22 stages')
  console.assert(getOverallWinRate(s3) === 57, `Overall win rate = 57, got ${getOverallWinRate(s3)}`)
  console.assert(getBestStage(s3).name === 'Arena', 'Best stage = Arena')
  console.assert(getWorstStage(s3).name === 'Genmaji Temple', 'Worst stage = Genmaji Temple')
  console.assert(getPreviousSeasonKey('S3') === null, 'S3 has no previous season')
  console.assert(barColor(70) === 'green', 'barColor 70 = green')
  console.assert(barColor(45) === 'amber', 'barColor 45 = amber')
  console.assert(barColor(44) === 'red',   'barColor 44 = red')
  const deltas = computeDeltas('S3', 'season')
  console.assert(deltas['Arena'] === null, 'S3 season delta = null (no S2)')
  const escaped = escapeHtml('<script>alert(1)</script>')
  console.assert(!escaped.includes('<script>'), 'escapeHtml strips script tags')
  console.log('[DataTests] All assertions passed')
}
runDataTests()
```

- [ ] **Step 2: Open index.html in browser — check console for `[DataTests] All assertions passed`**

Open DevTools console. Expected output: `[DataTests] All assertions passed` with no assertion errors or warnings.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add seed data, data layer functions, and escapeHtml XSS guard"
```

---

## Chunk 2: Rendering — season bar, summary cards, chart

### Task 3: State object and season bar

**Files:**
- Modify: `index.html` — append to `<script>` after `runDataTests()`

- [ ] **Step 1: Add state object and renderSeasonBar()**

Append after `runDataTests()`:

```js
// ── State ────────────────────────────────────────────────────────────────────
const state = {
  activeSeason: Object.keys(SEED_DATA).sort().at(-1), // default to last season
  activeTab: 'all',      // 'all' | 'trending-up' | 'trending-down'
  activeFilter: 'all',   // 'all' | 'strong' | 'weak'
  deltaMode: 'season',   // 'season' | 'session'
}

// ── Render: Season bar ───────────────────────────────────────────────────────
function renderSeasonBar() {
  document.getElementById('season-pills').innerHTML =
    Object.keys(SEED_DATA).sort().map(key => {
      const active = key === state.activeSeason ? ' active' : ''
      return `<button class="season-btn${active}" data-season="${escapeHtml(key)}">${escapeHtml(key)}</button>`
    }).join('')
}
```

- [ ] **Step 2: Temporarily append `renderSeasonBar()` at the bottom of the script, open browser**

Expected: "S3" pill visible, styled as active (lighter background, white text).

- [ ] **Step 3: Remove the temporary call**

---

### Task 4: Summary cards

**Files:**
- Modify: `index.html` — append to `<script>`

- [ ] **Step 1: Add renderSummaryCards()**

```js
// ── Render: Summary cards ────────────────────────────────────────────────────
function renderSummaryCards() {
  const current = buildCurrentState(state.activeSeason)
  const overall = getOverallWinRate(current)
  const best    = getBestStage(current)
  const worst   = getWorstStage(current)

  document.getElementById('summary-cards').innerHTML = `
    <div class="card">
      <div class="card-value white">${overall}%</div>
      <div class="card-label">Overall win rate</div>
    </div>
    <div class="card">
      <div class="card-value green">${escapeHtml(best.name)}</div>
      <div class="card-label">Best stage (${best.winRate}%)</div>
    </div>
    <div class="card">
      <div class="card-value red">${escapeHtml(worst.name)}</div>
      <div class="card-label">Worst stage (${worst.winRate}%)</div>
    </div>
  `
}
```

- [ ] **Step 2: Temporarily call `renderSeasonBar()` and `renderSummaryCards()` at the bottom of the script, open browser**

Expected: Three cards — "57% / Overall win rate", "Arena / Best stage (100%)" in green, "Genmaji Temple / Worst stage (33%)" in red/salmon.

- [ ] **Step 3: Remove temporary calls**

---

### Task 5: Chart section

**Files:**
- Modify: `index.html` — append to `<script>`

- [ ] **Step 1: Add renderChart() and its sub-functions**

```js
// ── Render: Chart section ────────────────────────────────────────────────────
function renderChart() {
  renderTabs()
  renderFilters()
  renderChartHeader()
  renderStageList()
}

function renderTabs() {
  const tabs = [
    { id: 'all',           label: 'All stages' },
    { id: 'trending-up',   label: 'Trending up' },
    { id: 'trending-down', label: 'Trending down' },
  ]
  document.getElementById('tabs').innerHTML = tabs.map(t => {
    const active = t.id === state.activeTab ? ' active' : ''
    return `<button class="tab-btn${active}" data-tab="${t.id}">${t.label}</button>`
  }).join('')
}

function renderFilters() {
  const filters = [
    { id: 'all',    label: 'All' },
    { id: 'strong', label: 'Strong (70%+)' },
    { id: 'weak',   label: 'Weak (sub 45%)' },
  ]
  document.getElementById('filters').innerHTML = filters.map(f => {
    const active = f.id === state.activeFilter ? ' active' : ''
    return `<button class="filter-btn${active}" data-filter="${f.id}">${f.label}</button>`
  }).join('')
}

function renderChartHeader() {
  const seasonToggle  = `<button class="delta-toggle${state.deltaMode === 'season'  ? ' active' : ''}" data-delta="season">last season</button>`
  const sessionToggle = `<button class="delta-toggle${state.deltaMode === 'session' ? ' active' : ''}" data-delta="session">last session</button>`
  document.getElementById('chart-header').innerHTML =
    `Stage &mdash; ${escapeHtml(state.activeSeason)} win rate &mdash; vs ${seasonToggle} / ${sessionToggle}`
}

function renderStageList() {
  const current = buildCurrentState(state.activeSeason)
  const deltas  = computeDeltas(state.activeSeason, state.deltaMode)

  // Sort by win rate descending
  let rows = Object.entries(current).sort((a, b) => b[1] - a[1])

  // Tab filter (trending requires a non-null delta)
  if (state.activeTab === 'trending-up') {
    rows = rows.filter(([name]) => deltas[name] !== null && deltas[name] > 0)
  } else if (state.activeTab === 'trending-down') {
    rows = rows.filter(([name]) => deltas[name] !== null && deltas[name] < 0)
  }

  // Strength filter
  if (state.activeFilter === 'strong') {
    rows = rows.filter(([, rate]) => rate >= 70)
  } else if (state.activeFilter === 'weak') {
    rows = rows.filter(([, rate]) => rate < 45)
  }

  if (!rows.length) {
    document.getElementById('stage-list').innerHTML =
      '<div class="no-stages">No stages match the current filters.</div>'
    return
  }

  document.getElementById('stage-list').innerHTML = rows.map(([name, rate]) => {
    const color = barColor(rate)
    const delta = deltas[name]
    let deltaHtml
    if (delta === null || delta === 0) {
      deltaHtml = `<span class="delta none">&mdash;</span>`
    } else if (delta > 0) {
      deltaHtml = `<span class="delta pos">+${delta}%</span>`
    } else {
      deltaHtml = `<span class="delta neg">${delta}%</span>`
    }
    return `
      <div class="stage-row">
        <span class="stage-name">${escapeHtml(name)}</span>
        <div class="bar-track"><div class="bar-fill ${color}" style="width:${rate}%"></div></div>
        <span class="win-rate">${rate}%</span>
        ${deltaHtml}
      </div>`
  }).join('')
}
```

- [ ] **Step 2: Temporarily call all render functions at the bottom of the script**

```js
renderSeasonBar()
renderSummaryCards()
renderChart()
```

- [ ] **Step 3: Open browser — verify chart renders 22 stages sorted high to low with correct bar colors**

Expected:
- Arena and Baobab Horizon at top with full-width green bars (100%)
- Coliseum of Fate through Genmaji Temple Daytime in amber zone (45–69%)
- Into the Stratosphere through Genmaji Temple at bottom with red bars (<45%)
- Delta column shows `—` for all rows (no S2 seed data)
- Tabs and filter buttons are clickable but have no effect yet (handlers added in Task 7)

- [ ] **Step 4: Remove temporary calls. Commit.**

```bash
git add index.html
git commit -m "feat: add renderSeasonBar, renderSummaryCards, renderChart"
```

---

## Chunk 3: Interactivity — entry form, event handlers, wiring

### Task 6: Entry form and save handler

**Files:**
- Modify: `index.html` — append to `<script>`

- [ ] **Step 1: Add renderEntryForm(), populateStageDropdown(), showToast(), handleSave()**

```js
// ── Render: Entry form ───────────────────────────────────────────────────────
function renderEntryForm() {
  const seasonSelect = document.getElementById('form-season')

  // Only rebuild season options if the set of seasons changed (avoid resetting user selection)
  const seasonKeys = Object.keys(SEED_DATA).sort()
  if (seasonSelect.options.length !== seasonKeys.length) {
    seasonSelect.innerHTML = seasonKeys.map(key =>
      `<option value="${escapeHtml(key)}">${escapeHtml(key)}</option>`
    ).join('')
  }
  seasonSelect.value = state.activeSeason
  populateStageDropdown(seasonSelect.value)
}

function populateStageDropdown(season) {
  const stages = Object.keys(SEED_DATA[season] || {}).sort()
  document.getElementById('form-stage').innerHTML = stages.map(s =>
    `<option value="${escapeHtml(s)}">${escapeHtml(s)}</option>`
  ).join('')
}

// ── Toast ────────────────────────────────────────────────────────────────────
let toastTimer = null
function showToast(msg = 'Saved!') {
  const toast = document.getElementById('toast')
  toast.textContent = msg
  toast.classList.add('show')
  clearTimeout(toastTimer)
  toastTimer = setTimeout(() => toast.classList.remove('show'), 2000)
}

// ── Save handler ─────────────────────────────────────────────────────────────
function handleSave() {
  const season  = document.getElementById('form-season').value
  const stage   = document.getElementById('form-stage').value
  const raw     = document.getElementById('form-winrate').value
  const winRate = parseInt(raw, 10)

  if (!season || !stage || raw === '' || isNaN(winRate) || winRate < 0 || winRate > 100) {
    showToast('Enter a win % between 0 and 100')
    return
  }

  saveEntry(season, stage, winRate)
  document.getElementById('form-winrate').value = ''
  showToast()
  render()
}
```

- [ ] **Step 2: Temporarily call `renderEntryForm()` at the bottom of the script, open browser**

Expected: Season dropdown shows "S3", stage dropdown shows all 22 stage names alphabetically. Win % input is empty.

- [ ] **Step 3: Remove temporary call**

---

### Task 7: Event delegation, render() orchestrator, and initial load

**Files:**
- Modify: `index.html` — append to `<script>`

- [ ] **Step 1: Add render(), event delegation, and initial render call**

```js
// ── Render orchestrator ──────────────────────────────────────────────────────
function render() {
  renderSeasonBar()
  renderSummaryCards()
  renderChart()
  renderEntryForm()
}

// ── Event delegation ─────────────────────────────────────────────────────────
document.addEventListener('click', e => {
  const seasonBtn = e.target.closest('[data-season]')
  if (seasonBtn) {
    state.activeSeason = seasonBtn.dataset.season
    render()
    return
  }

  const tabBtn = e.target.closest('[data-tab]')
  if (tabBtn) {
    state.activeTab = tabBtn.dataset.tab
    renderChart()
    return
  }

  const filterBtn = e.target.closest('[data-filter]')
  if (filterBtn) {
    state.activeFilter = filterBtn.dataset.filter
    renderChart()
    return
  }

  const deltaBtn = e.target.closest('[data-delta]')
  if (deltaBtn) {
    state.deltaMode = deltaBtn.dataset.delta
    renderChart()
    return
  }

  if (e.target.closest('#add-data-btn')) {
    document.getElementById('entry-panel').scrollIntoView({ behavior: 'smooth' })
    document.getElementById('form-winrate').focus()
    return
  }

  if (e.target.closest('#save-btn')) {
    handleSave()
    return
  }
})

// Re-populate stage dropdown when season changes in the form
document.getElementById('form-season').addEventListener('change', e => {
  populateStageDropdown(e.target.value)
})

// ── Initial render ────────────────────────────────────────────────────────────
render()
```

- [ ] **Step 2: Open browser — run full verification checklist**

1. Page loads — S3 active, 22 stages visible, sorted high to low, correct bar colors
2. Summary cards: 57%, Arena (green), Genmaji Temple (red)
3. Click "Trending up" tab — list shows "No stages match the current filters" (no prior season data, all deltas are `—`)
4. Click "All stages" tab, then "Strong (70%+)" filter — only 6 green-bar stages show (Arena, Baobab Horizon, Secluded Training Ground, Seaside Resort, Phoenix Gate, Descent into Subconscious)
5. Click "All" filter to reset, then click "last session" delta toggle — header underline moves to "last session"; all deltas still `—` (no prior-day entries)
6. Click "last season" toggle to reset
7. Use entry form: select S3, select Arena, enter 80, click Save — toast "Saved!" appears, Arena bar shrinks and turns amber
8. Reload page — Arena still shows 80% (localStorage persisted)
9. Check Genmaji Temple still shows 33% (seed baseline unchanged)
10. Click "+ Add Stage Data" — page scrolls to entry form

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add event handlers, render orchestrator, initial render — app complete"
```

---

### Task 8: GitHub Pages deployment

**Files:**
- No code changes

- [ ] **Step 1: Push branch to remote**

```bash
git push -u origin claude/strange-rosalind
```

- [ ] **Step 2: Enable GitHub Pages in repo settings**

In the GitHub repo → Settings → Pages → Source: set to the branch `claude/strange-rosalind` (or `main` after merging), root directory `/`. Save.

- [ ] **Step 3: Navigate to the Pages URL and run the same verification checklist as Task 7 Step 2**

Expected: identical behavior to local file:// — data renders, form works, localStorage persists across browser sessions on the same device.
