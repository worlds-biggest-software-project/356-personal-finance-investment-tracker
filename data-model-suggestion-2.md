# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Personal Finance & Investment Tracker · Created: 2026-05-25

## Philosophy

Core operational entities — users, linked accounts, transactions, securities — are relational tables with indexed columns for multi-account aggregation, date-range queries, and security lookups. Variable-structure data — account connection configs, holdings with tax lots, budget plans, goal projections, alternative asset details, tax event summaries — lives in JSONB columns with GIN indexes.

Personal finance platforms have a mixed access pattern: high-frequency transactional writes (daily balance syncs, new transactions) alongside read-heavy dashboard queries (net worth, portfolio allocation, budget progress). The hybrid model separates high-volume transactions into a partitioned relational table for fast date-range queries, while embedding portfolio state (holdings, tax lots, allocation) as JSONB on accounts for single-row portfolio reads. When a user opens their dashboard, loading all holdings for an account is a single-row read rather than a multi-table join across holdings, securities, and tax_lots.

The trade-off is that cross-account portfolio analytics (e.g., "total exposure to tech stocks across all accounts") requires JSONB extraction across multiple account rows rather than a simple JOIN. For users with fewer than 20 accounts — the vast majority — this is fast enough, and the schema simplicity pays for itself.

**Best for:** Teams building an MVP where rapid iteration on account types, minimal schema migrations, and fast dashboard loading are priorities over deep cross-account analytics.

**Trade-offs:**
- Pro: 6 tables — simple schema, fast to deploy
- Pro: Single-row account reads for portfolio display
- Pro: New account types and alternative assets require no schema migration
- Pro: Holdings and tax lots embedded per account simplify portfolio loading
- Con: Cross-account portfolio analytics require JSONB aggregation
- Con: Large accounts with many holdings can produce oversized rows
- Con: No FK enforcement on security references within JSONB
- Con: Tax lot calculations must be application-layer for cross-account wash sales

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FDX API | `linked_accounts.connection_json` stores FDX account IDs |
| PSD2 / NextGenPSD2 | `linked_accounts.connection_json` supports PSD2 providers |
| FAPI 2.0 | Token management config in connection details |
| OAuth 2.0 / OIDC | User auth; `linked_accounts.connection_json` stores OAuth tokens |
| ISIN (ISO 6166) | `securities.isin` for global identification |
| CUSIP | `securities.cusip` for US/Canadian identification |
| FIGI (ISO 4914) | `securities.figi` for open instrument mapping |
| OFX | Import format tracked in `linked_accounts.connection_json` |
| OpenAPI 3.1 | REST API documented in OpenAPI |
| GDPR | Data retention in `users.settings_json` |
| CCPA/CPRA | Privacy rights config in `users.settings_json` |
| CFPB 1033 | FDX-aligned connection support |
| ISO 27001 / SOC 2 | Audit logging |
| MCP | `users.mcp_json` exposes finance data as AI context |

---

## Users

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    password_hash   TEXT NOT NULL,
    household_json  JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "id": "uuid", "name": "Smith Family",
    --   "members": [
    --     {"user_id": "uuid", "email": "...", "display_name": "...",
    --      "role": "owner|editor|viewer", "joined_at": "..."}
    --   ]
    -- }
    budget_config_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "categories": [
    --     {"id": "uuid", "name": "Housing", "parent_id": null, "icon": "home",
    --      "colour_hex": "#4285F4", "is_income": false, "sort_order": 0},
    --     {"id": "uuid", "name": "Rent", "parent_id": "uuid", "sort_order": 0}
    --   ],
    --   "budgets": [
    --     {"category_id": "uuid", "period_type": "monthly",
    --      "amount_cents": 200000, "start_date": "2026-01-01"}
    --   ]
    -- }
    goals_json      JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Retire at 60", "goal_type": "retirement",
    --   "target_amount_cents": 200000000, "current_amount_cents": 85000000,
    --   "target_date": "2045-03-15",
    --   "monthly_contribution_cents": 300000,
    --   "linked_account_ids": ["uuid", "uuid"],
    --   "projection": {"monte_carlo_success_pct": 82,
    --                  "scenarios": [{"name": "optimistic", "return_pct": 10}]}
    -- }]
    tax_config_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "filing_status": "married_filing_jointly", "tax_bracket_pct": 24,
    --   "state_code": "CA", "country_code": "US",
    --   "accounting_method": "fifo",
    --   "wash_sale_monitoring": true
    -- }
    alerts_json     JSONB NOT NULL DEFAULT '[]',
    -- [{"id": "uuid", "type": "portfolio_drift", "message": "...",
    --   "severity": "warning", "created_at": "...", "read": false}]
    mcp_json        JSONB,
    -- {"enabled": true, "tools": [...], "resources": [...]}
    settings_json   JSONB NOT NULL DEFAULT '{}',
    -- {"default_currency": "USD", "locale": "en-US", "timezone": "America/New_York",
    --  "data_retention_days": 365, "gdpr_consent_at": "..."}
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_goals ON users USING GIN (goals_json);
```

---

## Linked Accounts

```sql
CREATE TABLE linked_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    account_name    TEXT NOT NULL,
    account_type    TEXT NOT NULL CHECK (account_type IN (
                        'checking','savings','credit_card','loan','mortgage',
                        'brokerage','ira_traditional','ira_roth','401k','403b',
                        'hsa','529','crypto_exchange','crypto_wallet',
                        'real_estate','vehicle','precious_metal','private_equity',
                        'other'
                    )),
    institution_name TEXT,
    currency        TEXT NOT NULL DEFAULT 'USD',
    current_balance_cents BIGINT NOT NULL DEFAULT 0,
    is_asset        BOOLEAN NOT NULL DEFAULT TRUE,
    is_investment   BOOLEAN NOT NULL DEFAULT FALSE,
    is_hidden       BOOLEAN NOT NULL DEFAULT FALSE,
    connection_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "aggregator": "plaid", "item_id": "...", "fdx_account_id": "...",
    --   "mask": "1234", "last_sync_at": "...",
    --   "sync_status": "connected", "sync_error": null
    -- }
    -- For manual: {"aggregator": "manual"}
    -- For PSD2: {"aggregator": "truelayer", "consent_id": "...", "institution_id": "..."}
    holdings_json   JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "security_id": "uuid", "ticker": "VTI", "name": "Vanguard Total Stock",
    --   "quantity": 150.5, "cost_basis_cents": 2800000, "market_value_cents": 3450000,
    --   "unrealised_gain_cents": 650000, "weight_pct": 32.5,
    --   "tax_lots": [
    --     {"id": "uuid", "acquired_date": "2022-03-15", "quantity": 50,
    --      "cost_per_unit_cents": 18500, "total_cost_cents": 925000,
    --      "current_value_cents": 1150000, "is_long_term": true,
    --      "source": "purchase"}
    --   ]
    -- }]
    tax_events_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "event_type": "realised_gain", "event_date": "2026-04-15",
    --   "security_id": "uuid", "ticker": "AAPL",
    --   "quantity": 10, "proceeds_cents": 185000, "cost_basis_cents": 150000,
    --   "gain_loss_cents": 35000, "is_long_term": true,
    --   "is_wash_sale": false, "tax_year": 2026, "form_8949_box": "D"
    -- }]
    alternative_asset_json JSONB,
    -- For real_estate: {"address": "...", "zillow_zestimate_cents": 4500000,
    --                   "purchase_price_cents": 3800000, "purchase_date": "2019-06-01",
    --                   "property_type": "single_family", "lot_size_sqft": 8500}
    -- For vehicle: {"vin": "...", "make": "Tesla", "model": "Model 3", "year": 2024,
    --              "estimated_value_cents": 3200000}
    -- For crypto_wallet: {"chain": "ethereum", "address": "0x...",
    --                     "tokens": [{"symbol": "ETH", "balance": 5.2, "value_cents": ...}]}
    performance_json JSONB NOT NULL DEFAULT '{}',
    -- {"twr_1m": 0.023, "twr_3m": 0.067, "twr_ytd": 0.089, "twr_1y": 0.124,
    --  "sharpe_ratio": 1.42, "beta": 0.95,
    --  "allocation": {"us_equity": 0.45, "intl_equity": 0.20, "fixed_income": 0.25, "cash": 0.10}}
    balance_history_json JSONB NOT NULL DEFAULT '[]',
    -- [{"date": "2026-05-25", "balance_cents": 3450000},
    --  {"date": "2026-05-24", "balance_cents": 3420000}]
    metadata_json   JSONB NOT NULL DEFAULT '{}',
    -- {"interest_rate": 0.045, "credit_limit_cents": 1500000,
    --  "beneficiary": {"name": "...", "relationship": "spouse"}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_accounts_user ON linked_accounts (user_id);
CREATE INDEX idx_accounts_type ON linked_accounts (account_type);
CREATE INDEX idx_accounts_holdings ON linked_accounts USING GIN (holdings_json);
CREATE INDEX idx_accounts_tax ON linked_accounts USING GIN (tax_events_json);
```

---

## Transactions

```sql
CREATE TABLE transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    account_id      UUID NOT NULL REFERENCES linked_accounts(id),
    transaction_date DATE NOT NULL,
    posted_date     DATE,
    description     TEXT NOT NULL,
    amount_cents    BIGINT NOT NULL,
    currency        TEXT NOT NULL DEFAULT 'USD',
    categorization_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "category_id": "uuid", "category_name": "Groceries",
    --   "merchant_name": "Whole Foods", "is_income": false,
    --   "is_transfer": false, "is_recurring": true,
    --   "recurring_group_id": "uuid",
    --   "auto_categorised": true, "confidence": 0.95,
    --   "budget_id": "uuid"
    -- }
    source          TEXT NOT NULL CHECK (source IN (
                        'aggregator','manual','csv_import','ofx_import'
                    )) DEFAULT 'aggregator',
    external_id     TEXT,
    is_excluded     BOOLEAN NOT NULL DEFAULT FALSE,
    notes           TEXT,
    tags            TEXT[] NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (transaction_date);

CREATE INDEX idx_txn_user ON transactions (user_id);
CREATE INDEX idx_txn_account ON transactions (account_id);
CREATE INDEX idx_txn_date ON transactions (transaction_date);
CREATE INDEX idx_txn_categorization ON transactions USING GIN (categorization_json);
CREATE INDEX idx_txn_tags ON transactions USING GIN (tags);
```

---

## Securities

```sql
CREATE TABLE securities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticker          TEXT,
    name            TEXT NOT NULL,
    security_type   TEXT NOT NULL CHECK (security_type IN (
                        'stock','etf','mutual_fund','bond','option',
                        'crypto','commodity','reit','index','other'
                    )),
    isin            TEXT,
    cusip           TEXT,
    figi            TEXT,
    exchange        TEXT,
    currency        TEXT NOT NULL DEFAULT 'USD',
    asset_class     TEXT CHECK (asset_class IN (
                        'us_equity','intl_equity','emerging_markets',
                        'fixed_income','real_estate','commodity',
                        'crypto','cash','alternative','other'
                    )),
    metadata_json   JSONB NOT NULL DEFAULT '{}',
    -- {"sector": "Technology", "expense_ratio": 0.0003,
    --  "dividend_yield": 0.015, "market_cap": "large",
    --  "last_price_cents": 23050, "last_price_at": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_securities_ticker ON securities (ticker)
    WHERE ticker IS NOT NULL;
CREATE INDEX idx_securities_isin ON securities (isin)
    WHERE isin IS NOT NULL;
CREATE INDEX idx_securities_cusip ON securities (cusip)
    WHERE cusip IS NOT NULL;
CREATE INDEX idx_securities_figi ON securities (figi)
    WHERE figi IS NOT NULL;
CREATE INDEX idx_securities_type ON securities (security_type);
```

---

## Financial Snapshots & Audit

```sql
CREATE TABLE financial_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    snapshot_date   DATE NOT NULL,
    net_worth_json  JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total_assets_cents": 650000000, "total_liabilities_cents": 340000000,
    --   "net_worth_cents": 310000000,
    --   "by_type": {"checking": 5000000, "savings": 20000000, "brokerage": 150000000,
    --               "retirement": 80000000, "real_estate": 450000000,
    --               "credit_cards": -4500000, "mortgage": -320000000}
    -- }
    portfolio_json  JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total_value_cents": 230000000,
    --   "allocation": {"us_equity": 0.45, "intl_equity": 0.20, "fixed_income": 0.25, "cash": 0.10},
    --   "twr_1m": 0.023, "twr_ytd": 0.089, "twr_1y": 0.124,
    --   "benchmark_sp500_ytd": 0.112, "benchmark_6040_ytd": 0.078,
    --   "total_fees_annual_cents": 12500, "fee_drag_pct": 0.054
    -- }
    budget_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "month": "2026-05", "total_income_cents": 1200000, "total_spending_cents": 850000,
    --   "by_category": [{"name": "Housing", "budgeted": 200000, "actual": 195000},
    --                   {"name": "Groceries", "budgeted": 80000, "actual": 92000}],
    --   "subscriptions_detected": 12, "subscription_total_cents": 45000
    -- }
    tax_summary_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "tax_year": 2026, "realised_gains_cents": 350000, "realised_losses_cents": -120000,
    --   "net_gain_cents": 230000, "estimated_tax_cents": 55200,
    --   "wash_sales_count": 1, "harvesting_opportunities": 3,
    --   "dividends_cents": 85000
    -- }
    goal_progress_json JSONB NOT NULL DEFAULT '[]',
    -- [{"goal_id": "uuid", "name": "Retire at 60", "progress_pct": 42.5,
    --   "current_cents": 85000000, "target_cents": 200000000}]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, snapshot_date)
) PARTITION BY RANGE (snapshot_date);

CREATE INDEX idx_snapshots_user ON financial_snapshots (user_id, snapshot_date);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    actor_type      TEXT NOT NULL CHECK (actor_type IN (
                        'user','system','ai','aggregator'
                    )),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    changes_json    JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_user ON audit_log (user_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
```

---

## Example Queries

### Asset allocation across all accounts

```sql
SELECT la.account_name, la.account_type,
       la.performance_json->>'allocation' AS allocation,
       la.current_balance_cents / 100.0 AS balance
FROM linked_accounts la
WHERE la.user_id = 'user-uuid'
  AND la.is_investment = TRUE
  AND la.is_hidden = FALSE
ORDER BY la.current_balance_cents DESC;
```

### Tax-loss harvesting from embedded tax lots

```sql
SELECT la.account_name,
       h->>'ticker' AS ticker,
       h->>'name' AS security_name,
       (h->>'unrealised_gain_cents')::BIGINT / 100.0 AS unrealised_gain,
       lot->>'acquired_date' AS acquired,
       (lot->>'is_long_term')::BOOLEAN AS long_term
FROM linked_accounts la,
     jsonb_array_elements(la.holdings_json) AS h,
     jsonb_array_elements(h->'tax_lots') AS lot
WHERE la.user_id = 'user-uuid'
  AND (h->>'unrealised_gain_cents')::BIGINT < 0
ORDER BY (h->>'unrealised_gain_cents')::BIGINT ASC;
```

### Budget vs actual spending

```sql
SELECT (b->>'name')::TEXT AS category,
       (b->>'budgeted')::BIGINT / 100.0 AS budgeted,
       (b->>'actual')::BIGINT / 100.0 AS actual,
       ((b->>'actual')::BIGINT - (b->>'budgeted')::BIGINT) / 100.0 AS variance
FROM financial_snapshots fs,
     jsonb_array_elements(fs.budget_json->'by_category') AS b
WHERE fs.user_id = 'user-uuid'
  AND fs.snapshot_date = CURRENT_DATE - 1
ORDER BY variance DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users | 1 | users (embeds household, budgets, goals, tax config, alerts, MCP) |
| Accounts | 1 | linked_accounts (embeds holdings, tax lots, tax events, alternative assets, performance, balance history) |
| Transactions | 1 | transactions (partitioned; embeds categorization) |
| Securities | 1 | securities (reference data with ISIN/CUSIP/FIGI) |
| Snapshots | 1 | financial_snapshots (partitioned; embeds net worth, portfolio, budget, tax, goals) |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **6** | |

---

## Key Design Decisions

1. **`holdings_json` with embedded `tax_lots` on linked_accounts** — the dashboard's primary operation is "show all holdings for this account with cost basis"; embedding holdings and their tax lots in a single JSONB column makes this a one-row read per account.

2. **`tax_events_json` on linked_accounts** — realised gains, losses, and wash sales are account-scoped events; embedding them keeps the per-account tax summary accessible without a separate join table.

3. **`alternative_asset_json` on linked_accounts** — real estate, vehicles, crypto wallets, and private equity have wildly different property schemas; JSONB absorbs this variation without separate tables or wide nullable columns.

4. **`categorization_json` on transactions** — transaction categorisation includes merchant name, category, auto-categorisation confidence, and budget linkage; JSONB accommodates evolving categorisation models without migration.

5. **`financial_snapshots` as a dedicated partitioned table** — daily snapshots capturing net worth, portfolio performance, budget progress, tax summary, and goal progress in JSONB provide point-in-time financial state without recomputing from current data.

6. **`budget_config_json` on users** — budget categories and spending limits are user-level configuration with hierarchical categories; JSONB accommodates variable category trees without a recursive relational table.

7. **`goals_json` on users** — financial goals with Monte Carlo projections are user-level, varying in number and type; JSONB accommodates goal creation/deletion without a separate table.

8. **`performance_json` on linked_accounts** — time-weighted returns, Sharpe ratio, beta, and allocation breakdown are computed periodically and cached as JSONB, avoiding real-time computation on every dashboard load.

9. **`securities` as a relational reference table** — securities are shared across users and need indexed lookups by ticker, ISIN, CUSIP, and FIGI; a relational table with unique indexes is the right choice for reference data.

10. **6 tables** — personal finance has a strongly user-centric access pattern (everything flows from user → accounts → transactions); embedding configuration and portfolio state into these core entities minimises joins for the dominant dashboard operations.
