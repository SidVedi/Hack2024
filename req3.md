# Stream A: Trainer Values Stabilization — Proposal

**Owner:** Siddharth Vedi (svedi@algosoftware.io)  
**Stream:** A — Trainer Values Inconsistent  
**Scope:** 9 bugs (A-1 through A-9)  
**Variants:** Holdem and Omaha  
**Date:** February 9, 2026

---

## 1. Executive Summary

Users reported inconsistent trainer values — EV discrepancies, missing combos, and incorrect feedback labels. After auditing both the frontend (Next.js) and backend (Django/DRF), the root causes are:

1. **V1/V2 API divergence** — Two backend code paths decode ranges, EV, and pot sizes through different logic, producing different results for the same spot.
2. **Frontend calculation drift** — The frontend recalculates `regretPercent` using a fundamentally different formula than the backend, causing non-debug users to see wrong feedback labels.

**Approach:** Stop serving wrong data first, investigate root causes second, polish and tech debt third. Test-first development at every phase.

---

## 2. Root Cause Analysis

### How the Trainer Works

The trainer has two API code paths that feed the same V3 frontend layout. The frontend decides which API to call based on whether `fromStrategiesConfig` is present (strategy page entry) or not (standalone trainer).

```
┌──────────────────┐         ┌──────────────────────────────┐
│  Standalone       │         │  Strategy Page Entry          │
│  Trainer          │         │  (via Train button)           │
│                   │         │                               │
│  No config        │         │  fromStrategiesConfig present │
└────────┬──────────┘         └──────────────┬────────────────┘
         │                                   │
         ▼                                   ▼
┌──────────────────┐         ┌──────────────────────────────┐
│  V1 API           │         │  V2 API                       │
│  /trainer/        │         │  /trainer/v2/                  │
│  generate-next-   │         │  generate-next-hand            │
│  hand             │         │                               │
│                   │         │  Uses StrategyTreeNodeResolver │
│  decode_range()   │         │  decode_or_calculate_range()   │
│  direct from node │         │  + expand_range()              │
│                   │         │  + apply_swaps()               │
└────────┬──────────┘         └──────────────┬────────────────┘
         │                                   │
         └──────────────┬────────────────────┘
                        ▼
            ┌───────────────────────┐
            │  _generate_hand_      │
            │  response()           │
            │                       │
            │  EV / regret calc     │
            │  pot_size adjustment  │
            └───────────┬───────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │  Frontend (V3 layout) │
            │                       │
            │  FeedbackPanel.tsx     │
            │  TrainerCenterPanel    │
            │  useTrainerMobile     │
            │  TrainerHistoryList   │
            │  FeedbackActions      │
            └───────────────────────┘
```

### Where Things Go Wrong

**Backend divergence:** V1 raw-decodes range data; V2 applies additional resolver transformations (sparse index expansion, action tree walking, isomorphism swaps). The same tree node can produce different range arrays through these two paths, leading to different combos and EV values.

**Frontend drift:** The backend computes regret using strategy-weighted EV across all actions. The frontend (for non-debug users) computes regret using only the selected action's EV — a fundamentally different quantity. This compounds the backend inconsistency.

---

## 3. Bug Inventory

| Bug  | Risk            | Type  | Description                                               | Priority | Effort    |
| ---- | --------------- | ----- | --------------------------------------------------------- | -------- | --------- |
| A-4  | HIGH            | FE    | Frontend regretPercent uses different formula than backend | P0       | 1 day     |
| A-1  | HIGH            | BE    | Range array decoded via different paths in V1 vs V2       | P0       | 2-3 days  |
| A-5  | HIGH            | FE    | Board randomization at wrong time with stale ref          | P1       | 2 days    |
| A-3  | MED             | BE+FE | Pot size differs between V1/V2 configs                    | P1       | 1-2 days  |
| A-2  | MED             | BE    | Pre-decoded vs fresh-decoded strategy/EV arrays           | P2       | 1 day     |
| A-6  | HIGH (postflop) | BE    | apply_swaps failures silently ignored                     | P2       | 0.5 day   |
| A-7  | HIGH (postflop) | BE    | 9s isomorphism bug in suit mapping                        | P2       | 1 day     |
| A-8  | MED             | BE    | Cache key missing user role                               | P2       | 0.5 day   |
| A-9  | LOW             | FE    | Spins config type safety (`as any` chain)                 | P2       | 0.5 day   |

### User Impact

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

## 4. P0 — Stop the Bleeding

### A-4: Remove Frontend regretPercent Fallback

**Effort:** 1 day

**Problem:** Non-debug users see a different regretPercent than debug users for the same action. The backend measures regret against optimal mixed play (strategy-weighted EV); the frontend fallback measures regret against the single best action. These are fundamentally different numbers.

**Scope:** 6 files contain the same conditional pattern — all must be updated consistently. Two additional files already use the backend value correctly and need no changes.

**Approach:**
1. Extract a single shared utility that always returns the backend-provided value (or null if missing).
2. Replace all 6 conditional blocks with the shared utility.
3. If backend value is missing, display "—" rather than a wrong number.
4. Pre-check: audit backend to confirm `regretPercent` is always present in the API response.

**Acceptance Criteria:**
- Non-debug and debug users see identical regretPercent for the same action
- No frontend code calculates its own regret formula
- Missing backend values show "—", not a fallback calculation
- Verified on Holdem and Omaha

---

### A-1: Range Array Divergence V1 vs V2

**Effort:** 2-3 days

**Problem:** V1 raw-decodes range data from the tree node. V2 uses the resolver which may apply sparse index expansion, action tree walking, and isomorphism remapping. The same tree node can produce different range arrays through these two paths, leading to different combos and EV values.

**Approach:**
1. **Investigation** (1-1.5 days): Trace both paths with the same tree node. Compare output arrays. Document which transformations V2 applies that V1 skips.
2. **Fix** (1-1.5 days): Standardize on one path. V2 is likely more correct — update V1 to route through the same resolver logic.

**Acceptance Criteria:**
- Both V1 and V2 produce identical range arrays for the same tree node
- Tested with standard preflop, sparse indices, and isomorphism scenarios
- Verified on Holdem and Omaha

---

## 5. P1 — Visible UX Fixes

### A-5: Board Randomization Timing

**Effort:** 2 days

**Problem:** Board cards are randomized at buffer consumption time, not at generation time. EV values were computed for the original board, but the user sees a randomized board. The strategy tree doesn't match what's displayed.

**Approach:**
1. **Ideal fix:** Move randomization to the backend during hand generation so the strategy tree is computed for the randomized board.
2. **Interim fix:** Randomize at buffer fill time instead of consume time; regenerate buffer when settings change.
3. Clean up stale ref logic (`isFirstHandRef`) that masks the timing issue.

**Acceptance Criteria:**
- Displayed board cards match the board used in strategy/EV computation
- First hand shows original board; subsequent hands randomize (when enabled)
- Correct card count per street; no duplicate cards
- Verified on Holdem and Omaha

---

### A-3: Pot Size Alignment V1/V2

**Effort:** 1-2 days

**Problem:** V1 and V2 source pot/rake config from different places. If rake structure or blind levels differ between standalone config and strategy-page config, pot sizes diverge, causing different feedback labels for the same spot.

**Approach:**
1. Audit both pot size sources and identify where they diverge.
2. Standardize on one source (likely V2's explicit game config).
3. Frontend pot value fallback is partially addressed by A-4.

**Acceptance Criteria:**
- V1 and V2 produce identical pot size for the same game config
- Rake adjustment logic handles all branches (enabled, disabled, finished, zero rake)
- Verified on Holdem and Omaha

---

## 6. P2 — Polish and Tech Debt

### A-2: EV Decode Precision

**Effort:** 1 day

**Problem:** V1 always decompresses EV data fresh (with 2-decimal rounding). V2 may use pre-decoded arrays from the resolver that were rounded at a different stage or not at all. This can cause subtle EV differences.

**Approach:** Compare float arrays from both paths. If precision differs, standardize rounding.

---

### A-6: apply_swaps Silent Failures

**Effort:** 0.5 day

**Problem:** Two locations catch swap failures at DEBUG level and continue with unswapped (wrong) data. Users see wrong combo ordering and EV with zero indication.

**Approach:** Escalate to WARNING; add monitoring metric; consider returning an error indicator to the frontend. Coordinate with Akhil.

---

### A-7: 9s Isomorphism Bug

**Effort:** 1 day

**Problem:** Suit mapping uses the current round's ISO board (e.g., TURN) instead of the FLOP ISO board. Precision strategies index by flop ISO, so the suit map is wrong on turn/river.

**Approach:** Fix to use flop round number. Existing test evidence documents the bug. Extend tests to verify fix. Coordinate with Akhil.

---

### A-8: Cache Key Missing Role

**Effort:** 0.5 day

**Problem:** Cache key includes role hash and path but not user ID. Users with the same role share cache, potentially serving wrong subscription tier data.

**Approach:** Add user ID to cache key, matching the pattern already used by the `user_role_cache()` decorator.

---

### A-9: Spins Config Type Safety

**Effort:** 0.5 day

**Problem:** Unsafe double `as any` cast when accessing spins config. If config structure changes, trainer silently uses wrong config.

**Approach:** Define a proper TypeScript interface. TypeScript compilation is the verification.

---

## 7. Testing Strategy

### 7.1 Approach Per Bug

| Bug    | Backend pytest | Frontend Jest | Playwright E2E | Rationale                                              |
| ------ | :------------: | :-----------: | :------------: | ------------------------------------------------------ |
| **A-4** |       —        |  **Primary**  |   Validation   | Pure FE logic; unit test the shared utility            |
| **A-1** |  **Primary**   |       —       |   Validation   | Backend decode paths; compare arrays                   |
| **A-5** |       —        |  **Primary**  |  **Primary**   | FE logic + E2E board verification                      |
| **A-3** |  **Primary**   |   Supporting  |       —        | Pot calculation is backend                             |
| **A-2** |  **Primary**   |       —       |       —        | Float precision; tolerance comparison                  |
| **A-6** |  **Primary**   |       —       |       —        | Log level verification                                 |
| **A-7** |  **Primary**   |       —       |       —        | Extend existing test suite                             |
| **A-8** |  **Primary**   |       —       |       —        | Key comparison                                         |
| **A-9** |       —        |  TypeScript   |       —        | `tsc` compilation is the test                          |

### 7.2 Test-First Workflow

For P0 bugs (A-4, A-1), the failing test is written **before** the fix to confirm the bug exists and prevent regressions. For P1+ bugs, tests are written alongside the fix.

### 7.3 New Test Files

**Frontend (Jest):**
- `components/Trainer/__tests__/FeedbackPanel.regret.test.tsx` — A-4 component tests
- `utils/__tests__/getRegretPercent.test.ts` — A-4 shared utility tests
- `components/Trainer/__tests__/processBoardCards.test.ts` — A-5 board card tests

**Frontend (Playwright):**
- `e2e/trainer-values.spec.ts` — A-4 and A-1 end-to-end validation
- `e2e/trainer-board-randomization.spec.ts` — A-5 flow verification

**Backend (pytest):**
- `trainer/tests/test_range_divergence_v1_v2.py` — A-1
- `trainer/tests/test_ev_decode_precision.py` — A-2
- `trainer/tests/test_pot_size_alignment.py` — A-3
- `trainer/tests/test_apply_swaps_error_handling.py` — A-6
- `strategies/tests/utils/test_cache_key_isolation.py` — A-8

### 7.4 Principles

- **E2E tests are validation, not primary.** Playwright tests are slow and flaky-prone. Use them only for flow verification, not calculation testing.
- **Mock external calls** in all Jest tests. No real network calls.
- **Both variants.** Every fix verified against Holdem and Omaha.
- **Follow existing patterns.** Jest tests follow `TrainerModalV2.test.tsx` conventions; Playwright tests follow `strategy-to-trainer.spec.ts` conventions; backend tests follow existing `trainer/tests/` patterns.

### 7.5 Estimated Test Output

| Phase   | Jest  | pytest | Playwright | Total   |
| ------- | :---: | :----: | :--------: | :-----: |
| P0      | 4-6   | 3-4    | 1          | 8-11    |
| P1      | 6-8   | 2-3    | 1          | 9-12    |
| P2      | —     | 5-6    | —          | 5-6     |
| **Total** | **10-14** | **10-13** | **2** | **22-29** |

---

## 8. Risks & Dependencies

### 8.1 Technical Risks

| Risk | Severity | Bugs Affected | Description | Mitigation |
| ---- | -------- | ------------- | ----------- | ---------- |
| **Backend does not always return `regretPercent`** | High | A-4 | If the API omits `regretPercent` for certain spots (e.g., edge cases, older strategy data), users will see "—" instead of any value after the fix. | Audit backend coverage before deploying. If gaps exist, add backend fallback before removing the frontend one. |
| **V1/V2 divergence is by design** | High | A-1 | The resolver transformations in V2 may be intentional — V1 may have been designed to skip them for performance. Standardizing could introduce new regressions. | Investigate thoroughly before changing. Compare results against known-correct outputs. Run both variants. |
| **Board randomization fix requires backend change** | Medium | A-5 | The ideal fix (backend randomization) touches hand generation logic and could affect buffer performance and strategy tree computation. | Start with frontend interim fix. Plan backend fix as a follow-up if interim is insufficient. |
| **apply_swaps fix may conflict with RTS work** | Medium | A-6, A-7 | Akhil is actively working on RTS which shares swap and isomorphism code. Changing error handling or suit mapping could conflict. | Coordinate with Akhil before making changes. Review his in-flight PRs first. |
| **Cache key change invalidates existing cache** | Low | A-8 | Adding user ID to cache key means all existing cached data becomes stale, causing a brief spike in cache misses and backend load. | Deploy during low-traffic window. Monitor cache hit rate after deployment. |
| **Spins config interface may not cover all variants** | Low | A-9 | Defining a TypeScript interface for spins config requires knowing all possible config shapes across game types. Incomplete typing could cause build failures. | Audit existing config usage before defining the interface. Use optional fields where uncertain. |

### 8.2 Execution Risks

| Risk | Severity | Description | Mitigation |
| ---- | -------- | ----------- | ---------- |
| **Cross-repo changes** | Medium | A-1, A-2, A-3 require changes in both frontend and backend repos. PRs must be coordinated to avoid deploying one without the other. | Sequence deployments: backend first, then frontend. Ensure backward compatibility. |
| **Missing test fixtures** | Medium | Backend tests for A-1, A-2, A-3 require real tree node data from the database. Synthetic fixtures may not cover edge cases. | Extract real fixtures during A-1 investigation. Supplement with synthetic data for edge cases. |
| **Shared file conflicts** | Low | `FeedbackActions.tsx` is modified by A-4 (Stream A) and potentially by Stream B (transitions). Parallel edits could cause merge conflicts. | Coordinate with Priyanka. Merge A-4 changes first since it's a quick win. |

### 8.3 Dependencies

| Dependency | Status | Blocks | Impact if Delayed | Mitigation |
| ---------- | ------ | ------ | ----------------- | ---------- |
| Backend `regretPercent` API audit | Can start immediately | A-4 deployment | Cannot safely remove frontend fallback | Audit on first day; if gaps found, fix backend first |
| Sample hands for validation | Pending | Realistic test fixtures | Tests use synthetic data only; may miss real-world edge cases | Start with synthetic; add real fixtures when available |
| Akhil RTS status report | Pending | A-6, A-7 scope | May duplicate work or introduce conflicts | Start investigation independently; hold off on code changes until aligned |
| Test data fixtures (tree nodes) | Extract during A-1 | Backend pytest tests | Cannot write meaningful backend tests | Create minimal synthetic fixtures to unblock; replace with real data later |
| A-5 fix approach decision | Needs team input | A-5 implementation | Frontend interim fix vs backend ideal fix changes effort and risk profile | Default to frontend interim fix; propose backend fix as follow-up |

### 8.4 Dependency Graph

```
A-4 (regret formula) ──── Independent, start immediately
A-1 (range divergence) ── Independent, start immediately
A-5 (board randomization) ← May share root cause with A-1
A-3 (pot alignment) ←──── Partially addressed by A-4
A-2 (EV precision) ←───── Depends on A-1 (range must be aligned first)
A-6 (swap logging) ←───── Coordinate with Akhil before changing
A-7 (9s isomorphism) ←─── Coordinate with Akhil; existing test evidence
A-8 (cache key) ────────── Independent, quick fix
A-9 (type safety) ──────── Independent, quick fix
```

---

## 9. Coordination Points

| With         | When            | Topic                                                      |
| ------------ | --------------- | ---------------------------------------------------------- |
| Akhil        | Before P2 work  | A-7 (9s iso), A-6 (apply_swaps), A-8 (cache key) — all touch strategy/RTS code |
| Srinivasan   | After P0        | A-2/D-10 (EV memory workaround affects both trainer and replayer) |
| Priyanka     | During P0       | `FeedbackActions.tsx` is shared between Stream A and B     |

---

## 10. Definition of Done

| # | Criterion                                                | Verification                     |
| - | -------------------------------------------------------- | -------------------------------- |
| 1 | Failing test written before fix (P0 mandatory)           | Test output in PR description    |
| 2 | Fix implemented                                          | Code diff in PR                  |
| 3 | All new and existing tests pass                          | `pnpm test` / `pytest` output    |
| 4 | `pnpm build` succeeds                                   | Build log                        |
| 5 | Verified against both Holdem and Omaha                   | Manual test or E2E               |
| 6 | Small, focused PR (one bug per PR where possible)        | Code review                      |
| 7 | No debug logs or TODOs left in code                      | Code review                      |

---

## 11. Open Items

- [ ] Audit backend to confirm `regretPercent` is always present in API response (blocks A-4)
- [ ] Obtain sample hands for realistic test fixtures
- [ ] Akhil to report RTS status (A-7/A-6 overlap)
- [ ] Decide interim vs ideal fix for A-5 (frontend buffer-fill vs backend randomization)
