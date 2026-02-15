# BudgetBud Refactor Plan

## Goal
Restructure the app from a single-file implementation into small, testable modules while preserving behavior (local-first budgeting with optional Google Drive sync).

## Current State Summary
- `index.html` contains markup, styles, and all JavaScript business logic.
- Persistence, forecast math, rendering, theme toggling, and Drive API calls are tightly coupled.
- There is no automated verification for forecast or scheduling logic.

---

## Target Architecture

### Files to introduce
- `index.html` — keep only app shell markup and script/style includes.
- `styles/main.css` — extracted styles.
- `src/config.js` — static config constants (OAuth scopes, storage keys).
- `src/state.js` — in-memory state and initialization helpers.
- `src/storage.js` — localStorage read/write + normalization/validation.
- `src/forecast.js` — pure forecasting math and recurring-date advancement.
- `src/health.js` — budget health scoring logic.
- `src/history.js` — history filtering/aggregation helpers.
- `src/sync.js` — Google auth + Drive upload/download logic.
- `src/ui.js` — DOM rendering and event binding only.
- `src/main.js` — bootstrap entrypoint.
- `tests/forecast.test.js` — forecast regression checks.
- `tests/storage.test.js` — data shape and normalization checks.

### Module boundaries
- **Pure logic modules**: `forecast.js`, `health.js`, `history.js`, `storage.js` helpers.
- **I/O modules**: `sync.js`, DOM in `ui.js`, browser bootstrap in `main.js`.
- **State ownership**: `state.js` is the single source of truth.

---

## Migration Plan (Incremental)

## Phase 1: Stabilize behavior before moving code
1. Add snapshot-like “behavior checklist” in README:
   - payday modal behavior
   - recurring vs once-off bill handling
   - 90-day chart projection assumptions
   - cloud pull/save behavior
2. Add smoke checks:
   - app loads with empty localStorage
   - adding a bill persists and re-renders
   - marking recurring bill advances date

**Deliverable**: Baseline confidence for parity during refactor.

## Phase 2: Extract pure logic first
1. Move monthly-income and score/rating logic into `src/health.js`.
2. Move date advancement and forecast progression into `src/forecast.js`.
3. Move history totals/filtering into `src/history.js`.

**Rules**:
- No DOM access inside these modules.
- Inputs/outputs are plain objects and arrays.

**Deliverable**: testable pure modules with no behavior changes.

## Phase 3: Extract persistence and validation
1. Create `src/storage.js` with:
   - `loadLocalState()`
   - `saveLocalState(state)`
   - `normalizeBill(bill)`
   - `normalizeState(raw)`
2. Enforce safe defaults on load:
   - invalid dates dropped or repaired
   - non-numeric amounts coerced or rejected
   - unknown bill types rejected

**Deliverable**: resilient startup even with malformed local/cloud payloads.

## Phase 4: Isolate cloud sync
1. Create `src/sync.js`:
   - token setup
   - file lookup
   - upload/patch
   - download/parse
2. Return structured errors to UI (`{ code, message }`) rather than only `console.error`.
3. Add user-facing status transitions for loading/success/failure.

**Deliverable**: clearer failure handling and easier future provider swap.

## Phase 5: Separate UI from logic
1. Move render functions to `src/ui.js`:
   - `renderBills(state)`
   - `renderHistory(state)`
   - `renderForecast(state)`
   - `renderHealth(score)`
2. Use event handlers that dispatch actions, e.g.:
   - `ADD_BILL`
   - `MARK_BILL_PAID`
   - `CONFIRM_PAYDAY`
   - `SET_THEME`
3. Keep direct mutation in one place (reducer-like update function in `state.js`).

**Deliverable**: maintainable update flow and easier bug isolation.

## Phase 6: Split HTML/CSS and bootstrap
1. Move styles to `styles/main.css`.
2. Keep `index.html` as structure only.
3. Add `src/main.js` to initialize:
   - state load
   - auth init
   - first render
   - event listeners

**Deliverable**: clean project layout suitable for growth.

---

## File-by-file Cutover Sequence

1. `index.html`
   - Remove inline `<style>` and inline `<script>`.
   - Replace inline `onclick` with IDs/data attributes for JS binding.
2. `styles/main.css`
   - Copy styles from current inline block and keep theme variables intact.
3. `src/health.js`
   - Port score thresholds and monthly conversion logic.
4. `src/forecast.js`
   - Port 90-day chart and weekly projection stepping logic.
5. `src/storage.js`
   - Port localStorage keys and read/write behavior.
6. `src/history.js`
   - Port filtering and total calculations.
7. `src/sync.js`
   - Port OAuth + Drive query/upload/download.
8. `src/ui.js`
   - Port all DOM rendering and tab/page switching.
9. `src/state.js`
   - Centralize state object and update helpers.
10. `src/main.js`
    - Wire modules together and run on load.

---

## Testing Strategy

## Unit tests (high priority)
- Forecast:
  - pay cycles across 7/14/30 frequencies
  - recurring bills increment correctly
  - once-off bills do not recur
- Storage:
  - corrupted JSON fallback
  - missing keys fallback
  - normalization of invalid bill entries
- Health score:
  - grade boundaries (A/B/C/D/F)

## Manual checks (release gate)
- Add/edit/delete bills works and persists.
- Payday modal appears only on/after pay date.
- Confirm payday updates balance, history, and next pay date.
- Chart renders in light/dark mode.
- Cloud load/save updates status and data as expected.

---

## Definition of Done
- No inline JavaScript in `index.html`.
- No inline CSS in `index.html`.
- Forecast and health logic covered by tests.
- Invalid local/cloud payloads do not break rendering.
- User-facing sync errors are visible and actionable.
- README includes architecture + local run/test instructions.

---

## Risks and Mitigations
- **Risk**: subtle forecast parity regressions.
  - **Mitigation**: create locked regression fixtures from current behavior before extraction.
- **Risk**: auth flow breakage during sync modularization.
  - **Mitigation**: preserve API call order and add explicit status/error states.
- **Risk**: event binding mismatches after removing inline handlers.
  - **Mitigation**: enforce element ID contract and boot-time assertions.

---

## Suggested Commit Plan
1. `chore: add refactor roadmap and module architecture`
2. `refactor: extract health and forecast pure modules`
3. `refactor: extract storage and data normalization`
4. `refactor: isolate google drive sync service`
5. `refactor: move rendering into ui module and bootstrap main`
6. `test: add forecast and storage regression tests`
7. `docs: update README with architecture and run/test guide`
