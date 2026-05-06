# Personal Finance & Investment Tracker

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open, AI-native platform that unifies net worth, investment performance, and tax optimisation in a single view.

Personal Finance & Investment Tracker is a candidate open-source project for individuals whose assets are spread across bank accounts, brokerages, retirement funds, property, cryptocurrency, and private investments. It targets the gap between budgeting-only and investment-only tools by providing a holistic dashboard with AI-driven insights across the full financial picture.

---

## Why Personal Finance & Investment Tracker?

- Most consumer tools focus on either budgeting (YNAB, Quicken Simplifi) or investments (Empower, PortfolioPilot), forcing users to manually reconcile across multiple apps.
- Comprehensive trackers like Kubera ($249–$2,499/yr) and full AI recommendation tiers like PortfolioPilot Platinum ($99/mo) are expensive, while affordable options lack advanced portfolio analytics.
- No mainstream product offers a holistic tax planning timeline that visualises Roth conversions, tax-loss harvesting windows, estimated tax payments, and capital gains realisation across the calendar year.
- Cross-account tax-loss harvesting for DIY investors is unaddressed: Wealthfront only harvests assets it manages directly.
- Privacy-conscious users have no local-first or self-hosted option among incumbent SaaS products.
- Non-US asset support (PSD2, UK, Australian, Asian account types) is poorly served by predominantly US-centric incumbents.

---

## Key Features

### Net Worth & Aggregation

- Multi-account aggregation via Plaid or MX covering bank, brokerage, credit card, and loan accounts
- Net worth dashboard with daily balance updates and historical trend chart
- Multi-currency support with automatic FX conversion
- CSV import fallback for institutions without API coverage

### Investment Analytics

- Holdings list, total portfolio value, asset allocation breakdown, and daily gain/loss
- Time-weighted return and benchmark comparison against indices such as S&P 500 and 60/40
- Fee drag analysis across all linked investment accounts
- Cost basis tracking (FIFO, LIFO, specific identification) with realised vs unrealised gains

### Tax Optimisation

- Tax event tracking, including realised gains/losses, estimated tax liability, and wash sale flagging
- Tax-loss harvesting opportunity identification across externally held accounts
- Account contribution optimisation across 401k, IRA, HSA, and taxable accounts
- Roth conversion and capital gains planning tools

### Budgeting & Cashflow

- Transaction auto-categorisation with manual correction and category learning
- Budget and spending plan creation with monthly tracking
- Recurring bill and subscription detection
- Cashflow forecasting

### Planning & AI Insights

- Retirement projections using Monte Carlo probability scoring
- Goal feasibility analysis and scenario modelling (e.g. retire at 60, pay off mortgage early)
- AI financial Q&A using the user's actual portfolio data
- Proactive alerts for portfolio drift, tax events, fee drags, and unusual spending

### Alternative Assets & Estate (backlog)

- Real estate (Zillow or manual), vehicles (VIN), private equity (Carta), precious metals, and crypto/DeFi balances
- Beneficiary and estate document tracking alongside asset records
- Document vault with OCR for statements and tax returns

---

## AI-Native Advantage

AI is applied where incumbents underuse it: continuous tax opportunity identification across all accounts, natural-language Q&A grounded in the user's real portfolio, proactive anomaly detection without user-defined rules, and goal feasibility analysis with actionable recommendations. JP Morgan research cited by PortfolioPilot indicates continuous tax optimisation can deliver an additional 1.94% yearly return — a margin that compounds significantly over decades. Document parsing via LLMs can extract data from uploaded tax returns and brokerage statements, removing manual entry friction.

---

## Tech Stack & Deployment

The project anticipates a cloud-hosted default with a privacy-first local or self-hosted deployment option for users unwilling to share financial data with third parties. Account aggregation will use open banking APIs (Plaid, MX, TrueLayer, FDX-compliant providers) with CSV import fallback. The portfolio analytics engine handles time-weighted return, IRR, Sharpe ratio, beta, and asset allocation. Security expectations include bank-level encryption, read-only access tokens, and SOC 2 alignment. AI features are framed as "insights" and "education" with disclaimers to avoid crossing into regulated investment advice.

---

## Market Context

The AI-powered personal finance management market was valued at $1.37 billion in 2024 and is projected to reach $2.36 billion by 2032 at a 7% CAGR ([Financial Deep Dive](https://financialdeepdive.com/investing/personal-finance/best-ai-tools-for-personal-finance-in-2026)). Incumbent pricing ranges from $35.88/yr (Quicken Simplifi) and $99.99/yr (Monarch Core) to $249–$2,499/yr (Kubera) and $99/mo (PortfolioPilot Platinum). Primary buyers are millennials and high-income professionals managing complex, multi-asset portfolios.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
