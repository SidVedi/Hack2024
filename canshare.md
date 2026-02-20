# Trainer V3 Landing Page

### 1. Reducing configuration complexity for a more reliable experience

The V3 landing page currently supports 5 street modes (Preflop / Flop / Turn / River / Custom), each with its own spot selection logic, config validation, and edge cases. The number of possible combinations across street, spot, and game config has grown beyond what we can confidently test and ship. Consolidating to two well-defined modes lets us deliver a reliable experience faster.

### 2. Custom Mode delivers a more accurate postflop training experience

When a user selects "Flop → SRP" from a dropdown, the backend must **infer** the exact preflop action sequence that led to that pot type. This inference has known limitations, particularly for 3-bet and 4-bet pots where raise sizes vary across solver configurations:

| User Goal | Dropdown Approach | Custom Mode Approach |
|-----------|-------------------|----------------------|
| Train SRP on Flop | Select Flop → SRP → position. Backend infers the preflop sequence. | Build exact sequence: CO opens → BB calls → select flop cards. User sees what happened preflop. |
| Train 3BP on Turn | Select Turn → 3BP → position. Backend infers raise sizes. | Build sequence with real sizes from the strategy tree. No inference needed. |
| Train 4BP on River | Select River → 4BP. Backend must discover both 3-bet and 4-bet sizes. | Full sequence built from actual strategy tree data. |

Custom Mode uses the same strategy tree API (`/get-player-next-actions`) that powers the Strategy page — the most thoroughly tested endpoint in the platform. Every action and raise size presented to the user comes directly from the solver's computed strategy, not from approximation.

**Users who want to train postflop spots can still do so — with higher accuracy and full visibility into the action sequence that created the pot.**

### 3. Building on proven foundations

Trainer V1 (preflop config + spot selection) and Trainer V2 (strategy-driven custom scenarios) have been running in production. Combining them into the V3 landing gives us a page built on proven foundations with well-understood behavior.

### 4. Shared endpoints reduce integration risk

By reusing the Strategy page's endpoints (`/games-new-config`, `/get-player-next-actions`) and config cascade logic, we ensure the trainer and the strategy browser always show consistent configuration options and produce matching results for the same setup.

---

## Approach: V1 + V2 Hybrid

Focus the landing page on two training modes:

- **Preflop** — V1-style config-driven flow. User selects game settings, preflop spot (RFI, Facing Open, Facing 3-Bet, etc.), and hero/villain positions.
- **Custom** — V2-style strategy-driven flow. User builds the exact action sequence, selects board cards, and trains on any street at any spot — with full control over the scenario.

---

## Architecture Overview

```mermaid
flowchart TB
  subgraph current ["Current V3 Landing"]
    SS["StreetSelector: 5 modes"]
    SD["SpotDiscovery API"]
    TGC["Trainer-specific config endpoint"]
    PF["Standalone postflop street modes with pot-type inference"]
  end

  subgraph reworked ["Reworked V3 Landing"]
    SS2["StreetSelector: 2 modes (Preflop | Custom)"]
    subgraph preflop ["Preflop Mode"]
      GC["Game Config Selectors (shared with Strategies)"]
      PS["Preflop Spot Selector (RFI, vsOpen, vs3Bet, vs4Bet)"]
      POS["Hero / Villain Position Selector"]
    end
    subgraph custom ["Custom Mode"]
      CSB["V3 Custom Spot Builder UI (horizontal action sequence)"]
      BCM["Board Card Selector"]
      GC2["Game Config (same selectors)"]
      DL["Deep-link from Strategy page Train button"]
    end
  end

  subgraph endpoints ["Shared Endpoints (same as Strategy page)"]
    GNC["GET /games-new-config"]
    GPNA["GET /get-player-next-actions"]
    NGC["GET /new-game-configuration"]
  end

  preflop --> GNC
  custom --> GNC
  custom --> GPNA
  custom --> NGC
```

---

## Detailed Changes

### 1. Focus the Street Selector

Consolidate to two modes by removing Flop, Turn, and River as standalone options:

- **Preflop** — Preflop-only training
- **Custom** — Build any scenario (including postflop) from the strategy tree

Postflop training is fully supported through Custom Mode, where users build the exact scenario they want. This removes the need for standalone postflop modes and their associated pot-type inference logic.

### 2. Preflop Mode (V1-style)

When the user selects "Preflop", show a clean configuration form:

- **Game config dropdowns** (Site, BB, Blind Structure, Players, Stack, Open Raise) — shared with the Strategy page
- **Preflop spot selector** (RFI, Facing Open, Facing 3-Bet, Facing 4-Bet, Squeeze)
- **Hero position** selection on the poker table preview
- **Difficulty** selector
- **Config endpoint**: `GET /games-new-config` (same as Strategy page), replacing the trainer-specific `GET /trainer/trainer-games-config`

### 3. Custom Mode (V3 UI + Strategy Page APIs)

When the user selects "Custom", the V3 Custom Spot Builder UI is shown — the horizontal, GTO Wizard-style action sequence with position cards, board card selectors, and progressive disclosure of actions per street.

**What stays (V3 UI/UX):**

- The horizontal action sequence layout (PositionCard + BoardCard components)
- Progressive disclosure: preflop actions appear first, postflop streets reveal as the sequence progresses
- Board card selection inline with the action sequence
- Hand-over detection and winner display
- Database-aware constraints (preflop-only, full tree, precision)
- Reset button and loading/error states

**What is stabilized (API alignment with Strategy page):**

The Custom Spot Builder already calls `/get-player-next-actions` — the same endpoint the Strategy page uses. The stabilization work ensures this integration is fully robust:

- **Config endpoint**: Switch from `GET /trainer/trainer-games-config` to `GET /games-new-config` (same as Strategy page) so the game config dropdowns draw from the same data source
- **Board card modal**: Adopt the Strategy page's `SelectBoardCardsModal` so board card selection behavior is identical across surfaces
- **Config cascade**: Align with the Strategy page's `useGameConfig` cascade logic to ensure available options (sites, BBs, stacks, etc.) are always consistent
- **Query parameter construction**: Ensure the `CustomSpotBuilder` constructs its `/get-player-next-actions` queries identically to the Strategy page — same parameter names, same defaults, same research type mapping
- **Edge case resolution**: Address known issues in the Custom Spot Builder (auto-correction loops, state synchronization between advanced settings and the action builder)

**Two ways to enter Custom mode:**

1. **On the trainer landing page** — User configures game settings, builds the action sequence using the horizontal builder, selects board cards, and starts training. Self-contained experience.
2. **From the Strategy page** — User clicks "Train" on the Strategy page. The current config (actions, board cards, game settings) transfers to the trainer via a deep link, pre-populating the Custom Spot Builder. No re-configuration needed.

### 4. Hand Generation: V1/V2 APIs + V3 Session Persistence

To further align with proven infrastructure, hand generation during gameplay will use the same V1 and V2 endpoints that have been running in production — while V3 continues to handle session management, statistics, history, and replayer integration.

**How it works:**

1. **Session creation** — `POST /trainer/v3/session/start` creates the session row (with a new `skipGeneration: true` flag). No hands are generated server-side.
2. **Hand dealing** — The frontend calls V1 or V2 hand generation endpoints directly:
   - **Preflop mode** → `GET /trainer/generate-next-hand` (V1) — simple, fast, proven
   - **Custom mode** → `GET /trainer/v2/generate-next-hand` (V2) — supports full action sequences, board cards, and site-specific configs
3. **Hand saving** — After the user plays each hand, results are saved via the existing `POST /trainer/v3/session/{id}/hands` endpoint, including a `replayContext` field for replayer integration.
4. **Session end** — The existing `POST /trainer/v3/session/{id}/end` closes the session and returns a summary.

**Why this is better than V3 server-side generation:**

- V1 and V2 hand generation endpoints are the same ones that power Trainer V1 and Trainer V2 in production — battle-tested with well-understood behavior.
- V3's server-side generation pipeline involves a multi-step chain (native path → strategy platform → board sampling → thread pool → transform). Bypassing it removes an entire category of potential issues without losing any functionality.
- Session management, statistics, hand history, and replayer all read from `TrainerHandResult` rows — they do not depend on how the hand was generated.

**Backend changes required (minimal):**

| Change | Description | Risk |
|--------|-------------|------|
| `skipGeneration` flag on session start serializer | Allows session creation without server-side hand generation | None — additive, defaults to `false` |
| Skip generation path in `StartSessionView` | Early return with empty hands array when flag is true | None — existing path untouched |
| `replayContext` fallback in `SaveHandResultsView` | Accept replay data from request body (V2 hands are not cached server-side) | None — falls back to existing cache first |
| `replayContext` field on `HandResultSerializer` | Validate the new optional field | None — optional field |

Zero model changes. Zero migrations. Fully backward-compatible — removing the `skipGeneration` flag reverts to the existing V3 generation pipeline.

---

## What Users Gain

| Current V3 | Reworked V3 |
|------------|-------------|
| 5 street modes with different config flows | 2 focused modes: Preflop and Custom |
| Postflop training relies on backend inference of action sequences | Users build exact sequences from real solver data |
| Trainer uses its own config endpoint | Shared config endpoint — same data as Strategy page |
| Board card selection with a separate implementation | Board card popup consistent with the Strategy page |
| Trainer-specific config cascade logic | Config cascade aligned with Strategy page — single source of truth |
| Custom builder queries the same API with different parameter construction | Query construction aligned with Strategy page for consistent results |

### Downstream impact: accurate pot sizes and bet sizes on the gameplay screen

This approach also improves the accuracy of pot sizes and bet sizes displayed during actual training gameplay.

When postflop training is configured via dropdowns (e.g., "Flop → 3-Bet Pot"), the backend must reconstruct the preflop action sequence to calculate the pot entering the flop. For 3-bet and 4-bet pots, the backend does not know the exact raise sizes from the solver — it attempts tree-based discovery first, but falls back to placeholder raise sizes when that fails. If the raise sizes are approximate, the pot size entering the flop is incorrect, and every bet size shown on the table (expressed as a fraction of pot) is calculated against that incorrect pot. This cascades through every subsequent street.

With Custom Mode, the user builds the exact action sequence from the strategy tree — including real raise sizes. That sequence is sent directly to the backend via `customConfig.preflopActions`, bypassing the inference and fallback logic entirely. The pot size is correct by construction, and all bet sizes downstream are accurate.

For Preflop mode, this is not a concern — the hand ends before postflop, so pot and bet size inference does not apply.

---

## What Stays Unchanged

- Poker table preview on the landing page
- Config summary bar (Database | Stake | Site | Stack)
- Custom Spot Builder visual design (horizontal action sequence, position cards, board cards)
- User preferences persistence
- Shareable URL support
- Difficulty selector
- The entire gameplay loop UI (everything after "Start Training")
- V3 session lifecycle endpoints (start, save hands, end, statistics, history)
- V1/V2 hand generation endpoints (no changes)
- `TrainerSession` and `TrainerHandResult` models (no migrations)

---

## Endpoint Summary

### Configuration (shared with Strategy page)

| Endpoint | Used By | Purpose |
|----------|---------|---------|
| `GET /games-new-config` | Strategy page, Trainer (both modes) | Fetch available game configurations (sites, BBs, stacks, etc.) |
| `GET /get-player-next-actions` | Strategy page, Trainer (Custom mode) | Fetch available actions at each decision point in the strategy tree |
| `GET /new-game-configuration` | Strategy page, Trainer (Custom mode) | Validate config and fetch strategy data for a specific game setup |

By routing both the Strategy page and the Trainer through the same configuration endpoints, any improvement to these APIs benefits both surfaces automatically.

### Hand generation (proven V1/V2 endpoints)

| Endpoint | Used By | Purpose |
|----------|---------|---------|
| `GET /trainer/generate-next-hand` | Trainer (Preflop mode) | Generate a single preflop training hand — same as production Trainer V1 |
| `GET /trainer/v2/generate-next-hand` | Trainer (Custom mode) | Generate a hand with full action sequence + board cards — same as production Trainer V2 |

### Session lifecycle (V3 — minimal changes)

| Endpoint | Change | Purpose |
|----------|--------|---------|
| `POST /trainer/v3/session/start` | Add `skipGeneration` flag | Create session row; hand generation handled by V1/V2 endpoints above |
| `POST /trainer/v3/session/{id}/hands` | Accept `replayContext` from body | Save hand results + replay data for replayer integration |
| `POST /trainer/v3/session/{id}/end` | No change | End session, return summary |
| `GET /trainer/v3/statistics` | No change | Aggregated training statistics |
| `GET /trainer/v3/sessions` | No change | Session history |

---

## Deep Link: Strategy Page → Trainer

The Strategy page's "Train" button currently passes preflop spot and game config to the trainer via URL parameters. This will be enhanced to also carry:

- Full action sequence (all streets)
- Board cards
- Active street
- Research / simulation type

This allows users to seamlessly move from browsing a strategy to training on that exact spot — Custom mode pre-populates with the full configuration and the user can start training immediately.

---

## Risk Assessment

| Area | Risk | Mitigation |
|------|------|-----------|
| Street selector consolidation | Low | Straightforward removal of standalone postflop options; postflop remains accessible via Custom mode |
| Preflop spot selector | Low | Scoping to preflop spots only, no new logic introduced |
| Switching config endpoint | Medium | Response shapes are similar but need field compatibility verification |
| Stabilizing Custom Spot Builder API calls | Medium | The builder already calls the correct endpoint — work is aligning query parameters and resolving edge cases, not a rewrite |
| Aligning board card modal | Low | Adopting an existing, well-tested component |
| Deep link enhancement | Low | Additive change to an existing working mechanism |
| Hybrid hand generation (V1/V2 + V3 sessions) | Low | V1/V2 endpoints are proven in production; backend changes are additive with zero migrations; `skipGeneration` defaults to false for full backward compatibility |
| V2 → V3 hand format transform on frontend | Medium | Mirrors the existing backend transform; needs thorough testing to ensure hand actions, pot sizes, and replay data map correctly |

---

## Implementation Steps

### Landing page (frontend)

1. Consolidate StreetSelector to 2 modes (Preflop | Custom), remove standalone Flop/Turn/River
2. Wire Preflop mode with Strategy page config selectors and preflop spot picker
3. Switch from `trainer-games-config` to `games-new-config` endpoint for both modes
4. Align Custom Spot Builder's `/get-player-next-actions` query construction with Strategy page
5. Adopt Strategy page's `SelectBoardCardsModal` for the Custom Spot Builder
6. Align config cascade with Strategy page's `useGameConfig` logic
7. Resolve known Custom Spot Builder edge cases (auto-correction loops, config drift between advanced settings and builder)
8. Enhance deep link to carry full action sequence and board cards for Custom mode pre-population
9. Remove standalone postflop mode code (spot discovery for individual streets, postflop pot type selector)
10. Streamline `TrainerLanding.tsx` conditional rendering for 2 modes

### Hand generation (frontend + backend)

11. Backend: Add `skipGeneration` flag to session start serializer and view
12. Backend: Accept `replayContext` from request body in save hand results view
13. Frontend: Create V2 API service function to call V1/V2 hand generation endpoints
14. Frontend: Build V2 → V3 hand format transformer (mirrors backend `transforms.py`)
15. Frontend: Wire session start to use `skipGeneration: true`, generate hands via V1/V2
16. Frontend: Include `replayContext` when saving hand results

### Testing

17. Update unit tests for reworked landing page
18. Test hybrid hand generation flow end-to-end (session start → V1/V2 hand → play → save → end)
19. Verify statistics, hand history, and replayer work correctly with hybrid-generated hands
