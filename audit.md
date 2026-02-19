# Trainer V3 Landing Page Revamp — Reviewed

**Target:** Phase 1 of the Trainer V3 Release Plan
**Scope:** Frontend, `components/UnifiedTrainer/` subtree + shared hooks
**Estimated Total Effort:** ~28h (revised from ~20h — see new tasks)

**Review date:** Feb 19, 2026
**Reviewed against:** codebase snapshot + Jamal's screenshot (Slack, 5:03 PM)

---

## Review Verdict

The original plan correctly identifies 8 gaps between the old Trainer V2 and the new UnifiedTrainer landing page. All 8 were verified in the codebase. However, **the plan misses 5 additional issues** — one of which (table display parity) is the primary concern from the screenshot. The plan also has 2 factual corrections and 1 task that should be redirected to use existing code.

Changes are marked:

- `[VERIFIED]` — plan item confirmed in codebase as described
- `[CORRECTED]` — plan item is right in spirit but has factual inaccuracies
- `[NEW]` — gap not in the original plan
- `[REDIRECTED]` — task should use different approach than described

---

## Why This Is a Revamp

The new UnifiedTrainer landing page was built with an optimistic design: it relies on Spot Discovery API + config cascade auto-correction to validate whether training can start. This is architecturally cleaner than the old trainer, but it **dropped critical edge-case handling** that the old Trainer V2 had.

The old trainer (`components/Trainer/useTrainer.tsx`, `utils.ts`, `TrainerModalV2.tsx`) guarded against 7 categories of failure that the new landing page does not:

### 1. No Error UI When Config Fails to Load `[VERIFIED]`

**Old:** `waitForTrainerConfig()` polls up to 5 seconds for `researchBasedData` and blocks start if it fails. Shows explicit error.

**New:** `trainerConfigError` is selected from Redux at line 342 (`error: trainerConfigError`) and used only as a fetch guard (line 355: `!trainerConfigError`). **It is never rendered**. The user sees a disabled Start button with no explanation.

### 2. No Board Card Count Validation `[VERIFIED]`

**Old:** Validates board cards are 0, 3, 4, or 5 before any API call. Invalid counts block the request.

**New:** `buildCustomConfig()` validates card _format_ (regex + duplicate check) but **not count**. A board with 1, 2, or 4 cards when street expects 3 would pass through.

### 3. Randomize Board Not Properly Gated `[CORRECTED]`

**Old:** Disables "Randomize Board" for preflop-only training (forced Off) and precision DB with non-check postflop actions.

**New:** `AdvancedSettingsModal.tsx` renders `RandomizeBoardSelector` with `disabled={false}` **hardcoded** (line 513). There is no `isPreflopStreet` check, no `shouldDisableRandomizeBoard` logic. The original plan stated the modal "shows Randomize Board unconditionally" — this is correct, but it also claimed the modal had some gating code. **It does not.** The fix is slightly larger than estimated.

### 4. No Config Rejection Recovery `[VERIFIED]`

**Old:** `handlePlayerActionsRejected()` catches backend 400/500 errors, resets all selectors, and sets `isInvalidConfig` flag.

**New:** Session start errors are shown in an error banner (`errorMessage` prop), but there is no automatic reset-to-defaults flow. User must manually change settings to recover.

### 5. No Essential Config Values Guard `[VERIFIED]`

**Old:** Checks `selectedCasino`, `selectedBB`, `selectedPlayers`, `selectedStack` exist before making API calls. Skips if any missing.

**New:** `buildSessionConfig()` falls back to `CANONICAL_TRAINER_DEFAULTS` with a console warning (line 283-293). This silently sends potentially wrong config (e.g., default 6-player when user intended 2-player but URL sync hadn't completed).

### 6. Position Validation Is Purely API-Dependent `[CORRECTED — existing hook unused]`

**Old:** Has explicit spot-specific position validation functions (`validateUnopenedSpot()`, `validateVsRfiSpot()`, etc.) in `components/Trainer/utils.ts`.

**New (plan says):** Relies entirely on Spot Discovery API.

**Actual:** A comprehensive `usePositionValidation` hook **already exists** at `components/UnifiedTrainer/hooks/usePositionValidation.ts` (267 lines) with rules for `rfi`, `vsOpen`, `vs3Bet`, `vs4Bet`, `squeeze`, `Any`. It returns `validHeroPositions`, `validVillainPositions`, `isValid`, and `errors[]`. **It is not wired into `TrainerLanding.tsx` or `SpotSelector.tsx`.** The task should wire this existing hook, not port the old `utils.ts` code.

### 7. RTS Not Integrated in Custom Mode `[VERIFIED]`

**Old (Strategies page):** Sends `realTimeCalc=true` when conditions are met.

**New (CustomSpotBuilder):** Does not send `realTimeCalc`. The `noSolutionForBoard` handling exists (line 707-720 in `CustomSpotBuilder.tsx`) but offers no RTS fallback.

### 8. Complete Dead-End Config Not Communicated `[VERIFIED]`

The config cascade in `useTrainerGameConfig` can reach a state where ALL options at a cascade layer are empty. `isCurrentConfigValid` is false, Start is disabled, but no guidance on _which_ setting to change.

---

## Gaps Not In The Original Plan

### 9. Table Display Parity During Gameplay `[NEW — P0]`

**Source:** Jamal's screenshot (Slack, 5:03 PM):
> "The trainer v3 landing chips, stack, cards and pot should match the strategies at any given point."

**Screenshot shows:** A 5-player table (BB: 99 BB, SB: 64.5 BB, Hero: 64.5 BB, BTN: 0 BB, CO: 100 BB, Pot: 136.5 BB, Board: K♠ Q♥ 7♣). BTN at 0 BB is suspicious — suggests stacks aren't being calculated the same way as the Strategies page.

**Root cause:** Two different data pipelines:

| Aspect | Trainer V3 | Strategies Page |
|--------|-----------|-----------------|
| **Pot** | Backend-provided `hand.potSizeBB` (raw) | Frontend-calculated: `sum(player.Chips[])` |
| **Stacks** | Backend-provided `hand.players[].stack` (raw) | Frontend-calculated: `StartingStack - totalChipsUsed` |
| **Source** | `v3HandAdapter.ts` → `TrainerTable2D` → `PokerTable2D` | `TrainerDataAdapter.ts` → `UnifiedPokerTable` |

The Trainer V3 trusts backend values directly. The Strategies page computes from chip arrays. If the backend sends stacks that represent different points in the hand (e.g., starting stacks vs current stacks after posting blinds), or if non-active players get 0 BB, the display diverges from the strategy page.

**This is not a landing page issue — it's a gameplay table issue.** The plan only covers landing page configuration; it does not address the gameplay rendering pipeline at all.

### 10. No Daily Limit Handling `[NEW — P1]`

**Old:** `useTrainer.tsx` has `handleDailyLimitReached()` (lines 111-135) that checks subscription-tier hand limits and shows `TrainerLimitModal` / `DailyLimitModal`.

**New:** Zero daily limit logic in `UnifiedTrainer/`. No references to daily limits, no modal integration. Free-tier users could exceed their hand quota without any frontend guard.

**Files:** `components/Modals/TrainerLimitModal.tsx`, `components/Modals/DailyLimitModal.tsx` (both exist, just not wired).

### 11. No Session Buffer Depletion Guard `[NEW — P1]`

**Old:** `useTrainer.tsx` validates `handBufferRef.current.length > 0` before consuming a hand. If the buffer is empty, it waits or generates more.

**New:** `useTrainingSession.ts` has buffer refill logic (`generateHands()`), but if `startTrainerSession()` returns a valid `sessionId` with 0 hands in the initial buffer, the UI enters a broken state — the session appears started but no hand is displayed.

The tracer detects this (stage 3: `handsCount === 0`), but the UI has no recovery path.

### 12. No Config Drift Detection `[NEW — P2]`

**Old:** `getConfigKey()` + `checkAndClearBufferIfConfigChanged()` detect when config changes mid-session and clear the hand buffer.

**New:** No mechanism to detect config changes during an active session. If URL params change while training (e.g., back-button navigation or shared link), stale hands from the old config may be served.

### 13. No Transient Failure Retry `[NEW — P2]`

**`useSpotDiscovery`:** No retry logic on network failures. 300ms debounce + 30s cache but no resilience.

**`startTrainerSession`:** Returns `null` on error with no retry. A single network timeout means the user must manually retry.

These are infrastructure-level gaps. Not landing-page-specific but affect the user experience during config and session start.

---

## Task Breakdown (Revised)

### Task 1.1 — Hide Standard Postflop Modes `[HIDE]` (P0, ~2h) `[VERIFIED]`

**Problem:** Phase 1 only supports preflop + custom mode. Flop/Turn/River dropdown options are visible but lead to incomplete flows.

**Change:**
- Add `hiddenStreets?: TrainingMode[]` prop to `StreetSelector.tsx` — filter `MODE_OPTIONS`, adjust grid columns
- `TrainerLanding.tsx` passes `hiddenStreets={['flop', 'turn', 'river']}`
- Remove postflop pot type auto-correction effects (dead code when postflop is hidden)
- Remove postflop hero position correction effects (same reason)

**Files:** `StreetSelector.tsx`, `TrainerLanding.tsx`

---

### Task 1.2 — Config Load Error UI `[FIX]` (P0, ~2h) `[VERIFIED]`

**Problem:** When `getTrainerGamesConfig()` fails, `trainerConfigError` is set in Redux but the landing page shows no error. User sees a disabled Start button with no explanation. (Gap #1)

**Change:**
- Read `trainerConfigError` from Redux (already selected at line 342)
- Show error banner when `trainerConfigError` is truthy: "Failed to load trainer configuration. Please refresh or try again."
- Add retry button that re-dispatches `getTrainerGamesConfig()`
- Disable all config selectors while config is loading (loading skeleton state)

**Files:** `TrainerLanding.tsx`

---

### Task 1.3 — Guard Against Empty Spot Discovery `[FIX]` (P0, ~2h) `[VERIFIED]`

**Problem:** Spot Discovery returning 0 scenarios disables the Start button, but no distinction between "still loading" vs "loaded but empty" vs "API error".

**Change:**
- Add explicit error UI when `spotDiscovery.error` is set
- Differentiate "loading" (spinner) vs "no strategies" (amber warning with suggestions) vs "API error" (red warning with retry)
- When `scenarios.length === 0` and not loading, show: "No strategies found. Try changing [Site], [Stack], or [Simulation] in Advanced Settings."
- Guard all `spotDiscovery.scenarios` access paths against empty/null

**Files:** `TrainerLanding.tsx`, `useSpotDiscovery.ts` (verify error propagation)

---

### Task 1.4 — Board Card Count Validation `[FIX]` (P0, ~1.5h) `[VERIFIED]`

**Problem:** No validation of `cardSelection` count before it reaches `buildCustomConfig()` or the session start API. (Gap #2)

**Change:**
- In `buildCustomConfig()` (`sessionConfigBuilder.ts`): validate board card count is 0, 3, 4, or 5 before including `boardCards` in the config. Log warning and omit if invalid.
- In `TrainerLanding.tsx` `handleStartTraining()`: validate board cards before dispatching. Show error if count is invalid.
- In Custom mode: add guard in `CustomSpotBuilder` `handleActionClick` street transition logic.

**Files:** `sessionConfigBuilder.ts`, `TrainerLanding.tsx`, `CustomSpotBuilder.tsx`

---

### Task 1.5 — Randomize Board Gating `[FIX]` (P1, ~1.5h) `[CORRECTED — slightly larger]`

**Problem:** Randomize Board has `disabled={false}` hardcoded. It's always visible and always enabled, even for preflop-only training. (Gap #3)

**Change:**
- `AdvancedSettingsModal.tsx`: Accept `trainingMode` prop. Compute `isPreflopOnly = trainingMode === 'preflop'`. Pass `disabled={isPreflopOnly}` to `RandomizeBoardSelector`. Add tooltip "Not applicable for preflop training" when disabled.
- Force `randomizeBoard: 'Off'` in `buildSessionConfig()` when `trainingMode === 'preflop'`.
- `TrainerLanding.tsx`: Pass `trainingMode` to the modal.

**Files:** `AdvancedSettingsModal.tsx`, `TrainerLanding.tsx`, `sessionConfigBuilder.ts`

---

### Task 1.6 — Essential Config Values Guard `[FIX]` (P0, ~1.5h) `[VERIFIED]`

**Problem:** `buildSessionConfig()` silently defaults to canonical values when Redux state is empty. (Gap #5)

**Change:**
- In `handleStartTraining()`: check that `selectedPlayers`, `selectedCasino`, `selectedBB`, `selectedStack` are populated in Redux BEFORE calling `onStartTraining()`. If any are missing, show error: "Configuration not ready. Please wait or refresh."
- This mirrors the old trainer's `hasEssentialConfigValues` check.

**Files:** `TrainerLanding.tsx`

---

### Task 1.7 — Wire usePositionValidation Hook `[FIX]` (P1, ~1.5h) `[REDIRECTED]`

**Problem:** Position validation relies entirely on Spot Discovery API. If API is slow, fails, or returns empty, there's no fallback. (Gap #6)

**Original plan:** Port old trainer's validation rules from `components/Trainer/utils.ts`.

**Redirected approach:** `usePositionValidation` hook already exists at `components/UnifiedTrainer/hooks/usePositionValidation.ts` with comprehensive rules for all spot types (rfi, vsOpen, vs3Bet, vs4Bet, squeeze). It returns `validHeroPositions`, `validVillainPositions`, `isValid`, and `errors[]`. **It is built but not wired.**

**Change:**
- In `SpotSelector.tsx` or `TrainerLanding.tsx`: call `usePositionValidation()` with current spot + player count.
- When `availableHeroPositions` from Spot Discovery is empty or the API hasn't returned, fall back to `usePositionValidation` results.
- When both sources agree a position is invalid, dim it on the MiniTable.

**Effort reduced:** No porting needed — just wiring + fallback logic (~1.5h instead of 2h).

**Files:** `SpotSelector.tsx`, `TrainerLanding.tsx`

---

### Task 1.8 — RTS Integration for Custom Mode `[FIX]` (P1, ~3h) `[VERIFIED]`

**Problem:** CustomSpotBuilder doesn't send `realTimeCalc` param. (Gap #7)

**Change:**
- Import `shouldEnableRealTimeCalc` from `utils/gameUtils.ts` into `CustomSpotBuilder.tsx`
- Add `realTimeCalc` to `buildBaseQueryParams()` when conditions are met
- When `noSolutionForBoard` is true and RTS is available, show "Computing strategy..." and trigger RTS
- When RTS is not available (PLO preflop/flop/turn), keep current message

**Phase 1 scope:** At minimum, add `realTimeCalc` param to the query. Full RTS WebSocket integration can follow.

**Files:** `CustomSpotBuilder.tsx`, potentially `hooks/rts/` integration

---

### Task 1.9 — Dead-End Config Guidance `[FIX]` (P2, ~1.5h) `[VERIFIED]`

**Problem:** When config cascade reaches a dead end, "No strategy data for this configuration" gives no guidance on which setting to change. (Gap #8)

**Change:**
- Extend `useTrainerGameConfig` to return `deadEndLayer?: string` indicating which cascade layer has no options
- In `TrainerLanding.tsx` validation message: "No strategies for [current site] with [current simulation]. Try changing {deadEndLayer} in Advanced Settings."
- Highlight the Settings icon when config is invalid

**Files:** `useTrainerGameConfig.ts`, `TrainerLanding.tsx`

---

### Task 1.10 — Align CustomSpotBuilder Query Params `[FIX]` (P0, ~1.5h) `[VERIFIED]`

**Problem:** CustomSpotBuilder's `buildBaseQueryParams()` is missing `BetRepresentation` and per-street bet sizing params that the Strategies page sends.

**Change:**
- Add `BetRepresentation` to `buildBaseQueryParams()` — read from Redux `gameReducer.betRepresentation`
- Add per-street bet sizing defaults
- No new files — just add missing params (~10-15 lines)

**Confirmed:** `BetRepresentation` grep returned 0 results in `components/UnifiedTrainer/`. It's entirely missing.

**Files:** `CustomSpotBuilder.tsx`

---

### Task 1.11 — Verify Custom Config Pipeline `[VERIFY]` (P0, ~2h) `[VERIFIED]`

**What to verify:**
- `buildCustomConfig()` joins `selectedActions.Preflop` as comma-separated string
- Flop/Turn/River actions included when present
- Board cards validated and concatenated
- `buildSessionConfig()` sets `trainingMode: 'custom'` + attaches `customConfig`
- `handleStartTraining()` dispatches `setSelectedTrainingMode('custom')`

**Test:** Add `sessionConfigBuilder.test.ts` covering `buildCustomConfig()` and `buildSessionConfig()` for preflop + custom modes.

**Files:** `sessionConfigBuilder.ts` (review only), new test file

---

### Task 1.12 — Table Display Parity `[FIX]` (P0, ~3h) `[NEW]`

**Source:** Jamal's screenshot — stacks/pot/chips on the gameplay table don't match what the Strategies page shows for the same spot.

**Root cause:** Two divergent data pipelines:

| Pipeline | Pot | Stacks |
|----------|-----|--------|
| **Trainer V3** | `hand.potSizeBB` (backend raw) | `hand.players[].stack` (backend raw) |
| **Strategies** | `sum(player.Chips[])` (frontend calc) | `StartingStack - totalChipsUsed` (frontend calc) |

**Suspected issues from screenshot:**
- BTN at 0 BB → backend may be sending remaining stack after posting/betting, or position is marked inactive
- Pot 136.5 BB but sum of visible stacks doesn't account for it → blind-posting and bet-posting may differ between pipelines

**Change:**
- Audit `v3HandAdapter.ts` `adaptV3HandToTrainerHand()` — verify that `potSizeBB` and `players[].stack` represent the same game state that the Strategies page displays (current pot + remaining stacks, not effective stacks)
- Compare with `TrainerDataAdapter.ts` (`UnifiedPokerPanel/adapters/`) calculation logic: `StartingStack - totalChipsUsed` for stacks, `sum(Chips[])` for pot
- If backend values differ from strategy-page values: add frontend normalization in the adapter to match strategy-page conventions
- Verify that non-active players (folded or not in hand) show stack correctly (not 0 BB)
- Add test cases to `v3HandAdapter.test.ts` comparing adapter output against expected strategy-page-equivalent values

**Files:** `adapters/v3HandAdapter.ts`, `Training/Table/TrainerTable2D.tsx`, `Training/TrainingTableInstance.tsx`

---

### Task 1.13 — Daily Limit Handling `[FIX]` (P1, ~1.5h) `[NEW]`

**Problem:** Old trainer checks subscription-tier daily hand limits via `handleDailyLimitReached()` and shows `TrainerLimitModal` / `DailyLimitModal`. New trainer has zero daily limit logic.

**Impact:** Free-tier users can exceed their hand quota without frontend guard. (Backend may enforce, but UX is a hard stop with no explanation.)

**Change:**
- Wire `TrainerLimitModal` or `DailyLimitModal` (both exist in `components/Modals/`) into the UnifiedTrainer flow
- Check daily limit before session start in `handleStartTraining()`
- Show upgrade modal when limit is reached
- Optionally: show remaining hands count in the UI

**Files:** `TrainerLanding.tsx`, import from `components/Modals/`

---

### Task 1.14 — Session Buffer Empty State `[FIX]` (P1, ~1h) `[NEW]`

**Problem:** If `startTrainerSession()` returns a valid `sessionId` but 0 hands in the initial buffer, the UI enters a broken state — session appears started but no hand is displayed.

**Change:**
- In `useTrainingSession.ts`: after session start, check `hands.length > 0`. If 0, show error: "No hands could be generated for this configuration. Try different settings."
- Do not transition to gameplay view if buffer is empty
- The tracer already captures this (stage 3: `handsCount === 0`) — surface it to the user

**Files:** `useTrainingSession.ts`, `TrainerLanding.tsx` (error display)

---

## Summary

| Task | Nature | Gap Addressed | Priority | Effort | Status |
|------|--------|---------------|----------|--------|--------|
| 1.1 Hide postflop modes | HIDE | Phase 1 scope | P0 | 2h | Verified |
| 1.2 Config load error UI | FIX | Gap #1: Silent config failure | P0 | 2h | Verified |
| 1.3 Empty discovery handling | FIX | Gap #8 partial: No feedback | P0 | 2h | Verified |
| 1.4 Board card validation | FIX | Gap #2: No count validation | P0 | 1.5h | Verified |
| 1.5 Randomize board gating | FIX | Gap #3: Shown when irrelevant | P1 | 1.5h | Corrected |
| 1.6 Essential config guard | FIX | Gap #5: Silent defaults | P0 | 1.5h | Verified |
| 1.7 Wire position validation | FIX | Gap #6: API-only validation | P1 | 1.5h | Redirected |
| 1.8 RTS for custom mode | FIX | Gap #7: No RTS in custom | P1 | 3h | Verified |
| 1.9 Dead-end config guidance | FIX | Gap #8: No guidance | P2 | 1.5h | Verified |
| 1.10 Query param alignment | FIX | Strategy page parity | P0 | 1.5h | Verified |
| 1.11 Verify custom pipeline | VERIFY | Confidence | P0 | 2h | Verified |
| **1.12 Table display parity** | **FIX** | **Gap #9: Screenshot issue** | **P0** | **3h** | **New** |
| **1.13 Daily limit handling** | **FIX** | **Gap #10: No quota guard** | **P1** | **1.5h** | **New** |
| **1.14 Buffer empty state** | **FIX** | **Gap #11: Broken start** | **P1** | **1h** | **New** |

**Deferred (P2, infrastructure-level):**

| Issue | Gap | Rationale for Deferral |
|-------|-----|----------------------|
| Config drift detection (Gap #12) | Mid-session config change | Low probability in Phase 1 (no postflop URL params) |
| Transient failure retry (Gap #13) | No network resilience | Refresh is acceptable MVP workaround |

---

## Revised Execution Order

### Day 1 — P0 (unblock training + screenshot fix):
1. **1.12** Table display parity (Jamal's screenshot — stakeholder-reported, highest visibility)
2. **1.1** Hide postflop modes (unblocks visual testing)
3. **1.2** Config load error UI (users can see what's wrong)
4. **1.6** Essential config guard (prevent bad payloads)
5. **1.10** Query param alignment (Custom mode parity)

### Day 2 — P0 continued + P1:
6. **1.11** Verify custom pipeline + add tests
7. **1.3** Empty discovery handling (better feedback)
8. **1.4** Board card validation (defensive)
9. **1.14** Buffer empty state (prevent broken start)
10. **1.13** Daily limit handling (subscription guard)

### Day 3 — P1 continued + P2:
11. **1.7** Wire position validation hook (offline validation)
12. **1.5** Randomize board gating (cleanup)
13. **1.8** RTS for custom mode (at minimum: add param)
14. **1.9** Dead-end config guidance (UX polish)
15. Final test pass

---

## Key Corrections From Review

1. **Task 1.5** — `AdvancedSettingsModal` has `disabled={false}` hardcoded on `RandomizeBoardSelector`. There is no `isPreflopStreet` variable, no `shouldDisableRandomizeBoard` logic. The fix is slightly larger than originally scoped (+0.5h).

2. **Task 1.7** — `usePositionValidation` hook already exists at `hooks/usePositionValidation.ts` with full spot-type validation. Do NOT port code from `components/Trainer/utils.ts`. Wire the existing hook instead. (-0.5h effort).

3. **Table display parity (Task 1.12)** — This is the screenshot's actual concern. The original plan covers only landing-page config; the screenshot shows a _gameplay_ table display mismatch. The data flows through `v3HandAdapter.ts` → `TrainerTable2D` → `PokerTable2D`, using raw backend values that may diverge from the Strategies page's frontend-calculated values.

---

*Document generated: Feb 19, 2026*
*Reviewed by: codebase cross-reference against plan + screenshot analysis*
