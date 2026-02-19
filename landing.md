# Trainer V3 Landing Page Revamp Plan

**Prepared by:** Engineering Team  
**Branch:** `feature/gl-proto`

---

## Executive Summary

This plan revamps the **Trainer V3 Landing page ** by narrowing the landing experience to **Preflop + Custom**, improving Custom flow through logical Strategy-page reuse, and hardening landing-state reliability so users can consistently configure and start training without crashes or mismatched payloads.

### Scope Summary

| In Scope (Landing Only) | Out of Scope |
|----------|--------------------------|
| Restrict landing street/mode selector to Preflop + Custom | Training View feature changes |
| Strategy-style UX reuse for Custom builder interactions | Villain range/matrix visibility work |
| Spot Discovery empty-state safety on landing | Postflop mode rollout beyond landing |
| Reliable custom sequence → session payload mapping from landing | Stats/history enhancements |
| Focused unit tests for landing/custom config behavior | Non-landing refactors |

### Custom Reuse Strategy Description

The **Custom** section on landing should logically reuse Strategy-page interaction patterns while preserving UnifiedTrainer contracts:

1. **Sequence-first UI** — Keep position-by-position action building flow familiar to Strategy users
2. **Progressive action updates** — Continue using real next-action API results for each decision point
3. **Consistent reset/recovery behavior** — Align reset/loading/retry states with Strategy conventions
4. **Safe handoff to trainer session** — Ensure selected action sequence maps correctly into `customConfig`

---

## Technical Approach

### Landing-Only Revamp Approach (Selected)

**Effort:** 8-14 hours

This approach keeps existing architecture and updates only landing-owned surfaces and landing-to-session mapping.

**Core changes:**
- Restrict mode selector and persisted-mode normalization to `preflop` + `custom`
- Refactor `CustomSpotBuilder` interaction/presentation with logical Strategy-style reuse
- Harden Spot Discovery empty-state behavior and Start CTA gating
- Validate and enforce robust `customConfig` generation from landing state
- Add targeted tests for landing and custom mapping logic

---

## Task Breakdown

### Track A: Landing Mode Constraints

| Task | File(s) | Effort | Priority |
|------|---------|--------|----------|
| A.1 Restrict visible mode options to Preflop + Custom | `components/UnifiedTrainer/Configuration/StreetSelector.tsx` | 1-2h | P0 |
| A.2 Normalize persisted/URL postflop mode values to Preflop | `components/UnifiedTrainer/hooks/useTrainerLandingConfig.ts` | 1h | P0 |
| A.3 Verify Start CTA behavior after mode narrowing | `components/UnifiedTrainer/TrainerLanding.tsx` | 0.5h | P1 |

### Track B: Custom Builder Strategy Reuse

| Task | File(s) | Effort | Priority |
|------|---------|--------|----------|
| B.1 Align Custom builder progression with Strategy interaction conventions | `components/UnifiedTrainer/Configuration/CustomSpot/CustomSpotBuilder.tsx` | 2-4h | P0 |
| B.2 Reuse Strategy sequence/action patterns logically without cross-feature coupling | `components/ActionSequence/PlayerSequence.tsx`, `components/UnifiedTrainer/Configuration/CustomSpot/CustomSpotBuilder.tsx` | 2-3h | P0 |
| B.3 Align reset/loading/retry behavior to Strategy-style UX | `components/UnifiedTrainer/Configuration/CustomSpot/CustomSpotBuilder.tsx` | 1-2h | P1 |

### Track C: Landing Robustness + Payload Reliability

| Task | File(s) | Effort | Priority |
|------|---------|--------|----------|
| C.1 Prevent landing crash when Spot Discovery returns empty | `components/UnifiedTrainer/TrainerLanding.tsx` | 1h | P0 |
| C.2 Improve fallback messaging when configuration has no available strategies | `components/UnifiedTrainer/Configuration/SpotSelector.tsx`, `components/UnifiedTrainer/TrainerLanding.tsx` | 1h | P1 |
| C.3 Harden custom sequence → `customConfig` mapping | `components/UnifiedTrainer/utils/sessionConfigBuilder.ts` | 1-2h | P0 |

### Track D: Tests and Verification

| Task | File(s) | Effort | Priority |
|------|---------|--------|----------|
| D.1 Add/update tests for mode restriction + normalization | `__tests__/hooks/useTrainerLandingConfig.test.ts`, `components/UnifiedTrainer/__tests__/TrainerLanding.configValidation.test.ts` | 1-2h | P0 |
| D.2 Add/update tests for custom payload mapping and reset/progression behavior | `components/UnifiedTrainer/utils/__tests__/sessionConfigBuilder.test.ts`, targeted CustomSpotBuilder tests | 1-2h | P0 |
| D.3 Run targeted lint/tests for changed files | Frontend test/lint commands | 0.5-1h | P0 |

---

## Detailed Implementation Notes

### 1. Restrict Landing Modes

Landing should only expose:

```typescript
const TRAINING_MODES = ['preflop', 'custom'];
```

Persisted or URL-initialized `flop/turn/river` values should be normalized to `preflop` on landing state init.

### 2. Custom Mode Reuse Flow

Custom builder keeps current API flow and aligns interaction patterns:

```
User selects Custom mode
  → Fetch initial next-actions from strategy tree API
  → Render sequence UI with Strategy-style progression semantics

User clicks actions position-by-position
  → Update selectedActions in Redux
  → Fetch next available actions
  → Maintain consistent active position and recovery states

User starts training
  → sessionConfigBuilder reads selectedActions + cardSelection
  → Builds safe customConfig (required preflop, optional postflop/board)
  → Session starts at the selected decision point
```

### 3. Empty Discovery Safety

If Spot Discovery returns zero scenarios:
- Landing does not crash
- Start CTA is deterministically gated
- Clear guidance prompts user to adjust configuration
- Custom mode remains available with transparent state messaging

### 4. Custom Payload Reliability

`sessionConfigBuilder` must produce safe `customConfig`:
- Include `preflopActions` only when valid
- Include `flopActions/turnActions/riverActions` only when present
- Include `boardCards` only when format and uniqueness checks pass
- Fail safe for partial/invalid states (no malformed payload)

---

## Risk Assessment

| Risk | Mitigation |
|------|------------|
| Strategy-style reuse introduces hidden coupling | Reuse interaction patterns selectively; keep UnifiedTrainer boundaries intact |
| Mode restriction causes persisted-state edge cases | Add explicit normalization in landing config hook |
| Empty discovery handling regresses CTA behavior | Add deterministic gating tests for unavailable scenarios |
| Custom payload mismatch between UI and session start | Add focused mapping tests for preflop-only, partial postflop, and board-card cases |

---

## Definition of Done

### Landing Page
- [ ] Mode selector shows only Preflop and Custom
- [ ] Persisted/URL postflop modes normalize to Preflop
- [ ] No landing crashes when Spot Discovery is empty
- [ ] CTA and guidance are deterministic for unavailable strategy states

### Custom Section on Landing
- [ ] Custom flow reflects Strategy-style interaction conventions
- [ ] Action progression/reset/retry behavior is stable
- [ ] Custom sequence remains backed by real next-actions API
- [ ] Start Training uses accurate custom sequence state

### Quality Gates
- [ ] `customConfig` mapping is reliable and fail-safe
- [ ] Unit tests added/updated for changed landing logic
- [ ] Targeted lint/tests pass for touched files

---

## Team Share Summary

```
Trainer V3 Landing Revamp Plan

Scope:
• Landing page only
• Mode constrained to Preflop + Custom
• Custom flow aligned with Strategy interaction patterns
• Spot Discovery empty-state hardening on landing
• Reliable custom sequence → customConfig mapping

Execution:
• Constrain mode options + normalize persisted mode state
• Refactor CustomSpotBuilder interactions with logical Strategy reuse
• Harden landing empty states and CTA gating
• Add focused tests for mode behavior and payload mapping
```
