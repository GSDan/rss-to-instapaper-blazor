# RSS to Instapaper Sync Appliance (SDD Project)

A containerized, self-hosted .NET Blazor WebAssembly application designed to ingest personal RSS feeds and push articles directly into Instapaper accounts.

## Key Features
- **Multi-Tenant User Isolation:** Admin-controlled, invite-only user registration.
- **Strict Feed Privacy:** Feeds are 100% private to each user (ideal for tokenized RSS feeds).
- **Flexible Content Delivery:** Push full HTML content directly or let Instapaper scrape the URL, with automatic fallback and content warnings.
- **Configurable Backfill:** Select historical backfill window (`[N] [Days/Weeks/Months]`) on feed creation with real-time item count preview.
- **Robust Background Sync:** Native `BackgroundService` with `PeriodicTimer` and exponential backoff retry semantics.
- **Zero-Config Homelab Deployment:** Single Docker container running ASP.NET Core + Blazor WASM + SQLite with volume-persisted Data Protection encryption keys.

## Specifications
Refer to the `specs/` directory for full Spec-Driven Development (SDD) documentation:
- [01. Architecture & SRS](specs/01-architecture-srs.md)
- [02. Data Models & Entities](specs/02-data-models.md)
- [03. API & Service Contracts](specs/03-api-contracts.md)
- [04. UI & UX (Blazor WASM)](specs/04-ui-ux-blazor.md)
- [05. Testing Strategy & Unit Tests](specs/05-testing-strategy.md)
- [06. Deployment & Docker](specs/06-deployment-docker.md)
