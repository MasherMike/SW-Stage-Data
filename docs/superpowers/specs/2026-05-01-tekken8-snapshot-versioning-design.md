# Tekken 8 Stage Tracker — Snapshot Versioning Design

**Status:** Design
**Date:** 2026-05-01
**Builds on:** `docs/superpowers/specs/2026-04-11-tekken8-stage-tracker-design.md`

## Summary

Extend the Tekken 8 Stage Tracker to support multiple time-indexed snapshots of seed data per season. The user can capture the state of a season at different points in time, switch between snapshots in the UI, and see deltas comparing one snapshot to the previous. The existing `vs last session` delta mode is removed; snapshot-to-snapshot comparison subsumes it.

## Motivation

The seed data baked into `index.html` represents the user's win rates at one point in time. As the user plays more, those numbers change. Today the only way to track change over time is via manual localStorage entries — which is fine for day-to-day micro-tracking but doesn't capture clean "state of the season" baselines. Snapshots fill that gap: each is a complete picture of all stage win rates on a specific date. Comparing snapshots answers: *"What changed between when I first imported this season and now?"*

## Design

### Data model

`SEED_DATA` restructures from `{ season: { stage: winRate } }` to `{ season: { date: { stage: winRate } } }`. Each season holds one or more dated snapshots. The newest snapshot represents the current authoritative baseline.

```js
const SEED_DATA = {
  S1: { "2026-04-11": { /* 20 stages */ } },
  S2: { "2026-04-11": { /* 22 stages */ } },
  S3: {
    "2026-04-11": { /* original 22 stages from initial import */ },
    "2026-05-01": { /* updated 22 stages from new screenshot */ }
  }
}
```

Dates are ISO `YYYY-MM-DD` strings. Sorting them lexically equals sorting them chronologically.

### LocalStorage compatibility

The existing `tekken_entries` localStorage shape (`{season, stage, winRate, date}`) is unchanged. Manual entries continue to layer on top of the **newest** snapshot — the live "current state" view.

**Stale-entry rule:** when computing the current view, localStorage entries with `entry.date < newestSnapshotDate` are ignored. Rationale: a fresh seed snapshot is the new authoritative baseline; older manual edits shouldn't shadow it. Practical effect on this rollout: any dev-test entries saved before 2026-05-01 stop appearing once 2026-05-01 becomes S3's newest snapshot. The localStorage rows aren't deleted, just filtered out of the merge.

### Data layer changes

New / changed functions:

- **`getSnapshotDates(season)`** — returns sorted ascending array of date strings for that season.
- **`getNewestSnapshotDate(season)`** — last element of the above (or `null` if season missing).
- **`buildCurrentState(season, snapshotDate = null)`**:
  - If `snapshotDate === null` → use newest snapshot, then layer non-stale localStorage entries on top (most recent per stage wins).
  - If `snapshotDate` is a date string → return that snapshot's stage map exactly. No localStorage layering.
- **`computeDeltas(season, mode, snapshotDate = null)`**:
  - `mode === 'last-season'` → compare the resolved snapshot to the newest snapshot of the previous season (alphabetic predecessor in `SEED_DATA`). Returns `null` per stage if no previous season exists.
  - `mode === 'previous-snapshot'` → compare the resolved snapshot to the snapshot immediately preceding it within the same season. Returns `null` per stage if no prior snapshot exists.

Removed:

- `getLastSessionDate()`
- All session-based delta logic in `computeDeltas`.
- The `'session'` branch of `state.deltaMode`.

### State

```js
const state = {
  activeSeason: 'S3',
  activeSnapshot: null,      // null = newest snapshot; or 'YYYY-MM-DD' for historical view
  activeTab: 'all',
  activeFilter: 'all',
  deltaMode: 'last-season',  // 'last-season' | 'previous-snapshot'
}
```

`activeSnapshot` is reset to `null` whenever `activeSeason` changes (switching seasons always lands on the newest snapshot of that season).

### UI

**Snapshot pill row** (rendered below the existing season pill row):

- Visible only when the active season has more than one snapshot.
- Same pill styling as season buttons but slightly smaller (12px font, 14px border-radius).
- Each pill labeled with its date. The newest pill carries an additional `current` tag.
- Clicking a non-newest pill sets `state.activeSnapshot` to that date and re-renders.
- Clicking the newest pill (or switching seasons) sets `state.activeSnapshot` back to `null`.

**Entry form and "+ Add Stage Data" button:**

- Hidden entirely (CSS `display: none`) when `state.activeSnapshot !== null`.
- Reappear immediately when the user returns to the newest snapshot.
- Rationale: historical snapshots are read-only views; eliminating the form removes any ambiguity about where saves would land.

**Delta toggle in chart header:**

- Shows two options: `vs last season` / `vs previous snapshot`.
- The `vs previous snapshot` button is rendered with a `disabled` attribute, reduced opacity, and `cursor: not-allowed` when the active season has only one snapshot. Clicks are ignored.
- The `last-season` toggle continues to be active by default.

**Summary cards and chart:**

- Both render from whichever snapshot is active. No special handling needed beyond passing `state.activeSnapshot` to `buildCurrentState`.

### Migration

In-file changes to `index.html`:

1. Wrap each existing season's seed map under its date key:
   - `S1: { "2026-04-11": { ...existing } }`
   - `S2: { "2026-04-11": { ...existing } }`
   - `S3: { "2026-04-11": { ...existing } }`
2. Add a new `"2026-05-01"` snapshot to S3 with the 22 updated win rates.
3. Update all `SEED_DATA[season]` accesses in the data layer to go through the new helpers.
4. Update `runDataTests()` assertions to match new shape.

The 2026-05-01 S3 snapshot:

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

### Testing

`runDataTests()` adds assertions:

- `buildCurrentState('S3')` returns 2026-05-01 values (e.g., Arena = 58 before any localStorage layering).
- `buildCurrentState('S3', '2026-04-11')` returns the original values (Arena = 100).
- `getSnapshotDates('S3')` returns `['2026-04-11', '2026-05-01']`.
- `getSnapshotDates('S1').length === 1`.
- `computeDeltas('S3', 'previous-snapshot')['Arena'] === -42` (58 - 100).
- `computeDeltas('S1', 'previous-snapshot')['Arena'] === null` (only one snapshot).
- `computeDeltas('S3', 'last-season')` is non-null for stages present in both S2 and S3 newest snapshots.
- A localStorage entry dated `2026-04-15` for S3 Arena does not appear in the merged current state (stale: predates newest 2026-05-01 snapshot).

Manual verification checklist:

1. Page loads on S3, newest snapshot — chart shows 2026-05-01 values, snapshot pills row visible with two pills, newest active.
2. Click `2026-04-11` snapshot pill — chart shows original values; entry form and "+ Add Stage Data" button hidden.
3. With historical snapshot active, click `vs previous snapshot` — all deltas show `—` (no prior snapshot before 2026-04-11).
4. Switch to newest snapshot, click `vs previous snapshot` — Arena shows -42%, Secluded Training Ground shows +10%.
5. Switch to S1 — snapshot pill row hidden; `vs previous snapshot` toggle disabled (greyed).
6. Switch back to S3 newest, save Arena = 99 via entry form — Arena bar reflects 99 immediately, persists across reload.
7. Switch to S3 historical (2026-04-11) — Arena still shows 100 (manual entries don't layer on historical views).

## Out of scope

- Editing or deleting historical snapshots from the UI. Snapshots are added by editing `SEED_DATA` in the source file.
- Snapshot creation UI. The user manually adds new dated entries to `SEED_DATA` when a new screenshot is imported.
- Reviving the `vs last session` mode. If finer-grained day-by-day tracking is wanted later, it can be re-added as a third toggle without conflicting with snapshots.
