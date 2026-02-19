# Trainer V3 Landing Revamp Plan (Phase 1)

**Target Date:** Wednesday, Feb 26, 2026  
**Prepared by:** Engineering Team  
**Branch:** `feature/gl-proto`

---

## Executive Summary

This plan focuses on revamping the Trainer V3 Landing experience for Phase 1 by constraining the experience to **Preflop + Custom**, hardening landing configuration behavior, and making Custom flow reuse Strategy-page interaction patterns where logical.

### Scope Summary

| In Scope | Out of Scope (Deferred) |
|----------|--------------------------|
| Restrict Landing street/mode to Preflop + Custom | Standard postflop mode exposure on landing (Flop/Turn/River) |
| Custom mode UI/UX aligned with Strategy-page interaction flow | Villain range/matrix work (Phase 2 track) |
| Custom sequence builder stability (reset/progression/empty states) | Training view simplification outside landing scope |
| Spot Discovery empty-state crash hardening on landing | Stats/history deep enhancements |
| Session payload reliability for custom sequences (`customConfig`) | Adaptive difficulty tuning |
| Focused unit test coverage for landing/custom mapping changes | Broad repo-wide refactors |

### Custom Reuse Strategy Description

The **Custom** experience should reuse Strategy-page logic and interaction conventions where possible (without breaking UnifiedTrainer contracts):

1. **Sequence-first interaction model** — Position-by-position action flow that mirrors Strategy-page expectations
2. **Consistent action progression** — Active player and step progression follows familiar Strategy behavior
3. **Reset and recovery parity** — Reset behavior, loading states, and retry messaging align with Strategy interaction standards
4. **Data contract preservation** — Continue using existing Trainer V3 landing/session flow and `get-player-next-actions` pipeline

---

## Technical Approach

### Approach A: Mode-Constrained Landing + Strategy-Style Custom Builder (Selected)

**Effort:** 8-14 hours

This approach keeps existing Trainer V3 architecture intact while tightening the landing scope and improving Custom UX by reusing Strategy interaction patterns.

**Core changes:**
- Restrict mode selector and state normalization to `preflop` + `custom`
- Refactor CustomSpotBuilder interaction/presentation toward Strategy-style sequence UX
- Harden Spot Discovery empty-state behavior and CTA gating
- Validate and enforce robust `customConfig` generation in session builder
- Add targeted tests for behavior and mapping

### Alternative Considered (Not Selected for this phase)

**Approach B: Full UI component extraction from Strategy page**

This was not selected for Phase 1 to avoid high-risk cross-feature coupling and excessive refactor scope.

---

## Task Breakdown

### Phase 1-A: Landing Mode Constraints

| Task | File(s) | Effort | Priority |
|------|---------|--------|----------|
| A.1 Restrict visible mode options to Preflop + Custom | `components/UnifiedTrainer/Configuration/StreetSelector.tsx` | 1-2h | P0 |
| A.2 Normalize persisted/URL postflop mode values to Preflop | `components/UnifiedTrainer/hooks/useTrainerLandingConfig.ts` | 1h | P0 |
| A.3 Verify Start CTA logic remains consistent after mode narrowing | `components/UnifiedTrainer/TrainerLanding.tsx` | 0.5h | P1 |

**Estimated:** 2.5-3.5 hours

### Phase 1-B: Custom Builder Reuse and UX Alignment

| Task | File(s) | Effort | Priority |
|------|---------|--------|----------|
| B.1 Align CustomSpotBuilder progression flow with Strategy sequence conventions | `components/UnifiedTrainer/Configuration/CustomSpot/CustomSpotBuilder.tsx` | 2-4h | P0 |
| B.2 Reuse existing Strategy interaction patterns/components logically (without breaking contracts) | `components/ActionSequence/PlayerSequence.tsx`, `components/UnifiedTrainer/Configuration/CustomSpot/CustomSpotBuilder.tsx` | 2-3h | P0 |
| B.3 Align reset/loading/retry behavior to Strategy-like UX | `components/UnifiedTrainer/Configuration/CustomSpot/CustomSpotBuilder.tsx` | 1-2h | P1 |

**Estimated:** 5-9 hours

### Phase 1-C: Robustness + Mapping Reliability

| Task | File(s) | Effort | Priority |
|------|---------|--------|----------|
| C.1 Prevent landing crash when Spot Discovery returns empty | `components/UnifiedTrainer/TrainerLanding.tsx` | 1h | P0 |
| C.2 Improve explicit fallback messaging and guidance for sparse/no strategy states | `components/UnifiedTrainer/Configuration/SpotSelector.tsx`, `components/UnifiedTrainer/TrainerLanding.tsx` | 1h | P1 |
| C.3 Verify and harden custom action sequence -> session payload mapping | `components/UnifiedTrainer/utils/sessionConfigBuilder.ts` | 1-2h | P0 |

**Estimated:** 3-4 hours

### Phase 1-D: Testing and Verification

| Task | File(s) | Effort | Priority |
|------|---------|--------|----------|
| D.1 Add/update tests for mode restriction and normalization | `__tests__/hooks/useTrainerLandingConfig.test.ts`, `components/UnifiedTrainer/__tests__/TrainerLanding.configValidation.test.ts` | 1-2h | P0 |
| D.2 Add/update tests for custom payload mapping and reset/progression logic | `components/UnifiedTrainer/utils/__tests__/sessionConfigBuilder.test.ts`, targeted CustomSpotBuilder tests | 1-2h | P0 |
| D.3 Run targeted lint/tests for changed files | Frontend test/lint commands | 0.5-1h | P0 |

**Estimated:** 2.5-5 hours

---

## Detailed Implementation Notes

### 1) Restrict Landing Modes

Landing should only expose:

```typescript
const TRAINING_MODES_PHASE1 = ['preflop', 'custom'];
```

Any persisted or URL-initialized `flop/turn/river` mode should normalize back to `preflop` for Phase 1.

### 2) Strategy-Style Custom Reuse

Custom builder keeps current API flow but follows Strategy interaction conventions:

```
User selects Custom mode
  → Fetch initial next-actions from strategy tree API
  → Render sequence cards/chips with Strategy-style progression semantics

User clicks actions position-by-position
  → Update selectedActions in Redux
  → Fetch next available actions
  → Advance active position/street with consistent reset/retry behavior

User starts training
  → sessionConfigBuilder reads selectedActions + cardSelection
  → Builds customConfig safely (preflop required; optional postflop/board)
  → Session starts at the selected decision point
```

### 3) Empty Discovery Safety

If Spot Discovery returns zero scenarios:
- No crash on landing
- Start button disabled when configuration is not trainable
- Clear guidance to modify database/site/stack inputs
- Custom mode remains available with transparent state messaging

### 4) Custom Payload Reliability

`sessionConfigBuilder` must produce a safe `customConfig`:
- Include `preflopActions` only when valid data exists
- Include optional `flopActions/turnActions/riverActions` when present
- Include `boardCards` only when format and uniqueness checks pass
- Fail safe on partial/invalid selections (no malformed payloads)

---

## Risk Assessment

| Risk | Mitigation |
|------|------------|
| Reuse effort introduces coupling to Strategy-specific assumptions | Reuse interaction patterns selectively; keep UnifiedTrainer contracts primary |
| Narrowing modes causes persistence edge cases from old saved configs | Add mode normalization in hook-level state initialization |
| Empty discovery handling regresses CTA behavior | Add deterministic gating tests around unavailable strategies |
| Custom payload drift between UI state and backend expectation | Add explicit mapping tests for preflop-only, partial postflop, and board-card cases |

---

## Definition of Done

### Landing
- [ ] Mode selector shows only Preflop and Custom
- [ ] Persisted/URL postflop modes normalize to Preflop safely
- [ ] No crashes when Spot Discovery is empty
- [ ] CTA and guidance behavior are deterministic in unavailable states

### Custom Mode
- [ ] Custom flow reflects Strategy-style interaction conventions
- [ ] Action progression/reset/retry behavior is stable
- [ ] Custom sequence remains backed by real strategy next-actions API
- [ ] Start Training uses accurate custom sequence state

### Session Mapping + Quality
- [ ] `customConfig` mapping is reliable and fail-safe
- [ ] Unit tests added/updated for all changed logic paths
- [ ] Targeted lint/tests pass for touched files

---

## Timeline

| Day | Focus | Deliverable |
|-----|-------|-------------|
| **Thu Feb 20** | Phase 1-A | Landing mode constraints + normalization |
| **Thu Feb 20** | Phase 1-B (start) | Custom builder reuse refactor in progress |
| **Fri Feb 21** | Phase 1-B + 1-C | Strategy-style custom UX + robustness fixes |
| **Fri Feb 21** | Phase 1-D | Test updates + verification |
| **Sat Feb 22** | Buffer | Bug fixes or polish from QA feedback |

---

## Slack Message Template

```
📢 Trainer V3 Landing Revamp Plan (Phase 1)

**Scope:**
• Landing constrained to Preflop + Custom
• Custom flow aligned with Strategy-style interaction patterns
• Spot Discovery empty-state hardening (no-crash + clear guidance)
• Reliable custom sequence → session payload mapping
• Focused unit-test coverage for changed flows

**Out of Scope (this pass):**
• Villain range/matrix updates
• Standard postflop landing modes
• Broader training-view refactors

**Execution Plan:**
• Constrain modes + normalize persisted state
• Refactor CustomSpotBuilder for Strategy-like UX behavior
• Harden empty discovery and CTA/fallback states
• Validate customConfig mapping and add tests

cc @Product @QA
```

---

## Next Phase Follow-ups

1. Reintroduce gated postflop landing modes when Phase 2 scope is ready
2. Hide/adjust villain range and matrix in training view per Phase 2 plan
3. Expand end-to-end coverage for custom postflop sequences

---

*Document generated: Feb 19, 2026*
