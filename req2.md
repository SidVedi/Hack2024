# Stream A: Trainer Values Stabilization — Proposal Plan

**Owner:** Siddharth Vedi (svedi@algosoftware.io)  
**Stream:** A — Trainer Values Inconsistent  
**Scope:** 9 bugs (A-1 through A-9), frontend and backend  
**Variants:** Holdem and Omaha  
**Timeline:** 3 weeks  
**Date:** February 9, 2026

---

## 1. Executive Summary

Users reported inconsistent trainer values — EV, strategies, combos missing, tables showing incorrect data. After auditing the Next.js frontend and Django/DRF backend, the root causes fall into two categories:

1. **V1/V2 API divergence** — Two backend code paths decode ranges, EV, and pot sizes through different logic, producing different results for the same poker spot.
2. **Frontend calculation drift** — The frontend independently recalculates `regretPercent` using a fundamentally different formula than the backend, causing non-debug users to see wrong feedback labels.

**Approach:** Quick wins first (stop serving known-wrong data), then root cause investigation, then polish — with test-first development at every phase.

---

## 2. Root Cause Analysis

### How the Trainer Works

The trainer has two API code paths feeding the same frontend layout:

```
┌──────────────────┐         ┌──────────────────────────────┐
│  Standalone       │         │  Strategy Page Entry          │
│  Trainer          │         │  (via Train button)           │
└────────┬──────────┘         └──────────────┬────────────────┘
         │                                   │
         ▼                                   ▼
┌──────────────────┐         ┌──────────────────────────────┐
│  V1 API           │         │  V2 API                       │
│  decode_range()   │         │  StrategyTreeNodeResolver     │
│  direct from node │         │  + expand + swaps             │
└────────┬──────────┘         └──────────────┬────────────────┘
         │                                   │
         └──────────────┬────────────────────┘
                        ▼
            ┌───────────────────────┐
            │  Shared hand response │
            │  EV / regret / pot    │
            └───────────┬───────────┘
                        ▼
            ┌───────────────────────┐
            │  Frontend (V3 layout) │
            │  6 files with regret  │
            │  calculation logic    │
            └───────────────────────┘
```

**Backend divergence:** V1 does a raw range decode. V2 routes through the resolver, which applies sparse index expansion, action tree walking, and isomorphism swaps — none of which V1 does.

**Frontend drift:** The backend computes regret as `(best_ev - strategy_weighted_ev) / pot × 100`. The frontend (for non-debug users) computes `(best_ev - selected_action_ev) / pot × 100`. These measure fundamentally different things.

---

## 3. Bug Inventory

| Bug  | Risk            | Type  | Description                                            | Phase | Effort   |
| ---- | --------------- | ----- | ------------------------------------------------------ | ----- | -------- |
| A-4  | HIGH            | FE    | Frontend regretPercent uses different formula than backend | P0    | 1 day    |
| A-1  | HIGH            | BE    | Range array decoded via different paths in V1 vs V2    | P0    | 3 days   |
| A-5  | HIGH            | FE    | Board randomization at wrong time with stale ref       | P1    | 2 days   |
| A-3  | MED             | BE+FE | Pot size differs between V1/V2 configs                 | P1    | 1 day    |
| A-2  | MED             | BE    | Pre-decoded vs fresh-decoded strategy/EV arrays        | P1    | 1 day    |
| A-6  | HIGH (postflop) | BE    | apply_swaps failures silently ignored                  | P2    | 0.5 day  |
| A-7  | HIGH (postflop) | BE    | 9s isomorphism bug in suit mapping                     | P2    | 1 day    |
| A-8  | MED             | BE    | Cache key missing user role                            | P2    | 0.5 day  |
| A-9  | LOW             | FE    | Spins config type safety (`as any` chain)              | P2    | 0.5 day  |

### User Impact Matrix

| Bug  | Frequency     | Visibility | Data Correctness | User Trust |
| ---- | ------------- | ---------- | ---------------- | ---------- |
| A-4  | Every hand    | High       | Wrong number     | Critical   |
| A-1  | Every hand    | Medium     | Wrong combos     | Critical   |
| A-5  | Postflop only | High       | Wrong board/EV   | Critical   |
| A-3  | Every hand    | Low        | Wrong pot        | High       |
| A-2  | Every hand    | Low        | Precision drift  | Medium     |
| A-6  | Postflop      | None       | Silent wrong     | Medium     |
| A-7  | Postflop 9s   | None       | Silent wrong     | Medium     |
| A-8  | Shared cache  | None       | Wrong tier data  | Low        |
| A-9  | Spins only    | None       | Potential crash  | Low        |

---

## 4. Testing Strategy

### 4.1 Testing Stack

| Layer                        | Tool                             | Command            |
| ---------------------------- | -------------------------------- | ------------------ |
| **Backend unit/integration** | pytest + Django TestCase         | `pytest`           |
| **Frontend unit**            | Jest 30 + @testing-library/react | `pnpm test`        |
| **Frontend E2E**             | Playwright 1.58 (7 viewports)   | `pnpm test:e2e`    |
| **Type checking**            | TypeScript                       | `pnpm build`       |

### 4.2 Test-First Workflow

For each bug:

1. Write a failing test that reproduces the bug
2. Confirm the test fails (bug is real)
3. Implement the fix
4. Confirm the test passes
5. Add edge-case and regression tests

This is **mandatory for P0** bugs and recommended for P1+.

### 4.3 Test Type Assignment

| Bug    | Backend pytest | Frontend Jest | Playwright E2E | Rationale                                           |
| ------ | :------------: | :-----------: | :------------: | --------------------------------------------------- |
| **A-4** |       —        |  **Primary**  |   Validation   | Pure FE logic; extract and unit test shared utility |
| **A-1** |  **Primary**   |       —       |   Validation   | Backend decode paths; compare arrays byte-for-byte  |
| **A-5** |       —        |  **Primary**  |  **Primary**   | FE logic + E2E flow verification                    |
| **A-3** |  **Primary**   |   Supporting  |       —        | Pot calculation is backend; FE just displays         |
| **A-2** |  **Primary**   |       —       |       —        | Float precision is backend-only                      |
| **A-6** |  **Primary**   |       —       |       —        | Exception handling; test log level escalation        |
| **A-7** |  **Primary**   |       —       |       —        | Extend existing test suite                           |
| **A-8** |  **Primary**   |       —       |       —        | Cache key logic; simple key comparison               |
| **A-9** |       —        |  TypeScript   |       —        | `tsc` compilation is the test                        |

### 4.4 Test Conventions

- **Jest:** Follow existing patterns in `components/Trainer/__tests__/` — mock Redux store with `configureStore`, mock PostHog/router, use `@testing-library/react`.
- **Playwright:** Follow `strategy-to-trainer.spec.ts` patterns — use `waitForPageLoad()` helpers, `page.route()` for API interception, existing Keycloak auth setup. Skip mobile viewport where inappropriate.
- **Backend pytest:** Follow existing patterns in `trainer/tests/` and `strategies/tests/` — use pytest fixtures, `np.testing.assert_array_equal` for array comparisons.
- **E2E scope:** Use Playwright only for flow validation and cross-API consistency — not for testing formulas or error handling.

### 4.5 New Test Files

| Layer          | File                                         | Bug  |
| -------------- | -------------------------------------------- | ---- |
| Jest           | `utils/__tests__/getRegretPercent.test.ts`   | A-4  |
| Jest           | `Trainer/__tests__/FeedbackPanel.regret.test.tsx` | A-4 |
| Jest           | `Trainer/__tests__/processBoardCards.test.ts` | A-5  |
| Playwright     | `e2e/trainer-values.spec.ts`                 | A-4, A-1 |
| Playwright     | `e2e/trainer-board-randomization.spec.ts`    | A-5  |
| pytest         | `trainer/tests/test_range_divergence_v1_v2.py` | A-1 |
| pytest         | `trainer/tests/test_pot_size_alignment.py`   | A-3  |
| pytest         | `trainer/tests/test_ev_decode_precision.py`  | A-2  |
| pytest         | `trainer/tests/test_apply_swaps_error_handling.py` | A-6 |
| pytest         | `strategies/tests/utils/test_cache_key_isolation.py` | A-8 |

### 4.6 Test Data

| Source         | What                        | How to Obtain                                         |
| -------------- | --------------------------- | ----------------------------------------------------- |
| DB fixtures    | Real tree nodes, range data | Extract during A-1 investigation; serialize to JSON   |
| API responses  | generate-next-hand JSON     | Capture via Playwright route interception              |
| Mock stores    | Redux state slices          | Build programmatically in test helpers                 |
| QA samples     | Specific hand scenarios     | Request from QA team (pending)                        |

### 4.7 Estimated Test Output

| Phase    | Jest | pytest | Playwright | Total  |
| -------- | :--: | :----: | :--------: | :----: |
| P0 (W1)  | 4-6  | 3-4    | 1          | 8-11   |
| P1 (W2)  | 6-8  | 3-4    | 1          | 10-13  |
| P2 (W3)  | —    | 3-4    | —          | 3-4    |
| **Total** | **10-14** | **9-12** | **2** | **21-28** |

---

## 5. Phase 1: Stop the Bleeding (P0) — Week 1

### 5.1 A-4: Remove Frontend `regretPercent` Fallback

**Risk:** HIGH | **Type:** FE | **Effort:** 1 day

**Problem:** Non-debug users see a different `regretPercent` than debug users for the exact same action. The frontend formula measures regret against the single best action; the backend measures regret against the optimal mixed strategy. These are fundamentally different numbers.

**Scope:** 6 files share the same conditional pattern (debug → use backend value; non-debug → recalculate). Two additional files already use backend values correctly.

**Approach:**
1. Create a shared utility that always returns the backend-provided `regretPercent`. If missing, return `null` (display "—").
2. Replace all 6 conditional blocks with the shared utility.
3. **Pre-check:** Audit backend to confirm `regretPercent` is always present in the API response.

**Acceptance Criteria:**
- Non-debug and debug users see identical `regretPercent` values
- Missing backend value shows "—", not a wrong calculated number
- No frontend formula calculates `(bestEV - selectedEV) / potValue` anywhere
- Shared utility has full branch coverage in unit tests
- Verified on both Holdem and Omaha

**Tests:** Jest unit tests for the shared utility (happy path, null/undefined, edge cases) + component test for FeedbackPanel rendering. Playwright E2E spec to validate displayed value matches API response.

---

### 5.2 A-1: Range Array Divergence V1 vs V2

**Risk:** HIGH | **Type:** BE | **Effort:** 3 days

**Problem:** V1 does a raw range decode directly from the tree node. V2 routes through the `StrategyTreeNodeResolver`, which applies sparse index expansion, action tree walking, and isomorphism swaps. These produce different range arrays for the same spot — leading to different hand selection and combos.

**Impact:** Likely the root cause of most "values are inconsistent" reports.

**Approach:**
1. **Investigation** (1-2 days): Write a pytest that feeds the same tree node through both paths and compares output arrays byte-for-byte. Document which transformations V2 applies that V1 skips.
2. **Fix** (1 day): Standardize on one path. V2 is likely more correct. Update V1 to route through the same decode pipeline.

**Acceptance Criteria:**
- Both V1 and V2 produce identical range arrays for the same tree node
- Tests cover: standard preflop, sparse indices, isomorphism mapping
- Backend tests added and passing

**Tests:** Backend pytest comparing V1 and V2 range output arrays for multiple tree node types. Playwright E2E spec validating that standalone trainer and strategy-entry trainer show consistent data.

---

## 6. Phase 2: Visible UX Fixes (P1) — Week 2

### 6.1 A-5: Board Randomization Timing

**Risk:** HIGH | **Type:** FE | **Effort:** 2 days

**Problem:** Board cards are randomized when a hand is consumed from the buffer, not when it was generated. EV values were computed for the original board, but the user sees a randomized board — the strategy tree doesn't match the displayed board.

**Approach:**
1. **Ideal fix (backend):** Move randomization to backend during hand generation so the strategy tree is computed for the randomized board.
2. **Interim fix (frontend):** Randomize at buffer fill time instead of consume time. Regenerate buffer when settings change.
3. Clean up redundant `isFirstHandRef` logic.

**Acceptance Criteria:**
- Displayed board cards match the board used in strategy/EV computation
- First hand shows original board; subsequent hands randomize when enabled
- No duplicate cards; correct card count per street (Flop: 3, Turn: 4, River: 5)

**Tests:** Jest unit tests for `processBoardCards()` covering all streets, randomization on/off, first hand behavior, and duplicate card prevention. Playwright E2E spec validating board cards align with API computation.

---

### 6.2 A-3: Pot Size Alignment V1/V2

**Risk:** MEDIUM | **Type:** BE+FE | **Effort:** 1 day

**Problem:** V1 and V2 source pot/rake config differently. The backend's rake adjustment logic may produce different pot sizes depending on which config path is used.

**Approach:** Audit both pot size sources, identify where they diverge, standardize on one source.

**Acceptance Criteria:**
- V1 and V2 produce identical `pot_size` for the same game config
- Rake adjustment logic tested for all branches (enabled/disabled, finished, zero rake)

**Tests:** Backend pytest for `adjust_pot_size_with_rake()` covering all code paths.

---

### 6.3 A-2: EV Decode Precision

**Risk:** MEDIUM | **Type:** BE | **Effort:** 1 day

**Problem:** V1 always fresh-decodes from brotli data with explicit rounding to 2 decimal places. V2 may use pre-decoded arrays from the resolver with different or no rounding applied.

**Approach:** Compare float arrays from both paths for the same spot. Standardize rounding.

**Acceptance Criteria:**
- Both paths produce float arrays that agree to at least 2 decimal places
- Rounding applied consistently in both paths

**Tests:** Backend pytest comparing fresh decode vs pre-decoded arrays with tolerance assertions.

---

## 7. Phase 3: Polish (P2) — Week 3

### 7.1 A-6: `apply_swaps` Silent Failures

**Risk:** HIGH (postflop) | **Type:** BE | **Effort:** 0.5 day

**Problem:** Two locations catch all exceptions at DEBUG level and continue with unswapped (wrong) data. Users see wrong combo ordering and EV values with zero indication anything failed.

**Approach:** Escalate to `logger.warning()`, add monitoring counter, consider returning an error indicator to the frontend. Coordinate with Akhil.

**Tests:** Backend pytest verifying failures log at WARNING level, not DEBUG.

---

### 7.2 A-7: 9s Isomorphism Bug

**Risk:** HIGH (postflop) | **Type:** BE | **Effort:** 1 day

**Problem:** `apply_swaps()` uses the current round number (e.g., TURN) to compute the ISO board, but precision strategies index by ISO flop only. The suit map should derive from the flop board. Comprehensive test documentation already exists.

**Approach:** Fix to use flop round number. Extend existing test suite. Coordinate with Akhil.

---

### 7.3 A-8: Cache Key Missing User ID

**Risk:** MEDIUM | **Type:** BE | **Effort:** 0.5 day

**Problem:** `cache_page_with_role()` builds a cache key with roles and path but no user ID. Users with the same role share cache, potentially serving wrong subscription tier data.

**Tests:** Backend pytest verifying different users with same role get different cache keys.

---

### 7.4 A-9: Spins Config Type Safety

**Risk:** LOW | **Type:** FE | **Effort:** 0.5 day

**Problem:** Unsafe `as any` chain when accessing spins game config. If config structure changes, trainer silently uses wrong config.

**Fix:** Define a proper TypeScript interface. `pnpm build` is the test.

---

## 8. Timeline Summary

| Week       | Phase    | Bugs              | Focus                                          |
| ---------- | -------- | ------------------ | ---------------------------------------------- |
| **Week 1** | P0       | A-4, A-1           | Remove wrong FE formula; investigate and fix range divergence |
| **Week 2** | P1       | A-5, A-3, A-2      | Fix board randomization; align pot sizes and EV precision |
| **Week 3** | P2       | A-6, A-7, A-8, A-9 | Swap error handling; isomorphism fix; cache key; type safety |

### Effort Breakdown

| Bug  | Investigation | Implementation | Testing  | Total      |
| ---- | :-----------: | :------------: | :------: | ---------- |
| A-4  | —             | 0.5 day        | 0.5 day  | **1 day**  |
| A-1  | 1.5 days      | 1 day          | 0.5 day  | **3 days** |
| A-5  | 0.5 day       | 1 day          | 0.5 day  | **2 days** |
| A-3  | 0.25 day      | 0.5 day        | 0.25 day | **1 day**  |
| A-2  | 0.25 day      | 0.5 day        | 0.25 day | **1 day**  |
| A-6  | —             | 0.25 day       | 0.25 day | **0.5 day** |
| A-7  | 0.25 day      | 0.5 day        | 0.25 day | **1 day**  |
| A-8  | —             | 0.25 day       | 0.25 day | **0.5 day** |
| A-9  | —             | 0.25 day       | 0.25 day | **0.5 day** |
| **Total** |          |                |          | **~10-11 working days** |

### Dependency Order

```
A-4 (regret formula) ──── Independent, start immediately
A-1 (range divergence) ── Independent, start immediately
        │
        ▼
A-5 (board randomization) ── May share root cause with A-1
A-3 (pot alignment) ──────── Partially addressed by A-4
A-2 (EV precision) ──────── Depends on A-1 (range alignment first)
        │
        ▼
A-6 (apply_swaps) ────────── Coordinate with Akhil
A-7 (9s isomorphism) ─────── Coordinate with Akhil; existing test evidence
A-8 (cache key) ──────────── Independent, quick fix
A-9 (type safety) ────────── Independent, quick fix
```

---

## 9. Cross-Stream Coordination

| Coordination            | Who               | When     | Topic                                                |
| ----------------------- | ----------------- | -------- | ---------------------------------------------------- |
| Siddharth ↔ Akhil       | Weekly            | Ongoing  | A-7 (isomorphism), A-6 (apply_swaps), A-8 (cache key) |
| Siddharth ↔ Srinivasan  | After P0          | Week 2+  | A-2/D-10 (EV memory workaround affects both streams) |
| Siddharth ↔ Priyanka    | As needed         | Week 1   | `FeedbackActions.tsx` shared between Stream A and B  |
| All streams             | Daily 10-min sync | Ongoing  | Flag blockers, share discoveries                     |

---

## 10. Dependencies and Blockers

| Item                                 | Status            | Impact                                  | Mitigation                              |
| ------------------------------------ | ----------------- | --------------------------------------- | --------------------------------------- |
| QA sample hands for validation       | Pending           | Cannot write realistic E2E fixtures     | Use synthetic test data initially       |
| Akhil RTS status update              | Pending           | May duplicate work on A-7, A-6          | Investigate first; coordinate before fix |
| Backend `regretPercent` coverage     | Can start now     | Blocks A-4 deployment                   | Audit on Day 1                          |
| Test data fixtures (tree nodes)      | Extract during A-1 | Required for backend pytest tests       | Create minimal synthetic fixtures first |

---

## 11. Definition of Done

| # | Criterion                                                  |
| - | ---------------------------------------------------------- |
| 1 | Failing test written and confirmed failing before fix      |
| 2 | Fix implemented                                            |
| 3 | All new and existing tests pass                            |
| 4 | `pnpm build` succeeds (no TypeScript errors)               |
| 5 | Verified against both Holdem and Omaha                     |
| 6 | Small, focused PR with test evidence in description        |
| 7 | No debug logs or new TODOs left in code                    |

---

## 12. Open Items

- [ ] Audit backend to confirm `regretPercent` is always present in API response (blocks A-4)
- [ ] Obtain sample hands from QA for realistic test fixtures
- [ ] Akhil to report RTS status so overlap with A-7/A-6 is understood
- [ ] Decide interim vs ideal fix for A-5 (frontend buffer-fill vs backend randomization)
- [ ] Confirm Priyanka and Dhwani assignment to Streams B and D
