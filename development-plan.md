# Personal Finance & Investment Tracker — Phased Development Plan

> Project: 356-personal-finance-investment-tracker · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three `data-model-suggestion-*.md` files. It adopts **Data Model Suggestion 1 (Entity-Centric Normalized Relational)** as the canonical schema: the project's core differentiators — cross-account tax-loss harvesting, fee drag analysis, cost-basis tracking, and household analytics — all depend on indexed SQL joins across first-class entities (`tax_lots`, `holdings`, `securities`, `tax_events`), which the normalized model serves directly. JSONB columns from Suggestion 2 are retained selectively (`goals.projection_json`, `net_worth_snapshots.breakdown_json`, `*.metadata_json`) where structure genuinely varies.

---

## Core Requirements (synthesised)

- **What it does**: An open, AI-native platform unifying net worth, investment performance analytics, budgeting, and tax optimisation across all of a user's accounts (bank, brokerage, retirement, crypto, alternative assets) in a single view.
- **Who uses it**: Millennials and high-income professionals with multi-asset portfolios; privacy-conscious users wanting self-hosting; DIY investors needing cross-account tax-loss harvesting.
- **Key differentiators**: (1) holistic tax planning timeline; (2) cross-account TLH for externally held accounts; (3) privacy-first self-hosted deployment; (4) AI financial coach grounded in real portfolio data; (5) MCP server exposing finance data to AI assistants — no incumbent offers this.
- **Deployment model**: Cloud-hosted default + self-hosted/local-first Docker option. API-first with a web dashboard.
- **Integration surface**: Plaid, MX, TrueLayer/GoCardless (aggregation); Alpha Vantage / Polygon.io (market data); CoinGecko (crypto); OpenFIGI/CUSIP (security ID mapping); an LLM provider via the Anthropic API; an MCP server.
- **Standards compliance**: FDX-aligned account model, OAuth 2.0 + PKCE + FAPI 2.0 for aggregation consent, ISIN/CUSIP/FIGI security identifiers, OpenAPI 3.1 for the API surface, SARIF-equivalent structured output not relevant, IRS Pub 550 / Form 8949 for tax math, GDPR/CCPA for privacy, MCP for AI integration, TLS 1.3 minimum.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | The product is analytics- and AI-heavy (Monte Carlo, time-weighted return, LLM Q&A, document parsing). Python has the strongest numerical/finance ecosystem (`numpy`, `pandas`, `scipy`, `numpy-financial`) and first-class Plaid/MX/Anthropic SDKs. |
| API framework | FastAPI | Async-native (essential for fan-out aggregator/market-data calls), generates OpenAPI 3.1 automatically (a stated standard), and integrates Pydantic v2 for request/response validation. |
| Data validation | Pydantic v2 | Enforces JSON Schema 2020-12-aligned models for accounts, transactions, securities, and snapshots; shared between API layer and domain layer. |
| ORM / DB access | SQLAlchemy 2.0 (async) + Alembic | Mature async ORM matching the normalized schema; Alembic gives versioned migrations required as account types evolve. |
| Database | PostgreSQL 16 | The chosen data model needs CHECK constraints, partial indexes, GIN indexes on JSONB/arrays, declarative partitioning (`transactions`, `tax_events`, `net_worth_snapshots`, `audit_log`), and `NUMERIC(20,8)` precision. SQLite is offered only as a reduced self-hosted single-user mode. |
| Task queue | Celery + Redis | Aggregator syncs, market-data refresh, snapshot computation, and LLM document parsing are async, scheduled, and retryable. Redis doubles as cache and Celery broker/result backend. |
| Scheduler | Celery Beat | Daily balance sync, nightly net-worth snapshot, periodic performance/fee recompute. |
| Frontend | Next.js 15 (App Router) + TypeScript + shadcn/ui + Tailwind + Recharts | Dashboard-first product (net worth, allocation, trend charts). Server Components for fast initial dashboard load; Recharts for time-series/allocation visualisations. |
| LLM provider | Anthropic API (Claude) via `anthropic` SDK, with prompt caching | AI coach, categorisation assist, document parsing, anomaly explanation. Prompt caching reduces cost on the large portfolio-context system prompt. |
| MCP server | Python MCP SDK (`mcp`) | Exposes balances, net worth, allocation, transactions, and goals as MCP resources/tools — a key differentiator from `standards.md`. |
| Money type | `int` cents (BIGINT) | Avoids float precision errors per the schema's Stripe convention. A `Money` value object wraps cents + currency. |
| Market data | Polygon.io (equities/ETF) + CoinGecko (crypto), behind a `PriceProvider` interface | Polygon free tier gives delayed OHLCV/dividends/splits; CoinGecko free tier covers crypto. Interface allows swapping to Alpha Vantage. |
| Security ID mapping | OpenFIGI (free) primary, CUSIP optional | Royalty-free FIGI↔ISIN↔ticker mapping; CUSIP requires a commercial licence so it is optional. |
| Auth | OAuth 2.0 / OIDC for user login (Authlib) + JWT sessions; aggregator consent via Plaid Link / OAuth+PKCE | FAPI 2.0 alignment for the open-banking surface; standard bearer JWT for the app API. |
| Secrets / token encryption | `cryptography` (Fernet/AES-GCM) with envelope encryption | Aggregator access tokens and document contents are encrypted at rest; supports per-user crypto-shredding for GDPR erasure. |
| Testing | pytest + pytest-asyncio + httpx.AsyncClient + factory_boy + testcontainers-postgres | Unit + integration with a real ephemeral Postgres; VCR.py / responses for mocking external HTTP. |
| Code quality | ruff (lint+format), mypy (strict), bandit (security) | Standard modern Python toolchain; bandit matters for a finance app. |
| Package manager | uv | Fast, reproducible locking for the Python backend. |
| Containerisation | Docker + docker-compose | Self-hosted deployment is a stated differentiator; compose wires api, worker, beat, postgres, redis, web. |
| Key libraries | `plaid-python`, `numpy-financial` (IRR/XIRR), `numpy`/`scipy` (Monte Carlo, Sharpe/beta), `pandas` (CSV/OFX import), `ofxparse`, `anthropic`, `mcp`, `pdfplumber`/`pypdf` (document OCR pre-pass) | Domain-specific dependencies tied to each phase. |

### Project Structure

```
personal-finance-tracker/
├── pyproject.toml                # uv-managed; deps, ruff, mypy, pytest config
├── uv.lock
├── Dockerfile                    # backend image (api + worker share it)
├── docker-compose.yml            # api, worker, beat, postgres, redis, web
├── docker-compose.selfhost.yml   # single-user self-hosted overlay
├── alembic.ini
├── .env.example
├── README.md
├── migrations/                   # Alembic revisions
│   └── versions/
├── src/
│   └── pftracker/
│       ├── __init__.py
│       ├── main.py               # FastAPI app factory, router registration
│       ├── config.py             # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py        # async engine/session
│       │   ├── base.py           # declarative base, naming conventions
│       │   └── models/           # SQLAlchemy ORM models (one file per group)
│       │       ├── user.py
│       │       ├── account.py
│       │       ├── transaction.py
│       │       ├── security.py
│       │       ├── tax.py
│       │       ├── planning.py
│       │       └── audit.py
│       ├── schemas/              # Pydantic request/response models
│       ├── domain/               # pure business logic (no I/O)
│       │   ├── money.py          # Money value object
│       │   ├── cost_basis.py     # FIFO/LIFO/HIFO/specific-id lot matching
│       │   ├── wash_sale.py      # §1091 detection
│       │   ├── performance.py    # TWR, IRR, Sharpe, beta
│       │   ├── allocation.py     # asset allocation rollup
│       │   ├── fees.py           # fee drag projection
│       │   ├── monte_carlo.py    # retirement projection
│       │   └── categorisation.py # rules + ML-assist interface
│       ├── api/
│       │   └── routes/           # FastAPI routers per resource
│       ├── services/             # orchestration: combine domain + repos + I/O
│       ├── integrations/
│       │   ├── aggregators/      # Plaid/MX/TrueLayer adapters (Aggregator protocol)
│       │   ├── market_data/      # Polygon/CoinGecko adapters (PriceProvider protocol)
│       │   ├── security_ids/     # OpenFIGI/CUSIP mapping
│       │   └── llm/              # Anthropic client wrapper, prompt templates
│       ├── tasks/                # Celery tasks + beat schedule
│       ├── mcp/                  # MCP server exposing finance resources/tools
│       ├── auth/                 # OIDC login, JWT, RBAC, token encryption
│       └── ingest/               # CSV/OFX importers
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/                 # sample Plaid payloads, OFX/CSV files, statements
└── web/                          # Next.js frontend
    ├── package.json
    ├── app/
    ├── components/
    └── lib/
```

The structure is grouped by concern (domain, services, integrations, api) so each phase adds modules without restructuring.

---

## Phase 1: Foundation — Project Skeleton, Config, Database, Auth

### Purpose
Establish the runnable backbone: a FastAPI app, a migrated PostgreSQL schema for the core entities, configuration, user authentication with RBAC, and the test harness. After this phase a user can register, log in, and the database is ready for accounts and transactions. Everything else builds on this.

### Tasks

#### 1.1 — Project scaffolding & configuration

**What**: Initialise the `uv` project, FastAPI app factory, settings, and CI-quality tooling.

**Design**:
- `pyproject.toml` declares dependencies and tool config (ruff line-length 100, mypy strict, pytest asyncio mode auto).
- `config.py` using Pydantic `BaseSettings`:
```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    jwt_secret: str
    jwt_alg: str = "HS256"
    access_token_ttl_min: int = 30
    encryption_key: str                 # base64 32-byte key for Fernet/AES-GCM
    anthropic_api_key: str | None = None
    plaid_client_id: str | None = None
    plaid_secret: str | None = None
    plaid_env: str = "sandbox"
    polygon_api_key: str | None = None
    coingecko_api_key: str | None = None
    deployment_mode: Literal["cloud", "selfhost"] = "cloud"
    model_config = SettingsConfigDict(env_file=".env")
```
- `main.py` exposes `create_app() -> FastAPI`, registers routers, adds middleware (request ID, CORS, TLS-enforced headers `Strict-Transport-Security`), and a `/healthz` endpoint.
- Error handling: a global exception handler maps domain exceptions to RFC 7231 status codes and a consistent JSON error envelope `{ "error": { "code": str, "message": str, "field": str | null } }`.

**Testing**:
- `Unit: Settings loads from env vars → correct typed values, defaults applied.`
- `Unit: missing required env (jwt_secret) → ValidationError naming the field.`
- `Integration: GET /healthz → 200 {"status":"ok"}.`
- `Integration: unhandled domain exception → JSON error envelope with mapped status.`

#### 1.2 — Database models & migrations (core entities)

**What**: Implement SQLAlchemy models and the first Alembic migration for `users`, `households`, `household_members`, `categories`, `linked_accounts`, plus partitioning helpers.

**Design**: Implement the DDL from Data Model Suggestion 1 for these tables verbatim (UUID PKs, `*_cents BIGINT`, CHECK constraints, indexes). Key models:
```python
class User(Base):
    __tablename__ = "users"
    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    email: Mapped[str] = mapped_column(unique=True)
    display_name: Mapped[str]
    password_hash: Mapped[str]
    default_currency: Mapped[str] = mapped_column(default="USD")
    tax_filing_status: Mapped[str | None]   # CHECK enforced at DB
    tax_bracket_pct: Mapped[float | None]
    state_code: Mapped[str | None]
    country_code: Mapped[str] = mapped_column(default="US")
    mcp_enabled: Mapped[bool] = mapped_column(default=False)
    gdpr_consent_at: Mapped[datetime | None]
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```
- `LinkedAccount` includes the full `account_type`/`aggregator`/`sync_status` CHECK enums, `current_balance_cents`, `is_asset`, `is_investment`, `metadata_json` (JSONB).
- Partition management: `transactions`, `tax_events`, `net_worth_snapshots`, `audit_log` are `PARTITION BY RANGE`. Provide a `create_monthly_partition(table, year, month)` helper and a Celery beat task (added in Phase 6) to pre-create next month's partitions. For Phase 1, create the current and next partition in the migration.
- A naming convention on `Base.metadata` ensures deterministic constraint names for Alembic autogenerate.

**Testing**:
- `Integration (testcontainers PG): run alembic upgrade head → all tables, indexes, partitions exist (introspect pg_catalog).`
- `Integration: insert linked_account with invalid account_type → IntegrityError (CHECK).`
- `Integration: insert two users with same email → IntegrityError (unique).`
- `Unit: create_monthly_partition generates correct CREATE TABLE ... PARTITION OF SQL for a given month.`

#### 1.3 — Authentication, sessions & RBAC

**What**: User registration/login with hashed passwords, JWT issuance, OIDC-ready login, and household role-based access control.

**Design**:
- Password hashing with `argon2` (`argon2-cffi`).
- Endpoints:
  - `POST /auth/register` → `{email, password, display_name}` → 201 `{user_id}`.
  - `POST /auth/login` → `{email, password}` → 200 `{access_token, token_type:"bearer", expires_in}`.
  - `GET /auth/me` → current user profile (requires bearer).
- JWT claims: `sub` (user id), `exp`, `households` (list of `{id, role}`).
- `require_user` FastAPI dependency decodes/validates JWT. `require_household_role(min_role)` dependency enforces RBAC (`owner > editor > viewer`) for household-scoped resources.
- Token encryption utility (`auth/crypto.py`): `encrypt(plaintext) -> str`, `decrypt(token) -> str` using AES-GCM with `settings.encryption_key`; used later for aggregator tokens. Per-user key derivation supports crypto-shredding (delete the user's key → data unrecoverable, satisfying GDPR erasure).
- OIDC login (`auth/oidc.py`) using Authlib is scaffolded with PKCE; concrete providers configured via env. FAPI 2.0 alignment documented (PAR/PKCE) for the aggregator flow in Phase 3.

**Testing**:
- `Unit: argon2 hash/verify round-trip; wrong password → False.`
- `Unit: encrypt then decrypt → original plaintext; tampered ciphertext → InvalidTag error.`
- `Integration: register → login → GET /auth/me returns the user, 200.`
- `Integration: GET /auth/me without token → 401; expired token → 401.`
- `Integration: viewer attempts editor-only household action → 403.`

### Definition of Done
Tasks 1.1–1.3 implemented; `alembic upgrade head` succeeds on a fresh PG; ruff/mypy/bandit clean; `docker compose up postgres redis api` boots; register→login→me works end-to-end.

---

## Phase 2: Accounts, Transactions & Categorisation Core

### Purpose
Build the manual/CSV-driven heart of the ledger before any external integration: create accounts, record transactions, and categorise them with a learning rule engine. This lets the product function fully in self-hosted/manual mode and provides the substrate that aggregators (Phase 3) merely populate.

### Tasks

#### 2.1 — Account & transaction CRUD

**What**: REST endpoints to manage `linked_accounts` and `transactions` manually.

**Design**: Implement `transactions`, `categories`, and `budgets` tables from Suggestion 1 (Alembic migration extending Phase 1).
- Endpoints (OpenAPI 3.1 auto-documented):
  - `POST /accounts`, `GET /accounts`, `GET /accounts/{id}`, `PATCH /accounts/{id}`, `DELETE /accounts/{id}` (soft via `is_hidden`).
  - `POST /accounts/{id}/transactions`, `GET /accounts/{id}/transactions?from=&to=&category_id=&page=`, `PATCH /transactions/{id}`, `DELETE /transactions/{id}`.
- Pagination via `Link` header (RFC 8288) + `?cursor=`/`?limit=` (default 50, max 200).
- Pydantic schemas use `amount_cents: int` and a `currency: str` validated against ISO 4217. A `Money` value object (`domain/money.py`) handles arithmetic; never use float.
- Updating an account balance writes an `audit_log` row (actor_type=`user`).

**Testing**:
- `Unit: Money(1050,"USD") + Money(295,"USD") == Money(1345,"USD"); mixed currency → raises.`
- `Integration: POST /accounts then GET returns it scoped to the user only.`
- `Integration: user A cannot GET user B's account → 404 (not 403, to avoid existence leak).`
- `Integration: list transactions paginated → correct Link: rel="next" header.`
- `Integration: invalid currency code → 422 with field name.`

#### 2.2 — Category model & rule-based auto-categorisation

**What**: A deterministic categorisation engine with user-correctable rules and category learning.

**Design** (`domain/categorisation.py`):
```python
@dataclass(frozen=True)
class CategorySuggestion:
    category_id: UUID | None
    confidence: float           # 0..1
    source: Literal["rule", "merchant_memory", "llm", "default"]

class Categoriser:
    def suggest(self, txn: TransactionInput, rules: list[CategoryRule],
                merchant_memory: dict[str, UUID]) -> CategorySuggestion: ...
```
- Resolution order: (1) exact `merchant_memory` hit (a normalised merchant → category map built from past user corrections) → confidence 1.0; (2) first matching `CategoryRule` (regex/substring on description/merchant) → 0.9; (3) LLM-assist (Phase 7, interface stubbed here, returns `None` if disabled); (4) `default` uncategorised.
- "Learning": when a user PATCHes a transaction's `category_id`, the service upserts `merchant_memory[normalise(merchant_name)] = category_id` (stored in a `merchant_category_map` table keyed by user_id). Future transactions from the same merchant auto-apply.
- System categories (Housing, Groceries, Transport, etc.) seeded per user on registration via a fixture.

**Testing**:
- `Unit: merchant_memory hit overrides matching rule (confidence 1.0).`
- `Unit: no rule, no memory → default uncategorised, confidence 0.`
- `Integration: PATCH txn category for "WHOLEFDS" → next imported "WHOLEFDS" txn auto-categorised.`
- `Unit: normalise("WHOLEFDS #123 SF") == normalise("WHOLEFDS #456 NY")` (strips store numbers/locations).

#### 2.3 — CSV & OFX import

**What**: Import transactions and balances from CSV and OFX/QFX files for institutions without API coverage.

**Design** (`ingest/`):
- `POST /accounts/{id}/import` multipart upload; `format` inferred from extension or explicit query param (`csv`|`ofx`).
- CSV: a `ColumnMapping` Pydantic model (`date_col`, `amount_col`, `description_col`, optional `category_col`); a preview endpoint `POST /accounts/{id}/import/preview` returns the first 10 parsed rows + detected mapping for user confirmation.
- OFX parsed with `ofxparse`; maps `<STMTTRN>` to transactions, dedupes on `external_id` = OFX `FITID`.
- Idempotency: dedupe on `(account_id, external_id)` or, when absent, a hash of `(date, amount_cents, description)`.
- Each imported transaction runs through `Categoriser` (2.2).

**Testing**:
- `Fixture: tests/fixtures/sample_chase.csv → N transactions created with correct cents conversion (e.g. "-$45.20" → -4520).`
- `Fixture: sample.ofx → transactions deduped on FITID; re-import imports 0 new rows.`
- `Unit: amount parsing handles parentheses negatives, thousands separators, leading currency symbol.`
- `Integration: import preview returns detected column mapping for an unlabelled CSV.`

### Definition of Done
Manual accounts/transactions fully usable; categorisation learns from corrections; CSV/OFX import idempotent; OpenAPI spec shows all new endpoints; tests green; lint/type/security clean.

---

## Phase 3: Account Aggregation (Plaid primary)

### Purpose
Connect real financial institutions so accounts, balances, transactions, and investment holdings populate automatically. This is the table-stakes feature that makes the product usable at scale. Built behind an `Aggregator` protocol so MX/TrueLayer can be added without touching callers.

### Tasks

#### 3.1 — Aggregator abstraction & Plaid adapter

**What**: A provider-agnostic aggregation interface with a concrete Plaid implementation.

**Design** (`integrations/aggregators/base.py`):
```python
class Aggregator(Protocol):
    name: str
    async def create_link_token(self, user_id: UUID) -> str: ...
    async def exchange_public_token(self, public_token: str) -> AggregatorItem: ...
    async def fetch_accounts(self, item: AggregatorItem) -> list[NormalisedAccount]: ...
    async def fetch_transactions(self, item: AggregatorItem,
                                 cursor: str | None) -> TransactionSync: ...
    async def fetch_holdings(self, item: AggregatorItem) -> list[NormalisedHolding]: ...

@dataclass
class AggregatorItem:
    provider: str
    item_id: str
    access_token_encrypted: str   # stored encrypted via auth/crypto
@dataclass
class NormalisedAccount:
    external_id: str; name: str; account_type: str; institution: str
    balance_cents: int; currency: str; mask: str | None; is_investment: bool
@dataclass
class NormalisedHolding:
    external_account_id: str; ticker: str | None; isin: str | None
    cusip: str | None; quantity: Decimal; cost_basis_cents: int
    market_value_cents: int; security_type: str
```
- Plaid adapter wraps `plaid-python` using `/link/token/create`, `/item/public_token/exchange`, `/accounts/balance/get`, `/transactions/sync` (cursor-based), `/investments/holdings/get`. Access tokens encrypted at rest; OAuth uses Plaid Link's hosted flow (PKCE, FAPI-aligned).
- `account_type` mapping table translates Plaid subtypes → the schema's CHECK enum.

**Testing**:
- `Integration (mocked Plaid via responses): exchange_public_token → AggregatorItem with encrypted token stored.`
- `Unit: Plaid subtype "401k" → "401k"; "cd"→"savings"; unknown→"other".`
- `Integration (mocked): /transactions/sync cursor advances; second call with cursor returns only added/modified.`
- `Integration (Plaid sandbox, marked @pytest.mark.real, skipped in CI): real link → accounts fetched.`

#### 3.2 — Link & sync orchestration

**What**: Endpoints and Celery tasks to connect an item and sync its data into the ledger.

**Design**:
- `POST /aggregator/link-token` → returns link token for the frontend Plaid Link widget.
- `POST /aggregator/exchange` `{public_token}` → exchanges, persists `linked_accounts` rows (one per `NormalisedAccount`), enqueues initial sync, returns account ids.
- Celery tasks: `sync_item_transactions(item_id)`, `sync_item_balances(item_id)`, `sync_item_holdings(item_id)`.
- Sync writes: upsert transactions (dedupe on `external_id`), update `current_balance_cents`, set `last_sync_at`/`sync_status`. Each new transaction runs through `Categoriser`.
- Webhook endpoint `POST /aggregator/webhook/plaid` verifies the Plaid JWT signature, then enqueues the relevant sync task (e.g. `SYNC_UPDATES_AVAILABLE`).
- Error handling: on `ITEM_LOGIN_REQUIRED`, set `sync_status="error"`, store `sync_error`, surface a re-auth prompt; retries with exponential backoff (Celery `autoretry_for`, max 3).

**Testing**:
- `Integration (mocked): exchange → linked_accounts created, initial sync enqueued.`
- `Integration (mocked): webhook with valid signature → sync task enqueued, 200.`
- `Integration: webhook with invalid signature → 401, no task enqueued.`
- `Integration (mocked): sync upserts; running twice imports each transaction once.`
- `Unit: ITEM_LOGIN_REQUIRED → sync_status="error" and sync_error populated.`

#### 3.3 — Securities resolution (OpenFIGI)

**What**: Resolve and dedupe `securities` from holdings using ISIN/CUSIP/ticker → FIGI mapping.

**Design** (`integrations/security_ids/openfigi.py`):
- On holding sync, for each `NormalisedHolding`, look up an existing `securities` row by ISIN→CUSIP→FIGI→ticker (in that order). If none, call OpenFIGI `POST /v3/mapping` to enrich (`name`, `security_type`, `exchange`, `figi`), then insert. Cache mappings in Redis (24h) and respect the 25 req/min free-tier limit with a token-bucket limiter.
- `asset_class` derived from FIGI security type + heuristics (ETF/mutual_fund → look up sector/asset_class metadata later in Phase 4).

**Testing**:
- `Integration (mocked OpenFIGI): unknown ISIN → mapping called, security inserted with figi.`
- `Integration: known ticker already in securities → reused, no API call.`
- `Unit: rate limiter blocks the 26th call within a minute.`
- `Fixture: OpenFIGI 429 response → retried after backoff.`

### Definition of Done
A user links a Plaid sandbox institution; accounts, balances, transactions, and holdings sync; securities are resolved and deduped; webhooks trigger incremental sync; sync errors surface re-auth state. Tests (mocked) green; sandbox test passes locally.

---

## Phase 4: Net Worth & Investment Analytics Engine

### Purpose
Turn raw accounts and holdings into the product's headline value: a net-worth dashboard with historical trend, and an investment analytics engine computing allocation, time-weighted return, benchmark comparison, and fee drag. This is the core that differentiates from balance-only trackers.

### Tasks

#### 4.1 — Holdings, securities pricing & market-data provider

**What**: Implement `securities` and `holdings` tables fully, a `PriceProvider` interface, and daily price refresh.

**Design**:
- `integrations/market_data/base.py`:
```python
class PriceProvider(Protocol):
    async def get_quote(self, security: SecurityRef) -> Quote: ...          # last price
    async def get_history(self, security: SecurityRef,
                          start: date, end: date) -> list[OHLCV]: ...
    async def get_dividends(self, security: SecurityRef,
                            start: date, end: date) -> list[Dividend]: ...
```
- `PolygonProvider` (equities/ETF) and `CoinGeckoProvider` (crypto), selected by `securities.security_type`. Quotes update `securities.last_price_cents`/`last_price_at`; `holdings.market_value_cents = quantity * last_price_cents` and `unrealised_gain_cents = market_value - cost_basis`.
- Celery beat: `refresh_prices` nightly + on-demand; batches requests, caches in Redis.

**Testing**:
- `Unit: market_value = round(quantity * price); unrealised = market_value - cost_basis.`
- `Integration (mocked Polygon): refresh updates last_price_cents and dependent holdings.`
- `Unit: crypto security routes to CoinGecko, equity to Polygon.`

#### 4.2 — Net worth snapshot computation

**What**: Compute and persist daily `net_worth_snapshots` with category breakdown.

**Design**:
- `services/net_worth.py::compute_snapshot(user_id, as_of)`:
  - Sum `current_balance_cents` over visible accounts; assets vs liabilities by `is_asset`.
  - `breakdown_json` keyed by a coarse class derived from `account_type` (checking, savings, brokerage, retirement, real_estate, crypto, credit_cards, mortgage, loans). Liabilities stored negative.
  - Upsert into `net_worth_snapshots` (unique `(user_id, snapshot_date)`).
- Celery beat `nightly_snapshots` iterates active users. Endpoint `GET /networth?from=&to=` returns the trend series; `GET /networth/current` returns latest + breakdown.

**Testing**:
- `Unit: assets=600000, liabilities=340000 → net_worth=260000; breakdown sums to net.`
- `Unit: hidden accounts excluded.`
- `Integration: compute twice same day → single snapshot (upsert).`
- `Integration: GET /networth returns ordered series for the range.`

#### 4.3 — Portfolio analytics (allocation, TWR, IRR, Sharpe, beta, benchmarks)

**What**: Compute investment performance and risk metrics across all investment accounts.

**Design** (`domain/performance.py`, `domain/allocation.py`):
```python
def time_weighted_return(periods: list[SubPeriod]) -> float: ...   # geometric link
def money_weighted_return(cashflows: list[Cashflow]) -> float: ... # XIRR via numpy_financial
def sharpe_ratio(returns: np.ndarray, rf: float) -> float: ...
def beta(asset_returns: np.ndarray, market_returns: np.ndarray) -> float: ...
def asset_allocation(holdings: list[HoldingView]) -> dict[str, float]: ...  # weights by asset_class
```
- TWR breaks the series at each external cashflow (deposit/withdrawal) and geometrically links sub-period returns, neutralising contribution timing. IRR/XIRR via `numpy_financial.irr` on dated cashflows.
- Benchmarks: S&P 500 (`SPY`/index) and a synthetic 60/40 (0.6·equity-index + 0.4·bond-index) computed over the same window for comparison.
- `GET /portfolio/performance?window=1m|3m|ytd|1y|max` returns TWR, benchmark TWRs, Sharpe, beta. `GET /portfolio/allocation` returns class weights + per-holding rows.
- Results cached on `holdings`/a `portfolio_analytics` cache; recomputed by a Celery task on price refresh.

**Testing**:
- `Unit: TWR with a mid-period deposit equals geometric link of sub-period returns (golden value).`
- `Unit: XIRR of [-1000 @ t0, +1100 @ t0+1y] ≈ 0.10.`
- `Unit: allocation weights sum to 1.0; uncovered asset_class bucketed to "other".`
- `Unit: beta of asset==market series == 1.0; Sharpe matches reference calc.`
- `Integration: GET /portfolio/performance returns benchmark comparison fields.`

#### 4.4 — Fee drag analysis

**What**: Compute annual fees and projected multi-decade fee drag (the Empower differentiator).

**Design** (`domain/fees.py`):
- `annual_fee_cents(holding) = market_value_cents * security.expense_ratio`.
- `project_fee_drag(portfolio_value, annual_fee_pct, growth_pct, years)` returns the cumulative difference between gross and net-of-fee compounded value.
- `GET /portfolio/fees` returns per-holding annual fees, total weighted expense ratio, and a 10/20/30-year projected drag series.

**Testing**:
- `Unit: holding value 3_450_000 cents @ 0.03% → annual_fee = 1035 cents.`
- `Unit: project_fee_drag monotonically increases with years; 0% fee → 0 drag.`
- `Integration: GET /portfolio/fees totals match sum of per-holding fees.`

### Definition of Done
Dashboard endpoints return net worth (current + trend), allocation, TWR vs benchmarks, risk metrics, and fee drag. Nightly snapshot and price-refresh tasks run. Golden-value unit tests for all financial math pass.

---

## Phase 5: Budgeting, Cashflow & Web Dashboard

### Purpose
Deliver the budgeting half of the value proposition and the user-facing Next.js dashboard. After this phase the product is a complete, demoable application covering the full MVP scope from `features.md`.

### Tasks

#### 5.1 — Budgets, recurring detection & cashflow forecast

**What**: Budget creation/tracking, subscription detection, and forward cashflow projection.

**Design**:
- `budgets` table (from Suggestion 1) with monthly/weekly/yearly periods. `GET /budgets/progress?month=` joins `transactions` to `budgets` by category, returning budgeted vs actual vs variance.
- Recurring detection (`services/recurring.py`): group transactions by normalised merchant; flag as recurring when ≥3 occurrences at a near-constant amount and a regular interval (monthly/weekly/annual) within tolerance. Persist `recurring_group_id` and surface a subscriptions list with detected cadence and price-change flags.
- Cashflow forecast: project N months forward from average recurring income minus recurring bills minus budgeted discretionary spend; return a monthly series.

**Testing**:
- `Unit: 3 monthly $9.99 charges → recurring detected, frequency="monthly".`
- `Unit: amount jump $9.99→$12.99 → price_changed flag true.`
- `Integration: GET /budgets/progress returns actual summed from transactions for the month.`
- `Unit: forecast = income − bills − discretionary per month over horizon.`

#### 5.2 — Next.js dashboard

**What**: A web dashboard rendering net worth, allocation, performance, budgets, and transactions.

**Design** (`web/`):
- App Router pages: `/` (net worth + allocation + headline metrics), `/investments` (holdings, performance vs benchmark, fees), `/transactions` (paginated list, inline recategorise), `/budgets` (progress bars), `/accounts` (link via Plaid Link, manual add, CSV import).
- Server Components fetch from the FastAPI API using the user's JWT (httpOnly cookie). Recharts for the net-worth trend (line), allocation (donut), and fee-drag projection (area).
- Plaid Link integrated via `react-plaid-link` using the link token from `POST /aggregator/link-token`.
- shadcn/ui components; responsive; dark mode.

**Testing**:
- `Component (React Testing Library): NetWorthCard renders formatted currency from props.`
- `Component: recategorise dropdown PATCHes the transaction and optimistically updates.`
- `E2E (Playwright, mocked API): login → dashboard shows net worth → navigate to investments → allocation donut renders.`
- `E2E: add manual account → appears in accounts list and net worth updates.`

### Definition of Done
Full MVP is demoable in the browser: link/add accounts, see net worth + trend, investment analytics, budgets, and categorised transactions. Playwright happy-path E2E passes against the running stack.

---

## Phase 6: Tax Optimisation Engine

### Purpose
Implement the project's strongest differentiator: cost-basis tracking, realised/unrealised gain calculation, wash-sale detection, and cross-account tax-loss-harvesting identification — none of which incumbents do across externally held accounts. Built on the `tax_lots` and `tax_events` tables.

### Tasks

#### 6.1 — Tax lots & cost-basis accounting

**What**: Track purchase lots and compute cost basis under FIFO/LIFO/HIFO/specific-id.

**Design** (`domain/cost_basis.py`): implement `tax_lots` table from Suggestion 1.
```python
@dataclass
class Lot:
    id: UUID; acquired: date; quantity: Decimal
    cost_per_unit_cents: int
def match_sale(lots: list[Lot], qty_sold: Decimal,
               method: Literal["fifo","lifo","hifo","specific_id"],
               specific_ids: list[UUID] | None = None
               ) -> list[LotConsumption]: ...
```
- Lots created from holding-sync purchases and dividend reinvestments. A sale consumes lots per the method, producing `tax_events` rows (`realised_gain`/`realised_loss`) with `proceeds_cents`, `cost_basis_cents`, `gain_loss_cents`, `is_long_term` (acquired >1yr before sale), and `form_8949_box`.
- Long-term determination uses acquisition→sale dates per IRS rules.

**Testing**:
- `Unit: FIFO consumes oldest lots first; HIFO consumes highest cost first (golden).`
- `Unit: specific_id consumes named lots; insufficient quantity → error.`
- `Unit: lot acquired 2024-01-01 sold 2026-01-02 → is_long_term True; sold 2024-06-01 → False.`
- `Unit: realised gain = proceeds − consumed cost basis (per lot, summed).`

#### 6.2 — Wash-sale detection (§1091)

**What**: Flag wash sales when a substantially identical security is bought within 30 days before/after a loss sale.

**Design** (`domain/wash_sale.py`):
- For each `realised_loss` event, scan purchases of the same `security_id` (across all the user's accounts — the cross-account differentiator) in the ±30-day window. If found, mark `is_wash_sale=True`, compute `wash_sale_disallowed_cents` (proportional to replacement quantity), and adjust the replacement lot's basis.
- Runs as a service after each sale and as a nightly reconciliation task.

**Testing**:
- `Unit: loss sale with a repurchase 5 days later (same security) → wash sale, disallowed amount computed.`
- `Unit: repurchase 31 days later → not a wash sale.`
- `Unit: cross-account repurchase (different account, same security) → still flagged.`
- `Unit: partial repurchase → proportional disallowance.`

#### 6.3 — Tax-loss-harvesting opportunities & tax dashboard

**What**: Identify TLH opportunities across all accounts and surface a tax-year dashboard.

**Design** (`services/tax.py`):
- TLH scan: find tax lots with `unrealised_gain_cents < 0`, exclude securities sold at a loss in the last 30 days (would create a wash sale), rank by harvestable loss, estimate tax savings = loss × marginal rate (from `users.tax_bracket_pct`), and suggest non-substantially-identical replacement candidates (same asset_class, different security).
- `GET /tax/harvesting` returns ranked opportunities. `GET /tax/summary?year=` returns net realised short/long-term gains, estimated liability (using filing status + bracket), dividends, wash-sale count, and contribution headroom by account type (401k/IRA/HSA limits as config).
- `GET /tax/form-8949?year=` exports realised events grouped by Form 8949 box (CSV).

**Testing**:
- `Unit: harvesting excludes a security sold at a loss <30 days ago.`
- `Unit: estimated savings = loss * tax_bracket_pct.`
- `Integration: GET /tax/summary nets short/long gains and losses correctly for the year.`
- `Fixture: Form 8949 export groups events into boxes A–F correctly.`

#### 6.4 — Partition maintenance task

**What**: Auto-create monthly partitions for partitioned tables.

**Design**: Celery beat `ensure_partitions` runs monthly, calling `create_monthly_partition` (Phase 1.2) for next month on `transactions`, `tax_events`, `net_worth_snapshots`, `audit_log`.

**Testing**:
- `Integration: running ensure_partitions creates next month's partitions; idempotent on re-run.`

### Definition of Done
Realised/unrealised gains, wash sales, and cross-account TLH opportunities compute correctly; tax-year summary and Form 8949 export available; partition maintenance automated. Golden-value tests for all tax math pass.

---

## Phase 7: AI Insights, Coach & Categorisation Assist

### Purpose
Layer the AI-native differentiators on top of the now-complete financial data: natural-language Q&A grounded in real portfolio data, proactive anomaly/insight detection, LLM-assisted categorisation, and document parsing. Framed as "insights/education" with disclaimers to stay clear of regulated advice.

### Tasks

#### 7.1 — LLM client, portfolio context builder & prompt caching

**What**: An Anthropic client wrapper that assembles a cached portfolio-context system prompt.

**Design** (`integrations/llm/`):
- `build_portfolio_context(user_id) -> str`: a compact, structured summary (net worth, allocation, top holdings, recent realised gains, budget status, goals) — the large, stable block placed in a **cached** system prompt segment (`cache_control: ephemeral`) to cut cost across a session.
- A standing disclaimer system instruction: "Provide educational insights, not personalised investment advice."
- `ask(user_id, question, history) -> AIResponse` streams the answer; tool-use lets the model call read-only functions (`get_transactions`, `get_holdings`, `run_tlh_scan`) rather than hallucinate numbers.

**Testing**:
- `Unit: build_portfolio_context includes net worth, allocation, top holdings and stays under a token budget.`
- `Integration (mocked Anthropic): ask() includes the disclaimer and the cached context block with cache_control.`
- `Integration (mocked): tool-call request → correct read-only function dispatched, result returned to model.`

#### 7.2 — AI Q&A endpoint & categorisation assist

**What**: Expose the coach via API and wire LLM categorisation into the Phase 2 engine.

**Design**:
- `POST /ai/ask` `{question, conversation_id?}` → streamed response; persists turns; rate-limited per user.
- Categorisation: when rule + merchant-memory miss, `Categoriser` calls the LLM with merchant/description to suggest a category (confidence from the model), used only if confidence ≥ threshold; user corrections still feed `merchant_memory`.

**Testing**:
- `Integration (mocked): POST /ai/ask streams tokens; conversation persisted.`
- `Integration: rate limit exceeded → 429.`
- `Unit: LLM suggestion below threshold → falls back to uncategorised.`

#### 7.3 — Proactive insights & anomaly detection

**What**: Generate alerts for portfolio drift, tax events, fee drags, unusual spending, and subscription price hikes.

**Design** (`services/insights.py`): a nightly Celery task evaluates rule-based detectors (allocation drift > X% from target; new fee drag; spending anomaly = txn > k·category mean; TLH opportunity above threshold; subscription price increase) and stores `alerts` (JSONB per Suggestion 2's `alerts_json`, or a small `alerts` table). The LLM only writes human-readable explanations for flagged items; detection itself is deterministic (auditable, cheap).

**Testing**:
- `Unit: allocation 55% vs 45% target → drift alert; within band → none.`
- `Unit: transaction 5× category mean → anomaly flagged.`
- `Integration (mocked LLM): insights task produces alerts with explanations; deterministic detection independent of LLM.`

#### 7.4 — Document parsing (statements / tax forms)

**What**: Extract structured data from uploaded PDFs (brokerage statements, 1099s) to reduce manual entry.

**Design** (`ingest/documents.py`): `POST /documents` stores the encrypted file in `documents`; a Celery task runs a text pre-pass (`pdfplumber`/`pypdf`), sends extracted text to the LLM with a structured-output schema (holdings, realised gains, dividends), validates against Pydantic, and stages proposed entries for user confirmation before writing to ledger/tax tables.

**Testing**:
- `Fixture (mocked LLM): sample 1099-B text → structured realised-gain rows matching schema.`
- `Unit: extracted data failing Pydantic validation → staged as needs_review, not auto-applied.`
- `Integration: uploaded document stored encrypted; decryptable by owner only.`

### Definition of Done
AI coach answers grounded questions with tool use and disclaimers; categorisation falls back to LLM; nightly insights produce auditable alerts; document upload extracts reviewable structured data. Prompt caching active. Mocked-LLM tests green.

---

## Phase 8: Goals, Retirement Projection & MCP Server

### Purpose
Complete the planning layer (goals, Monte Carlo retirement projection, FIRE/scenario modelling) and ship the standout integration differentiator: an MCP server exposing the user's finance data to AI assistants — something no incumbent offers.

### Tasks

#### 8.1 — Goals & Monte Carlo retirement projection

**What**: Goal tracking with probability-scored projections and scenario modelling.

**Design** (`domain/monte_carlo.py`): implement `goals` table (Suggestion 1).
- `monte_carlo(start_value, monthly_contribution, years, mean_return, volatility, trials=10_000) -> ProjectionResult` returns success probability vs target, percentile end-value bands, and the projected achievement date. Uses `numpy` vectorised simulation (lognormal returns).
- `POST /goals`, `GET /goals/{id}/projection`, and scenario endpoint `POST /goals/{id}/scenario` (e.g., "retire at 60", "+$500/mo", "pay off mortgage early") recomputing with overridden parameters. Results cached in `goals.projection_json`.

**Testing**:
- `Unit: deterministic seed → reproducible success_pct; higher contribution → higher success_pct (monotonic).`
- `Unit: percentile bands ordered p10 < p50 < p90.`
- `Integration: scenario override changes projection without mutating base goal.`

#### 8.2 — MCP server

**What**: An MCP server exposing read-only finance resources and tools to AI assistants.

**Design** (`mcp/server.py`, Python `mcp` SDK):
- Resources: `networth`, `allocation`, `accounts`, `goals`, `tax-summary`. Tools: `get_transactions(from,to,category)`, `run_tlh_scan()`, `get_performance(window)`.
- Auth: a per-user MCP token (issued only when `users.mcp_enabled`), scoped read-only; all calls go through the same RBAC + the audit log (actor_type=`ai`).
- Runs as a separate process/entrypoint; documented for connecting Claude Desktop / other MCP clients.

**Testing**:
- `Integration (MCP test client): list resources → expected set; read networth → current snapshot.`
- `Integration: get_transactions tool → rows scoped to the authenticated user.`
- `Integration: MCP call when mcp_enabled=False → unauthorised; calls recorded in audit_log.`

### Definition of Done
Goals with Monte Carlo projections and scenario modelling work; MCP server exposes scoped read-only finance data to an MCP client; all MCP access audited. Tests green.

---

## Phase 9: Hardening, Self-Hosting & Compliance

### Purpose
Make the platform production- and self-host-ready: multi-currency, audit/compliance, GDPR/CCPA data rights, security hardening, and packaged deployment. This phase realises the privacy-first differentiator and the SOC 2 / ISO 27001 alignment expectations.

### Tasks

#### 9.1 — Multi-currency & FX

**What**: Normalise balances and holdings across currencies into the user's `default_currency`.

**Design**: an `fx_rates` cache (daily rates from a provider behind an `FxProvider` interface). Net-worth and portfolio rollups convert each account/holding to the base currency at the latest rate; original currency retained. Snapshots store the rate used.

**Testing**:
- `Unit: EUR account converts to USD at the cached rate; missing rate → flagged, excluded from total with warning.`
- `Integration: mixed-currency net worth equals sum of converted balances.`

#### 9.2 — GDPR/CCPA data rights & audit completeness

**What**: Data export, right-to-erasure (crypto-shredding), and complete audit logging.

**Design**:
- `GET /me/export` produces a full JSON/CSV bundle (data portability).
- `DELETE /me` performs crypto-shredding (destroy the user's encryption key, schedule row deletion) — satisfies GDPR erasure for encrypted aggregator tokens/documents.
- Audit middleware writes `audit_log` for all mutating actions and all AI/MCP/aggregator reads of sensitive data.

**Testing**:
- `Integration: export contains accounts, transactions, holdings, tax events for the user only.`
- `Integration: DELETE /me → encrypted documents/tokens become undecryptable; user rows removed.`
- `Integration: every mutating endpoint writes an audit_log row.`

#### 9.3 — Security hardening & self-hosted packaging

**What**: TLS 1.3 enforcement, secret management, rate limiting, and a one-command self-hosted deployment.

**Design**:
- Enforce `Strict-Transport-Security`, secure cookies, CSP; bandit clean; dependency scanning in CI.
- Per-user/IP rate limiting (Redis) on auth, AI, and aggregator endpoints.
- `docker-compose.selfhost.yml`: single-user mode (`deployment_mode=selfhost`), optional SQLite fallback for the smallest footprint, no external aggregator required (manual/CSV/OFX only), data kept local. README self-hosting guide.

**Testing**:
- `Integration: HTTP request → redirected/refused per HSTS; response has security headers.`
- `Integration: exceed auth rate limit → 429.`
- `E2E: docker compose -f docker-compose.selfhost.yml up → register, add manual account, import CSV, see net worth — fully offline.`

### Definition of Done
Multi-currency rollups correct; data export and erasure work; all sensitive actions audited; security headers and rate limits enforced; bandit/dependency scans clean; self-hosted compose boots and runs offline end-to-end.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (skeleton, DB, auth) ─── required by everything
    │
Phase 2: Accounts, Transactions, Categorisation ─── requires Phase 1
    │
Phase 3: Aggregation (Plaid) ─── requires Phase 2
    │
Phase 4: Net Worth & Investment Analytics ─── requires Phase 3
    ├── Phase 5: Budgeting & Web Dashboard ─── requires Phase 4 (UI can start against mocked API after Phase 4 contracts)
    └── Phase 6: Tax Optimisation Engine ─── requires Phase 4 (holdings/securities); parallel with Phase 5
         │
Phase 7: AI Insights, Coach, Categorisation Assist ─── requires Phases 4 & 6 (needs portfolio + tax data)
    │
Phase 8: Goals, Monte Carlo & MCP Server ─── requires Phase 4 (Monte Carlo) & Phase 7 (MCP exposes AI-relevant data)
    │
Phase 9: Hardening, Self-Hosting & Compliance ─── requires all; finalises the platform
```

Parallelism opportunities:
- **Phase 5 (Web Dashboard)** and **Phase 6 (Tax Engine)** can be developed concurrently once Phase 4 ships, as they share no code (frontend vs backend tax math).
- Within Phase 3, the **MX/TrueLayer adapters** (post-MVP) can be added in parallel once the `Aggregator` protocol (3.1) is fixed.
- **Frontend work in Phase 5** can begin against the OpenAPI contract (mocked) as soon as Phase 4 endpoints are specified, ahead of full backend completion.
- **Phase 8.1 (Monte Carlo)** depends only on Phase 4 and can be built in parallel with Phase 7.

Mapping to `features.md` scope: Phases 1–5 deliver the **MVP (Must-have)**. Phase 6 + parts of 7 (AI Q&A, fee drag, tax tracking, crypto via CoinGecko) and Phase 5's household sharing deliver **v1.1 (Should-have)**. Document vault/OCR (7.4), alternative assets, estate/beneficiary tracking, holistic tax calendar, and local-first deployment (Phase 9) cover the **backlog (Nice-to-have)**.

---

## Definition of Done (per phase)

Every phase must satisfy this checklist before it is considered complete:

1. All tasks in the phase implemented.
2. All unit and integration tests pass; new financial-math functions have golden-value tests.
3. `ruff check` and `ruff format --check` pass.
4. `mypy --strict` passes.
5. `bandit` reports no high/medium issues on new code.
6. The feature works end-to-end (verified via integration or E2E test).
7. New config options documented in `.env.example` and README.
8. New/changed API endpoints appear in the auto-generated OpenAPI 3.1 spec.
9. Database changes have an Alembic migration that upgrades and downgrades cleanly on a fresh PostgreSQL.
10. `docker compose build` succeeds; affected services boot via `docker compose up`.
11. External integrations are exercised by mocked integration tests; real-dependency tests are marked and runnable locally.
12. Sensitive actions write to `audit_log`; secrets/tokens are stored encrypted.
```
