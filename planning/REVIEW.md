# Review of PLAN.md

Date: 2026-04-27
Reviewer: GitHub Copilot

## Summary

The plan is strong on product vision, UX goals, and high-level architecture. The biggest risks are in execution safety and operational consistency: trade/accounting precision, concurrent trade correctness, and a Docker volume instruction mismatch. Addressing these now will prevent hard-to-debug defects later.

## Findings (ordered by severity)

### 1. High: Monetary precision risk from floating-point storage

- Location: PLAN.md lines 217 and 228
- Issue: Positions and trades use REAL for quantity and price-related values. Floating-point math can produce rounding drift in cash balance, average cost, and P&L over many operations.
- Risk: Portfolio values can become inconsistent over time, especially with fractional shares and repeated buy/sell cycles.
- Recommendation:
  - Store money in integer minor units (cents) or use Decimal semantics end-to-end.
  - If fractional shares are required, define fixed precision rules explicitly (for example: quantity to 1e-6, price to cents, cash to cents).
  - Add deterministic rounding policy in one place and document it.

### 2. High: Concurrency and atomicity for trade execution are underspecified

- Location: PLAN.md lines 273, 315, 316
- Issue: Manual trades and AI-triggered trade batches can both mutate cash and positions, but the plan does not require transactional boundaries or locking strategy.
- Risk: Race conditions can cause overspending, negative holdings, or inconsistent trade history under concurrent requests.
- Recommendation:
  - Require each trade (and each AI batch step) to run in a single DB transaction with re-read + validate + write.
  - For AI batch semantics, clarify whether partial success is allowed and how rollback behaves.
  - Add idempotency keys for POST trade requests to avoid accidental duplicate executions.

### 3. High: Docker volume instructions conflict

- Location: PLAN.md lines 416 and 419
- Issue: The example uses a named volume (-v finally-data:/app/db), but the text immediately says the project-root db directory maps to /app/db (bind mount behavior).
- Risk: Setup confusion and inconsistent persistence behavior across environments.
- Recommendation:
  - Pick one approach and document it consistently:
  - Named volume example and wording, or bind mount example and wording.
  - If both are supported, show both commands and explain when to use each.

### 4. Medium: SSE update cadence may be unnecessarily noisy with slow upstream polling

- Location: PLAN.md lines 162, 165, 178
- Issue: Upstream can update every 15 seconds on free tier, but SSE pushes every ~500ms for all watchlist tickers.
- Risk: Clients process frequent unchanged payloads and backend does extra work for little user value.
- Recommendation:
  - Emit only on change (or heartbeat + deltas), not full snapshots at fixed high frequency when data is stale.
  - Document SSE payload mode explicitly: full snapshot vs changed symbols only.

### 5. Medium: Unbounded chat history growth

- Location: PLAN.md lines 239, 311, 316
- Issue: portfolio_snapshots retention is defined, but chat_messages retention is not.
- Risk: DB bloat, slower chat history reads, and larger prompts over time.
- Recommendation:
  - Define retention/archival policy for chat_messages (for example keep 7-30 days, or cap at N messages).
  - Add index guidance for common access patterns (user_id, created_at).

### 6. Medium: Environment exposure assumption is not explicit

- Location: PLAN.md lines 13 and 416
- Issue: No auth is intentional, but network exposure constraints are not clearly stated in deployment/run instructions.
- Risk: If port publishing is not constrained, others on the network may access and trade in the demo app.
- Recommendation:
  - State that default run mode is local-only development and should bind to localhost when possible.
  - Add a warning that no-auth mode is for simulation/dev, not internet exposure.

### 7. Low: Initialization trigger wording is ambiguous

- Location: PLAN.md line 188
- Issue: Startup (or first request) leaves behavior undefined.
- Risk: Different implementations may initialize at different times, causing test and operational drift.
- Recommendation:
  - Define one source of truth: app startup lifecycle hook or explicit lazy-init gate.
  - If lazy-init is retained, define race-safe first-request behavior.

### 8. Low: Build reproducibility can be improved

- Location: PLAN.md line 398
- Issue: npm install is less reproducible than npm ci in CI/container builds.
- Risk: Dependency drift between environments.
- Recommendation:
  - Prefer npm ci when lockfile exists.

## Test Coverage Gaps to Add

1. Numeric invariants over long trade sequences
- Verify cash + market value equals expected portfolio total within defined precision policy.
- Include many fractional-share operations and alternating buys/sells.

2. Concurrency tests for trade correctness
- Simulate parallel POST /api/portfolio/trade and chat-triggered trades on same ticker.
- Assert no negative cash/quantity and consistent trade log ordering.

3. SSE efficiency and semantics
- Verify stream behavior when upstream data is unchanged for multiple intervals.
- Verify reconnect behavior preserves correctness for watchlist add/remove events.

4. Persistence mode tests
- Validate behavior under named volume and bind mount modes (if both documented).

## What is already strong

- Clear UX contract and single-command startup goal.
- Sensible single-container architecture for a teaching/demo project.
- Good separation of market-data provider behind one interface.
- Explicit LLM structured-output contract and mock mode for testing.

## Suggested edits in next revision

1. Add a short Non-Functional Requirements section covering precision policy, transactionality, idempotency, and retention.
2. Resolve the Docker persistence wording conflict.
3. Clarify SSE payload semantics (delta vs snapshot) and cadence policy.
4. Add a Security and Exposure note for no-auth local development mode.
5. Expand testing section with numeric and concurrency invariants.
