# Trainer V3 Phase 1 Release Plan

**Target Date:** Wednesday, Feb 26, 2026  
**Prepared by:** Engineering Team  
**Branch:** `feature/gl-proto`

---

## Executive Summary

This plan outlines a Phase 1 release of Trainer V3 focusing on **preflop training** and **custom spot building** with session persistence. We will disable/hide incomplete features (villain range, standard postflop modes) to ship a stable product.

### Scope Summary

| In Scope | Out of Scope (Hidden/Disabled) |
|----------|-------------------------------|
| Preflop training (RFI, Facing Open, vs3Bet, vs4Bet, Squeeze) | Standard postflop modes (Flop/Turn/River from dropdown) |
| **Custom mode** — build action sequences like Strategy screen | Villain range/matrix display |
| Hero range display | Street/Full Hand game modes |
| Landing page with config selection | Statistics drill-down |
| Session persistence (Track A or B) | Adaptive difficulty |
| Basic training loop (deal hand → select action → feedback) | History tab deep features |
| Crash fixes for the above paths | |

### Custom Mode Description

The **Custom** mode allows users to build their own training scenarios by constructing action sequences similar to the Strategy screen. Users can:

1. **Select actions position-by-position** — Each position shows available actions (fold, call, raise with specific sizing) fetched from the real strategy tree API
2. **Build to any decision point** — Stop at any point in the sequence to train from that exact spot
3. **Progressive street disclosure** — After completing preflop, users can add flop cards and continue building turn/river actions
4. **Database-aware constraints**:
   - Preflop-only database → Only preflop actions available
   - Full tree (expand) → Preflop → Flop → Turn → River
   - Precision database → All streets with auto-switched strategies

This mirrors the `/get-player-next-actions` API flow used by the Strategies module.

---

## Technical Approach: Two-Track Strategy

We have two options for session persistence. **Track A is preferred** — attempt it first, fall back to Track B only if Track A has fundamental issues.

### Track A: Fix Existing V3 Session API (Recommended)

**Effort:** 4-8 hours

The `trainer_v3/` Django app already has complete implementation:
- `POST /trainer/v3/session/start` — creates session + generates hands internally
- `POST /trainer/v3/session/{id}/hands` — saves hand results  
- `POST /trainer/v3/session/{id}/end` — ends session with reason
- Database models exist with migrations applied

**Why it may not be working:**
- Config validation rejecting requests
- Auth/permission issues
- Network errors being swallowed
- Frontend `startTrainerSession()` failing silently

**Track A Tasks:**
| Task | Effort | Priority |
|------|--------|----------|
| A.1 Test `POST /trainer/v3/session/start` via curl/Postman | 1h | P0 |
| A.2 Check backend logs for `trainer_v3` errors | 1h | P0 |
| A.3 Verify frontend `startTrainerSession()` is called | 1h | P0 |
| A.4 Fix any config/auth issues found | 2-4h | P0 |
| A.5 Verify session persistence end-to-end | 1h | P0 |

**If Track A works:** Skip Track B entirely, proceed to UI fixes.

---

### Track B: Hybrid V2 API + Session Storage (Fallback)

**Effort:** 14-20 hours

If V3 session API has fundamental backend issues, use V2 for hand generation while keeping V3 for session storage only.

**Architecture:**
```
Frontend: Click Start
  → POST /trainer/v3/session/start (creates session row, skipGeneration=true)
  → GET /trainer/v2/generate-next-hand (separate call for hands)
  
Frontend: Complete Hand
  → POST /trainer/v3/session/{id}/hands (save result)
  
Frontend: End Session
  → POST /trainer/v3/session/{id}/end
```

**Track B Tasks:**

| Task | File(s) | Effort | Priority |
|------|---------|--------|----------|
| B.1 Add `skipGeneration` flag to StartSessionView | `trainer_v3/views/session_views.py` | 2h | P0 |
| B.2 Create `useTrainerHandV2` hook | New: `hooks/useTrainerHandV2.ts` | 4h | P0 |
| B.3 Modify `useTrainerPersistence` to use V2 for hands | `useTrainerPersistence.ts` | 3h | P0 |
| B.4 Wire session creation → V2 hand fetch → session save | `useTrainingSession.ts` | 4h | P0 |
| B.5 Adapt V2 response to V3 hand format | `v3HandAdapter.ts` | 3h | P0 |
| B.6 Test full flow end-to-end | Manual testing | 2h | P0 |

---

## API Reference

### V3 Session API (Track A)

```
POST /trainer/v3/session/start
Body: { config: TrainerSessionConfig, batchSize: 25 }
Response: { sessionId, hands[], hasMore, config }

POST /trainer/v3/session/{id}/hands  
Body: { hands: HandResult[] }
Response: { saved, stats }

POST /trainer/v3/session/{id}/end
Body: { reason: 'user_exit' | 'completed' }
Response: { success }
```

### V2 Hand Generation API (Track B fallback)

```
GET /trainer/v2/generate-next-hand
  ?numPlayers=6
  &spot=unopened|facingOpen|facing3Bet|facing4Bet|facingOpenPlusCaller
  &heroPosition=BTN
  &villainPosition=CO
  &BB=5
  &stackInBB=100

Response: { playerHandsPlayers[], fold_list, handActions, configValues, ... }
```

### Spot Mapping (Frontend → Backend)

```typescript
const SPOT_MAP: Record<string, string> = {
  'rfi': 'unopened',
  'vsOpen': 'facingOpen',
  'vs3Bet': 'facing3Bet',
  'vs4Bet': 'facing4Bet',
  'squeeze': 'facingOpenPlusCaller',
};
```

---

## Task Breakdown

### Phase 0: API Validation (Day 1 Morning)

| Task | Effort | Priority |
|------|--------|----------|
| 0.1 Test V3 session/start via curl with valid config | 1h | P0 |
| 0.2 Check Django logs for errors | 30m | P0 |
| 0.3 Test V3 session/hands save endpoint | 30m | P0 |
| 0.4 **Decision Point:** Track A works? → Phase 1. Track A broken? → Track B | - | P0 |

**Estimated:** 2 hours

### Phase 1: Landing Page Fixes (Day 1-2)

| Task | File(s) | Effort | Priority |
|------|---------|--------|----------|
| 1.1 Hide standard postflop modes (Flop/Turn/River) from street selector | `TrainerLanding.tsx` | 2h | P0 |
| 1.2 Keep Custom mode enabled — verify it works end-to-end | `SpotSelector.tsx`, `CustomSpotBuilder.tsx` | 3h | P0 |
| 1.3 Fix crash when Spot Discovery returns empty | `TrainerLanding.tsx`, `useTrainerLandingConfig.ts` | 2h | P0 |
| 1.4 Validate player count + spot combinations | `SpotSelector.tsx` | 1h | P1 |
| 1.5 Remove/hide Advanced Settings that don't apply to preflop/custom | `AdvancedSettingsModal.tsx` | 2h | P2 |
| 1.6 Verify CustomSpotBuilder action sequence → session config mapping | `sessionConfigBuilder.ts` | 2h | P0 |

**Estimated:** 12 hours

### Phase 2: Training View Simplification (Day 2-3)

| Task | File(s) | Effort | Priority |
|------|---------|--------|----------|
| 2.1 Verify/fix session start flow (Track A) OR implement Track B | See Track sections above | 4-8h | P0 |
| 2.2 Hide villain range panel (set `villainRange: null`) | `RightStrategyPanel.tsx`, `useStrategyData.ts` | 2h | P0 |
| 2.3 Hide Villain Matrix button/section | `TrainingViewV2.tsx` | 1h | P0 |
| 2.4 Keep hero range working (uses `/list-combos`) | Verify existing flow | 1h | P1 |
| 2.5 Fix ErrorBoundary reset flow | `TrainerErrorBoundary.tsx` | 1h | P1 |

**Estimated:** 9-13 hours (depends on Track)

### Phase 3: Hand Evaluation & Feedback (Day 3-4)

| Task | File(s) | Effort | Priority |
|------|---------|--------|----------|
| 3.1 Verify GTO feedback shows correctly | `FeedbackPanel.tsx` | 2h | P0 |
| 3.2 Fix action button rendering for preflop | `EnhancedActionBar.tsx` | 2h | P1 |
| 3.3 Verify stats update on hand completion | `useSessionStats.ts`, `SessionStatsPanel.tsx` | 2h | P1 |
| 3.4 Test "Next Hand" flow | Manual testing | 1h | P0 |

**Estimated:** 7 hours

### Phase 4: Testing & Polish (Day 4-5)

| Task | Effort | Priority |
|------|--------|----------|
| 4.1 Manual QA: All 5 preflop spots × 2-6 players | 3h | P0 |
| 4.2 Manual QA: Custom mode — build various action sequences | 3h | P0 |
| 4.3 Fix any crashes found in QA | Variable | P0 |
| 4.4 Verify hero range loads for each spot (preflop + custom) | 2h | P1 |
| 4.5 Test custom mode with board cards (postflop in custom) | 2h | P1 |
| 4.6 Verify session persists across page refresh | 1h | P0 |
| 4.7 Code review & cleanup | 2h | P1 |

**Estimated:** 13+ hours

---

## Detailed Implementation Notes

### 1. Hide Standard Postflop Modes (Keep Custom)

In `TrainerLanding.tsx`, filter the street options to allow preflop and custom:

```typescript
// Before
const TRAINING_MODES = ['preflop', 'flop', 'turn', 'river', 'custom'];

// After (Phase 1) — Keep preflop + custom
const TRAINING_MODES_PHASE1 = ['preflop', 'custom'];
```

### 2. Custom Mode Action Sequence Flow

The `CustomSpotBuilder` component fetches real actions from the strategy tree API:

```
User opens Custom mode
  → CustomSpotBuilder fetches /get-player-next-actions (root)
  → Shows position cards with available actions (fold, call, raise sizes)
  
User clicks action for position
  → Redux updates selectedActions[street][index]
  → Fetches next available actions from API
  → Sequence builds horizontally
  
User completes preflop (or any street)
  → If postflop: prompt for board cards
  → Continue building turn/river actions
  
User clicks "Start Training"
  → sessionConfigBuilder reads from playerActionsReducer
  → Builds customConfig with preflopActions, flopActions, etc.
  → Session starts at the exact decision point user built to
```

### 3. Hide Villain Range

In `RightStrategyPanel.tsx` or `useStrategyData.ts`:

```typescript
// Phase 1: Always return null for villain data
villainComboData: null,
villainComboLoading: false,
villainRange: null,
```

### 4. Track B: V2 Hand Generation Hook

If Track B is needed, create `useTrainerHandV2.ts`:

```typescript
import { GET_TRAINER_HAND_V2 } from '../../../store/apiEndpoints';
import { apiServiceInstance } from '../../../store/apiUtils';

export async function fetchTrainerHandV2(params: {
  numPlayers: number;
  spot: string;
  heroPosition: string;
  villainPosition?: string;
  BB: string;
  stackInBB: number;
}) {
  const searchParams = new URLSearchParams({
    numPlayers: String(params.numPlayers),
    spot: params.spot,
    heroPosition: params.heroPosition,
    BB: params.BB,
    stackInBB: String(params.stackInBB),
  });
  if (params.villainPosition) {
    searchParams.set('villainPosition', params.villainPosition);
  }
  
  const response = await apiServiceInstance.get(
    `${GET_TRAINER_HAND_V2}?${searchParams.toString()}`
  );
  return response.data;
}
```

---

## Risk Assessment

| Risk | Mitigation |
|------|------------|
| Track A V3 API has fundamental issues | Fall back to Track B (V2 + session storage) |
| Track B takes longer than estimated | Cut P2 tasks; limit custom mode to preflop-only |
| V2 API response format incompatible | Create adapter layer to map V2 → V3 types |
| Custom mode postflop crashes | Limit custom mode to preflop sequences only for Phase 1 |
| CustomSpotBuilder action sequence not mapping to session | Verify `sessionConfigBuilder.ts` buildCustomConfig flow |
| Spot Discovery failures | Add fallback to static spot list when API fails |
| Strategy tree API errors in custom mode | Add error boundary + retry UI in CustomSpotBuilder |

---

## Definition of Done

### Preflop Mode
- [ ] User can select preflop spot (RFI, Facing Open, vs3Bet, vs4Bet, Squeeze)
- [ ] User can select player count (2-6) and hero position
- [ ] Click "Start Training" creates a session and generates hands
- [ ] Session is persisted to database (survives page refresh)
- [ ] Action buttons display correct GTO options
- [ ] Selecting an action shows feedback (correct/incorrect + EV loss)
- [ ] Hand result is saved to database
- [ ] "Next Hand" loads a new hand
- [ ] Hero range panel shows combo data

### Custom Mode
- [ ] User can select Custom from training mode dropdown
- [ ] CustomSpotBuilder loads and fetches initial actions from strategy tree
- [ ] User can build action sequences position-by-position
- [ ] Preflop actions work correctly (fold, call, raise with sizes)
- [ ] Board card selection works (for postflop in custom mode)
- [ ] Action sequence is correctly passed to session config
- [ ] Training starts at the exact decision point user built to
- [ ] Hero range loads for the custom spot

### General
- [ ] No crashes in the happy path (both preflop and custom modes)
- [ ] Villain range/matrix is hidden (no errors, just not shown)
- [ ] Standard postflop modes (Flop/Turn/River dropdown) are hidden from UI

---

## Timeline

| Day | Focus | Deliverable |
|-----|-------|-------------|
| **Thu Feb 20** | Phase 0 + Track decision | API validated, Track A or B chosen |
| **Thu Feb 20** | Phase 1 | Landing page working, postflop hidden |
| **Fri Feb 21** | Phase 2 | Training view with session persistence |
| **Sat Feb 22** | Phase 3 | Full training loop working |
| **Sun Feb 23** | Buffer | Catch-up / overflow |
| **Mon Feb 24** | Phase 4 | QA & fixes |
| **Tue Feb 25** | Polish | Final fixes, code review |
| **Wed Feb 26** | **Release** | Deploy to production |

---

## Slack Message Template

```
📢 Trainer V3 Phase 1 Release Plan — Target: Wednesday Feb 26

**Scope:**
• Preflop training: RFI, Facing Open, vs3Bet, vs4Bet, Squeeze
• Custom mode: Build action sequences like Strategy screen to train any spot
• 2-6 player configurations  
• Hero range display
• GTO feedback on actions
• Session persistence (hands saved to database)

**Deferred (hidden in UI):**
• Standard postflop modes (Flop/Turn/River dropdown)
• Villain range/matrix
• Street/Full Hand game modes
• Statistics drill-down

**Technical Approach:**
Two-track strategy:
• Track A (preferred): Use existing V3 session API — already implemented, needs debugging
• Track B (fallback): Hybrid V2 hand generation + V3 session storage

**Timeline:**
• Thu: Validate APIs, choose track
• Thu-Fri: Landing page + training view fixes
• Sat-Mon: Integration + QA
• Tue: Final polish
• Wed: Release

**Risks:** 
• Custom mode postflop may need additional fixes
• Track B adds 1-2 days if Track A fails

cc @Product @QA
```

---

## Phase 2+ Roadmap (Post Phase 1)

1. **Phase 2:** Enable standard postflop modes (Flop → Turn → River dropdown)
2. **Phase 3:** Implement villain range display
3. **Phase 4:** Add Street/Full Hand game modes
4. **Phase 5:** Statistics drill-down & history features
5. **Phase 6:** Adaptive difficulty

---

*Document generated: Feb 18, 2026*
