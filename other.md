# Trainer V3:Landing Page Revamp


---

## Executive Summary

The Trainer V3 frontend currently generates hands and retrieves strategy data through a newly built pipeline that **reconstructs** critical parameters client-side — parameters that the proven V1 and V2 pipelines already return directly from the backend. This reconstruction layer (`configValuesBridge`) is the single largest source of bugs, mismatched strategy data, and silent failures in the trainer today.

The proposed revamp does **not** discard V3's product vision. It preserves the session model, the new UI, and the training configuration experience. What it replaces is the fragile plumbing underneath: hand generation and strategy parameter resolution are routed through battle-tested V1/V2 backend endpoints that already return everything the frontend needs, eliminating the reconstruction step entirely.

The result is a trainer that is **shippable in Phase 1** with two reliable training modes (Preflop and Custom Spot), a clear upgrade path for postflop and advanced features in later phases, and significantly fewer integration bugs to chase.

---

## 1. The Core Problem: Parameter Reconstruction

### How V1/V2 Work (Proven)

In V1 and V2, when the backend generates a hand it also returns a `configValues` object — a complete set of parameters (site, BB, number of players, action sequences, board cards, etc.) that the frontend passes directly to the strategy endpoints (`/list-combos`, `/hand-categorization`). This is a single source of truth: the backend resolves all parameters, and the frontend simply forwards them.

### How V3 Works Today (Fragile)

V3 introduced a new hand generation endpoint (`POST /v3/session/start`) that returns hands **without** `configValues`. The frontend must then **reconstruct** those parameters by combining data from three separate sources:

1. The session-level configuration (what the user selected)
2. Per-hand data from the backend (hero position, board cards, player actions)
3. Application-level context (game variant, bet sizes)

This reconstruction happens in a module called `configValuesBridge`. It involves string-matching on action names, deriving action sequences from player data, and reconciling session-level settings with hand-level overrides.

### Why This Matters

Every mismatch in the reconstructed parameters means:

- **Wrong strategy data** — the hero range display shows incorrect frequencies or fails to load entirely
- **Silent failures** — the UI appears to work but the underlying data does not match what the solver computed
- **Difficult debugging** — errors surface far from their origin, making each bug investigation time-consuming

The reconstruction layer is not a small utility — it is a complex translation step that duplicates logic the backend already handles correctly in V1/V2.

---

## 2. What the Revamp Changes (and What It Preserves)

### Preserved (No User-Facing Regression)

| Feature | Status |
|---------|--------|
| Trainer Landing page configuration UI | **Kept** — same user experience |
| Custom Spot Builder (action sequence selection) | **Kept** — already uses proven strategy APIs |
| Session persistence (start, save hands, end) | **Kept** — V3 session model stays |
| Session history and listing | **Kept** |
| New strategy data clients (hero range, hand categorization) | **Kept** — well-written, reused as-is |
| User preferences | **Kept** |

### Changed (Internal Plumbing Only)

| Current V3 Approach | Revamped Approach |
|---------------------|-------------------|
| `POST /v3/session/start` generates hands AND creates session | Session creation only (`skipGeneration` flag); hand generation via V1/V2 endpoints |
| Frontend reconstructs strategy parameters from fragments | Frontend receives complete `configValues` inline from V1/V2 — no reconstruction needed |
| `configValuesBridge` performs complex parameter derivation | `configValuesBridge` simplified to a pass-through |
| Buffer refill calls new `/v3/generate` endpoint | Buffer refill calls proven V1/V2 endpoints |

### Removed from Phase 1 Scope (Deferred, Not Deleted)

| Feature | Reason for Deferral |
|---------|--------------------|
| Standard postflop dropdown modes (Flop/Turn/River) | Custom Mode already covers every postflop scenario with better accuracy (see Section 4) |
| Multi-decision game modes (Street, Full Hand) | Requires the V3 `submit-action` pipeline, which is not needed for Phase 1's Spot mode |
| Statistics drill-down and pattern analysis | Valuable feature, lower priority than a reliable core training loop |
| Adaptive difficulty (accuracy profile) | Depends on sufficient session history data; premature before core loop stabilizes |
| Villain range panel | Requires additional parameter resolution; planned for Phase 3 |

---

## 3. Identified Issues in the Current Pipeline

These are concrete problems observed during integration testing — not theoretical concerns.

### Frontend Issues

| Issue | User Impact |
|-------|-------------|
| Strategy parameter mismatch from reconstruction | Hero range shows wrong data or fails to load |
| Custom mode configuration not fully passed to backend | Hands generated for the wrong spot |
| Shared state conflicts with Strategy page | Navigating between Strategy and Trainer can corrupt action state |
| Silent buffer refill failures | Training stops unexpectedly after the initial batch of hands |
| Position validation gaps ("Any" position handling) | Backend may generate hands for invalid hero/spot combinations |

### Backend Issues

| Issue | User Impact |
|-------|-------------|
| Heuristic action sequence building for postflop | Wrong raise sizes for 3-bet/4-bet pots (falls back to placeholder values) |
| Decimal BB values cause server errors | Certain configurations crash the backend (`int("5.00")` failure) |
| Missing `resolvedParams` in hand responses | Frontend has no ground truth, reconstruction guesses incorrectly |

### Why V1/V2 Endpoints Avoid These

V1 and V2 endpoints were developed over a longer period, have been in production on `staging`, and critically, they return `configValues` as a single resolved object. The frontend never needs to guess or reconstruct — it receives exactly what the strategy endpoints expect.

---

## 4. Why Custom Mode Replaces Standard Postflop Dropdowns (Phase 1)

This is not a feature cut — it is a **better user experience with higher correctness**.

When a user selects "Flop → SRP" from a dropdown, the backend must **guess** the exact preflop action sequence that led to that pot type. This guesswork has known failure modes, particularly for 3-bet and 4-bet pots where raise sizes vary across solver configurations.

Custom Mode eliminates this entirely:

| User Goal | Dropdown Approach | Custom Mode Approach |
|-----------|-------------------|---------------------|
| Train SRP on Flop | Select Flop → SRP → position. Backend guesses the preflop sequence. | Build exact sequence: CO opens → BB calls → select flop. User sees what happened preflop. |
| Train 3BP on Turn | Select Turn → 3BP → position. Backend guesses raise sizes. | Build sequence with real sizes from the strategy tree. No guessing. |
| Train 4BP on River | Select River → 4BP. Backend must discover both 3-bet and 4-bet sizes. | Full sequence built from the actual strategy tree data. |

Custom Mode uses the **same strategy tree API** (`/get-player-next-actions`) that powers the Strategy page — the most thoroughly tested endpoint in the platform. Every action and raise size shown to the user comes directly from the solver's computed strategy, not from heuristic approximation.

**Phase 2+ plan:** If standard postflop shortcuts return, they will auto-populate the Custom Spot Builder with tree-verified sequences — giving users the convenience of a dropdown with the correctness of tree-based selection.

---

## 5. Effort and Timeline

| Task | Priority | Estimate |
|------|----------|----------|
| Hide postflop dropdown modes (Preflop + Custom only) | P0 | 2h |
| Wire Preflop mode to V1 hand generation endpoint | P0 | 4h |
| Wire Custom mode to V2 hand generation endpoint | P0 | 4h |
| Simplify config bridge to use inline `configValues` | P0 | 3h |
| Backend: add `skipGeneration` flag to session start | P0 | 2h |
| Session persistence wrapper (start → save → end) | P0 | 4h |
| Verify Custom Spot Builder → V2 parameter mapping | P0 | 3h |
| End-to-end testing across both modes | P0 | 4h |
| Minor cleanup (error boundary reset, villain range hide) | P1 | 2h |
| **Total** | | **~28 hours** |

---

## 6. Risk Assessment

| Risk | Mitigation |
|------|------------|
| V1/V2 endpoints do not support all V3 config options (board filters, adaptive difficulty) | These features are explicitly deferred to later phases. The core training loop ships first. |
| Session persistence adds latency | Session creation is fire-and-forget (async). Hand result saves are batched. Measured overhead is negligible. |
| Users expect postflop dropdown modes | Custom Mode covers all postflop scenarios with better accuracy. Dropdowns return in Phase 2 as shortcuts. |
| Shared Redux state between Strategy page and Trainer | Custom Spot Builder already resets state on mount. Verified during integration testing. |

---

## 7. What Happens If We Do Not Revamp

If the current V3 pipeline ships as-is, the team should expect:

1. **Ongoing strategy data mismatches** — the config bridge reconstruction will continue to produce incorrect parameters for edge cases (3-bet/4-bet pots, non-standard blind structures, specific position combinations)
2. **Higher bug investigation cost** — each mismatch surfaces as a UI-level symptom (wrong range, missing data) but originates in the reconstruction layer, requiring end-to-end pipeline debugging
3. **Blocked postflop modes** — the standard postflop dropdown modes depend on backend heuristics that have known failure modes for raise size discovery, meaning they would ship with known inaccuracies
4. **Difficult Phase 2+ upgrades** — building advanced features (multi-decision modes, adaptive difficulty) on top of the reconstruction layer compounds the fragility

The revamp eliminates the reconstruction layer as a failure point, provides a stable foundation for Phase 2+ features, and ships a reliable product in Phase 1.

---

## 8. Summary

The Trainer V3 revamp is a **reliability intervention**, not a rewrite. The user-facing product stays the same. The internal plumbing switches from a fragile parameter reconstruction approach to proven backend pipelines that already return exactly what the frontend needs. The estimated effort is ~28 hours, the risk profile is low (we are adopting battle-tested code, not writing new code), and the outcome is a trainer that ships with confidence.
