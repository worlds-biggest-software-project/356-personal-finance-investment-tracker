# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Personal Finance & Investment Tracker · Created: 2026-05-25

## Philosophy

Every state change — an account linked, a balance synced, a transaction imported, a holding purchased or sold, a tax lot created, a budget set, a goal created, a net worth snapshot taken — is an immutable event appended to a single `event_store` table. The event store is the sole source of truth. Read-optimised materialised views (read models) are projected from events to serve the dashboard, portfolio analytics, and tax planning.

Personal finance is inherently temporal: "what was my net worth on January 1st?", "what was my portfolio allocation when I made that rebalancing decision?", "how has my spending pattern changed since I started budgeting?" Event sourcing makes these temporal queries first-class: every balance change, every transaction, every holding update is a permanent record. When a user asks the AI assistant "why did my net worth drop in March?", the system can replay events from that period to identify the specific balance changes, market movements, or spending patterns that contributed.

The trade-off is infrastructure complexity — event replay, projection management, snapshot optimisation — but the payoff is a system where nothing is lost, every financial state is reconstructable, and new analytics (e.g., "what if I had rebalanced quarterly instead of annually?") can be answered by replaying events with different parameters.

**Best for:** Teams building a platform where full financial history, temporal queries, AI-driven what-if analysis, and privacy-first audit trails are core requirements.

**Trade-offs:**
- Pro: Complete financial history — every balance, transaction, and holding change is permanent
- Pro: Temporal queries: "what was my allocation on date X?"
- Pro: What-if analysis by replaying events with different parameters
- Pro: AI assistant can ground answers in specific historical events
- Pro: Privacy-first: event store can be encrypted per-user for local-first deployment
- Con: Read model projection infrastructure required
- Con: Eventual consistency between event writes and dashboard updates
- Con: Event schema evolution requires versioning strategy
- Con: Higher storage volume than mutable-row approaches

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | All events conform to CloudEvents spec |
| FDX API | Account link events include FDX account IDs |
| PSD2 / NextGenPSD2 | PSD2 consent events tracked |
| OAuth 2.0 | Token lifecycle events for account connections |
| ISIN (ISO 6166) | Security identifiers in holding events |
| CUSIP | US security identifiers in holding events |
| FIGI (ISO 4914) | Open instrument IDs in holding events |
| OpenAPI 3.1 | REST API documented in OpenAPI; read models serve responses |
| GDPR | Right-to-erasure via event crypto-shredding |
| CCPA/CPRA | Consumer deletion events |
| CFPB 1033 | FDX migration events |
| MCP | MCP config changes tracked as events |

---

## Event Infrastructure

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN (
                        'account','transaction','portfolio','tax',
                        'budget','goal','snapshot','config','user'
                    )),
    stream_id       UUID NOT NULL,
    version         BIGINT NOT NULL,
    event_type      TEXT NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- {"user_id": "uuid", "actor_type": "user|system|ai|aggregator",
    --  "ip_address": "...", "correlation_id": "uuid", "causation_id": "uuid"}
    ce_source       TEXT NOT NULL,
    ce_type         TEXT NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, version)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_stream ON event_store (stream_id, version);
CREATE INDEX idx_events_type ON event_store (event_type);
CREATE INDEX idx_events_stream_type ON event_store (stream_type, created_at);
CREATE INDEX idx_events_user ON event_store ((metadata->>'user_id'), created_at);

CREATE TABLE stream_snapshots (
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    version         BIGINT NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, version)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_version    BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Types by Stream

### Account Stream
Events for linked account lifecycle:

```
account_linked              — new account connected; captures aggregator, institution, account_type, mask
account_balance_synced      — daily balance update; captures balance_cents, available_balance_cents
account_sync_failed         — sync error; captures error_message, retry_count
account_reconnected         — re-authenticated after failure
account_renamed             — user renames account
account_hidden              — user hides from dashboard
account_unlinked            — account disconnected
fdx_migration_completed     — migrated from scraping to FDX API
```

### Transaction Stream
Events for transaction lifecycle:

```
transaction_imported        — new transaction from aggregator; captures amount_cents, description, date
transaction_categorised     — auto or manual category assignment; captures category, confidence, source
transaction_recategorised   — user corrects category; captures old_category, new_category
transaction_split           — split into multiple categories; captures splits array
transaction_tagged          — tag added; captures tag
transaction_excluded        — excluded from budgets/analytics
recurring_detected          — subscription/recurring pattern identified; captures frequency, merchant
recurring_price_changed     — subscription price change detected; captures old_amount, new_amount
```

### Portfolio Stream
Events for investment holdings:

```
holding_added               — new position; captures security_id, ticker, quantity, cost_basis_cents
holding_updated             — position quantity or value changed; captures old/new quantity, value
holding_removed             — position closed; captures final_value, gain_loss
tax_lot_created             — purchase lot recorded; captures acquired_date, quantity, cost_per_unit
tax_lot_sold                — lot (partially) sold; captures quantity_sold, proceeds_cents, gain_loss
market_value_updated        — daily price update; captures security_id, new_price, new_value
allocation_calculated       — asset allocation recomputed; captures allocation breakdown
performance_calculated      — returns computed; captures twr, sharpe, beta, benchmark comparison
fee_analysis_completed      — fee drag calculated; captures total_fees, fee_drag_pct
```

### Tax Stream
Events for tax tracking:

```
gain_realised               — capital gain/loss realised; captures proceeds, cost_basis, gain_loss, is_long_term
wash_sale_detected          — wash sale identified; captures disallowed_amount, triggering lots
dividend_received           — dividend income; captures amount, qualified vs ordinary
harvesting_opportunity_found — TLH opportunity identified; captures security, unrealised_loss, replacement
roth_conversion_modelled    — Roth conversion scenario; captures conversion_amount, tax_impact
estimated_tax_calculated    — quarterly estimate; captures q1-q4 amounts, total_liability
```

### Budget Stream
```
budget_set                  — budget created/updated for category; captures category, amount, period
spending_threshold_reached  — 80% or 100% of budget hit; captures category, actual, budgeted
anomaly_detected            — unusual charge; captures transaction, reason, severity
```

### Goal Stream
```
goal_created                — new financial goal; captures name, type, target, target_date
goal_progress_updated       — periodic progress check; captures current_amount, progress_pct
projection_calculated       — Monte Carlo run; captures success_pct, scenarios
goal_achieved               — target reached
goal_modified               — target or date changed
```

---

## Read Models

```sql
CREATE TABLE rm_net_worth (
    user_id         UUID PRIMARY KEY,
    total_assets_cents BIGINT NOT NULL DEFAULT 0,
    total_liabilities_cents BIGINT NOT NULL DEFAULT 0,
    net_worth_cents BIGINT NOT NULL DEFAULT 0,
    accounts_json   JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Chase Checking", "type": "checking",
    --   "institution": "JPMorgan Chase", "balance_cents": 5000000,
    --   "is_asset": true, "sync_status": "connected", "last_sync_at": "..."
    -- }]
    allocation_json JSONB NOT NULL DEFAULT '{}',
    -- {"checking": 5000000, "savings": 20000000, "brokerage": 150000000,
    --  "retirement": 80000000, "real_estate": 450000000,
    --  "credit_cards": -4500000, "mortgage": -320000000}
    history_json    JSONB NOT NULL DEFAULT '[]',
    -- [{"date": "2026-05-25", "net_worth_cents": 310000000},
    --  {"date": "2026-05-24", "net_worth_cents": 308500000}]
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rm_portfolio_analytics (
    user_id         UUID NOT NULL,
    period          DATE NOT NULL,
    total_value_cents BIGINT NOT NULL DEFAULT 0,
    holdings_json   JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "account_id": "uuid", "account_name": "Fidelity Brokerage",
    --   "security_id": "uuid", "ticker": "VTI", "name": "Vanguard Total Stock",
    --   "quantity": 150.5, "market_value_cents": 3450000,
    --   "cost_basis_cents": 2800000, "unrealised_gain_cents": 650000,
    --   "weight_pct": 32.5, "asset_class": "us_equity"
    -- }]
    allocation_json JSONB NOT NULL DEFAULT '{}',
    -- {"us_equity": 0.45, "intl_equity": 0.20, "fixed_income": 0.25,
    --  "real_estate": 0.05, "cash": 0.05}
    performance_json JSONB NOT NULL DEFAULT '{}',
    -- {"twr_1m": 0.023, "twr_3m": 0.067, "twr_ytd": 0.089, "twr_1y": 0.124,
    --  "sharpe_ratio": 1.42, "beta": 0.95,
    --  "benchmark_sp500_ytd": 0.112, "benchmark_6040_ytd": 0.078}
    fees_json       JSONB NOT NULL DEFAULT '{}',
    -- {"total_annual_fees_cents": 12500, "fee_drag_pct": 0.054,
    --  "by_holding": [{"ticker": "ARKK", "expense_ratio": 0.0075, "annual_fee_cents": 3750}]}
    config_at_period JSONB,
    PRIMARY KEY (user_id, period)
);
CREATE INDEX idx_rm_portfolio_user ON rm_portfolio_analytics (user_id, period);

CREATE TABLE rm_budget_dashboard (
    user_id         UUID NOT NULL,
    period          DATE NOT NULL,
    total_income_cents BIGINT NOT NULL DEFAULT 0,
    total_spending_cents BIGINT NOT NULL DEFAULT 0,
    savings_rate    REAL,
    categories_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "category_id": "uuid", "name": "Housing", "budgeted_cents": 200000,
    --   "actual_cents": 195000, "variance_cents": -5000,
    --   "transaction_count": 3
    -- }]
    subscriptions_json JSONB NOT NULL DEFAULT '[]',
    -- [{"merchant": "Netflix", "amount_cents": 2299, "frequency": "monthly",
    --   "detected_at": "...", "price_changed": false}]
    anomalies_json  JSONB NOT NULL DEFAULT '[]',
    -- [{"transaction_id": "uuid", "description": "...", "amount_cents": 45000,
    --   "reason": "unusually_large", "severity": "warning"}]
    PRIMARY KEY (user_id, period)
);
CREATE INDEX idx_rm_budget_user ON rm_budget_dashboard (user_id, period);

CREATE TABLE rm_tax_dashboard (
    user_id         UUID NOT NULL,
    tax_year        INTEGER NOT NULL,
    realised_gains_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "short_term_gains_cents": 150000, "short_term_losses_cents": -45000,
    --   "long_term_gains_cents": 350000, "long_term_losses_cents": -120000,
    --   "net_gain_cents": 335000, "estimated_tax_cents": 80400,
    --   "wash_sales": [{"security": "TSLA", "disallowed_cents": 12000, "date": "2026-03-15"}]
    -- }
    dividends_json  JSONB NOT NULL DEFAULT '{}',
    -- {"qualified_cents": 65000, "ordinary_cents": 20000, "total_cents": 85000}
    harvesting_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "security_id": "uuid", "ticker": "ARKK", "unrealised_loss_cents": -35000,
    --   "account": "Fidelity Brokerage", "is_long_term": true,
    --   "potential_tax_savings_cents": 8400,
    --   "replacement_candidates": ["VTI", "SCHB"]
    -- }]
    contributions_json JSONB NOT NULL DEFAULT '{}',
    -- {"401k_cents": 2300000, "ira_cents": 700000, "hsa_cents": 415000,
    --  "remaining_401k": 0, "remaining_ira": 0, "remaining_hsa": 0}
    estimated_payments_json JSONB NOT NULL DEFAULT '{}',
    -- {"q1": 20100, "q2": 20100, "q3": 20100, "q4": 20100, "total": 80400}
    PRIMARY KEY (user_id, tax_year)
);
CREATE INDEX idx_rm_tax_user ON rm_tax_dashboard (user_id);

CREATE TABLE rm_cost_dashboard (
    user_id         UUID NOT NULL,
    period          DATE NOT NULL,
    api_calls       INTEGER NOT NULL DEFAULT 0,
    aggregator_syncs INTEGER NOT NULL DEFAULT 0,
    ai_queries      INTEGER NOT NULL DEFAULT 0,
    ai_input_tokens BIGINT NOT NULL DEFAULT 0,
    ai_output_tokens BIGINT NOT NULL DEFAULT 0,
    ai_cost_cents   BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (user_id, period)
);
CREATE INDEX idx_rm_cost_user ON rm_cost_dashboard (user_id, period);
```

---

## Example Queries

### Replay net worth changes in a specific month

```sql
SELECT event_type, payload, ce_time
FROM event_store
WHERE stream_type = 'account'
  AND event_type = 'account_balance_synced'
  AND metadata->>'user_id' = 'user-uuid'
  AND ce_time >= '2026-03-01'
  AND ce_time < '2026-04-01'
ORDER BY ce_time ASC;
```

### What was my portfolio allocation when I made a rebalancing trade?

```sql
SELECT e.payload AS allocation_snapshot
FROM event_store e
WHERE e.stream_type = 'portfolio'
  AND e.event_type = 'allocation_calculated'
  AND e.metadata->>'user_id' = 'user-uuid'
  AND e.ce_time <= '2026-02-15'
ORDER BY e.version DESC
LIMIT 1;
```

### Detect spending pattern changes after starting a budget

```sql
WITH budget_start AS (
    SELECT ce_time AS start_time
    FROM event_store
    WHERE stream_type = 'budget'
      AND event_type = 'budget_set'
      AND metadata->>'user_id' = 'user-uuid'
    ORDER BY ce_time ASC
    LIMIT 1
)
SELECT
    CASE WHEN e.ce_time < bs.start_time THEN 'before_budget' ELSE 'after_budget' END AS period,
    COUNT(*) AS transaction_count,
    SUM((e.payload->>'amount_cents')::BIGINT) AS total_cents,
    AVG((e.payload->>'amount_cents')::BIGINT) AS avg_amount_cents
FROM event_store e
CROSS JOIN budget_start bs
WHERE e.stream_type = 'transaction'
  AND e.event_type = 'transaction_imported'
  AND e.metadata->>'user_id' = 'user-uuid'
  AND e.ce_time >= bs.start_time - INTERVAL '3 months'
  AND e.ce_time <= bs.start_time + INTERVAL '3 months'
GROUP BY CASE WHEN e.ce_time < bs.start_time THEN 'before_budget' ELSE 'after_budget' END;
```

### Tax-loss harvesting opportunities from tax events

```sql
SELECT ticker, unrealised_loss,
       potential_savings, replacement_candidates
FROM (
    SELECT h->>'ticker' AS ticker,
           (h->>'unrealised_loss_cents')::BIGINT / 100.0 AS unrealised_loss,
           (h->>'potential_tax_savings_cents')::BIGINT / 100.0 AS potential_savings,
           h->>'replacement_candidates' AS replacement_candidates
    FROM rm_tax_dashboard,
         jsonb_array_elements(harvesting_json) AS h
    WHERE user_id = 'user-uuid'
      AND tax_year = 2026
) sub
ORDER BY unrealised_loss ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_net_worth, rm_portfolio_analytics, rm_budget_dashboard, rm_tax_dashboard, rm_cost_dashboard |
| **Total** | **8** | 3 infrastructure + 5 read models |

---

## Key Design Decisions

1. **Account balance syncs as events** — every daily balance update is an immutable event; this enables temporal net worth queries ("what was my net worth on date X?") by replaying balance events up to that date, critical for the AI assistant's ability to explain net worth changes.

2. **Transaction categorisation as events** — `transaction_categorised` and `transaction_recategorised` events create a learning signal; the AI can identify which auto-categorisations users correct most frequently and adjust the categorisation model.

3. **`holding_added` / `tax_lot_created` / `tax_lot_sold` events** — the full purchase and sale lifecycle is captured as events, enabling cost basis recalculation under different accounting methods (FIFO vs HIFO) by replaying lots with different sort orders.

4. **`wash_sale_detected` as a first-class event** — wash sale detection events reference the triggering lots and disallowed amounts, enabling tax dashboard projections and alerts when a planned sale would trigger a wash sale within the 30-day window.

5. **`rm_tax_dashboard` with `harvesting_json`** — tax-loss harvesting opportunities are projected from portfolio events and include replacement security candidates, enabling the AI to suggest specific tax-optimising trades.

6. **`rm_portfolio_analytics` with `config_at_period`** — snapshots of the user's investment config (target allocation, rebalancing frequency) at each period enable before/after analysis when the user changes their strategy.

7. **`recurring_detected` and `recurring_price_changed` events** — subscription detection and price change tracking are event-driven, enabling proactive alerts and spending trend analysis.

8. **`anomaly_detected` events on the budget stream** — unusual charges are flagged as events by the AI, providing a permanent record of flagged items and their resolution for model improvement.

9. **Event-sourced what-if analysis** — because the full portfolio history is replayable, the AI can answer "what if I had contributed $500/month more to my 401k since 2023?" by replaying contribution events with modified amounts and projecting forward.

10. **Privacy-first event store** — events can be encrypted per-user with a key the user controls, enabling a local-first deployment where the event store lives on the user's device and read models are computed locally — a key differentiator for privacy-conscious users.
