# Consolidated Technical Debt Backlog (Frontend + Backend)

This document summarizes the agreed refactoring proposals across
frontend and backend, prioritized for **performance**, **stability**,
and **agent-friendly development**. The summary table comes first,
followed by detailed descriptions of each item.

------------------------------------------------------------------------

## Summary Priority Table

  ------------------------------------------------------------------------------------------------
  Priority   Refactor          Source     Impact   Effort   Risk     ROI          Agent-Friendly
  ---------- ----------------- ---------- -------- -------- -------- ------------ ----------------
  P0-1       Production-safe   Frontend   HIGH     LOW      LOW      ⭐⭐⭐⭐⭐   ⭐⭐⭐⭐
             logging                                                              

  P0-2       React compiler    Frontend   HIGH     LOW      LOW      ⭐⭐⭐⭐⭐   ⭐⭐⭐⭐
             optimizations                                                        

  P0-3       Faster JSON       Backend    HIGH     LOW      LOW      ⭐⭐⭐⭐     ⭐⭐⭐
             serialization                                                        

  P1-4       Hand upload       Backend    HIGH     HIGH     MED      ⭐⭐⭐⭐⭐   ⭐⭐⭐⭐⭐
             runnable locally                                                     

  P1-5       Remove hand       Backend    HIGH     HIGH     MED      ⭐⭐⭐⭐     ⭐⭐⭐⭐⭐
             upload race                                                          
             conditions                                                           

  P1-6       RT validation +   Backend    HIGH     MED      MED      ⭐⭐⭐⭐     ⭐⭐⭐⭐⭐
             structured errors                                                    

  P1-7       Remove unsafe     Frontend   HIGH     MED      LOW      ⭐⭐⭐⭐⭐   ⭐⭐⭐⭐⭐
             TypeScript usage                                                     

  P1-8       Reduce rerenders  Frontend   HIGH     MED      LOW      ⭐⭐⭐⭐     ⭐⭐⭐⭐
             from selectors                                                       

  P1-9       Consolidate       Frontend   HIGH     MED      MED      ⭐⭐⭐⭐     ⭐⭐⭐⭐
             WebSocket                                                            
             management                                                           

  P2-10      Typed schemas at  FE + BE    HIGH     HIGH     LOW      ⭐⭐⭐⭐     ⭐⭐⭐⭐⭐
             service                                                              
             boundaries                                                           

  P2-11      Public API        FE + BE    MED      HIGH     LOW      ⭐⭐⭐⭐     ⭐⭐⭐⭐⭐
             documentation                                                        

  P2-12      ADR decision      Backend    LOW      LOW      LOW      ⭐⭐⭐       ⭐⭐⭐⭐⭐
             notes                                                                

  P2-13      Reduce UI utility Frontend   MED      MED      LOW      ⭐⭐⭐⭐     ⭐⭐⭐⭐
             dependencies                                                         

  P2-14      Consolidate       Backend    MED      MED      MED      ⭐⭐⭐       ⭐⭐⭐⭐
             backend utilities                                                    

  P2-15      Standardize       Frontend   MED      MED      LOW      ⭐⭐⭐       ⭐⭐⭐⭐⭐
             component props                                                      

  P2-16      Feature-based     Frontend   MED      HIGH     MED      ⭐⭐⭐       ⭐⭐⭐⭐⭐
             frontend folders                                                     

  P3-20      Route-based code  Frontend   MED      MED      LOW      ⭐⭐⭐       ⭐⭐⭐
             splitting                                                            

  P3-23      API snapshot      Backend    LOW      LOW      LOW      ⭐⭐⭐       ⭐⭐⭐⭐
             tests                                                                
  ------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## Detailed Refactor Descriptions

### P0-1: Production-safe logging (Frontend)

Introduce a consistent logging approach that clearly distinguishes
between development and production behavior. This removes noisy console
output in production, improves signal quality when debugging, and gives
both humans and agents a predictable logging surface. It also reduces
accidental performance regressions caused by excessive logging.

### P0-2: React compiler optimizations (Frontend)

Enable compiler-level optimizations to automatically reduce unnecessary
re-renders. This lowers the need for manual memoization, simplifies
component code, and improves runtime performance without adding
conceptual overhead. Agents benefit from simpler, more declarative
components that are easier to reason about.

### P0-3: Faster JSON serialization (Backend)

Adopt a faster JSON serialization strategy for API responses to reduce
CPU usage and latency. This improves throughput under load and lowers
response times, while remaining largely transparent to application logic
and developer workflows.

------------------------------------------------------------------------

### P1-4: Hand upload runnable locally (Backend)

Simplify the hand upload pipeline so it can be executed end-to-end in a
local environment without relying on cloud-specific infrastructure. This
dramatically improves developer and agent productivity by making the
full flow observable, debuggable, and testable without external
dependencies.

### P1-5: Remove hand upload race conditions (Backend)

Eliminate race conditions caused by concurrent updates and fragile
coordination patterns in the hand upload pipeline. The goal is
deterministic processing with clear ownership of state transitions,
reducing intermittent production issues and making the system easier for
agents to modify safely.

### P1-6: RT validation + structured errors (Backend)

Introduce upfront validation and structured error handling for real-time
calculations. All required inputs are validated before computation
begins, preventing crashes mid-flow. Errors become predictable,
machine-readable, and easier for agents to interpret and act on.

### P1-7: Remove unsafe TypeScript usage (Frontend)

Eliminate `any` types and type suppressions to strengthen static
guarantees across the UI. This improves refactor safety, reduces runtime
errors, and gives agents much clearer signals about data shapes, intent,
and invariants throughout the codebase.

### P1-8: Reduce rerenders from selectors (Frontend)

Standardize state selection patterns to avoid unnecessary component
re-renders. This improves UI performance and responsiveness while also
making data dependencies more explicit and understandable to both humans
and agents.

### P1-9: Consolidate WebSocket management (Frontend)

Unify WebSocket usage behind a single, well-defined management layer.
This reduces duplicated connections, clarifies lifecycle handling, and
makes real-time data flow easier to trace and reason about.

------------------------------------------------------------------------

### P2-10: Typed schemas at service boundaries (Frontend + Backend)

Adopt explicit, shared schemas for request and response payloads at
service boundaries. This ensures end-to-end type safety, reduces
integration bugs, and allows agents to reason about system behavior
without guessing implicit contracts.

### P2-11: Public API documentation (Frontend + Backend)

Document all exported APIs and key modules with clear descriptions and
examples. This significantly improves onboarding speed, code navigation,
and agent comprehension, especially when making cross-cutting changes.

### P2-12: ADR decision notes (Backend)

Capture important architectural decisions in short, lightweight records.
These provide long-term context for why the system looks the way it
does, helping agents avoid repeating past mistakes or undoing
intentional tradeoffs.

### P2-13: Reduce UI utility dependencies (Frontend)

Trim the frontend dependency surface by favoring native utilities or
smaller, focused libraries. This reduces bundle size, simplifies mental
models, and lowers maintenance overhead for agents.

### P2-14: Consolidate backend utilities (Backend)

Group scattered helper functions into a clear, well-defined core
structure. This improves discoverability, reduces duplication, and makes
backend logic easier to navigate and extend.

### P2-15: Standardize component props (Frontend)

Define and apply consistent conventions for component props across the
UI. Predictable interfaces improve reuse, testing, and agent-driven
refactors.

### P2-16: Feature-based frontend folders (Frontend)

Reorganize frontend code around feature boundaries instead of technical
layers. This improves contextual understanding, reduces navigation time,
and aligns well with how agents reason about product behavior.

------------------------------------------------------------------------

### P3-20: Route-based code splitting (Frontend)

Improve route-level code splitting to reduce initial bundle size and
speed up first meaningful paint. This provides incremental performance
gains without significant architectural change.

### P3-23: API snapshot tests (Backend)

Add snapshot tests for API responses to lock down external behavior.
This creates a safety net for refactors, allowing agents to make changes
confidently without unintentionally breaking contracts.
