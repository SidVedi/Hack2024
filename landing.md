# Trainer V3 Landing Page Revamp

**Target:** Phase 1 of the Trainer V3 Release Plan (tasks 1.1-1.7)
**Scope:** Frontend-only, scoped to `components/UnifiedTrainer/` subtree
**Estimated Total Effort:** ~15h

---

## Current State

- **StreetSelector** (`components/UnifiedTrainer/Configuration/StreetSelector.tsx`) renders all 5 modes (Preflop, Flop, Turn, River, Custom). It supports a `disabledStreets` prop but has no mechanism to *hide* options entirely.
- **SpotSelector** (`components/UnifiedTrainer/Configuration/SpotSelector.tsx`) shows both preflop and postflop spots based on `SPOTS_BY_DATABASE`. Postflop spots (SRP, 3BP, 4BP) should not appear in Phase 1.
- **TrainerLanding** (`components/UnifiedTrainer/TrainerLanding.tsx`) drives the full config flow. It already computes `disabledStreets` from the database type (line 685) but does not hide streets outright.
- **AdvancedSettingsModal** (`components/UnifiedTrainer/Configuration/AdvancedSettingsModal.tsx`) shows Board Filter, Randomize Board, and Difficulty regardless of training mode.
- **CustomSpotBuilder** (`components/UnifiedTrainer/Configuration/CustomSpot/CustomSpotBuilder.tsx`) is functionally complete -- fetches real strategy tree actions and builds sequences. However, its `buildBaseQueryParams()` diverges from the Strategies page's query building (`getSpinsGameConfigPayload` in `useGameTypeStrategiesPage.ts`) -- see Task 1.7.
- **sessionConfigBuilder** (`components/UnifiedTrainer/utils/sessionConfigBuilder.ts`) has `buildCustomConfig()` reading from `playerActionsReducer` and `buildSessionConfig()` reading from Redux.

---

## Strategy Page / Custom Mode Parity Analysis

Both the Strategies page and CustomSpotBuilder call the same API (`/get-player-next-actions`) and use the same Redux slice (`playerActionsReducer`). The critical difference is **how they build query params**:

| Param | Strategies (`useGameTypeStrategiesPage`) | CustomSpotBuilder (`buildBaseQueryParams`) | Impact |
|-------|---|---|---|
| Core config (variant, numPlayers, stackInBB, BB, site, blindStructure, openRaise, research) | Yes | Yes | Aligned |
| 3bet / 4bet | Yes | Yes | Aligned |
| `selectedActionQuery()` | Yes | Yes | Aligned (same utility) |
| boardCards | Yes | Yes (added separately) | Aligned |
| BetRepresentation | Yes | **Missing** | Affects display format of bet sizes |
| Per-street bet sizing (flopIPBets, flopIPRaises, etc.) | Yes | **Missing** | Could affect which bet options backend returns for postflop |
| realTimeCalc (RTS) | Yes | **Missing** | RTS not needed for trainer, safe to omit |

The missing params don't affect preflop actions, but **do affect postflop** custom sequences. Task 1.7 addresses this.

---

## Task 1.1 -- Hide Standard Postflop Modes (P0, ~2h)

**Goal:** Users see only "Preflop" and "Custom" in the street selector.

**Approach:** Add a `hiddenStreets` prop to `StreetSelector` (distinct from `disabledStreets`) and filter `MODE_OPTIONS` before rendering. In `TrainerLanding`, pass `hiddenStreets={['flop', 'turn', 'river']}`.

**Files:**

- `components/UnifiedTrainer/Configuration/StreetSelector.tsx` -- add `hiddenStreets?: TrainingMode[]` prop; filter `MODE_OPTIONS` to exclude hidden items; adjust grid columns dynamically (`grid-cols-${visibleCount}`)
- `components/UnifiedTrainer/TrainerLanding.tsx` -- pass `hiddenStreets` to `StreetSelector`

**Why not just filter MODE_OPTIONS statically?** A prop keeps StreetSelector reusable and makes re-enabling postflop in Phase 2 a one-line change (remove the prop).

---

## Task 1.3 -- Fix Crash on Empty Spot Discovery (P0, ~2h)

**Goal:** Landing page does not crash when Spot Discovery returns 0 scenarios.

**Current behavior:** `spotDiscovery.scenarios` can be empty when the backend has no strategies for the current config. The code already handles this for the Start button gate (`isSelectedSpotAvailable`) but there may be unguarded `.find()` / `.map()` calls.

**Approach:**

- Audit `TrainerLanding.tsx` for every access to `spotDiscovery.scenarios` -- add null/empty guards
- Audit `getDiscoveryAvailablePositions()` -- already returns `[]` on empty, which is safe
- Audit `discoveryToFrontendSpotSet()` -- verify it returns `null` (not an empty Set) for empty scenarios so the "no constraint" fallback works
- Add a user-visible fallback message when no scenarios are available ("No strategies found for this configuration")

**Files:**

- `components/UnifiedTrainer/TrainerLanding.tsx` -- add guards + fallback UI
- `components/UnifiedTrainer/hooks/useTrainerLandingConfig.ts` -- ensure no crash if the auto-correction effects receive empty data
- `components/UnifiedTrainer/utils/spotDiscoveryMapping.ts` -- verify edge case behavior

**Test:** Add a pure-logic test (like the existing `TrainerLanding.configValidation.test.ts` pattern) that verifies no exception is thrown when scenarios are `[]`.

---

## Task 1.7 -- Align Custom Mode with Strategies Page API Flow (P0, ~3h)

**Goal:** CustomSpotBuilder must produce the **exact same API query** as the Strategies page for the same configuration, so users get identical action options in both places.

**Problem:** CustomSpotBuilder's `buildBaseQueryParams()` builds a simpler query than the Strategies page's `getSpinsGameConfigPayload()`. Both call `/get-player-next-actions` but with different params -- which means the backend could return different available actions (especially for postflop bet sizing).

**Approach: Extract shared query builder**

1. Create a shared utility `buildStrategyQueryParams(config)` that both the Strategies page and CustomSpotBuilder can call. This replaces CustomSpotBuilder's `buildBaseQueryParams()` with the same logic the Strategies page uses.
2. The shared function takes a config object with: `variant`, `numPlayers`, `stackInBB`, `BB`, `game`, `site`, `blindStructure`, `openRaise`, `research`, `3bet`, `4bet`, plus optional `BetRepresentation` and bet sizing params.
3. CustomSpotBuilder calls the shared builder, passing its props. The Strategies page refactors to also call it (or we extract from `getSpinsGameConfigPayload` -- whichever is smaller diff).

**Key params to add to CustomSpotBuilder:**

- `BetRepresentation` -- read from Redux `gameReducer.betRepresentation` (like Strategies does)
- Per-street bet sizing -- read from Redux `trainerReducer` or use defaults matching what Strategies sends

**Files:**

- New: `utils/buildStrategyQueryParams.ts` (or co-locate in `components/PlayerActions/utils.ts`)
- `components/UnifiedTrainer/Configuration/CustomSpot/CustomSpotBuilder.tsx` -- replace `buildBaseQueryParams()` with shared builder
- `hooks/useGameTypeStrategiesPage.ts` -- verify parity (minimal change -- just confirm the shared builder matches)

**Test:** Unit test that both the old Strategies query and the new shared builder produce identical param strings for the same input config.

---

## Task 1.2 -- Verify Custom Mode End-to-End (P0, ~3h)

**Goal:** Confirm Custom mode works from landing through session start, using the shared query builder from Task 1.7.

**Approach:** Manual + automated verification of the CustomSpotBuilder flow:

- CustomSpotBuilder loads initial actions from `/get-player-next-actions` using the shared query builder
- User builds action sequence (fold/call/raise per position)
- Board card selection works for postflop-capable databases
- "Start Training" dispatches correct Redux state and `buildCustomConfig()` produces valid `customConfig`
- **Parity check**: Same config in Strategies page and Custom mode produce the same initial actions from the API

**Files to verify/fix:**

- `components/UnifiedTrainer/Configuration/CustomSpot/CustomSpotBuilder.tsx` -- uses shared query builder (from 1.7)
- `components/UnifiedTrainer/utils/sessionConfigBuilder.ts` -- verify `buildCustomConfig()` reads `selectedActions` and `cardSelection` correctly
- `components/UnifiedTrainer/TrainerLanding.tsx` -- verify `handleStartTraining()` dispatches all required Redux state for custom mode (lines 727-829)

**Test:** Add a unit test for `buildCustomConfig()` with mocked Redux state containing preflop + flop actions and board cards.

---

## Task 1.6 -- Verify Custom Config to Session Config Mapping (P0, ~2h)

**Goal:** `buildCustomConfig()` correctly maps CustomSpotBuilder state to `TrainerSessionConfig.customConfig`.

**Verification checklist:**

- `selectedActions.Preflop` actions are joined as comma-separated string (e.g., `"r35,c,f"`)
- `selectedActions.Flop/Turn/River` are included when present
- `cardSelection` board cards are concatenated and validated (regex + duplicate check)
- `buildSessionConfig()` sets `trainingMode: 'custom'` and attaches `customConfig`
- `handleStartTraining()` in TrainerLanding dispatches `setSelectedTrainingMode('custom')` -- confirmed at line 816

**Files:**

- `components/UnifiedTrainer/utils/sessionConfigBuilder.ts` -- review `buildCustomConfig()` (lines 364-414)
- Add unit tests for `buildCustomConfig()` and `buildSessionConfig()` with custom mode Redux state

---

## Task 1.4 -- Validate Player Count + Spot Combinations (P1, ~1h)

**Goal:** Spots that require more players than currently selected are properly disabled.

**Current behavior:** `SpotSelector` already has `minPlayers` enforcement and `isSpotCompatibleWithHeroes()` validation. This task is mostly verification + edge-case hardening.

**Approach:**

- Verify `squeeze` (minPlayers: 3) is disabled at 2 players -- already enforced via `SPOT_REGISTRY`
- Verify `setPlayerCount` in `useTrainerLandingConfig.ts` auto-switches from `squeeze` to `vsOpen` when count drops below 3 -- already implemented (line 363)
- Test: spot buttons at 2-player count show `squeeze` as disabled with correct tooltip

**Files:**

- `components/UnifiedTrainer/Configuration/SpotSelector.tsx` -- verify, likely no changes needed
- Existing test in `TrainerLanding.configValidation.test.ts` -- extend with player count validation cases

---

## Task 1.5 -- Hide Irrelevant Advanced Settings (P2, ~2h)

**Goal:** Advanced Settings modal only shows settings that matter for preflop/custom training.

**What to hide:**

- **Board Filter panel** -- already conditionally shown via `isPostflop` prop, but should also be hidden when `trainingMode === 'custom'` and no board cards are selected (currently guarded by `onBoardFilterChange` existence)
- **Randomize Board** -- irrelevant for preflop-only training; show only when training mode could produce boards
- **Difficulty** -- keep (applies to all modes)
- **Simulation / Site / BB / Stack / Blinds / Open Raise** -- keep (all affect strategy lookup)

**Files:**

- `components/UnifiedTrainer/Configuration/AdvancedSettingsModal.tsx` -- add `trainingMode` prop; conditionally render Board Filter and Randomize Board
- `components/UnifiedTrainer/TrainerLanding.tsx` -- pass `trainingMode` to the modal

---

## Test Plan Summary

- Extend `TrainerLanding.configValidation.test.ts` with empty-discovery and player-count edge cases
- Add `sessionConfigBuilder.test.ts` covering `buildCustomConfig()` and `buildSessionConfig()` for preflop and custom modes
- Add `buildStrategyQueryParams.test.ts` verifying parity with Strategies page query output
- All tests use MSW for API mocking per project conventions
- Run `pnpm test -- --testPathPattern="UnifiedTrainer"` to verify

---

## Execution Order

1. **1.1** (Hide postflop) -- unblocks visual testing of the landing page
2. **1.3** (Empty discovery guards) -- defensive, prevents crashes during later testing
3. **1.7** (Strategy parity) -- extract shared query builder before verifying Custom mode
4. **1.2** (Custom mode E2E) -- verify with the shared builder in place
5. **1.6** (Config mapping) -- verify session config pipeline
6. **1.4** (Player/spot validation) -- P1 hardening
7. **1.5** (Advanced settings cleanup) -- P2 polish
8. **Tests** -- throughout, but final pass at the end

---

*Document generated: Feb 19, 2026*
