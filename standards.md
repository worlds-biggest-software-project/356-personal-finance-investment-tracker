# Standards & API Reference

> Project: Personal Finance & Investment Tracker · Generated: 2026-05-04

## Industry Standards & Specifications

### Open Banking & Data-Sharing Standards

**FDX API (Financial Data Exchange)**
- URL: https://financialdataexchange.org/
- The primary US open banking standard. A royalty-free, interoperable API specification covering over 600 data elements across banking, investment, tax, and insurance domains. Built on OAuth 2.0 with FAPI security profiles. As of early 2026 over 130 million consumer accounts are connected via FDX API. Recognised by the CFPB as the official US open banking standards body (January 2025). Any US-market aggregation layer should align with FDX API.

**PSD2 (Payment Services Directive 2 — EU)**
- URL: https://plaid.com/open-banking/ · https://www.openbanking.org.uk/
- European regulatory framework mandating banks to give licensed third-party providers (TPPs) API access to consumer accounts with explicit consent. Governs open banking integrations for any product serving EU or UK users. Implementations use the Berlin Group NextGenPSD2 framework for cross-border European interoperability.

**UK Open Banking Standard (OBIE)**
- URL: https://standards.openbanking.org.uk/
- Developed by the Open Banking Implementation Entity (OBIE) following the CMA ruling (2018). Comprehensive standards covering APIs, data formats, security, and strong customer authentication (SCA) for UK institutions. Mandatory for the nine largest UK banks; widely adopted by UK fintechs.

**Berlin Group NextGenPSD2**
- URL: https://www.berlin-group.org/nextgenpsd2-downloads
- Pan-European API framework building on PSD2 to standardise open banking APIs across EU member states. Widely implemented by European banks and aggregators (Powens, TrueLayer, Salt Edge).

**OFX (Open Financial Exchange)**
- URL: https://ofx.net/
- Older US standard for financial data exchange used by desktop tools such as Quicken and Microsoft Money. Still relevant for CSV/OFX import compatibility with legacy brokerage accounts and tax software.

---

### Security & Authentication Standards

**FAPI 2.0 (Financial-grade API Security Profile)**
- URL: https://openid.net/specs/fapi-security-profile-2_0-final.html · https://fapi.openid.net/
- OpenID Foundation security profile designed for high-value financial API interactions. Builds on OAuth 2.0 and OpenID Connect, mandating Pushed Authorization Requests (PAR) and PKCE. Required for FDX and PSD2-compliant implementations. FAPI 2.0 simplifies and strengthens FAPI 1.0 with reduced optionality.

**OAuth 2.0 (RFC 6749)**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundation authorisation framework for all major open banking APIs (FDX, PSD2, OBIE). Enables consumers to grant third-party apps read-only access to financial accounts without exposing bank credentials. All major account aggregators (Plaid, MX, TrueLayer) implement OAuth 2.0 flows.

**PKCE — Proof Key for Code Exchange (RFC 7636)**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- Mandatory extension for OAuth 2.0 public clients, required by FAPI 2.0. Prevents authorisation code interception attacks, which is particularly important in mobile apps where redirect-based OAuth flows are common.

**OpenID Connect (OIDC)**
- URL: https://openid.net/connect/
- Identity layer on top of OAuth 2.0. Used for user authentication in financial applications; required by FAPI profiles. Provides standardised ID tokens and UserInfo endpoints.

**ISO/IEC 27001 — Information Security Management**
- URL: https://www.iso.org/standard/27001
- The international standard for information security management systems (ISMS). Essential for any platform handling sensitive financial data. SOC 2 Type II certification (AICPA) is the US-market equivalent and is the de-facto expectation for B2C fintech products.

**TLS 1.3 (RFC 8446)**
- URL: https://datatracker.ietf.org/doc/html/rfc8446
- Minimum transport security standard for all API communication. Replaces TLS 1.2 in financial-grade implementations. Required by FAPI 2.0.

---

### Securities & Financial Data Standards

**ISO 20022 — Universal Financial Industry Message Scheme**
- URL: https://www.iso20022.org/iso-20022
- The global standard for financial messaging, covering payments, securities, trade services, and cash management. Adopted by SWIFT and the US Federal Reserve (Fedwire, July 2025 onwards). Relevant to the investment tracking layer for data interchange with custodians, clearing houses, and portfolio management systems.

**ISIN — International Securities Identification Number (ISO 6166)**
- URL: https://www.iso.org/standard/6166
- 12-character global standard for uniquely identifying securities (equities, bonds, ETFs). Used by international brokerages, data providers, and portfolio analytics engines to identify holdings unambiguously across markets and currencies.

**CUSIP — Committee on Uniform Security Identification Procedures**
- URL: https://www.cusip.com/
- 9-character identifier for US and Canadian securities. The standard identifier used by US brokerages and the SEC. Portfolio tracking tools must map CUSIP to ISIN for cross-market holdings. CUSIP Global Services provides a Data API and ISIN mapping service.

**FIGI — Financial Instrument Global Identifier (ISO 4914)**
- URL: https://www.openfigi.com/
- Open, royalty-free global identifier for financial instruments maintained by the Object Management Group (OMG) and published by Bloomberg. Free alternative to CUSIP for financial instrument lookup and mapping.

**XBRL — eXtensible Business Reporting Language**
- URL: https://www.xbrl.org/
- Standard for tagging financial statements (balance sheets, income statements) in machine-readable format. SEC requires XBRL filing for public company financial reports. Relevant if the platform ingests or displays company financial data alongside portfolio holdings.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The standard for documenting RESTful APIs. Any public or internal API surface should be described in OpenAPI 3.1. Enables automatic SDK generation, mock servers, and API documentation tools.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/
- Standard for describing and validating JSON data structures. Used to define the data models for transactions, accounts, securities, and net worth snapshots.

**RFC 7231 — HTTP/1.1 Semantics**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines the semantics of HTTP methods (GET, POST, PUT, DELETE, PATCH) and status codes. Foundational reference for REST API design in financial services.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Standard for hypermedia link relations. Used in REST APIs for pagination (Link: header) and HATEOAS-style resource navigation.

---

### Compliance & Privacy Frameworks

**GDPR (General Data Protection Regulation)**
- URL: https://gdpr.eu/
- EU regulation governing the collection, storage, and processing of personal data. Applies to any product serving EU residents. Requires lawful basis for data processing, explicit consent, data minimisation, right to erasure, and data portability. Financial account data is sensitive personal data under GDPR Article 9.

**CCPA / CPRA (California Consumer Privacy Act / California Privacy Rights Act)**
- URL: https://oag.ca.gov/privacy/ccpa
- California's primary data privacy law, strengthened by CPRA (effective January 2026). Requires opt-out mechanisms for data sale/sharing, consumer rights to access and deletion, and specific obligations for service providers handling financial data.

**CFPB Section 1033 (Open Banking Rule)**
- URL: https://www.consumerfinance.gov/
- US Consumer Financial Protection Bureau rule requiring financial institutions to provide consumers and authorised third parties with access to financial account data via standardised APIs. Aligned with FDX API. Implemented progressively from 2026, starting with the largest institutions.

**SOC 2 Type II (AICPA Trust Services Criteria)**
- URL: https://www.aicpa.org/resources/landing/system-and-organization-controls-soc-suite-of-services
- US audit framework assessing security, availability, processing integrity, confidentiality, and privacy of a service organisation. The market expectation for consumer fintech products storing sensitive financial data. Requires a 6–12 month observation period with independent auditor assessment.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic's open protocol for connecting AI models to external data sources and tools. A personal finance tracker MCP server could expose: account balances, net worth, portfolio allocation, transaction history, and goal progress as structured resources that LLMs can query. Enables AI assistant integrations with full financial context without requiring credential re-entry in each AI session.

---

## Similar Products — Developer Documentation & APIs

### Plaid

- **Description:** The dominant US financial data aggregation API, powering connections for over 150 million consumers across 12,000+ financial institutions. Provides transaction data, account balances, investment holdings, liabilities, and identity verification.
- **API Documentation:** https://plaid.com/docs/
- **SDKs/Libraries:** JavaScript, Python, Ruby, Java, Go — https://plaid.com/docs/api/libraries/
- **Developer Guide:** https://plaid.com/docs/quickstart/
- **Standards:** REST/JSON; FDX-aligned; OAuth 2.0 for end-user authorisation
- **Authentication:** OAuth 2.0 (Plaid Link token flow); API key (client_id + secret) for server-to-server

### MX Technologies

- **Description:** Financial data platform aggregating accounts from 16,000+ institutions, providing transaction enrichment, categorisation, and white-label PFM (personal finance management) widgets. Second-largest US aggregator after Plaid.
- **API Documentation:** https://docs.mx.com/
- **SDKs/Libraries:** Ruby, Python, JavaScript, Java, PHP — https://docs.mx.com/api-reference/sdks/
- **Developer Guide:** https://docs.mx.com/api-reference/platform-api/overview/
- **Standards:** REST/JSON; FDX-aligned (75%+ direct API connections with institutions); OpenAPI documented
- **Authentication:** OAuth 2.0 for user authorisation; Basic Auth (client_id:client_secret) for server calls

### TrueLayer

- **Description:** European open banking API platform providing PSD2-compliant account data and payment initiation across the UK and EU. Leading aggregator for European market applications.
- **API Documentation:** https://docs.truelayer.com/
- **SDKs/Libraries:** Python, JavaScript, Java, Ruby, PHP — https://docs.truelayer.com/docs/client-libraries
- **Developer Guide:** https://docs.truelayer.com/docs/quickstart-retrieve-data
- **Standards:** REST/JSON; PSD2 compliant; Berlin Group NextGenPSD2 aligned; FAPI 1.0 security profile
- **Authentication:** OAuth 2.0 with PKCE for end-user authorisation; client credentials for server flows

### Nordigen / GoCardless Bank Account Data

- **Description:** Free-tier open banking data API (now GoCardless Bank Account Data) providing PSD2-compliant account and transaction data across 2,100+ European banks. Popular for cost-sensitive personal finance applications in Europe.
- **API Documentation:** https://developer.gocardless.com/bank-account-data/overview
- **SDKs/Libraries:** Python, JavaScript, PHP — https://developer.gocardless.com/bank-account-data/client-libraries
- **Developer Guide:** https://developer.gocardless.com/bank-account-data/quick-start-guide
- **Standards:** REST/JSON; PSD2 compliant; OpenAPI 3.0 documented
- **Authentication:** JWT bearer tokens; API key issuance via developer portal

### CUSIP Global Services (CGS) Data API

- **Description:** Official provider of CUSIP and ISIN securities identifiers for US and Canadian securities. API enables lookup, validation, and cross-referencing of security identifiers for portfolio management.
- **API Documentation:** https://www.cusip.com/swagger/cusipApi.html
- **SDKs/Libraries:** REST-only; no official SDK (standard HTTP)
- **Developer Guide:** https://www.cusip.com/identifiers.html
- **Standards:** REST/JSON; ISO 6166 (ISIN); ISO 4914 (FIGI mapping available)
- **Authentication:** API key (commercial licence required for production use)

### OpenFIGI (Bloomberg)

- **Description:** Free, open API for mapping between FIGI, CUSIP, ISIN, ticker, and exchange identifiers. Royalty-free alternative to CUSIP for instrument lookup in portfolio tracking applications.
- **API Documentation:** https://www.openfigi.com/api
- **SDKs/Libraries:** Python — https://github.com/OpenFIGI/figi-py; community libraries in JavaScript, Ruby
- **Developer Guide:** https://www.openfigi.com/api#post-v3-mapping
- **Standards:** REST/JSON; ISO 4914 (FIGI)
- **Authentication:** API key (free tier: 25 requests/minute; higher tiers available)

### Alpha Vantage

- **Description:** Financial market data API providing real-time and historical equity, ETF, forex, and cryptocurrency price data. Commonly used in personal finance applications for portfolio valuation and benchmark comparison.
- **API Documentation:** https://www.alphavantage.co/documentation/
- **SDKs/Libraries:** Python, JavaScript, R — https://www.alphavantage.co/code_chart/
- **Developer Guide:** https://www.alphavantage.co/documentation/#api-key
- **Standards:** REST/JSON; no formal standard — proprietary schema
- **Authentication:** API key (free tier: 25 requests/day; premium tiers from $50/month)

### Polygon.io

- **Description:** Real-time and historical market data API covering equities, options, forex, and crypto. Provides WebSocket feeds for live price updates and REST endpoints for historical OHLCV data, dividends, and splits.
- **API Documentation:** https://polygon.io/docs/
- **SDKs/Libraries:** Python, JavaScript, Go, Kotlin — https://polygon.io/docs/stocks/getting-started
- **Developer Guide:** https://polygon.io/docs/stocks/getting-started
- **Standards:** REST/JSON and WebSocket; OpenAPI documented
- **Authentication:** API key (free tier: unlimited calls, 15-minute delayed data; paid from $29/month for real-time)

### CoinGecko API

- **Description:** Free cryptocurrency market data API covering prices, market cap, trading volume, and token metadata for 10,000+ cryptocurrencies. The standard reference for consumer-facing crypto balance valuation.
- **API Documentation:** https://www.coingecko.com/api/documentation
- **SDKs/Libraries:** Python, JavaScript — https://github.com/man-c/pycoingecko
- **Developer Guide:** https://docs.coingecko.com/v3.0.1/reference/introduction
- **Standards:** REST/JSON; OpenAPI 3.0 documented
- **Authentication:** API key (free tier: 30 calls/minute; Pro from $129/month)

### Zillow API (Bridge Interactive / ATTOM)

- **Description:** Real estate valuation data. Zillow's direct API access is limited; ATTOM Data Solutions and Bridge Interactive (Zillow-owned) provide property valuation APIs for integration. Used by Kubera and Copilot for real estate net worth tracking.
- **API Documentation:** https://api.bridgeinteractive.com/ · https://api.gateway.attomdata.com/
- **SDKs/Libraries:** REST-only; no official SDK
- **Developer Guide:** https://bridgedataoutput.com/docs/explorer/
- **Standards:** REST/JSON; RESO Data Dictionary standard for property data
- **Authentication:** API key (commercial licence; pricing on request)

---

## Notes

**Aggregator coverage and cost**: Plaid and MX are the dominant US aggregators. Plaid's February 2026 $8 billion valuation round and JPMorgan's paid API deal signal that US open banking may evolve toward a paid-access model, increasing costs for aggregation-dependent applications. Self-hosted aggregation alternatives (Actual Budget, Firefly III) avoid these costs but require users to maintain bank connections manually.

**CFPB 1033 and FDX adoption**: The CFPB's Section 1033 rule (phased from 2026) will mandate FDX-compliant APIs at major US banks, reducing reliance on credential-based scraping. This lowers long-term aggregation risk but requires ongoing compliance monitoring.

**MCP opportunity**: No mainstream personal finance product currently exposes an MCP server. An open-source tracker with an MCP interface could enable tight integration with AI assistants (Claude, ChatGPT, Copilot) without requiring custom API connectors, a significant developer-experience advantage.

**Crypto data sourcing**: CoinGecko's free tier is sufficient for MVP crypto valuation. For DeFi position tracking (on-chain balances, staking yields), dedicated providers such as Zerion or Zapper offer APIs but at higher complexity and cost.

**Tax calculation standards**: The IRS defines cost basis accounting methods (FIFO, LIFO, HIFO, specific identification) and wash sale rules (Section 1091) in regulatory guidance, not in formal technical standards. Implementation should reference IRS Publication 550 (Investment Income and Expenses) and Form 8949 instructions.
