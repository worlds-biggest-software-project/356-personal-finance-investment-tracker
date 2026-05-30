# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Personal Finance & Investment Tracker · Created: 2026-05-25

## Philosophy

Every concept in the personal finance tracker — users, households, linked accounts, transactions, securities, holdings, tax lots, tax events, budgets, categories, goals, net worth snapshots, alternative assets, documents — is a first-class relational table with foreign keys, indexed columns, and CHECK constraints. This produces a schema where every relationship is explicit and every query can be answered with standard SQL joins.

Personal finance platforms have a complex web of relationships: a user owns accounts, accounts hold securities, securities have tax lots, tax lots generate tax events, transactions map to budget categories, and all of this rolls up into net worth. The normalized model makes each of these relationships a queryable first-class entity: cost basis calculations join `tax_lots` to `securities`, portfolio analytics join `holdings` to market data, and budget tracking joins `transactions` to `categories`. This enables queries that cross entity boundaries — "what is my total unrealised gain on all positions across all accounts?" — with standard indexed joins.

The trade-off is schema complexity — 16 tables — but the payoff is full referential integrity, column-level constraints on financial data (BIGINT cents for amounts, CHECK constraints on account types), and the ability to answer any cross-entity financial query with standard SQL.

**Best for:** Teams building a production platform where data integrity, complex cross-account analytics (tax-loss harvesting, fee analysis, rebalancing), and regulatory compliance are priorities.

**Trade-offs:**
- Pro: Full referential integrity across all financial entities
- Pro: Column-level constraints prevent invalid financial state
- Pro: Cost basis and tax calculations via standard SQL joins
- Pro: Cross-account portfolio analytics via indexed joins
- Con: 16 tables — more complex schema to manage
- Con: Schema migrations required for new asset types or account properties
- Con: Net worth computation requires joining multiple tables
- Con: High transaction volume requires careful partitioning

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FDX API | `linked_accounts.fdx_account_id` for FDX-compliant connections |
| PSD2 / NextGenPSD2 | `linked_accounts.aggregator` supports PSD2-compliant providers |
| FAPI 2.0 | Security profile for open banking token management |
| OAuth 2.0 / OIDC | User auth; open banking consent flows |
| ISIN (ISO 6166) | `securities.isin` for global security identification |
| CUSIP | `securities.cusip` for US/Canadian security identification |
| FIGI (ISO 4914) | `securities.figi` for open instrument mapping |
| ISO 20022 | Transaction message format alignment |
| OFX | CSV/OFX import compatibility tracked in `linked_accounts` |
| OpenAPI 3.1 | REST API documented in OpenAPI |
| GDPR | Data retention and consent on `users` |
| CCPA/CPRA | Consumer privacy rights compliance |
| CFPB 1033 | FDX-aligned account access |
| ISO 27001 / SOC 2 | Audit logging, encryption, access control |
| MCP | `users.mcp_enabled` exposes finance data as AI context |

---

## Users & Households

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    password_hash   TEXT NOT NULL,
    default_currency TEXT NOT NULL DEFAULT 'USD',
    locale          TEXT NOT NULL DEFAULT 'en-US',
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    tax_filing_status TEXT CHECK (tax_filing_status IN (
                        'single','married_filing_jointly','married_filing_separately',
                        'head_of_household','qualifying_widow'
                    )),
    tax_bracket_pct REAL,
    state_code      TEXT,
    country_code    TEXT NOT NULL DEFAULT 'US',
    mcp_enabled     BOOLEAN NOT NULL DEFAULT FALSE,
    data_retention_days INTEGER,
    gdpr_consent_at TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE households (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    owner_id        UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE household_members (
    household_id    UUID NOT NULL REFERENCES households(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    role            TEXT NOT NULL CHECK (role IN (
                        'owner','editor','viewer'
                    )) DEFAULT 'viewer',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (household_id, user_id)
);
CREATE INDEX idx_hh_members_user ON household_members (user_id);
```

---

## Linked Accounts

```sql
CREATE TABLE linked_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    household_id    UUID REFERENCES households(id),
    account_name    TEXT NOT NULL,
    account_type    TEXT NOT NULL CHECK (account_type IN (
                        'checking','savings','credit_card','loan','mortgage',
                        'brokerage','ira_traditional','ira_roth','401k','403b',
                        'hsa','529','crypto_exchange','crypto_wallet',
                        'real_estate','vehicle','precious_metal','private_equity',
                        'other'
                    )),
    institution_name TEXT,
    aggregator      TEXT CHECK (aggregator IN (
                        'plaid','mx','truelayer','nordigen','salt_edge',
                        'manual','csv_import','ofx_import'
                    )),
    aggregator_item_id TEXT,
    fdx_account_id  TEXT,
    currency        TEXT NOT NULL DEFAULT 'USD',
    current_balance_cents BIGINT NOT NULL DEFAULT 0,
    available_balance_cents BIGINT,
    credit_limit_cents BIGINT,
    interest_rate   REAL,
    is_asset        BOOLEAN NOT NULL DEFAULT TRUE,
    is_investment   BOOLEAN NOT NULL DEFAULT FALSE,
    mask            TEXT,
    last_sync_at    TIMESTAMPTZ,
    sync_status     TEXT NOT NULL CHECK (sync_status IN (
                        'connected','syncing','error','disconnected','manual'
                    )) DEFAULT 'manual',
    sync_error      TEXT,
    metadata_json   JSONB NOT NULL DEFAULT '{}',
    is_hidden       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_accounts_user ON linked_accounts (user_id);
CREATE INDEX idx_accounts_household ON linked_accounts (household_id)
    WHERE household_id IS NOT NULL;
CREATE INDEX idx_accounts_type ON linked_accounts (account_type);
CREATE INDEX idx_accounts_aggregator ON linked_accounts (aggregator, aggregator_item_id)
    WHERE aggregator_item_id IS NOT NULL;
```

---

## Transactions & Budgeting

```sql
CREATE TABLE categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    name            TEXT NOT NULL,
    parent_id       UUID REFERENCES categories(id),
    icon            TEXT,
    colour_hex      TEXT,
    is_income       BOOLEAN NOT NULL DEFAULT FALSE,
    is_transfer     BOOLEAN NOT NULL DEFAULT FALSE,
    is_system       BOOLEAN NOT NULL DEFAULT FALSE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, name)
);
CREATE INDEX idx_categories_user ON categories (user_id);
CREATE INDEX idx_categories_parent ON categories (parent_id)
    WHERE parent_id IS NOT NULL;

CREATE TABLE transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    account_id      UUID NOT NULL REFERENCES linked_accounts(id),
    category_id     UUID REFERENCES categories(id),
    transaction_date DATE NOT NULL,
    posted_date     DATE,
    description     TEXT NOT NULL,
    merchant_name   TEXT,
    amount_cents    BIGINT NOT NULL,
    currency        TEXT NOT NULL DEFAULT 'USD',
    is_income       BOOLEAN NOT NULL DEFAULT FALSE,
    is_transfer     BOOLEAN NOT NULL DEFAULT FALSE,
    is_recurring    BOOLEAN NOT NULL DEFAULT FALSE,
    recurring_group_id UUID,
    is_excluded     BOOLEAN NOT NULL DEFAULT FALSE,
    source          TEXT NOT NULL CHECK (source IN (
                        'aggregator','manual','csv_import','ofx_import'
                    )) DEFAULT 'aggregator',
    external_id     TEXT,
    notes           TEXT,
    tags            TEXT[] NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (transaction_date);

CREATE INDEX idx_txn_user ON transactions (user_id);
CREATE INDEX idx_txn_account ON transactions (account_id);
CREATE INDEX idx_txn_category ON transactions (category_id)
    WHERE category_id IS NOT NULL;
CREATE INDEX idx_txn_date ON transactions (transaction_date);
CREATE INDEX idx_txn_merchant ON transactions (merchant_name)
    WHERE merchant_name IS NOT NULL;
CREATE INDEX idx_txn_recurring ON transactions (recurring_group_id)
    WHERE is_recurring = TRUE;
CREATE INDEX idx_txn_tags ON transactions USING GIN (tags);

CREATE TABLE budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    category_id     UUID NOT NULL REFERENCES categories(id),
    period_type     TEXT NOT NULL CHECK (period_type IN (
                        'monthly','weekly','yearly'
                    )) DEFAULT 'monthly',
    amount_cents    BIGINT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_budgets_user ON budgets (user_id);
CREATE INDEX idx_budgets_category ON budgets (category_id);
```

---

## Securities & Holdings

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
    sector          TEXT,
    asset_class     TEXT CHECK (asset_class IN (
                        'us_equity','intl_equity','emerging_markets',
                        'fixed_income','real_estate','commodity',
                        'crypto','cash','alternative','other'
                    )),
    expense_ratio   REAL,
    last_price_cents BIGINT,
    last_price_at   TIMESTAMPTZ,
    metadata_json   JSONB NOT NULL DEFAULT '{}',
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

CREATE TABLE holdings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id      UUID NOT NULL REFERENCES linked_accounts(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    security_id     UUID NOT NULL REFERENCES securities(id),
    quantity        NUMERIC(20,8) NOT NULL,
    cost_basis_cents BIGINT NOT NULL DEFAULT 0,
    market_value_cents BIGINT NOT NULL DEFAULT 0,
    unrealised_gain_cents BIGINT NOT NULL DEFAULT 0,
    weight_pct      REAL,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_holdings_account ON holdings (account_id);
CREATE INDEX idx_holdings_user ON holdings (user_id);
CREATE INDEX idx_holdings_security ON holdings (security_id);
```

---

## Tax Lots & Tax Events

```sql
CREATE TABLE tax_lots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    holding_id      UUID NOT NULL REFERENCES holdings(id),
    account_id      UUID NOT NULL REFERENCES linked_accounts(id),
    security_id     UUID NOT NULL REFERENCES securities(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    acquired_date   DATE NOT NULL,
    quantity        NUMERIC(20,8) NOT NULL,
    cost_per_unit_cents BIGINT NOT NULL,
    total_cost_cents BIGINT NOT NULL,
    current_value_cents BIGINT,
    unrealised_gain_cents BIGINT,
    is_long_term    BOOLEAN NOT NULL DEFAULT FALSE,
    accounting_method TEXT NOT NULL CHECK (accounting_method IN (
                        'fifo','lifo','hifo','specific_id'
                    )) DEFAULT 'fifo',
    source          TEXT NOT NULL CHECK (source IN (
                        'purchase','transfer','dividend_reinvest',
                        'split','gift','inheritance'
                    )) DEFAULT 'purchase',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_lots_holding ON tax_lots (holding_id);
CREATE INDEX idx_lots_account ON tax_lots (account_id);
CREATE INDEX idx_lots_security ON tax_lots (security_id);
CREATE INDEX idx_lots_user ON tax_lots (user_id);
CREATE INDEX idx_lots_acquired ON tax_lots (acquired_date);

CREATE TABLE tax_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    account_id      UUID NOT NULL REFERENCES linked_accounts(id),
    security_id     UUID NOT NULL REFERENCES securities(id),
    tax_lot_id      UUID REFERENCES tax_lots(id),
    event_type      TEXT NOT NULL CHECK (event_type IN (
                        'realised_gain','realised_loss','dividend',
                        'interest','wash_sale','roth_conversion',
                        'distribution','contribution'
                    )),
    event_date      DATE NOT NULL,
    quantity        NUMERIC(20,8),
    proceeds_cents  BIGINT,
    cost_basis_cents BIGINT,
    gain_loss_cents BIGINT,
    is_long_term    BOOLEAN,
    is_wash_sale    BOOLEAN NOT NULL DEFAULT FALSE,
    wash_sale_disallowed_cents BIGINT,
    tax_year        INTEGER NOT NULL,
    form_8949_box   TEXT CHECK (form_8949_box IN ('A','B','C','D','E','F')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (event_date);

CREATE INDEX idx_tax_events_user ON tax_events (user_id);
CREATE INDEX idx_tax_events_account ON tax_events (account_id);
CREATE INDEX idx_tax_events_security ON tax_events (security_id);
CREATE INDEX idx_tax_events_year ON tax_events (tax_year);
CREATE INDEX idx_tax_events_type ON tax_events (event_type);
CREATE INDEX idx_tax_events_wash ON tax_events (is_wash_sale)
    WHERE is_wash_sale = TRUE;
```

---

## Net Worth, Goals & Documents

```sql
CREATE TABLE net_worth_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    snapshot_date   DATE NOT NULL,
    total_assets_cents BIGINT NOT NULL,
    total_liabilities_cents BIGINT NOT NULL,
    net_worth_cents BIGINT NOT NULL,
    breakdown_json  JSONB NOT NULL DEFAULT '{}',
    -- {"checking": 50000, "savings": 200000, "brokerage": 1500000,
    --  "retirement": 800000, "real_estate": 4500000, "crypto": 150000,
    --  "credit_cards": -45000, "mortgage": -3200000, "loans": -120000}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, snapshot_date)
) PARTITION BY RANGE (snapshot_date);

CREATE INDEX idx_nw_user ON net_worth_snapshots (user_id, snapshot_date);

CREATE TABLE goals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    name            TEXT NOT NULL,
    goal_type       TEXT NOT NULL CHECK (goal_type IN (
                        'retirement','home_purchase','education','emergency_fund',
                        'debt_payoff','vacation','custom'
                    )),
    target_amount_cents BIGINT NOT NULL,
    current_amount_cents BIGINT NOT NULL DEFAULT 0,
    target_date     DATE,
    monthly_contribution_cents BIGINT,
    linked_account_ids UUID[],
    projection_json JSONB,
    -- {"monte_carlo_success_pct": 82, "projected_date": "2045-03-15",
    --  "scenarios": [{"name": "optimistic", "return_pct": 10, "end_value": ...}]}
    is_achieved     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_goals_user ON goals (user_id);

CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    account_id      UUID REFERENCES linked_accounts(id),
    document_type   TEXT NOT NULL CHECK (document_type IN (
                        'brokerage_statement','tax_return','w2','1099',
                        'bank_statement','insurance_policy','will','trust',
                        'deed','other'
                    )),
    filename        TEXT NOT NULL,
    storage_path    TEXT NOT NULL,
    file_size_bytes BIGINT,
    content_type    TEXT,
    tax_year        INTEGER,
    extracted_data_json JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_docs_user ON documents (user_id);
CREATE INDEX idx_docs_account ON documents (account_id)
    WHERE account_id IS NOT NULL;
CREATE INDEX idx_docs_type ON documents (document_type);

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

### Net worth trend over 12 months

```sql
SELECT snapshot_date, net_worth_cents / 100.0 AS net_worth,
       total_assets_cents / 100.0 AS total_assets,
       total_liabilities_cents / 100.0 AS total_liabilities
FROM net_worth_snapshots
WHERE user_id = 'user-uuid'
  AND snapshot_date >= CURRENT_DATE - INTERVAL '12 months'
ORDER BY snapshot_date;
```

### Tax-loss harvesting opportunities

```sql
SELECT s.ticker, s.name, tl.unrealised_gain_cents / 100.0 AS unrealised_gain,
       la.account_name, tl.acquired_date,
       CASE WHEN tl.is_long_term THEN 'long_term' ELSE 'short_term' END AS term
FROM tax_lots tl
JOIN holdings h ON h.id = tl.holding_id
JOIN securities s ON s.id = tl.security_id
JOIN linked_accounts la ON la.id = tl.account_id
WHERE tl.user_id = 'user-uuid'
  AND tl.unrealised_gain_cents < 0
  AND NOT EXISTS (
      SELECT 1 FROM tax_events te
      WHERE te.security_id = tl.security_id
        AND te.user_id = tl.user_id
        AND te.event_type = 'realised_loss'
        AND te.event_date >= CURRENT_DATE - INTERVAL '30 days'
  )
ORDER BY tl.unrealised_gain_cents ASC
LIMIT 20;
```

### Fee drag analysis across investment accounts

```sql
SELECT la.account_name, s.ticker, s.name,
       s.expense_ratio,
       h.market_value_cents / 100.0 AS market_value,
       ROUND((h.market_value_cents * s.expense_ratio / 100.0)::NUMERIC, 2) AS annual_fee
FROM holdings h
JOIN securities s ON s.id = h.security_id
JOIN linked_accounts la ON la.id = h.account_id
WHERE h.user_id = 'user-uuid'
  AND s.expense_ratio IS NOT NULL
  AND s.expense_ratio > 0
ORDER BY annual_fee DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users | 3 | users, households, household_members |
| Accounts | 1 | linked_accounts |
| Transactions | 3 | transactions (partitioned), categories, budgets |
| Investments | 2 | securities, holdings |
| Tax | 2 | tax_lots, tax_events (partitioned) |
| Planning | 3 | net_worth_snapshots (partitioned), goals, documents |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **15** | |

---

## Key Design Decisions

1. **`tax_lots` as a separate table** — cost basis accounting (FIFO, LIFO, HIFO, specific ID) requires tracking individual purchase lots with acquisition dates and per-unit costs; this enables IRS Form 8949 reporting and wash sale detection via SQL joins across lots and tax events.

2. **`securities` as a reference table** — ISIN, CUSIP, and FIGI identifiers are stored once per security, shared across all users' holdings; this enables cross-account analytics and deduplication when multiple accounts hold the same security.

3. **`tax_events` partitioned by event_date** — realised gains, losses, dividends, and wash sales are append-heavy and frequently queried by tax year; time-range partitioning keeps tax-year queries fast.

4. **`transactions` partitioned by transaction_date** — high-volume users with many linked accounts generate thousands of transactions per month; partitioning enables efficient date-range queries for budgeting and spending analysis.

5. **`amount_cents` as BIGINT** — all monetary values are stored in the smallest currency unit (cents) to avoid floating-point precision issues in financial calculations, following the Stripe convention.

6. **`households` and `household_members`** — household sharing with role-based access (owner, editor, viewer) enables collaborative financial management without credential sharing, an underserved feature in the market.

7. **`net_worth_snapshots` with `breakdown_json`** — daily snapshots store both the total net worth and a per-category breakdown, enabling trend charts and allocation analysis without recomputing from account balances.

8. **`goals.projection_json`** — Monte Carlo simulation results and scenario projections are stored as JSONB because the structure varies by goal type and the number of scenarios; this avoids a separate projections table.

9. **`linked_accounts.aggregator`** — tracking which aggregator (Plaid, MX, TrueLayer, manual) connected each account enables per-provider error handling, sync scheduling, and migration when CFPB 1033 / FDX APIs become available.

10. **`tax_events.form_8949_box`** — storing the IRS Form 8949 reporting box (A through F) per tax event enables direct export to tax filing software without recomputation.
