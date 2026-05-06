# Personal Finance & Investment Tracker — Feature & Functionality Survey

> Candidate #356 · Researched: 2026-05-04

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Empower (Personal Capital) | Desktop + Mobile | Free (wealth management upsell) | https://www.empower.com |
| Monarch Money | Web + Mobile | SaaS — $99.99/yr (Core), $199/yr (Plus) | https://www.monarch.com |
| Kubera | Web | SaaS — $249/yr (Essentials), $2,499/yr (Black) | https://www.kubera.com |
| PortfolioPilot | Web | Freemium — Free / $29/mo Gold / $99/mo Platinum | https://portfoliopilot.com |
| Copilot Money | iOS + Web | SaaS — $95/yr | https://www.copilot.money |
| Quicken (Simplifi) | Desktop + Mobile | SaaS — $35.88/yr | https://www.quicken.com |
| Quicken Business & Personal | Desktop + Mobile | SaaS — $47.88/yr | https://www.quicken.com |
| YNAB (You Need A Budget) | Web + Mobile | SaaS — $109/yr | https://www.ynab.com |
| Tiller Money | Google Sheets / Excel | SaaS — $79/yr | https://tiller.com |
| Finary | Web + Mobile | SaaS — €54.99/yr (Lite), €149.99/yr (Plus) | https://finary.com |
| Wealthfront | Web + Mobile | Managed robo-advisor — 0.25% AUM | https://www.wealthfront.com |
| Betterment | Web + Mobile | Managed robo-advisor — 0.25–0.40% AUM | https://www.betterment.com |

---

## Feature Analysis by Solution

### Empower (formerly Personal Capital)

**Core features**
- Free unified financial dashboard connecting bank, investment, retirement, and loan accounts
- Net worth tracking with real-time updates throughout the trading day
- Investment Checkup: compares current allocation to a target allocation based on age and risk tolerance
- Fee Analyzer: calculates total expense ratios across all linked investment accounts and projects cumulative fee drag over time
- Retirement Planner using Monte Carlo simulations to project retirement readiness as a probability score
- Asset allocation breakdown by class (US equities, international equities, bonds, alternatives, cash)
- Cash flow tracking and spending categorisation

**Differentiating features**
- Fee Analyzer is the most prominent differentiator — visualises how seemingly small fees compound over decades
- Monte Carlo retirement projections with probability-of-success scores
- Completely free for all financial tracking tools; monetises via wealth management services for accounts over $100,000

**UX patterns**
- Dashboard-first: all key metrics visible at a glance on login
- Account linking via Plaid and direct integrations with major brokerages
- Clear drill-down from portfolio summary to individual holding level
- Guided "Investment Checkup" wizard to prompt users to review allocation

**Integration points**
- Plaid for bank and investment account connectivity
- Direct integrations with Fidelity, Vanguard, Schwab, and other major brokerages
- No public API; proprietary data aggregation

**Known gaps**
- Limited budgeting features compared to dedicated budgeting apps
- No cryptocurrency tracking beyond simple balance display
- Retirement planner limited to US-centric accounts (401k, IRA, Social Security)
- No collaborative/household sharing for non-advisory clients
- Investment advice features gated behind paid wealth management tier

**Licence / IP notes**
- Proprietary, closed-source SaaS. No open-source components exposed.

---

### Monarch Money

**Core features**
- Unified dashboard combining budgeting, net worth, and investment tracking
- Two budgeting modes: Flex Budgeting (envelope-style) and Category Budgeting (traditional)
- Portfolio performance tracking: gains/losses, asset allocation overview
- Account aggregation via Plaid and Finicity
- Collaborative household budgets (multiple users under one subscription)
- Net worth tracking across accounts
- Recurring transaction detection and subscription management

**Differentiating features**
- "Plus" tier (launched 2026) adds financial modelling for future scenarios and small-business tracking in one workspace
- Strong collaborative features: household budgeting with role-based permissions
- Smooth onboarding with guided budget setup wizard

**UX patterns**
- Conversational setup flow asks about goals before revealing features
- Color-coded spending categories for fast visual scanning
- Mobile-first design with parity on web

**Integration points**
- Plaid and Finicity for account connections
- CSV import fallback for unsupported institutions
- Zapier integration for automation workflows

**Known gaps**
- No advanced portfolio analytics (no fee analysis, no Monte Carlo, no benchmark comparison)
- Investment tracking is display-only — no rebalancing recommendations
- Limited tax planning features
- No document vault or estate planning tools

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Kubera

**Core features**
- Balance-sheet-style net worth tracker for complex, multi-asset portfolios
- Supports: global stocks and ETFs, crypto and DeFi (including staking, lending, and NFTs), real estate (Zillow integration for US), vehicles (VIN lookup), domain names, precious metals, private equity (Carta integration), and manual rows for anything else
- Multi-currency support with automatic FX conversion
- Bank and brokerage connectivity via Plaid, Salt Edge, and direct integrations
- Beneficiary tracking: record account beneficiaries alongside assets for estate planning
- "Safe" document vault: attach scanned documents (insurance policies, wills, trust documents) to assets
- Mortality planning: designate a trusted contact to receive access in the event of death

**Differentiating features**
- Broadest asset class support of any consumer tracker — DeFi, NFTs, domain names, private equity via Carta
- Beneficiary and estate documentation built into asset records
- Mortality/legacy planning feature (a unique differentiator in the consumer market)
- "Black" tier supports nested portfolios for family offices and advisors managing multiple clients

**UX patterns**
- Spreadsheet-inspired layout (rows and columns) rather than card-based dashboards
- Manual entry is a first-class workflow, not an afterthought
- Data export to CSV/Excel for users who want to analyse in their own tools

**Known gaps**
- No budgeting or transaction categorisation features
- No cashflow tracking or spending insights
- No tax loss harvesting or investment recommendations
- No retirement planning tools
- Real estate valuation limited to US (Zillow); manual entry only for non-US

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### PortfolioPilot

**Core features**
- AI-powered portfolio assessment and personalised investment recommendations
- Aggregates net worth including brokerage, 401k, IRA, crypto, cash, real estate, private equity, and precious metals
- Portfolio scoring based on expected returns, risk-adjusted performance, diversification, and fee drag
- Tax impact analysis and tax-loss harvesting identification (Gold and above)
- Scenario modelling: "what if" analysis for asset allocation changes
- Efficient Frontier optimizer: shows the optimal risk/return allocation for your current holdings
- AI Assistant for natural language investment Q&A (5 queries/day on Gold, 100/day on Platinum)
- Smart stock/fund screeners with semantic AI equity search

**Differentiating features**
- Hybrid AI combining machine learning with hedge-fund-inspired quantitative models
- Continuous economic monitoring to provide macro-aware portfolio scoring
- Estate planning tools included in the free tier
- Semantic AI equity search (Platinum): natural language queries to find investments matching specific criteria

**UX patterns**
- Investment-first UX: onboarding centres on portfolio connection rather than budgeting
- "Portfolio Score" displayed prominently as a single headline metric
- Recommendations presented as actionable items rather than raw data

**Integration points**
- Account aggregation via Plaid
- CSV import for manual upload
- No public API

**Known gaps**
- No integrated budgeting or cashflow management
- Expensive for full AI recommendation access ($99/mo Platinum)
- Limited budgeting features compared to dedicated tools
- No document vault or beneficiary tracking

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Copilot Money

**Core features**
- AI-powered automatic transaction categorisation that learns from user corrections
- Investment tracking: stocks, ETFs, and crypto with live performance estimates throughout the day
- Asset allocation view by asset type
- Real estate tracking via Zillow integration
- Net worth dashboard
- Spending analytics with month-over-month and year-over-year comparisons
- Subscription tracking: detects and highlights recurring charges
- AI money assistant (public beta as of April 2026)

**Differentiating features**
- Best-in-class auto-categorisation that learns continuously from corrections
- Visual design is widely praised as the most polished of any personal finance app
- iOS-native with full web access added in December 2025
- Benchmark investment comparison against market indexes

**UX patterns**
- Strong mobile-first design with emphasis on visual clarity
- Onboarding is fast; account linking and categorisation begin immediately
- Smart notification system for unusual spending or subscription changes

**Integration points**
- Plaid for account connectivity
- Zillow for real estate valuation
- No public API

**Known gaps**
- No advanced investment analytics (no fee analysis, Monte Carlo, or rebalancing)
- No collaborative/household sharing
- No tax planning features
- Android app lacking compared to iOS (web added as partial workaround)
- Limited budgeting structure compared to YNAB or Monarch

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Quicken Simplifi

**Core features**
- Personalised Spending Plan auto-generated from income minus bills and subscriptions
- Connects to 14,000+ financial institutions
- Transaction auto-categorisation with user-correction learning
- Investment dashboard tracking individual holdings including cryptocurrency
- Net worth tracking
- Spending plan up to 12 months forward
- Savings goal tracking
- Retirement planner with dynamic timeline visualisation
- AI money assistant (public beta as of April 2026)

**Differentiating features**
- Best value pricing in the market ($35.88/yr)
- Spending Plan model is more prescriptive and actionable than traditional budgets
- Part of Quicken ecosystem, which also covers small business accounting

**UX patterns**
- Dashboard shows safe-to-spend amount prominently on open
- Bills and subscriptions surface automatically
- Forward-looking financial calendar

**Integration points**
- Plaid and direct bank connections
- Quicken ecosystem (shares data model with Quicken Business & Personal)
- No public API

**Known gaps**
- Investment features are basic display-only
- No advanced portfolio analytics or tax-loss harvesting
- No collaborative household sharing across separate accounts
- Limited crypto support

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### YNAB (You Need A Budget)

**Core features**
- Zero-based budgeting: every dollar of income is assigned a category before being spent
- Four Rules methodology: Give Every Dollar a Job, Embrace True Expenses, Roll With the Punches, Age Your Money
- "Age of Money" metric: tracks how long money sits before being spent (goal: 30+ days)
- Reports: spending history, income vs. expenses, net worth trends
- Shared budgets via YNAB Together (up to 6 users)
- Loan tracking
- Debt payoff planning with avalanche/snowball views

**Differentiating features**
- Strongest budgeting methodology of any app — builds genuine financial habits
- Education ecosystem: live workshops, YouTube content, community support
- YNAB Together: household sharing with granular permission controls

**UX patterns**
- Requires active participation: users must assign every dollar manually (though auto-import speeds this up)
- Prominent onboarding tutorials and methodology education
- "Roll with the Punches" UX: when you overspend in a category, app guides you to cover it from elsewhere

**Integration points**
- Bank import via Plaid and direct bank connections
- CSV import
- Public API for read/write access to budgets (OAuth-based)
- Zapier and Make (Integromat) support via third-party connectors

**Known gaps**
- No investment tracking beyond simple account balance display
- No net worth dashboard with holdings detail
- No retirement planning or portfolio analytics
- Learning curve is steep for users unfamiliar with zero-based budgeting

**Licence / IP notes**
- Proprietary SaaS. YNAB API is available but closed-source. No open-source components.

---

### Tiller Money

**Core features**
- Automated daily transaction sync to Google Sheets or Microsoft Excel
- 21,000+ connected financial institutions
- Foundation Template: pre-built spreadsheet with budget tracking, net worth, and spending dashboards
- AutoCat: 100% customisable auto-categorisation rules within Google Sheets
- Prebuilt templates for debt tracking, savings goals, net worth, and more
- Transactions Sheet as single source of truth for all financial data

**Differentiating features**
- Only mainstream personal finance product that delivers data directly into spreadsheets
- Ultimate flexibility: users can build any analysis on top of their own data in Excel/Sheets
- AutoCat is far more customisable than any app-based categorisation engine
- Complete data ownership: all data stays in the user's Google Drive or OneDrive

**UX patterns**
- Spreadsheet-native: no proprietary interface; users work in their familiar tool
- Community-contributed templates extend functionality significantly
- Data export is implicit — data is always in the spreadsheet

**Integration points**
- Plaid for bank connectivity
- Google Workspace Marketplace add-on (Google Sheets)
- Excel add-in (Microsoft AppSource)
- No dedicated API (data accessible via Google Sheets API or Excel APIs)

**Known gaps**
- No mobile app; purely desktop/web via spreadsheet
- No investment portfolio analytics beyond balance display
- No retirement planning tools
- Requires spreadsheet literacy — not suitable for all users
- No collaborative budgeting beyond native Sheets sharing

**Licence / IP notes**
- Proprietary SaaS for data sync service. Google Sheets and Excel are third-party platforms with their own licences. Community templates are typically shared under Creative Commons.

---

### Finary

**Core features**
- 360° wealth tracking: bank accounts, savings, PEA, life insurance, PER (French pension), loans, employee savings, business accounts, crypto, and real estate
- Connects to 20,000+ banks, brokers, and crypto platforms
- Portfolio diversification scoring with actionable insights
- Dividend tracker: projected annual income, upcoming payments, historical dividend data
- Budget and cashflow management with AI-powered categorisation
- Financial independence (FIRE) planning: projects net worth over 30 years
- Secure aggregation via Powens (Budget Insight) and Plaid — no credential exposure

**Differentiating features**
- Leading personal finance tracker designed for European users, with strong support for French financial products (PEA, PER, life insurance)
- FIRE planning built into the core product
- Dividend-focused analytics as a core feature rather than an afterthought

**UX patterns**
- Dashboard-first with net worth as headline metric
- Mobile apps on iOS and Android with full parity
- Clean visual design similar to Copilot Money

**Integration points**
- Powens (Budget Insight) for European bank connectivity (PSD2-compliant)
- Plaid for additional institution coverage
- No public API

**Known gaps**
- Tax optimisation features limited outside of France
- US market not a primary focus
- No advanced rebalancing or Monte Carlo modelling
- No document vault

**Licence / IP notes**
- Proprietary SaaS. French-registered company; subject to GDPR. No open-source components.

---

### Wealthfront

**Core features**
- Automated investing with tax-loss harvesting (daily, across all accounts)
- Direct indexing at $100,000+ AUM (own individual stocks comprising an index, enabling enhanced tax-loss harvesting)
- Asset allocation across: US equities, international equities, emerging markets, bonds, real estate (REITs), dividend equities
- Path: financial planning tool projecting retirement date, Social Security optimisation, and home purchase modelling
- Autopilot: automatically sweeps excess cash from checking to investment account above a set threshold
- Cash account with high-yield interest rate
- Risk parity and smart beta portfolio options

**Differentiating features**
- Daily automated tax-loss harvesting across the full portfolio — most sophisticated of any consumer product
- Autopilot cash sweeping: unique feature that automates savings/investment transfers
- Direct indexing (tax-optimised individual stock ownership) at $100K threshold

**UX patterns**
- Automated-first: the product minimises required user decisions
- Path planning tool provides guided questions to model financial scenarios
- Clean, minimal interface — investment decisions are largely handled by the engine

**Integration points**
- No public API
- External account linking for Path planning tool only (read-only, via Plaid)
- Not designed as an aggregator — primarily manages assets held at Wealthfront

**Known gaps**
- Not a tracker — assets must be held at Wealthfront (not aggregation of existing accounts)
- No budgeting features
- No cryptocurrency support
- No collaborative household investing features
- High minimum for direct indexing ($100K)

**Licence / IP notes**
- Proprietary, regulated investment adviser (SEC-registered RIA). Not open-source.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Multi-account aggregation via Plaid or equivalent open banking API
- Net worth dashboard with real-time or daily balance updates
- Spending transaction categorisation (auto and manual correction)
- Investment portfolio display (holdings, allocation, total value)
- Budget or spending plan creation and tracking
- iOS and Android mobile apps with web parity
- Bank-level encryption and read-only access to linked accounts

### Differentiating Features
- AI-powered personalised investment recommendations with economic context (PortfolioPilot)
- Automated daily tax-loss harvesting across all accounts (Wealthfront)
- Fee drag analysis projected over decades (Empower)
- Ultra-broad alternative asset support including DeFi, NFTs, private equity, domains, and vehicles (Kubera)
- Estate planning and beneficiary tracking alongside asset records (Kubera)
- Best-in-class auto-categorisation that learns continuously from corrections (Copilot)
- Spreadsheet-native data delivery with full user ownership (Tiller)
- FIRE planning and dividend income projections (Finary)
- Retirement Monte Carlo simulations with probability scores (Empower, PortfolioPilot)
- Semantic AI equity search with natural language investment discovery (PortfolioPilot Platinum)

### Underserved Areas / Opportunities
- **Holistic tax planning timeline**: no product visualises the full calendar-year tax event sequence (Roth conversions, estimated tax payments, tax-loss harvesting windows, capital gains realisation)
- **Cross-account tax-loss harvesting for DIY investors**: Wealthfront does this only for assets held with them; no tracker does this across externally held accounts
- **Estate and legacy planning integration**: Kubera's beneficiary tracking is basic; no product guides users through account titling, beneficiary optimisation, or estate tax exposure
- **Privacy-first / local-first deployment**: no consumer product offers on-premises or local-first operation
- **Non-US asset support**: most tools are US-centric; European (PSD2), UK, Australian, and Asian account types are poorly served
- **AI financial coach with full portfolio context**: PortfolioPilot's assistant is limited in free tiers; no product provides unrestricted conversational AI using the user's real data
- **Collaborative household / adviser access**: sharing is limited to budgets (YNAB, Monarch); no product enables read-only adviser access without credential sharing
- **Crypto and DeFi analytics beyond balance display**: only Kubera tracks DeFi positions; no product offers on-chain analytics or staking yield projections

### AI-Augmentation Candidates
- **Automatic transaction categorisation**: already a strong AI application but still imperfect; LLM understanding of merchant context could improve accuracy further
- **Natural language financial Q&A**: "Can I afford to retire at 60?" answered using the user's real data — only partially addressed today
- **Proactive anomaly detection**: identifying unusual charges, subscription price increases, or portfolio drift without requiring user-defined rules
- **Tax opportunity identification**: cross-account capital gain/loss netting, wash sale avoidance, and Roth conversion windows derived from real portfolio data
- **Goal feasibility analysis**: assessing whether stated goals (retire at 55, pay off mortgage in 10 years) are achievable given current trajectory, with specific actionable recommendations
- **Document parsing**: extracting data from uploaded tax returns, brokerage statements, and insurance policies to populate the net worth model without manual entry

---

## Legal & IP Summary

No patents or proprietary IP concerns were identified during this research that would block development of an open-source alternative. All features identified are functional capabilities — budgeting methodologies, portfolio analytics, and aggregation techniques — rather than patented inventions. Account aggregation relies on Plaid, MX, TrueLayer, or FDX-compliant bank APIs, all of which are commercially available to third-party developers. YNAB publishes a public OAuth-based API under terms that permit third-party integration. Tax calculation methodologies (FIFO, HIFO, wash sale tracking) are mathematical conventions defined by the IRS, not proprietary IP. An open-source implementation should comply with applicable financial regulations (not constituting regulated investment advice) and relevant data privacy laws (GDPR, CCPA/CPRA, and equivalents).

---

## Recommended Feature Scope

**Must-have (MVP)**
- Multi-account aggregation via Plaid or MX (bank accounts, brokerage accounts, credit cards, loans)
- Net worth dashboard with daily balance updates and historical trend chart
- Investment portfolio display: holdings list, total value, asset allocation breakdown, and daily gain/loss
- Transaction auto-categorisation with manual correction and category learning
- Budget/spending plan creation and monthly tracking
- Basic investment performance metrics: time-weighted return, benchmark comparison (S&P 500, 60/40)

**Should-have (v1.1)**
- Retirement planning with Monte Carlo probability projections
- Fee drag analysis across all linked investment accounts
- Tax event tracking: realised gains/losses, estimated tax liability, wash sale flagging
- Cryptocurrency and DeFi balance tracking (Ethereum, Bitcoin, Solana)
- Collaborative household access with role-based permissions (view/edit)
- AI financial Q&A using the user's actual portfolio data

**Nice-to-have (backlog)**
- Alternative asset tracking: real estate (Zillow/manual), vehicles (VIN), private equity (Carta), precious metals
- Beneficiary and estate document tracking alongside asset records
- Holistic tax planning calendar: visualising full-year tax events and action windows
- Local-first / privacy-first deployment mode (data stored on-device or self-hosted)
- Document vault with OCR: extract data from uploaded statements and tax returns automatically
- FIRE planning with financial independence date projection
