# Tekken 8 Stage Tracker — Snapshot Versioning Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add time-indexed snapshot support to the Tekken 8 Stage Tracker — `SEED_DATA` becomes date-keyed, a snapshot pill row lets the user view historical state, deltas can compare snapshot-to-snapshot, and the `vs last session` mode is removed.

**Architecture:** Incremental edits to the single `index.html` file. Migrates `SEED_DATA` from `{season: {stage: rate}}` to `{season: {date: {stage: rate}}}`, threads an `activeSnapshot` field through state and render functions, replaces session-based delta logic with snapshot-based logic, and adds a snapshot pill row + entry-form hiding for read-only historical views. All changes verified via `runDataTests()` console assertions and a manual UI checklist served at `http://localhost:8080`.

**Tech Stack:** Vanilla HTML5/CSS/JavaScript (ES6+). No frameworks, no build, no dependencies. Local dev server: `python -m http.server 8080` from repo root.

**Spec:** `docs/superpowers/specs/2026-05-01-tekken8-snapshot-versioning-design.md`

**Repository state:** Single file `index.html` at repo root. Existing dev server config at `.claude/launch.json`. Working branch: `claude/strange-rosalind` (worktree).

---

## File Structure

All changes occur in **`D:\SW-Stage-Data\index.html`**:

| Section in file | Change |
|---|---|
| `SEED_DATA` constant (~line 272) | Restructure to date-keyed; add 2026-05-01 S3 snapshot |
| Data layer (~lines 320-435) | Add `getSnapshotDates`, `getNewestSnapshotDate`, `getSnapshot`; update `buildCurrentState`; rewrite `computeDeltas`; remove `getLastSessionDate` |
| `runDataTests()` (~line 437) | Expand assertions for new shape, snapshot navigation, stale rule |
| `state` constant (~line 460) | Add `activeSnapshot`; rename `deltaMode` values |
| HTML body (~line 230) | Add empty snapshot-pills div row below season pills |
| CSS (~line 27) | Add `.snapshot-row`, `.snapshot-btn` styles; add `.delta-toggle:disabled` style; hide-class for entry panel |
| Render functions (~lines 467-606) | New `renderSnapshotPills`; update `renderChartHeader`, `renderEntryForm`, all callers of data layer |
| Event delegation (~line 645) | Handle `data-snapshot`; reset rules on season change; ignore disabled delta toggle |

**Note on innerHTML usage:** the existing codebase builds DOM via `innerHTML` with `escapeHtml()` wrapping all untrusted strings (stage names, season keys, dates). New code in this plan follows the same convention — every interpolated string from `SEED_DATA` keys or localStorage passes through `escapeHtml()` before insertion. Do not switch to `textContent` ad-hoc; it would break the established pattern and the existing tests.

---

## Chunk 1: Data layer migration

### Task 1: Restructure SEED_DATA to date-keyed shape

**Files:**
- Modify: index.html — SEED_DATA constant (~lines 272-301)

- [ ] **Step 1: Replace the SEED_DATA constant**

Find the entire `const SEED_DATA = { ... }` block and replace with the new date-keyed shape. Each season key now points to an object whose keys are ISO dates (YYYY-MM-DD) mapping to stage win-rate maps. The newest date is the current authoritative baseline.

- S1: wrap the existing 20-stage map under key `"2026-04-11"`.
- S2: wrap the existing 22-stage map under key `"2026-04-11"`.
- S3: wrap the existing 22-stage map under key `"2026-04-11"` AND add a new `"2026-05-01"` key with these 22 win rates:

| Stage | Win % |
|---|---|
| Arena | 58 |
| Arena Underground | 62 |
| Baobab Horizon | 66 |
| Celebration on the Seine | 66 |
| Coliseum of Fate | 63 |
| Descent into Subconscious | 35 |
| Elegant Palace | 66 |
| Fallen Destiny | 43 |
| Genmaji Temple | 33 |
| Genmaji Temple Daytime | 61 |
| Into the Stratosphere | 52 |
| Midnight Siege | 53 |
| Ortiz Farm | 57 |
| Pac-Pixels | 50 |
| Phoenix Gate | 62 |
| Rebel Hangar | 30 |
| Sanctum | 61 |
| Seaside Resort | 62 |
| Secluded Training Ground | 81 |
| Urban Square | 47 |
| Urban Square Evening | 58 |
| Yakushima | 58 |

Add a comment above the constant: "Each season holds one or more dated snapshots. Newest is current baseline. Add new snapshots by inserting another date key under the same season."

Also remove the existing trailing comment `// Add future seasons here: S4: { "Stage Name": winRate, ... }` — it documents the old flat shape and is no longer accurate.

- [ ] **Step 2: Open browser at http://localhost:8080**

Expected: page is **broken**. Console shows assertion failures because `buildCurrentState` still expects the old shape. Chart empty or NaN. **This is the expected intermediate state.** Tasks 2-4 fix it.

- [ ] **Step 3: Do NOT commit yet** — wait until Task 4. We commit Tasks 1-4 together as one logical migration.

---

### Task 2: Add snapshot helpers and rewrite buildCurrentState

**Files:**
- Modify: index.html — data layer section (~lines 320-346)

- [ ] **Step 1: Insert three new helpers immediately before the existing `buildCurrentState` function**

Add functions:

- `getSnapshotDates(season)` — returns `Object.keys(SEED_DATA[season] || {}).sort()`. Returns sorted ascending array of ISO date strings, or `[]` if season missing.
- `getNewestSnapshotDate(season)` — returns last element of `getSnapshotDates(season)`, or `null` if empty.
- `getSnapshot(season, date)` — returns `(SEED_DATA[season] && SEED_DATA[season][date]) || {}`.

All raw snapshot access in the codebase must go through `getSnapshot` rather than direct `SEED_DATA[season][date]` reads. This includes `buildCurrentState`, `computeDeltas`, and `populateStageDropdown` (~line 601, currently reads `Object.keys(SEED_DATA[season] || {})` — change to `Object.keys(getSnapshot(season, getNewestSnapshotDate(season)))` so it lists stages from the newest snapshot of the chosen season). Without this update the entry form's stage dropdown will be empty after the migration.

- [ ] **Step 2: Replace `buildCurrentState` body — new signature `buildCurrentState(season, snapshotDate = null)`**

Logic:

1. If `snapshotDate !== null`, return `{ ...getSnapshot(season, snapshotDate) }` and exit. **No localStorage layering** for historical views.
2. Compute `newestDate = getNewestSnapshotDate(season)`. If `null`, return `{}`.
3. Initialize `base = { ...getSnapshot(season, newestDate) }`.
4. Get all entries via `getEntries()`, map each to `{...e, _idx: i}` to preserve insertion order.
5. Filter to entries where `e.season === season` AND `e.date >= newestDate`. **Strict less-than stale rule** keeps same-day entries.
6. Sort: newer date first via `b.date.localeCompare(a.date)`; same-date tie broken by larger `_idx` first (latest save wins).
7. Walk sorted entries: first occurrence per stage wins, write into `base`.
8. Return `base`.

Doc comment on the function must call out:
- "Stale-entry rule: localStorage entries with `entry.date < newestSnapshotDate` are dropped (strict less-than). Same-day entries DO layer on top."
- "Historical view (snapshotDate non-null) ignores localStorage entirely — read-only."

- [ ] **Step 3: No isolated test yet** — runs after Task 4.

---

### Task 3: Rewrite computeDeltas, delete getLastSessionDate

**Files:**
- Modify: index.html — `getLastSessionDate`, `computeDeltas` (~lines 357-405)

- [ ] **Step 1: Delete `getLastSessionDate` entirely.** It is no longer referenced anywhere.

- [ ] **Step 2: Replace `computeDeltas` with new signature `computeDeltas(season, mode, snapshotDate = null)`**

Logic:

1. `current = buildCurrentState(season, snapshotDate)`.
2. `reference = {}` initially.
3. **If `mode === 'last-season'`:** `prevKey = getPreviousSeasonKey(season)`. If `prevKey` exists, `reference = getSnapshot(prevKey, getNewestSnapshotDate(prevKey))`. **Note:** always compares to previous season's newest, regardless of `snapshotDate`.
4. **Else if `mode === 'previous-snapshot'`:** `dates = getSnapshotDates(season)`. `viewedDate = snapshotDate !== null ? snapshotDate : getNewestSnapshotDate(season)`. `idx = dates.indexOf(viewedDate)`. If `idx > 0`, `reference = getSnapshot(season, dates[idx - 1])`. Otherwise reference stays `{}` (no prior snapshot — all deltas null).
5. Build deltas object: for each stage in `current`, `deltas[stage] = (stage in reference) ? current[stage] - reference[stage] : null`.
6. Return `deltas`.

Old `'season'` and `'session'` string values are gone. Any caller still using them must be updated (state init in Task 5).

---

### Task 4: Update runDataTests for the new shape

**Files:**
- Modify: index.html — `runDataTests()` (~line 437)

- [ ] **Step 1: Replace `runDataTests` body with the expanded assertion suite**

The new test function must verify all of the following. The test injects a stub for `getEntries` while testing localStorage behavior, then restores the original — required so assertions are deterministic regardless of real localStorage state.

Assertion groups:

1. **Snapshot navigation:**
   - `getSnapshotDates('S3').length === 2`
   - `getSnapshotDates('S3')[0] === '2026-04-11'`
   - `getSnapshotDates('S3')[1] === '2026-05-01'`
   - `getSnapshotDates('S1').length === 1`
   - `getNewestSnapshotDate('S3') === '2026-05-01'`
   - `getNewestSnapshotDate('S1') === '2026-04-11'`

2. **buildCurrentState newest (no localStorage):** stub `getEntries` to return `[]`.
   - `buildCurrentState('S3').Arena === 58`
   - `buildCurrentState('S3')['Genmaji Temple'] === 33`
   - `buildCurrentState('S3')['Secluded Training Ground'] === 81`
   - `Object.keys(buildCurrentState('S3')).length === 22`

3. **buildCurrentState historical:**
   - `buildCurrentState('S3', '2026-04-11').Arena === 100`
   - `buildCurrentState('S3', '2026-04-11')['Secluded Training Ground'] === 71`

4. **Summary helpers (newest with empty localStorage):**
   - `getBestStage(buildCurrentState('S3')).name === 'Secluded Training Ground'`
   - `getWorstStage(buildCurrentState('S3')).name === 'Rebel Hangar'`
   - `getBestStage(buildCurrentState('S3', '2026-04-11')).name === 'Arena'`

5. **Previous-season helpers:**
   - `getPreviousSeasonKey('S3') === 'S2'`
   - `getPreviousSeasonKey('S2') === 'S1'`
   - `getPreviousSeasonKey('S1') === null`

6. **Delta: previous-snapshot:**
   - `computeDeltas('S3', 'previous-snapshot').Arena === -42` (58 - 100)
   - `computeDeltas('S3', 'previous-snapshot')['Secluded Training Ground'] === 10` (81 - 71)
   - `computeDeltas('S3', 'previous-snapshot', '2026-04-11').Arena === null` (oldest snapshot, no prior)
   - `computeDeltas('S1', 'previous-snapshot').Arena === null` (single-snapshot season)

7. **Delta: last-season:**
   - `computeDeltas('S3', 'last-season').Arena === 8` (58 - 50)
   - `computeDeltas('S3', 'last-season', '2026-04-11').Arena === 50` (100 - 50; historical view still compares vs S2 newest)
   - `computeDeltas('S1', 'last-season').Arena === null` (no previous season)

8. **barColor & escapeHtml unchanged:**
   - `barColor(70) === 'green'`, `barColor(45) === 'amber'`, `barColor(44) === 'red'`
   - `escapeHtml(...)` strips a literal `<script>` tag (assemble the test string by concatenation so the source file itself stays parser-safe).

9. **Stale-entry rule (strict <):** wrap each sub-test in a stub-and-restore.
   - Inject one entry: `{season:'S3', stage:'Arena', winRate:99, date:'2026-04-15'}`. Assert `buildCurrentState('S3').Arena === 58` (stale, dropped).
   - Inject one entry: `{season:'S3', stage:'Arena', winRate:99, date:'2026-05-01'}`. Assert `buildCurrentState('S3').Arena === 99` (same-day layers).
   - Inject two entries (insertion order matters): `[{...,winRate:99,date:'2026-05-01'}, {...,winRate:77,date:'2026-05-01'}]`. Assert `buildCurrentState('S3').Arena === 77` (later same-day save wins).
   - Inject one entry on 2026-05-01. Assert `buildCurrentState('S3', '2026-04-11').Arena === 100` (historical view ignores localStorage entirely).

Implementation note: replace `getEntries` via reassignment. To support that, the original `getEntries` must be a `let` binding, OR the test must close over a stub via a top-level `let _entriesStub = null` plus a small modification to `getEntries` that returns `_entriesStub` when set. Choose whichever is cleanest; either is fine. Document the choice in a comment. **Wrap each stubbed sub-test in a `try { ... } finally { _restoreGetEntries() }`** so that an assertion failure does not leak the stub into the live UI render that follows `runDataTests()`. Without this, a single failing test could leave the page rendering against fake localStorage data and produce confusing follow-on failures.

End with `console.log('[DataTests] All assertions passed')`.

- [ ] **Step 2: Hard reload http://localhost:8080 and verify console**

Expected: `[DataTests] All assertions passed`. Zero `Assertion failed:` messages. The visible UI may still look wrong (state still uses old `'session'` literal, snapshot row not rendered yet, etc.). That's expected — UI tasks are next.

- [ ] **Step 3: Commit data layer migration**

```bash
git add index.html
git commit -m "feat(data): migrate SEED_DATA to date-keyed snapshots and rewrite delta logic"
```

---

## Chunk 2: State, snapshot pill row, and rendering

### Task 5: Update state object — add activeSnapshot, rename deltaMode values

**Files:**
- Modify: index.html — `state` constant (~line 460)

- [ ] **Step 1: Replace state constant**

New shape:

- `activeSeason` — unchanged (defaults to last season key alphabetically).
- `activeSnapshot: null` — new. `null` means "newest snapshot of activeSeason"; otherwise an ISO date string for historical view.
- `activeTab: 'all'` — unchanged.
- `activeFilter: 'all'` — unchanged.
- `deltaMode: 'last-season'` — new default. Old values `'season'` / `'session'` are gone; new values are `'last-season'` / `'previous-snapshot'`.

- [ ] **Step 2: Reload and confirm console still shows `[DataTests] All assertions passed`.** No commit yet.

---

### Task 6: Add snapshot pill row HTML, CSS, renderer, and entry-panel hide logic

**Files:**
- Modify: index.html — body (~line 230), CSS (~line 27), JS render section (~line 467)

- [ ] **Step 1: Insert snapshot row markup in the HTML body**

Immediately after the closing `</div>` of `.season-bar` and before the `<!-- Summary cards -->` HTML comment marker (anchor by that comment, not by line number — it currently sits at line ~234), insert a new section:

- A wrapping `<div>` with `class="snapshot-row" id="snapshot-row" hidden`.
- Inside: a `<span class="snapshot-row-label">Snapshots</span>` and a `<div class="snapshot-pills" id="snapshot-pills"></div>`.

The `hidden` attribute is the default state; the renderer toggles it.

- [ ] **Step 2: Add CSS rules**

Insert these style rules into the existing `<style>` block, just before the `/* Summary cards */` section:

- `.snapshot-row` — flex row, gap 10px, margin-bottom 24px, padding-left 2px.
- `.snapshot-row[hidden]` — `display: none`.
- `.snapshot-row-label` — uppercase 11px, color #666, letter-spacing 0.05em, margin-right 4px.
- `.snapshot-pills` — flex row, gap 6px, wrap.
- `.snapshot-btn` — transparent bg, 1px solid #333 border, color #888, border-radius 14px, padding 4px 12px, cursor pointer, font-size 12px, transition (background, color, border-color) 0.15s.
- `.snapshot-btn:hover` — border-color #666, color #ccc.
- `.snapshot-btn.active` — background #2a2a2a, border-color #777, color #eee.
- `.snapshot-btn .tag` — color #888, margin-left 4px, font-size 11px.
- `.snapshot-btn.active .tag` — color #aaa.

Also add disabled-state styling for the existing delta toggle:

- `.delta-toggle:disabled, .delta-toggle[disabled]` — opacity 0.4, cursor not-allowed, text-decoration none.
- `.delta-toggle:disabled:hover` — color #666 (don't lighten on hover).

And add hide-overrides for the entry panel:

- `.entry-panel[hidden], .add-data-btn[hidden]` — `display: none !important`.

- [ ] **Step 3: Add `renderSnapshotPills` function**

Insert immediately after `renderSeasonBar`'s closing brace.

Logic:

1. `dates = getSnapshotDates(state.activeSeason)`.
2. If `dates.length <= 1`, set the row's `hidden` attribute to true, clear the pills container's HTML, return.
3. Otherwise, set `hidden = false`. Compute `newest = dates[dates.length - 1]`.
4. For each date in `dates`, build a button with `class="snapshot-btn"` plus `' active'` when the pill represents the active view. Active means: `(state.activeSnapshot === null && date === newest) || state.activeSnapshot === date`.
5. The newest pill gets a trailing `<span class="tag">current</span>`.
6. Each button has `data-snapshot="<escaped date>"` and an escaped date label.
7. Build the markup and assign to the pills container.

All untrusted strings (the date itself comes from `SEED_DATA` keys — controlled, but escape anyway for consistency with the existing pattern).

- [ ] **Step 4: Add `renderEntryPanelVisibility` function**

Insert immediately before `// Render orchestrator`. Logic: read `state.activeSnapshot`; set `entry-panel.hidden = (activeSnapshot !== null)` and `add-data-btn.hidden = (activeSnapshot !== null)`. Single line of comment: "Historical snapshots are read-only — hide entry form and Add button."

- [ ] **Step 5: Wire both new renderers into `render()`**

Update the `render()` orchestrator to call (in this order):

1. `renderSeasonBar()`
2. `renderSnapshotPills()` — new
3. `renderSummaryCards()`
4. `renderChart()`
5. `renderEntryForm()`
6. `renderEntryPanelVisibility()` — new

- [ ] **Step 6: Reload — verify pills visible only on S3**

Expected: page loads on S3 newest. Two snapshot pills visible: `2026-04-11` (inactive) and `2026-05-01 current` (active). Switching season pills to S1 or S2 hides the snapshot row. Click handlers not wired yet — clicks do nothing. No commit yet.

---

### Task 7: Update renderSummaryCards, renderStageList, renderChartHeader to thread activeSnapshot

**Files:**
- Modify: index.html — render functions (~lines 477-584)

- [ ] **Step 1: Pass `state.activeSnapshot` into data calls**

In `renderSummaryCards`, change `buildCurrentState(state.activeSeason)` to `buildCurrentState(state.activeSeason, state.activeSnapshot)`.

In `renderStageList`, do the same for the `current` line, AND change `computeDeltas(state.activeSeason, state.deltaMode)` to `computeDeltas(state.activeSeason, state.deltaMode, state.activeSnapshot)`.

- [ ] **Step 2: Replace `renderChartHeader` body**

New behavior:

1. Determine `seasonActive` and `snapActive` flags from `state.deltaMode`.
2. Determine `hasMultiSnapshot = getSnapshotDates(state.activeSeason).length > 1`. If false, the `previous-snapshot` toggle button is rendered with the `disabled` attribute.
3. Compute the displayed date label: `state.activeSnapshot` if non-null, otherwise `getNewestSnapshotDate(state.activeSeason) || '—'`. Escape it.
4. Build the header HTML: `Stage — <season> (<date>) win rate — vs [last season toggle] / [previous snapshot toggle]`.
5. Both toggle buttons carry `data-delta="last-season"` or `data-delta="previous-snapshot"` accordingly. The disabled attribute is conditional on the multi-snapshot check.

- [ ] **Step 3: Reload — verify the chart updates correctly**

Expected:
- Default S3 newest: chart shows 2026-05-01 values (Secluded Training Ground 81 at top, Rebel Hangar 30 at bottom). Header reads `S3 (2026-05-01) win rate — vs last season / previous snapshot`. `last season` is the active toggle. `previous snapshot` is enabled but inactive.
- Switching to S1: header `S1 (2026-04-11) ...`. `previous snapshot` toggle is greyed out (disabled attribute applied).
- Pill clicks still do nothing (Task 8 wires events).

---

## Chunk 3: Event handling and verification

### Task 8: Wire snapshot pill clicks, season-change reset, disabled-toggle skip

**Files:**
- Modify: index.html — event delegation block (~lines 644-684)

- [ ] **Step 1: Replace the click delegation handler**

The new handler must process these targets in order, with `e.target.closest(...)` checks:

1. **`[data-season]`** — set `state.activeSeason = element.dataset.season`. **Reset rules:**
   - Set `state.activeSnapshot = null` (always land on newest of new season).
   - If `state.deltaMode === 'previous-snapshot'` AND `getSnapshotDates(state.activeSeason).length <= 1`, auto-fall-back: `state.deltaMode = 'last-season'`.
   - Call `render()`.
2. **`[data-snapshot]`** — read `date = element.dataset.snapshot`. Compute `newest = getNewestSnapshotDate(state.activeSeason)`. Set `state.activeSnapshot = (date === newest) ? null : date`. Call `render()`.
3. **`[data-tab]`** — unchanged: set `activeTab`, call `renderChart()`.
4. **`[data-filter]`** — unchanged: set `activeFilter`, call `renderChart()`.
5. **`[data-delta]`** — **Add disabled-skip:** if `element.disabled` is truthy, return without changes. Otherwise set `state.deltaMode = element.dataset.delta`, call `renderChart()`.
6. **`#add-data-btn`** — unchanged: scroll entry panel into view, focus win-rate input.
7. **`#save-btn`** — unchanged: call `handleSave()`.

Each block ends with `return` to avoid falling through.

- [ ] **Step 2: Reload — quick smoke test**

Expected (on S3 newest):
1. Click `2026-04-11` pill → chart shows historical data; header date label shows `2026-04-11`; entry panel and `+ Add Stage Data` button are hidden.
2. Click `2026-05-01 current` pill → returns to current view; entry panel reappears.
3. Click `vs previous snapshot` toggle → Arena delta `-42%`, Secluded Training Ground `+10%`.
4. Switch to S1 (with `previous-snapshot` mode active) → mode auto-resets to `last-season`. Toggle is greyed.
5. Click greyed `previous snapshot` toggle on S1 → no effect.
6. Switch back to S3 → snapshot row reappears; mode stays `last-season` (no auto-restore — confirmed one-way).

- [ ] **Step 3: Commit UI and event changes**

```bash
git add index.html
git commit -m "feat(ui): add snapshot pill row, historical read-only view, snapshot-aware deltas"
```

---

### Task 9: Full manual verification checklist

**Files:**
- No code changes

- [ ] **Step 1: Hard reload (Ctrl+Shift+R) http://localhost:8080 and verify each item**

1. Console shows `[DataTests] All assertions passed`. No errors or warnings.
2. Page opens on **S3 newest (2026-05-01)** by default. Chart sorted by win rate desc: Secluded Training Ground (81%) at top, Rebel Hangar (30%) at bottom.
3. Summary cards reflect 2026-05-01 data (Best = Secluded Training Ground green, Worst = Rebel Hangar red, Overall ≈ 55%).
4. Snapshot row visible with two pills; `2026-05-01 current` highlighted. Header reads `S3 (2026-05-01) win rate — ...`.
5. Click `vs previous snapshot` toggle → Arena `-42%`, Secluded Training Ground `+10%`, Genmaji Temple `—` (33-33=0; existing `renderStageList` treats `delta === 0` the same as `null` and renders the em-dash via the `.delta.none` branch — confirmed against current code at ~line 569).
6. Click `Trending up` tab → only stages with positive deltas shown.
7. Click `Trending down` tab → only stages with negative deltas shown (Arena, Baobab Horizon, Phoenix Gate, etc.).
8. Click `All stages` tab to reset.
9. Click `2026-04-11` snapshot pill → chart shows old data (Arena 100, Baobab Horizon 100). Header reads `S3 (2026-04-11)`. Entry panel and `+ Add Stage Data` are **hidden**.
10. With historical S3 active, click `vs last season` → deltas computed against S2 newest (Arena: 100-50 = +50).
11. With historical S3 active, click `vs previous snapshot` → all deltas show `—` (no snapshot before 2026-04-11).
12. Click `2026-05-01 current` pill → returns to current view; entry panel reappears.
13. Switch to S2 → snapshot row hides. Header reads `S2 (2026-04-11)`. `vs previous snapshot` toggle is greyed.
14. Click greyed `vs previous snapshot` toggle on S2 → no effect.
15. Switch back to S3 → snapshot row reappears.
16. Set deltaMode to `vs previous snapshot` (S3 newest), then switch to S1 → mode auto-resets to `vs last season`.
17. Save Arena = 99 in entry form (S3, current view) → toast "Saved!", Arena bar updates immediately.
18. Reload page → Arena still 99 (localStorage same-day layering works).
19. Click `2026-04-11` historical pill → Arena reverts to 100 (historical view ignores localStorage).
20. DevTools → Application → Local Storage → confirm `tekken_entries` contains the saved entry.

- [ ] **Step 2: Confirm clean working tree**

```bash
git status
# Expected: nothing to commit, working tree clean
```

---

### Task 10: Push and deploy

**Files:**
- No code changes

- [ ] **Step 1: Push the worktree branch**

```bash
git push
```

- [ ] **Step 2: Wait ~30-60s for GitHub Pages rebuild, then open the live URL**

Expected: same behavior as local — all 20 manual checklist items pass on the deployed site.

- [ ] **Step 3: Merge to main if Pages serves from main**

If GitHub Pages is configured to serve from `main`:

```bash
git checkout main
git merge claude/strange-rosalind --no-ff -m "feat: snapshot versioning for stage tracker"
git push
```

Skip if Pages serves directly from the worktree branch. To check the current Pages config without guessing, run `gh api repos/:owner/:repo/pages` (replace owner/repo) or visit the repository Settings → Pages page in the browser.

---

## Notes for the implementing engineer

- This is a **single-file static site**. No build, no transpile, no test framework. Tests are `console.assert` calls inside `runDataTests()` that run on every page load.
- The dev server is `python -m http.server 8080` from `D:\SW-Stage-Data`. Config lives at `.claude/launch.json`.
- `escapeHtml()` MUST wrap any string from `localStorage` or `SEED_DATA` keys before going into a markup template that gets assigned to an element's HTML content. Stage names, season keys, and snapshot dates all flow through this guard.
- The pre-Task-4 state has a broken UI by design — only Task 4's commit point produces a working build. Tasks 1-3 form one logical migration; commit them together as instructed in Task 4.
- If `runDataTests()` fails after Task 4, **stop and fix before moving on**. Failing tests block all UI tasks.
- Subagent-driven development: each task is sized to be one fresh subagent invocation. Hand off the spec path plus the relevant prior commit's contents so the subagent has full context.
