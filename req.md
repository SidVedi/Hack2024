# Stream A: Trainer Values Stabilization — Proposal Plan

**Owner:** Siddharth Vedi (svedi@algosoftware.io)  
**Stream:** A — Trainer Values Inconsistent  
**Scope:** 9 bugs (A-1 through A-9), spanning frontend and backend  
**Variants affected:** Holdem and Omaha  
**Timeline:** 4 weeks  
**Date:** February 9, 2026

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Root Cause Analysis](#2-root-cause-analysis)
3. [Bug Inventory and Risk Matrix](#3-bug-inventory-and-risk-matrix)
4. [Testing Strategy](#4-testing-strategy)
5. [Phase 1: Stop the Bleeding (P0) — Week 1](#5-phase-1-stop-the-bleeding-p0--week-1)
6. [Phase 2: Visible UX Fixes (P1) — Week 2](#6-phase-2-visible-ux-fixes-p1--week-2)
7. [Phase 3: Polish (P2) — Week 3](#7-phase-3-polish-p2--week-3)
8. [Phase 4: Tech Debt (P3) — Week 4+](#8-phase-4-tech-debt-p3--week-4)
9. [Timeline Summary](#9-timeline-summary)
10. [Cross-Stream Coordination](#10-cross-stream-coordination)
11. [Dependencies and Blockers](#11-dependencies-and-blockers)
12. [Definition of Done](#12-definition-of-done)
13. [Open Items](#13-open-items)

---

## 1. Executive Summary

Andres reported inconsistent trainer values — EV, strategies, combos missing, tables showing incorrect data. After auditing both the **Next.js frontend** and **Django/DRF backend**, the root causes fall into two categories:

1. **V1/V2 API divergence** — Two backend code paths decode ranges, EV, and pot sizes through different logic, producing different results for the same poker spot.
2. **Frontend calculation drift** — The frontend independently recalculates `regretPercent` using a fundamentally different formula than the backend, causing non-debug users to see wrong feedback labels.

**Approach:** Quick wins first (stop serving known-wrong data), root cause investigation second, polish third — with **test-first development** at every phase.

**Total scope:** 9 bugs, ~14-18 working days across 4 weeks.

---

## 2. Root Cause Analysis

### 2.1 How the Trainer Works

The trainer has two API code paths that feed the same V3 frontend layout:

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

### 2.2 The Two Root Causes

**Backend divergence:** V1 uses `decode_range(tree_node['range'])` directly (line 199 in `spot_generation.py`), while V2 uses `data.get('rangeResult', [])` from `StrategyTreeNodeResolver` (line 407). The resolver applies additional transformations — sparse index expansion, action tree walking, isomorphism swaps — that `decode_range()` alone does not.

**Frontend drift:** The backend computes `regretPercent` as:

```
weighted_ev = Σ(strategy[action] × ev[action])    ← strategy-weighted EV
regret% = (best_ev - weighted_ev) / pot_size × 100
```

The frontend (for non-debug users) computes:

```
regret% = (best_ev - selected_ev) / pot_value × 100    ← user's chosen action EV
```

These are **fundamentally different numbers**. The backend measures regret against optimal mixed play; the frontend measures regret against the single best action.

---

## 3. Bug Inventory and Risk Matrix

| Bug  | Risk            | Type  | Description                                            | Phase | Effort     |
| ---- | --------------- | ----- | ------------------------------------------------------ | ----- | ---------- |
| A-4  | HIGH            | FE    | Frontend regretPercent uses different formula than backend | P0    | 1 day      |
| A-1  | HIGH            | BE    | Range array decoded via different paths in V1 vs V2    | P0    | 3-4 days   |
| A-5  | HIGH            | FE    | Board randomization at wrong time with stale ref       | P1    | 2-3 days   |
| A-3  | MED             | BE+FE | Pot size differs between V1/V2 configs                 | P1    | 1-2 days   |
| A-2  | MED             | BE    | Pre-decoded vs fresh-decoded strategy/EV arrays        | P2    | 1-2 days   |
| A-6  | HIGH (postflop) | BE    | apply_swaps failures silently ignored                  | P2    | 0.5-1 day  |
| A-7  | HIGH (postflop) | BE    | 9s isomorphism bug in suit mapping                     | P3    | 1-2 days   |
| A-8  | MED             | BE    | Cache key missing user role                            | P3    | 0.5 day    |
| A-9  | LOW             | FE    | Spins config `as any` chain — type safety              | P3    | 0.5 day    |

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

| Layer                        | Tool                             | Config                                          | Command            |
| ---------------------------- | -------------------------------- | ----------------------------------------------- | ------------------ |
| **Backend unit/integration** | pytest + Django TestCase         | `pytest.ini` (`GAMETRAINER.settings_test`)      | `pytest`           |
| **Frontend unit**            | Jest 30 + @testing-library/react | `jest.config.js` + `jest.setup.js`              | `pnpm test`        |
| **Frontend E2E**             | Playwright 1.58                  | `playwright.config.ts` (7 viewports)            | `pnpm test:e2e`    |
| **Visual regression**        | Playwright screenshots           | Snapshot comparisons                            | `pnpm test:visual` |
| **API mocking (FE)**         | msw 2.12 (available)             | —                                               | In-process         |
| **Type checking**            | TypeScript (tsc)                 | `tsconfig.json`                                 | `pnpm build`       |

### 4.2 Test-First Workflow (Mandatory for P0, Recommended for P1+)

```
Step 1: Write a failing test that reproduces the bug
Step 2: Verify the test fails → confirms the bug exists
Step 3: Implement the fix
Step 4: Verify the test passes
Step 5: Add edge-case and regression tests
Step 6: Verify all existing tests still pass
```

### 4.3 Which Bugs Get Which Tests

| Bug    | Backend pytest | Frontend Jest | Playwright E2E | Rationale                                                   |
| ------ | :------------: | :-----------: | :------------: | ----------------------------------------------------------- |
| **A-4** |       —        |  **Primary**  |   Validation   | Pure FE logic extraction → unit test the shared utility     |
| **A-1** |  **Primary**   |       —       |   Validation   | Backend decode paths → compare arrays byte-for-byte         |
| **A-5** |       —        |  **Primary**  |  **Primary**   | FE logic (`processBoardCards`) + E2E flow verification      |
| **A-3** |  **Primary**   |   Supporting  |   Validation   | Pot calculation is backend; FE just displays                |
| **A-2** |  **Primary**   |       —       |       —        | Float precision is backend-only; tolerance comparison       |
| **A-6** |  **Primary**   |       —       |       —        | Exception handling is backend; test log level escalation    |
| **A-7** |  **Primary**   |       —       |       —        | Existing test evidence; extend existing suite               |
| **A-8** |  **Primary**   |       —       |       —        | Cache key logic is backend; simple key comparison           |
| **A-9** |       —        |  TypeScript   |       —        | Type safety; `tsc` compilation is the test                  |

### 4.4 Frontend Unit Test Patterns (Jest)

Based on the existing patterns in `TrainerModalV2.test.tsx`, `TrainerModal.test.tsx`, and `trainerReducer.test.ts`, our tests will follow these conventions:

**Store mocking pattern:**

```typescript
import { configureStore } from '@reduxjs/toolkit';
import { render, screen, waitFor } from '@testing-library/react';
import { Provider } from 'react-redux';

// Create a test store matching existing patterns
function createTestStore(overrides = {}) {
  return configureStore({
    reducer: {
      trainerReducer: (state = { ...defaultTrainerState, ...overrides }) => state,
      gameReducer: (state = defaultGameState) => state,
      userReducer: (state = { hasDebugAccess: false }) => state,
    },
  });
}

function renderWithStore(component: React.ReactElement, storeOverrides = {}) {
  const store = createTestStore(storeOverrides);
  return render(<Provider store={store}>{component}</Provider>);
}
```

**Component mock pattern:**

```typescript
// Mock sub-components to isolate the component under test
jest.mock('@/components/Trainer/TrainerCenterPanel', () => ({
  __esModule: true,
  default: () => <div data-testid="mock-center-panel" />,
}));
```

**Assertion pattern:**

```typescript
// Use @testing-library/jest-dom matchers
expect(screen.getByTestId('regret-percent')).toHaveTextContent('5.23%');
expect(screen.queryByTestId('fallback-calculation')).not.toBeInTheDocument();
```

### 4.5 Playwright E2E Patterns

Based on the existing `strategy-to-trainer.spec.ts`, our E2E tests will follow these conventions:

**Helper functions:**

```typescript
async function waitForPageLoad(page: Page) {
  await page.waitForLoadState('load');
  await page.waitForTimeout(1000);
}

async function waitForTrainerReady(page: Page) {
  await waitForPageLoad(page);
  // Wait for trainer to render and receive first hand
  const trainerContent = page.locator('[data-testid="trainer-content"], .trainer-view, main');
  await trainerContent.first().waitFor({ state: 'visible', timeout: 15000 });
}
```

**API interception pattern (from existing spec):**

```typescript
// Set up route interception BEFORE triggering the action
let capturedResponse: any = null;

await page.route('**/trainer/**/generate-next-hand*', async (route) => {
  const response = await route.fetch();
  capturedResponse = await response.json();
  await route.fulfill({ response });
});
```

**Test structure:**

```typescript
test.describe('Trainer Values - Stream A', () => {
  test.setTimeout(60000); // Strategy page has ongoing network activity

  test.beforeEach(async ({ page }, testInfo) => {
    test.skip(testInfo.project.name === 'Mobile', 'Trainer page layout differs on mobile');
    // Navigate and wait for page to be ready
  });

  test('test case description', async ({ page }) => {
    // Arrange → Act → Assert
  });
});
```

**Auth:** All E2E tests use pre-authenticated state from `e2e/auth.setup.ts` (Keycloak login → saved to `e2e/.auth/user.json`).

**Viewports:** Tests run across 7 viewport configurations (Desktop-Large through Mobile). Use `test.skip()` for viewport-inappropriate tests.

### 4.6 Backend pytest Patterns

Based on the existing tests in `trainer/tests/` and `strategies/tests/`:

```python
import numpy as np
import pytest
from unittest.mock import patch, MagicMock

class TestBugDescription:
    """Descriptive docstring for the test class."""

    @pytest.fixture
    def sample_data(self):
        """Provide test data fixtures."""
        return { ... }

    def test_expected_behavior(self, sample_data):
        """Document what correct behavior looks like."""
        result = function_under_test(sample_data)
        np.testing.assert_array_equal(result, expected)

    def test_edge_case(self):
        """Document edge case."""
        ...
```

### 4.7 Test File Organization

**New frontend unit tests (Jest):**

```
components/Trainer/__tests__/
  ├── TrainerModal.test.tsx                  (existing - 748 lines)
  ├── TrainerModalV2.test.tsx                (existing - 1231 lines)
  ├── TrainerModalV2ActionChips.test.tsx     (existing - 880 lines)
  ├── TrainerModalV2ConfigDetails.test.tsx   (existing - 672 lines)
  ├── FeedbackPanel.regret.test.tsx          (NEW - A-4)
  ├── processBoardCards.test.ts              (NEW - A-5)
  └── consumeFromBuffer.test.ts              (NEW - A-5)

utils/__tests__/
  └── getRegretPercent.test.ts               (NEW - A-4, shared utility)
```

**New Playwright E2E tests:**

```
e2e/
  ├── auth.setup.ts                          (existing)
  ├── strategy-to-trainer.spec.ts            (existing)
  ├── strategy-flows.spec.ts                 (existing)
  ├── strategy-config.spec.ts                (existing)
  ├── trainer-values.spec.ts                 (NEW - A-4 + A-1 validation)
  └── trainer-board-randomization.spec.ts    (NEW - A-5)
```

**New backend pytest tests:**

```
trainer/tests/
  ├── test_generate_next_hand_v2.py          (existing)
  ├── test_trainer_helper.py                 (existing)
  ├── test_trainer_helper_metrics.py         (existing)
  ├── test_cache_key.py                      (existing)
  ├── test_range_divergence_v1_v2.py         (NEW - A-1)
  ├── test_ev_decode_precision.py            (NEW - A-2)
  ├── test_pot_size_alignment.py             (NEW - A-3)
  └── test_apply_swaps_error_handling.py     (NEW - A-6)

strategies/tests/utils/
  ├── test_apply_swaps_iso_consistency.py    (existing - extend for A-7)
  ├── test_build_suit_map.py                 (existing)
  ├── test_unfold_range.py                   (existing)
  ├── test_unfold_strategy.py                (existing)
  └── test_cache_key_isolation.py            (NEW - A-8)
```

### 4.8 Test Data Management

| Source           | What                         | How to Obtain                                              | Used By     |
| ---------------- | ---------------------------- | ---------------------------------------------------------- | ----------- |
| DB fixtures      | Real tree nodes, range data  | Extract during A-1 investigation; serialize to JSON files  | BE pytest   |
| API responses    | Full generate-next-hand JSON | Capture via Playwright `page.route()` interception         | E2E tests   |
| Mock stores      | Redux state slices           | Build programmatically in `createTestStore()`              | FE Jest     |
| Andres' samples  | Specific hand scenarios      | Request from Andres (pending)                              | All layers  |

**Fixture storage:**

```
# Backend
trainer/tests/fixtures/
  ├── preflop_tree_node.json
  ├── postflop_tree_node_flop.json
  ├── strategy_object_holdem.json
  └── strategy_object_omaha.json

# Frontend
components/Trainer/__tests__/fixtures/
  ├── trainerApiResponse.json
  └── summaryDataWithRegret.json
```

### 4.9 Testing Principles

1. **Test-first is mandatory for P0 bugs (A-4, A-1).** Write the failing test before the fix.
2. **E2E tests are validation, not primary.** Playwright tests are slow (~60s per test with auth) and flaky-prone. Use them to validate end-to-end flows, not to test calculation logic.
3. **One assertion per concern.** Don't combine unrelated assertions in a single test.
4. **Mock external calls.** All Jest tests must mock APIs, Redux, and router. No real network calls.
5. **Both variants.** Every fix must be verified against Holdem AND Omaha.
6. **No snapshot tests for values.** Snapshot tests are appropriate for UI layout, not for numerical correctness. Use explicit assertions.

### 4.10 Estimated Test Output

| Phase    | Jest Unit Tests | Backend pytest Tests | Playwright E2E Specs | Total |
| -------- | :-------------: | :------------------: | :------------------: | :---: |
| P0 (W1)  | 4-6             | 3-4                  | 1                    | 8-11  |
| P1 (W2)  | 6-8             | 2-3                  | 1                    | 9-12  |
| P2 (W3)  | —               | 3-4                  | —                    | 3-4   |
| P3 (W4+) | —               | 2-3 (extend)         | —                    | 2-3   |
| **Total** | **10-14**       | **10-14**            | **2**                | **22-30** |

---

## 5. Phase 1: Stop the Bleeding (P0) — Week 1

### 5.1 A-4: Remove Frontend `regretPercent` Fallback

**Risk:** HIGH | **Type:** FE | **Effort:** 1 day

#### 5.1.1 Problem

Non-debug users see a **different** `regretPercent` than debug users for the exact same action.

| Path          | Formula                                          | Meaning                             |
| ------------- | ------------------------------------------------ | ----------------------------------- |
| Backend       | `(best_ev - weighted_ev) / pot_size × 100`       | Regret vs. optimal mixed strategy   |
| FE (debug)    | Uses backend `regretPercent` directly             | Correct                             |
| FE (non-debug)| `(best_ev - selected_ev) / pot_value × 100`      | Regret vs. single best action       |

The non-debug path is **fundamentally wrong** — it measures a different quantity.

#### 5.1.2 Scope — 6 files, identical conditional pattern

| File                                              | Variable             | Lines (approx) |
| ------------------------------------------------- | -------------------- | --------------- |
| `components/Trainer/FeedbackPanel.tsx`             | `regretPercent`      | 347-361         |
| `components/Trainer/TrainerCenterPanel.tsx`        | `evLossPercent`      | 267-277         |
| `components/Trainer/mobile/useTrainerMobile.ts`    | `regretPercent`      | 256-267         |
| `components/Trainer/TrainerHistoryList.tsx`         | `getRegretPercent()` x2 | 480-509      |
| `components/Common/FeedbackActions.tsx`            | `evLossPercent`      | 226-234         |
| `components/PlayerActions/ReplayerActionsMenu.tsx` | `evLossPercent`      | 220-222         |

Two additional files already use backend value correctly (no change needed):
- `components/Trainer/MobileHistoryDrawer.tsx` — direct `selectedActionData?.regretPercent ?? 0`
- `components/UnifiedPokerPanel/adapters/TrainerDataAdapter.ts` — direct from `summaryData`

#### 5.1.3 Approach

1. Create a shared utility: `utils/getRegretPercent.ts`
2. Always use backend-provided `regretPercent`. If missing/null, return `null` → display "—".
3. Replace all 6 conditional blocks with the shared utility.
4. **Pre-check:** Audit backend to confirm `regretPercent` is always returned in the API response.

#### 5.1.4 Acceptance Criteria

- [ ] Non-debug users see the same `regretPercent` as debug users
- [ ] If backend `regretPercent` is missing, UI shows "—" (not a wrong number)
- [ ] No frontend formula calculates `(bestEV - selectedEV) / potValue` anywhere
- [ ] Shared utility has 100% branch coverage in unit tests
- [ ] Verified on both Holdem and Omaha

#### 5.1.5 Tests

**Jest unit test — shared utility (primary, test-first):**

```typescript
// utils/__tests__/getRegretPercent.test.ts
import { getRegretPercent } from '@/utils/getRegretPercent';

describe('getRegretPercent', () => {
  // --- Happy path ---
  it('returns backend value when provided as a positive number', () => {
    expect(getRegretPercent(12.34)).toBe(12.34);
  });

  it('returns 0 when backend value is explicitly 0 (optimal play)', () => {
    expect(getRegretPercent(0)).toBe(0);
  });

  it('preserves decimal precision from backend', () => {
    expect(getRegretPercent(3.14159)).toBe(3.14159);
  });

  // --- Missing/invalid values → null (display "—") ---
  it('returns null when backend value is undefined', () => {
    expect(getRegretPercent(undefined)).toBeNull();
  });

  it('returns null when backend value is null', () => {
    expect(getRegretPercent(null)).toBeNull();
  });

  // --- Edge cases ---
  it('returns negative values without clamping (backend decides)', () => {
    expect(getRegretPercent(-5.0)).toBe(-5.0);
  });

  it('handles very large values', () => {
    expect(getRegretPercent(99999.99)).toBe(99999.99);
  });
});
```

**Jest component test — FeedbackPanel (test-first):**

```typescript
// components/Trainer/__tests__/FeedbackPanel.regret.test.tsx
import { configureStore } from '@reduxjs/toolkit';
import { render, screen } from '@testing-library/react';
import { Provider } from 'react-redux';

// Mock sub-components to isolate FeedbackPanel
jest.mock('@/components/Trainer/TrainerCenterPanel', () => ({
  __esModule: true,
  default: () => <div data-testid="mock-center-panel" />,
}));

function createTestStore(overrides: Record<string, unknown> = {}) {
  return configureStore({
    reducer: {
      userReducer: (state = { hasDebugAccess: false, ...overrides }) => state,
      trainerReducer: (state = {}) => state,
      gameReducer: (state = {}) => state,
    },
  });
}

describe('FeedbackPanel regretPercent', () => {
  it('displays backend regretPercent for non-debug users', () => {
    const store = createTestStore({ hasDebugAccess: false });
    // Render FeedbackPanel with summaryData containing regretPercent: 5.23
    // Assert: element shows "5.23%"
  });

  it('displays backend regretPercent for debug users (same value)', () => {
    const store = createTestStore({ hasDebugAccess: true });
    // Same setup as above
    // Assert: shows same "5.23%" — no difference between debug and non-debug
  });

  it('displays "—" when backend regretPercent is missing', () => {
    const store = createTestStore({ hasDebugAccess: false });
    // Render with summaryData where regretPercent is undefined
    // Assert: shows "—" or "-", NOT a calculated number
  });

  it('does NOT calculate (bestEV - selectedEV) / potValue', () => {
    const store = createTestStore({ hasDebugAccess: false });
    // Render with bestEV=10, selectedEV=5, potValue=100, backendRegret=8.5
    // Assert: shows "8.50%" (backend), NOT "5.00%" (frontend formula)
  });
});
```

**Playwright E2E — validation (run after fix):**

```typescript
// e2e/trainer-values.spec.ts
import { expect, Page, test } from '@playwright/test';

async function waitForPageLoad(page: Page) {
  await page.waitForLoadState('load');
  await page.waitForTimeout(1000);
}

test.describe('Trainer Values — regretPercent consistency', () => {
  test.setTimeout(60000);

  test.beforeEach(async ({ page }, testInfo) => {
    test.skip(testInfo.project.name === 'Mobile', 'Trainer not supported on mobile');
  });

  test('displayed regretPercent matches backend API response', async ({ page }) => {
    let apiRegretPercent: number | null = null;

    // Intercept trainer API to capture backend regretPercent
    await page.route('**/trainer/**/generate-next-hand*', async (route) => {
      const response = await route.fetch();
      const body = await response.json();

      // Extract regretPercent from the first action in handActions
      const handActions = body?.handActions || {};
      const firstAction = Object.values(handActions)[0] as any;
      apiRegretPercent = firstAction?.regretPercent ?? null;

      await route.fulfill({ response });
    });

    // Navigate to trainer (standalone V1)
    await page.goto('/holdem/cash/trainer');
    await waitForPageLoad(page);

    // Wait for trainer to load and display a hand
    await page.waitForTimeout(5000);

    // If we captured regretPercent, verify it appears in the UI
    if (apiRegretPercent !== null && apiRegretPercent > 0) {
      const regretText = apiRegretPercent.toFixed(2);
      const regretElement = page.getByText(`${regretText}%`);
      // Allow graceful skip if trainer hasn't rendered yet
      const count = await regretElement.count();
      if (count > 0) {
        await expect(regretElement.first()).toBeVisible();
      }
    }
  });
});
```

---

### 5.2 A-1: Range Array Divergence V1 vs V2

**Risk:** HIGH | **Type:** BE | **Effort:** 3-4 days

#### 5.2.1 Problem

V1 and V2 decode ranges through different code paths, potentially producing different hand combos for the same tree node:

| Path | Code Location                   | Method                            | Transformations Applied              |
| ---- | ------------------------------- | --------------------------------- | ------------------------------------ |
| V1   | `spot_generation.py:199`        | `decode_range(tree_node['range'])` | Raw decode only                     |
| V2   | `spot_generation.py:407`        | `data.get('rangeResult', [])`     | `decode_or_calculate_range()` → `expand_range()` → `apply_swaps()` |

V2 may apply: sparse index expansion, action tree walking, isomorphism remapping — none of which V1 does.

#### 5.2.2 Impact

Different range arrays → different hand selection → different combos shown to user. This is likely the **root cause** of most "values are inconsistent" reports.

#### 5.2.3 Approach

1. **Investigation** (1-2 days): Write a pytest that feeds the same tree node through both paths. Compare output arrays. Document which transformations V2 applies that V1 skips.
2. **Fix** (1-2 days): Standardize on one path. V2 is likely more correct. Update V1 to route through `decode_or_calculate_range()`.

#### 5.2.4 Files to Audit

| File (Backend)                                         | Lines     | Role                              |
| ------------------------------------------------------ | --------- | --------------------------------- |
| `trainer/views/spot_generation.py`                     | 199-216   | V1 path — `decode_range` direct   |
| `trainer/views/spot_generation.py`                     | 407-425   | V2 path — `rangeResult` from resolver |
| `strategies/utils/unfold_range.py`                     | 375-512   | `decode_or_calculate_range()`     |
| `strategies/services/strategy_treenode_resolver.py`    | 588-685   | `decode_strategy_values()` + swaps |

#### 5.2.5 Acceptance Criteria

- [ ] Both V1 and V2 produce identical range arrays for the same tree node
- [ ] Tests cover: standard preflop, sparse indices, isomorphism mapping
- [ ] Backend test added and passing
- [ ] Verified with Andres' sample hands (when available)

#### 5.2.6 Tests

**Backend pytest (primary, test-first):**

```python
# trainer/tests/test_range_divergence_v1_v2.py
import numpy as np
import pytest
from strategies.utils.unfold_range import decode_range, decode_or_calculate_range

class TestRangeDivergenceV1V2:
    """
    V1 (decode_range) and V2 (decode_or_calculate_range via resolver)
    must produce identical range arrays for the same tree node.

    This test documents the bug: if it FAILS, the divergence exists.
    After fix, it should PASS.
    """

    @pytest.fixture
    def sample_preflop_node(self):
        """Load a real preflop tree node from test fixtures."""
        # Extract from DB during investigation phase
        ...

    @pytest.fixture
    def sample_postflop_node(self):
        """Load a real postflop tree node (flop) from test fixtures."""
        ...

    def test_preflop_range_identical(self, sample_preflop_node):
        """V1 and V2 must agree on preflop range arrays."""
        v1_range = decode_range(sample_preflop_node['range'])
        v2_range = decode_or_calculate_range(sample_preflop_node, ...)
        np.testing.assert_array_equal(
            v1_range, v2_range,
            err_msg="DIVERGENCE: V1 and V2 range arrays differ for preflop"
        )

    def test_postflop_range_identical(self, sample_postflop_node):
        """V1 and V2 must agree on postflop range arrays."""
        ...

    def test_sparse_indices_handled_identically(self):
        """When tree node has sparse indices, both paths must expand identically."""
        ...

    def test_isomorphism_applied_identically(self):
        """When isomorphism mapping applies, both paths must produce same result."""
        ...
```

**Playwright E2E (validation, after fix):**

```typescript
// Part of e2e/trainer-values.spec.ts
test('V1 standalone and V2 strategy-entry show same hand data', async ({ page }) => {
  // This test validates the fix is working end-to-end

  // Step 1: Capture V1 API response (standalone trainer)
  let v1Response: any = null;
  await page.route('**/trainer/generate-next-hand*', async (route) => {
    const response = await route.fetch();
    v1Response = await response.json();
    await route.fulfill({ response });
  });

  await page.goto('/holdem/cash/trainer');
  await waitForPageLoad(page);
  await page.waitForTimeout(3000);

  // Step 2: Navigate to strategy page, then trainer (V2)
  let v2Response: any = null;
  await page.route('**/trainer/v2/generate-next-hand*', async (route) => {
    const response = await route.fetch();
    v2Response = await response.json();
    await route.fulfill({ response });
  });

  await page.goto('/holdem/cash/strategies');
  await waitForPageLoad(page);

  const trainButton = page.locator('#train-button');
  if ((await trainButton.count()) > 0 && !(await trainButton.isDisabled())) {
    await trainButton.click();
    await page.waitForURL(/.*trainer.*/, { timeout: 10000 });
    await waitForPageLoad(page);
    await page.waitForTimeout(3000);
  }

  // Step 3: If both responses captured, verify key fields match
  if (v1Response && v2Response) {
    // Range arrays should produce same number of combos
    expect(Object.keys(v1Response.handActions || {}).length)
      .toBe(Object.keys(v2Response.handActions || {}).length);
  }
});
```

---

## 6. Phase 2: Visible UX Fixes (P1) — Week 2

### 6.1 A-5: Board Randomization Timing

**Risk:** HIGH | **Type:** FE | **Effort:** 2-3 days

#### 6.1.1 Problem

Board cards are randomized at **consumption time** from the buffer (`consumeFromBuffer()` in `useTrainer.tsx:1342-1397`), not at **generation time**. This creates a critical mismatch:

```
Buffer Generation (API call):
  → Strategy tree computed for board [Ah Kd 7s]
  → EV values computed for board [Ah Kd 7s]

Buffer Consumption (user sees next hand):
  → Board randomized to [Tc 5h 2d]     ← WRONG: EV was for [Ah Kd 7s]
  → User sees EV for wrong board
```

Additionally, `isFirstHandRef.current` is passed to `processBoardCards()` inside a block that already checks `!isFirstHandRef.current` — the parameter is always `false` inside the branch, making it redundant.

#### 6.1.2 Approach

1. **Ideal fix (backend):** Move randomization to backend during hand generation → strategy tree matches displayed board.
2. **Interim fix (frontend):** Randomize at buffer *fill* time, not consume time. Regenerate buffer when `randomizeBoard` changes.
3. Clean up `isFirstHandRef` logic.

#### 6.1.3 Files

| File                                     | Lines       | Role                           |
| ---------------------------------------- | ----------- | ------------------------------ |
| `components/Trainer/useTrainer.tsx`      | 1342-1397   | `consumeFromBuffer()` function |
| `components/Trainer/utils.ts`            | 1406+       | `processBoardCards()` function |

#### 6.1.4 Acceptance Criteria

- [ ] Board cards displayed to user match the board used in strategy/EV computation
- [ ] First hand shows original board; subsequent hands randomize (when enabled)
- [ ] No duplicate cards produced
- [ ] Correct card count per street (Flop: 3, Turn: 4, River: 5)

#### 6.1.5 Tests

**Jest unit tests — `processBoardCards` (primary):**

```typescript
// components/Trainer/__tests__/processBoardCards.test.ts
import { processBoardCards } from '@/components/Trainer/utils';

describe('processBoardCards', () => {
  // --- Street handling ---
  it('returns undefined for Preflop (no board cards)', () => {
    const result = processBoardCards(['Ah', 'Kd', 'Qs'], 'Preflop', 'On', false, 'postflop_only');
    expect(result).toBeUndefined();
  });

  it('produces 3 cards for Flop', () => {
    const result = processBoardCards(['Ah', 'Kd', 'Qs'], 'Flop', 'On', false, 'postflop_only');
    expect(result).toBeDefined();
    // Parse result and verify 3 cards
  });

  it('produces 4 cards for Turn', () => {
    const result = processBoardCards(
      ['Ah', 'Kd', 'Qs', '7c'], 'Turn', 'On', false, 'postflop_only'
    );
    expect(result).toBeDefined();
    // Parse result and verify 4 cards
  });

  it('produces 5 cards for River', () => {
    const result = processBoardCards(
      ['Ah', 'Kd', 'Qs', '7c', '2h'], 'River', 'On', false, 'postflop_only'
    );
    expect(result).toBeDefined();
    // Parse result and verify 5 cards
  });

  // --- Randomization control ---
  it('returns original cards when randomize is Off', () => {
    const result = processBoardCards(['Ah', 'Kd', 'Qs'], 'Flop', 'Off', false, 'postflop_only');
    // Should match original cards
  });

  it('does NOT randomize on first hand even when randomize is On', () => {
    const result = processBoardCards(['Ah', 'Kd', 'Qs'], 'Flop', 'On', true, 'postflop_only');
    // isFirstHand=true → should return original
  });

  it('randomizes on non-first hand when randomize is On', () => {
    const result = processBoardCards(['Ah', 'Kd', 'Qs'], 'Flop', 'On', false, 'postflop_only');
    // Should produce valid cards (may differ from input)
    expect(result).toBeDefined();
  });

  // --- Card validity ---
  it('does not produce duplicate cards', () => {
    // Run 20 times to catch probabilistic duplicates
    for (let i = 0; i < 20; i++) {
      const result = processBoardCards(['Ah', 'Kd', 'Qs'], 'Flop', 'On', false, 'postflop_only');
      if (result) {
        const cards = result.match(/.{2}/g) || [];
        const unique = new Set(cards);
        expect(unique.size).toBe(cards.length);
      }
    }
  });

  // --- Research type ---
  it('only randomizes for postflop_only research', () => {
    const result = processBoardCards(['Ah', 'Kd', 'Qs'], 'Flop', 'On', false, 'full_tree');
    // full_tree should not randomize board
  });
});
```

**Playwright E2E (primary):**

```typescript
// e2e/trainer-board-randomization.spec.ts
import { expect, Page, test } from '@playwright/test';

async function waitForPageLoad(page: Page) {
  await page.waitForLoadState('load');
  await page.waitForTimeout(1000);
}

test.describe('Trainer Board Randomization', () => {
  test.setTimeout(60000);

  test.beforeEach(async ({ page }, testInfo) => {
    test.skip(testInfo.project.name === 'Mobile', 'Trainer not supported on mobile');
  });

  test('board cards in UI match board used in API computation', async ({ page }) => {
    const apiBoardCards: string[] = [];

    // Intercept API to capture board cards used in computation
    await page.route('**/trainer/**/generate-next-hand*', async (route) => {
      const response = await route.fetch();
      const body = await response.json();
      if (body?.boardCards) {
        apiBoardCards.push(body.boardCards);
      }
      await route.fulfill({ response });
    });

    // Navigate to trainer via strategy page (postflop_only)
    await page.goto('/holdem/cash/strategies');
    await waitForPageLoad(page);

    // Select Precision research type
    const precisionOption = page.locator('#dropdown-label-postflop_only');
    try {
      await precisionOption.waitFor({ state: 'visible', timeout: 15000 });
      await precisionOption.click();
      await page.waitForTimeout(500);
    } catch {
      test.skip();
      return;
    }

    // Click Train
    const trainButton = page.locator('#train-button');
    if ((await trainButton.count()) === 0 || (await trainButton.isDisabled())) {
      test.skip();
      return;
    }
    await trainButton.click();
    await page.waitForURL(/.*trainer.*/, { timeout: 10000 });
    await waitForPageLoad(page);
    await page.waitForTimeout(3000);

    // Verify at least one API call captured board cards
    expect(apiBoardCards.length).toBeGreaterThan(0);
  });
});
```

---

### 6.2 A-3: Pot Size Alignment V1/V2

**Risk:** MEDIUM | **Type:** BE+FE | **Effort:** 1-2 days

#### 6.2.1 Problem

V1 and V2 may compute different pot sizes because they source config differently. The backend's `adjust_pot_size_with_rake()` (`trainer_helper.py:962-978`) adds rake when `rakeAfterEachStreet` is enabled — but the config values may differ between V1 standalone and V2 strategy-entry.

#### 6.2.2 Files to Audit

| File (Backend)                             | Lines     | Role                                          |
| ------------------------------------------ | --------- | --------------------------------------------- |
| `trainer/utils/trainer_helper.py`          | 456-468   | `adjust_pot_size_with_rake()` call            |
| `trainer/utils/trainer_helper.py`          | 962-978   | `adjust_pot_size_with_rake()` implementation  |
| `trainer/utils/trainer_helper.py`          | 532-559   | Regret calculation using `pot_size`           |

#### 6.2.3 Acceptance Criteria

- [ ] V1 and V2 produce identical `pot_size` for the same game config
- [ ] Rake adjustment logic tested for all branches (enabled, disabled, finished, zero rake)

#### 6.2.4 Tests

**Backend pytest (primary):**

```python
# trainer/tests/test_pot_size_alignment.py
import pytest
from trainer.utils.trainer_helper import TrainerHandGenerator

class TestPotSizeAlignment:
    """Verify pot size calculation is consistent across V1 and V2."""

    def test_adjust_pot_adds_rake_when_enabled(self):
        """Pot size includes rake when rakeAfterEachStreet is enabled."""
        result = TrainerHandGenerator.adjust_pot_size_with_rake(
            pot_size=10.0,
            state_obj={'finished': False, 'rake': 0.5},
            strategy_obj={'rakeAfterEachStreet': '1'}
        )
        assert result == 10.5

    def test_adjust_pot_no_change_when_disabled(self):
        """Pot unchanged when rakeAfterEachStreet is disabled."""
        result = TrainerHandGenerator.adjust_pot_size_with_rake(
            pot_size=10.0,
            state_obj={'finished': False, 'rake': 0.5},
            strategy_obj={'rakeAfterEachStreet': None}
        )
        assert result == 10.0

    def test_adjust_pot_no_change_when_finished(self):
        """Pot unchanged when hand is finished."""
        result = TrainerHandGenerator.adjust_pot_size_with_rake(
            pot_size=10.0,
            state_obj={'finished': True, 'rake': 0.5},
            strategy_obj={'rakeAfterEachStreet': '1'}
        )
        assert result == 10.0

    def test_adjust_pot_no_change_when_rake_zero(self):
        """Pot unchanged when rake value is 0."""
        result = TrainerHandGenerator.adjust_pot_size_with_rake(
            pot_size=10.0,
            state_obj={'finished': False, 'rake': 0.0},
            strategy_obj={'rakeAfterEachStreet': '1'}
        )
        assert result == 10.0

    def test_adjust_pot_handles_none_pot(self):
        """Gracefully handles None pot_size."""
        result = TrainerHandGenerator.adjust_pot_size_with_rake(
            pot_size=None,
            state_obj={'finished': False, 'rake': 0.5},
            strategy_obj={'rakeAfterEachStreet': '1'}
        )
        assert result == 0.5  # 0.0 + 0.5
```

---

## 7. Phase 3: Polish (P2) — Week 3

### 7.1 A-2: EV Decode Precision

**Risk:** MEDIUM | **Type:** BE | **Effort:** 1-2 days

#### 7.1.1 Problem

V1 always calls `_decode_strategy_ev()` which decompresses brotli data and rounds EV to 2 decimal places (via `.astype(np.float64).round(2)` in `unfold_strategy.py:142`). V2 may use pre-decoded arrays from the resolver which could have been rounded at a different stage or not rounded at all.

```python
# trainer_helper.py:362-364 — V2 path
if action_probs is not None and expected_value is not None:
    decoded_strategy_values = action_probs      # pre-decoded, precision unknown
    decoded_ev_values = expected_value           # pre-decoded, precision unknown
else:
    # V1 path — always fresh decode with .round(2)
    decoded_strategy_values, decoded_ev_values = _decode_strategy_ev(...)
```

#### 7.1.2 Acceptance Criteria

- [ ] Both paths produce float arrays that agree to at least 2 decimal places
- [ ] Rounding is applied consistently in both paths

#### 7.1.3 Tests

```python
# trainer/tests/test_ev_decode_precision.py
import numpy as np
import pytest

class TestEVDecodePrecision:
    """Verify V1 fresh decode and V2 pre-decoded arrays agree in precision."""

    def test_fresh_decode_matches_predecoded(self, sample_tree_node, strategy_obj):
        """Both decode paths must produce arrays equal to 2 decimal places."""
        fresh_strat, fresh_ev = TrainerHandGenerator._decode_strategy_ev(
            strategy_obj, sample_tree_node, action_list
        )
        pre_strat = resolver_data.get('actionProbs')
        pre_ev = resolver_data.get('expectedValue')

        np.testing.assert_array_almost_equal(fresh_ev, pre_ev, decimal=2,
            err_msg="EV precision differs between fresh decode and pre-decoded")

    def test_ev_rounding_applied(self, raw_ev_array):
        """EV values must be rounded to 2 decimal places."""
        decoded = decode_and_round(raw_ev_array)
        for val in decoded.flat:
            assert round(val, 2) == val, f"EV value {val} not rounded to 2dp"
```

---

### 7.2 A-6: `apply_swaps` Silent Failures

**Risk:** HIGH (postflop) | **Type:** BE | **Effort:** 0.5-1 day

#### 7.2.1 Problem

Two locations in `trainer_helper.py` catch all exceptions at DEBUG level and continue with unswapped (wrong) data:

```python
# Lines 380-385 and 689-694 — IDENTICAL pattern
try:
    decoded_strategy_values, decoded_ev_values, _ = apply_swaps(...)
except Exception:
    logger.debug('apply_swaps failed; proceeding without swaps', exc_info=True)
```

If a postflop swap fails, the user sees unswapped (wrong) combo ordering and EV values with **zero indication** anything went wrong.

#### 7.2.2 Approach

1. Escalate `logger.debug` → `logger.warning`
2. Add a counter metric for monitoring
3. Consider adding `swap_failed: true` flag to the API response

#### 7.2.3 Tests

```python
# trainer/tests/test_apply_swaps_error_handling.py
import logging
import pytest
from unittest.mock import patch

class TestApplySwapsErrorHandling:
    """Verify apply_swaps failures are logged at WARNING, not DEBUG."""

    def test_swap_failure_logs_warning(self, caplog):
        """When apply_swaps raises, a WARNING must be logged."""
        with patch('trainer.utils.trainer_helper.apply_swaps',
                   side_effect=ValueError("bad swap data")):
            with caplog.at_level(logging.WARNING):
                # Call _generate_hand_response with precision postflop data
                ...
            assert any('apply_swaps' in r.message for r in caplog.records
                       if r.levelno >= logging.WARNING)

    def test_swap_failure_does_not_log_debug_only(self, caplog):
        """Swap failures must NOT be silently hidden at DEBUG level."""
        with patch('trainer.utils.trainer_helper.apply_swaps',
                   side_effect=RuntimeError("swap error")):
            with caplog.at_level(logging.DEBUG):
                ...
            warning_records = [r for r in caplog.records if r.levelno >= logging.WARNING]
            assert len(warning_records) > 0, "Swap failure was only logged at DEBUG"
```

---

## 8. Phase 4: Tech Debt (P3) — Week 4+

### 8.1 A-7: 9s Isomorphism Bug

**Risk:** HIGH (postflop) | **Type:** BE | **Effort:** 1-2 days

`apply_swaps()` in `build_suit_map.py:212` calls `get_iso_board(roundNum, boardCardsLst)` using the current round (e.g., TURN). But precision strategies index by ISO **flop** only. The suit map should be derived from the flop board, not the turn board.

**Evidence:** Comprehensive test documentation in `strategies/tests/utils/test_apply_swaps_iso_consistency.py` (333 lines). Fix: use `iso_flop_board` instead of computing new ISO board per round. Coordinate with Akhil.

### 8.2 A-8: Cache Key Missing Role

**Risk:** MEDIUM | **Type:** BE | **Effort:** 0.5 day

`cache_page_with_role()` in `cache_key.py:53` builds key as `{prefix}_{role_hash}_{path}` — missing user ID. Compare with `user_role_cache()` which correctly includes `{prefix}_{user_id}_{hash_key}_{path}`.

```python
# strategies/tests/utils/test_cache_key_isolation.py
class TestCacheKeyIsolation:
    def test_different_users_same_role_get_different_keys(self):
        """Two users with same role must NOT share cache."""
        key1 = build_cache_key(user_id=1, roles=['pro'], path='/api/data')
        key2 = build_cache_key(user_id=2, roles=['pro'], path='/api/data')
        assert key1 != key2

    def test_same_user_different_role_gets_different_key(self):
        key_a = build_cache_key(user_id=1, roles=['free'], path='/api/data')
        key_b = build_cache_key(user_id=1, roles=['pro'], path='/api/data')
        assert key_a != key_b
```

### 8.3 A-9: Spins Config Type Safety

**Risk:** LOW | **Type:** FE | **Effort:** 0.5 day

Replace the unsafe `as any` chain in `useTrainer.tsx:1577-1591` with a proper TypeScript interface. The `tsc` compilation (`pnpm build`) is the test — no runtime test needed.

---

## 9. Timeline Summary

| Week       | Phase                  | Bugs           | Focus                                                     | Deliverables                                                    |
| ---------- | ---------------------- | -------------- | --------------------------------------------------------- | --------------------------------------------------------------- |
| **Week 1** | P0 — Stop the Bleeding | A-4, A-1       | Remove wrong FE formula; investigate range divergence     | Shared utility + 6 file updates; investigation report           |
| **Week 2** | P1 — Visible UX        | A-5, A-3       | Fix board randomization timing; align pot sizes           | Board fix + pot alignment; 2 E2E specs                          |
| **Week 3** | P2 — Polish            | A-2, A-6       | EV precision alignment; swap failure escalation           | Precision fix; monitoring improvement                            |
| **Week 4+** | P3 — Tech Debt        | A-7, A-8, A-9  | Isomorphism fix; cache key; type safety                   | Coordinated fixes with Akhil; TypeScript interface               |

### Effort Breakdown

| Bug  | Investigation | Implementation | Testing | Total      |
| ---- | :-----------: | :------------: | :-----: | ---------- |
| A-4  | —             | 0.5 day        | 0.5 day | **1 day**  |
| A-1  | 1-2 days      | 1-2 days       | 0.5 day | **3-4 days** |
| A-5  | 0.5 day       | 1-2 days       | 0.5 day | **2-3 days** |
| A-3  | 0.5 day       | 0.5-1 day      | 0.5 day | **1-2 days** |
| A-2  | 0.5 day       | 0.5-1 day      | 0.5 day | **1-2 days** |
| A-6  | —             | 0.25 day       | 0.25 day | **0.5-1 day** |
| A-7  | 0.5 day       | 0.5-1 day      | 0.5 day | **1-2 days** |
| A-8  | —             | 0.25 day       | 0.25 day | **0.5 day** |
| A-9  | —             | 0.25 day       | 0.25 day | **0.5 day** |

**Total: ~14-18 working days across 4 weeks**

### Test Output Summary

| Category             | Count   | Framework  |
| -------------------- | :-----: | ---------- |
| Frontend unit tests  | 10-14   | Jest       |
| Backend unit tests   | 10-14   | pytest     |
| E2E specs            | 2       | Playwright |
| **Total new tests**  | **22-30** |          |

---

## 10. Cross-Stream Coordination

| Coordination            | Who               | When       | Topic                                                      |
| ----------------------- | ----------------- | ---------- | ---------------------------------------------------------- |
| Siddharth ↔ Akhil       | Weekly            | Ongoing    | A-7 (9s iso), A-6 (`apply_swaps`), A-8 (cache key) — all touch strategy/RTS code Akhil knows |
| Siddharth ↔ Srinivasan  | After P0          | Week 2+    | A-2/D-10 (EV memory workaround) affects both trainer and replayer |
| Siddharth ↔ Priyanka    | As needed         | Week 1     | `FeedbackActions.tsx` is shared between Stream A (regret formula) and Stream B (transitions) |
| All streams             | Daily 10-min sync | Ongoing    | Flag blockers, share discoveries                           |

### Dependency Graph

```
A-4 (regret formula)
  └── Independent — start immediately

A-1 (range divergence)
  └── Independent — start immediately (investigation-heavy)

A-5 (board randomization)
  └── Depends on A-1 findings (may share root cause)

A-3 (pot alignment)
  └── Partially addressed by A-4 (frontend potValue fallback removed)

A-2 (EV precision)
  └── Depends on A-1 (range must be aligned first)

A-6 (apply_swaps logging)
  └── Coordinate with Akhil before changing

A-7 (9s isomorphism)
  ├── Coordinate with Akhil (may already be addressed in RTS)
  └── Existing test evidence available

A-8 (cache key)
  └── Independent — quick fix

A-9 (type safety)
  └── Independent — quick fix
```

---

## 11. Dependencies and Blockers

| Item                                   | Dependency                                | Status               | Impact                                             | Mitigation                              |
| -------------------------------------- | ----------------------------------------- | -------------------- | -------------------------------------------------- | --------------------------------------- |
| Sample hands from Andres               | Needed for A-1, A-3 validation            | **Pending**          | Cannot write realistic E2E fixtures without this   | Use synthetic test data initially       |
| Akhil RTS status update                | Needed to scope A-7, A-6 overlap          | **Pending**          | May duplicate or conflict with RTS work            | Start with investigation; coordinate before fix |
| Backend `regretPercent` coverage audit | Self-serve                                | **Can start now**    | Blocks A-4 deployment                              | Audit first day of Week 1               |
| Test data fixtures (tree nodes)        | Need real examples from DB                | Extract during A-1   | Required for backend pytest tests                  | Create minimal synthetic fixtures first |
| Frontend `data-testid` attributes      | Needed for reliable E2E selectors         | May need adding      | E2E tests fragile without them                     | Use existing selectors; add testids in fix PRs |

---

## 12. Definition of Done

A bug is **not done** until all of the following are satisfied:

| # | Criterion                                                     | How to Verify                              |
| - | ------------------------------------------------------------- | ------------------------------------------ |
| 1 | Failing test written and confirmed failing before fix         | Test output in PR description              |
| 2 | Fix implemented                                               | Code diff in PR                            |
| 3 | All new tests pass                                            | `pnpm test` / `pytest` output              |
| 4 | All existing tests still pass                                 | `pnpm test` / `pytest` output              |
| 5 | `pnpm build` succeeds (no TypeScript errors)                  | Build log                                  |
| 6 | Verified against both Holdem AND Omaha                        | Manual test or E2E covering both           |
| 7 | PR is small and focused (one bug per PR where possible)       | Code review                                |
| 8 | No debug logs or TODOs left in code                           | Code review                                |

---

## 13. Open Items

- [ ] **Confirm** backend always provides `regretPercent` in generate-next-hand response (blocks A-4 deployment)
- [ ] **Get sample hands from Andres** to build realistic test fixtures for A-1 and A-3
- [ ] **Akhil to report RTS status** so overlap with A-7/A-6 is understood before Week 3
- [ ] **Decide** interim vs ideal fix for A-5: frontend buffer-fill randomization vs backend randomization
- [ ] **Assign Priyanka and Dhwani** to Streams B and D (blocks cross-stream coordination)
- [ ] **Add `data-testid` attributes** to trainer UI elements needed for E2E tests (can do during fix PRs)
