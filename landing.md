# Trainer V3 Landing Page Revamp

**Target:** Phase 1 of the Trainer V3 Release Plan
**Scope:** Frontend, `components/UnifiedTrainer/` subtree + shared hooks
**Estimated Total Effort:** ~20h

---

## Why This Is a Revamp, Not Small Fixes

The new UnifiedTrainer landing page was built with an optimistic design: it relies on Spot Discovery API + config cascade auto-correction to validate whether training can start. This is architecturally cleaner than the old trainer, but it **dropped critical edge-case handling** that the old Trainer V2 had.

The old trainer (`components/Trainer/useTrainer.tsx`, `utils.ts`, `TrainerModalV2.tsx`) guarded against 7 categories of failure that the new landing page does not:

### 1. No Error UI When Config Fails to Load

**Old:** `waitForTrainerConfig()` polls up to 5 seconds for `researchBasedData` and blocks start if it fails. Shows explicit error.

**New:** `trainerConfigError` is set in Redux but **has no UI**. The user sees a disabled Start button with no explanation. (`TrainerLanding.tsx` line 337-364 fetches config but never renders `trainerConfigError`.)

### 2. No Board Card Count Validation

**Old:** Validates board cards are 0, 3, 4, or 5 before any API call. Invalid counts block the request.

**New:** No validation in the landing page. Relies entirely on `SelectBoardCardsModal` to enforce counts, but nothing stops a malformed `cardSelection` array from reaching `buildCustomConfig()` or the session start API.

### 3. Randomize Board Not Properly Gated

**Old:** Disables "Randomize Board" for:
- Preflop-only training (forced Off)
- Precision DB with non-check postflop actions (structural conflict)

**New:** `AdvancedSettingsModal` shows Randomize Board unconditionally. A user can set Randomize Board = "On" for preflop-only training, where it has no effect and could confuse the backend.

### 4. No Config Rejection Recovery

**Old:** `handlePlayerActionsRejected()` catches backend 400/500 errors, shows "Invalid Game configuration, resetting default values", resets all selectors, and sets `isInvalidConfig` flag.

**New:** Session start errors are shown in an error banner (`errorMessage` prop), but there is no automatic reset-to-defaults flow. User must manually change settings to recover from a bad config.

### 5. No Essential Config Values Guard

**Old:** Checks `selectedCasino`, `selectedBB`, `selectedPlayers`, `selectedStack` exist before making API calls. Skips call if any are missing.

**New:** `buildSessionConfig()` falls back to `CANONICAL_TRAINER_DEFAULTS` with a console warning (line 283-293), but proceeds anyway. This can send incorrect config to the backend (e.g., default 6-player when user intended 2-player but the URL sync hadn't completed).

### 6. Position Validation Is Purely API-Dependent

**Old:** Has explicit spot-specific position validation functions (`validateUnopenedSpot()`, `validateVsRfiSpot()`, etc.) that work offline. These are deterministic rules that don't require an API call.

**New:** Relies entirely on Spot Discovery API `availablePositions`. If the API is slow, fails, or returns empty, there's no fallback -- positions appear unvalidated.

### 7. RTS Not Integrated in Custom Mode

**Old (Strategies page):** Sends `realTimeCalc=true` when conditions are met (NLH from flop, PLO on river). This enables on-demand solving for boards without precomputed strategies.

**New (CustomSpotBuilder):** Does not send `realTimeCalc` at all. For NLH custom spots that build into postflop, this means:
- Only precomputed boards work
- Boards without precomputed solutions silently fail or show `noSolutionForBoard`
- The `noSolutionForBoard` message exists (line 707-720) but doesn't offer RTS as an alternative

### 8. Complete Dead-End Config Not Communicated

**New issue:** The config cascade in `useTrainerGameConfig` can reach a state where ALL options at a cascade layer are empty (e.g., switching to a simulation type with no data for the current player count). When this happens:
- `isCurrentConfigValid` is false
- Start button is disabled
- The validation message says "No strategy data for this configuration"
- But **which** setting is wrong is not communicated -- the user has no guidance

---

## Task Breakdown

### Task 1.1 -- Hide Standard Postflop Modes `[HIDE]` (P0, ~2h)

**Problem:** Phase 1 only supports preflop + custom mode. Flop/Turn/River dropdown options are visible but lead to incomplete flows.

**Change:**
- Add `hiddenStreets?: TrainingMode[]` prop to `StreetSelector.tsx` -- filter `MODE_OPTIONS`, adjust grid columns
- `TrainerLanding.tsx` passes `hiddenStreets={['flop', 'turn', 'river']}`
- Remove postflop pot type auto-correction effects (dead code when postflop is hidden)
- Remove postflop hero position correction effects (same reason)

**Files:** `StreetSelector.tsx`, `TrainerLanding.tsx`

---

### Task 1.2 -- Config Load Error UI `[FIX]` (P0, ~2h)

**Problem:** When `getTrainerGamesConfig()` fails, `trainerConfigError` is set in Redux but the landing page shows no error. User sees a disabled Start button with no explanation. (Gap #1)

**Change:**
- Read `trainerConfigError` from Redux (already selected at line 343)
- Show error banner when `trainerConfigError` is truthy: "Failed to load trainer configuration. Please refresh or try again."
- Add retry button that re-dispatches `getTrainerGamesConfig()`
- Disable all config selectors while config is loading (loading skeleton state)

**Files:** `TrainerLanding.tsx`

---

### Task 1.3 -- Guard Against Empty Spot Discovery `[FIX]` (P0, ~2h)

**Problem:** Spot Discovery returning 0 scenarios disables the Start button, but:
- No distinction between "still loading" vs "loaded but empty"
- No guidance on what to change
- API failure is silent

**Change:**
- Add explicit error UI when `spotDiscovery.error` is set
- Differentiate "loading" (spinner) vs "no strategies" (amber warning with suggestions) vs "API error" (red warning with retry)
- When `scenarios.length === 0` and not loading, show: "No strategies found. Try changing [Site], [Stack], or [Simulation] in Advanced Settings."
- Guard all `spotDiscovery.scenarios` access paths against empty/null

**Files:** `TrainerLanding.tsx`, `useSpotDiscovery.ts` (verify error propagation)

---

### Task 1.4 -- Board Card Count Validation `[FIX]` (P0, ~1.5h)

**Problem:** No validation of `cardSelection` before it reaches `buildCustomConfig()` or the session start API. Malformed board state (e.g., 1 or 2 cards) could cause backend errors. (Gap #2)

**Change:**
- In `buildCustomConfig()` (`sessionConfigBuilder.ts`): validate board card count is 0, 3, 4, or 5 before including `boardCards` in the config. Log warning and omit if invalid.
- In `TrainerLanding.tsx` `handleStartTraining()`: validate board cards before dispatching. Show error if count is invalid (e.g., "Please select 3 cards for flop or clear the board").
- In Custom mode: `CustomSpotBuilder` already enforces via the `BoardCard` component, but add a guard in `handleActionClick` street transition logic.

**Files:** `sessionConfigBuilder.ts`, `TrainerLanding.tsx`, `CustomSpotBuilder.tsx`

---

### Task 1.5 -- Randomize Board Gating `[FIX]` (P1, ~1h)

**Problem:** Randomize Board is shown and editable for preflop-only training where it has no effect. For precision DB with non-check postflop actions, it creates a structural conflict. (Gap #3)

**Change:**
- `AdvancedSettingsModal.tsx`: Add `trainingMode` prop. Hide Randomize Board when `trainingMode === 'preflop'`. Disable when precision DB + postflop non-check actions.
- Force `randomizeBoard: 'Off'` in `buildSessionConfig()` when `trainingMode === 'preflop'`.
- `TrainerLanding.tsx`: Pass `trainingMode` to the modal.

**Files:** `AdvancedSettingsModal.tsx`, `TrainerLanding.tsx`, `sessionConfigBuilder.ts`

---

### Task 1.6 -- Essential Config Values Guard `[FIX]` (P0, ~1.5h)

**Problem:** `buildSessionConfig()` silently defaults to canonical values when Redux state is empty (e.g., URL sync race). This can send wrong config to the backend. (Gap #5)

**Change:**
- In `handleStartTraining()`: Check that essential values are populated in Redux BEFORE calling `onStartTraining()`. If any are missing, show error: "Configuration not ready. Please wait or refresh."
- Essential values: `selectedPlayers`, `selectedCasino`, `selectedBB`, `selectedStack`
- This mirrors the old trainer's `hasEssentialConfigValues` check.

**Files:** `TrainerLanding.tsx`

---

### Task 1.7 -- Fallback Position Validation `[FIX]` (P1, ~2h)

**Problem:** Position validation relies entirely on Spot Discovery API. If API is slow, fails, or returns empty `availablePositions`, there's no fallback. (Gap #6)

**Change:**
- Port the old trainer's deterministic position validation rules from `components/Trainer/utils.ts` (`validateUnopenedSpot`, `validateVsRfiSpot`, etc.) into the new `SpotSelector.tsx` as a fallback.
- When `availableHeroPositions` from Spot Discovery is empty, fall back to these rules.
- The rules are: RFI hero != BB; vsOpen hero needs someone before; vs3Bet hero can't be last; vs4Bet hero needs someone before; squeeze hero needs 2 before.
- Note: `isSpotCompatibleWithHeroes()` already exists in `SpotSelector.tsx` (lines 57-76) with the same rules. Verify it's used as a fallback when discovery data is absent.

**Files:** `SpotSelector.tsx`, `TrainerLanding.tsx`

---

### Task 1.8 -- RTS Integration for Custom Mode `[FIX]` (P1, ~3h)

**Problem:** CustomSpotBuilder doesn't send `realTimeCalc` param. For NLH custom spots going into postflop, only precomputed boards work. The `noSolutionForBoard` message appears but doesn't offer RTS as a fallback. (Gap #7)

**Change:**
- Import `shouldEnableRealTimeCalc` from `utils/gameUtils.ts` into `CustomSpotBuilder.tsx`
- Add `realTimeCalc` to `buildBaseQueryParams()` when conditions are met (variant + board card count)
- When `noSolutionForBoard` is true and RTS is available, show "Computing strategy..." instead of "No precomputed solution" and trigger RTS calculation
- When RTS is not available (PLO preflop/flop/turn), keep current message

**Files:** `CustomSpotBuilder.tsx`, potentially `hooks/rts/` integration

**Note:** This can be scoped down for Phase 1 -- at minimum, add `realTimeCalc` param to the query. Full RTS WebSocket integration can follow.

---

### Task 1.9 -- Dead-End Config Guidance `[FIX]` (P2, ~1.5h)

**Problem:** When config cascade reaches a dead end (all options empty at a layer), the user sees "No strategy data for this configuration" with no guidance on which setting to change. (Gap #8)

**Change:**
- Extend `useTrainerGameConfig` to return `deadEndLayer?: string` indicating which cascade layer has no options (e.g., "site", "bb", "stack")
- In `TrainerLanding.tsx` validation message: "No strategies for [current site] with [current simulation]. Try changing {deadEndLayer} in Advanced Settings."
- Highlight the Settings icon when config is invalid to draw attention to Advanced Settings.

**Files:** `useTrainerGameConfig.ts`, `TrainerLanding.tsx`

---

### Task 1.10 -- Align CustomSpotBuilder Query Params `[FIX]` (P0, ~1.5h)

**Problem:** CustomSpotBuilder's `buildBaseQueryParams()` is missing `BetRepresentation` and per-street bet sizing params that the Strategies page sends. Postflop custom sequences may get different bet options.

**Change:**
- Add `BetRepresentation` to `buildBaseQueryParams()` -- read from Redux `gameReducer.betRepresentation`
- Add per-street bet sizing defaults -- read from Redux or use same defaults as Strategies page
- No new files, no shared utility extraction -- just add the missing params (~10-15 lines)

**Files:** `CustomSpotBuilder.tsx`

---

### Task 1.11 -- Verify Custom Config Pipeline `[VERIFY]` (P0, ~2h)

**What to verify (all existing code):**
- `buildCustomConfig()` joins `selectedActions.Preflop` as comma-separated string
- Flop/Turn/River actions included when present
- Board cards validated and concatenated
- `buildSessionConfig()` sets `trainingMode: 'custom'` + attaches `customConfig`
- `handleStartTraining()` dispatches `setSelectedTrainingMode('custom')`

**Test:** Add `sessionConfigBuilder.test.ts` covering `buildCustomConfig()` and `buildSessionConfig()` for preflop + custom modes with mocked Redux state.

**Files:** `sessionConfigBuilder.ts` (review only), new test file

---

## Summary

| Task | Nature | Gap Addressed | Priority | Effort |
|------|--------|---------------|----------|--------|
| 1.1 Hide postflop modes | HIDE | Phase 1 scope | P0 | 2h |
| 1.2 Config load error UI | FIX | Gap #1: Silent config failure | P0 | 2h |
| 1.3 Empty discovery handling | FIX | Gap #8 partial: No feedback | P0 | 2h |
| 1.4 Board card validation | FIX | Gap #2: No count validation | P0 | 1.5h |
| 1.5 Randomize board gating | FIX | Gap #3: Shown when irrelevant | P1 | 1h |
| 1.6 Essential config guard | FIX | Gap #5: Silent defaults | P0 | 1.5h |
| 1.7 Fallback position rules | FIX | Gap #6: API-only validation | P1 | 2h |
| 1.8 RTS for custom mode | FIX | Gap #7: No RTS in custom | P1 | 3h |
| 1.9 Dead-end config guidance | FIX | Gap #8: No guidance | P2 | 1.5h |
| 1.10 Query param alignment | FIX | Strategy page parity | P0 | 1.5h |
| 1.11 Verify custom pipeline | VERIFY | Confidence | P0 | 2h |

---

## Execution Order

**Day 1 -- P0 (unblock training):**
1. **1.1** Hide postflop modes (unblocks visual testing)
2. **1.2** Config load error UI (users can see what's wrong)
3. **1.6** Essential config guard (prevent bad payloads)
4. **1.10** Query param alignment (Custom mode parity)
5. **1.11** Verify custom pipeline + add tests

**Day 2 -- P0 continued + P1:**
6. **1.3** Empty discovery handling (better feedback)
7. **1.4** Board card validation (defensive)
8. **1.7** Fallback position rules (offline validation)
9. **1.5** Randomize board gating (cleanup)
10. **1.8** RTS for custom mode (at minimum: add param)

**Day 3 -- P2 + polish:**
11. **1.9** Dead-end config guidance (UX polish)
12. Final test pass

---

*Document generated: Feb 19, 2026*
