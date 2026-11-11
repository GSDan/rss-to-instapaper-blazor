# CLAUDE.md - Development & Architectural Guidelines

## Project Overview
**RSS-to-Instapaper** is a self-hosted, multi-user .NET Blazor WebAssembly appliance designed for homelab/Docker deployments. It continuously monitors private/public RSS feeds and syncs articles to users' personal Instapaper accounts via the Instapaper API v2 (`https://www.instapaper.com/api/2/bookmarks`).

## Core Tech Stack
- **Target Framework:** .NET 9 / .NET 10 (`net9.0`)
- **Frontend:** Blazor WebAssembly (hosted via ASP.NET Core Host / BFF pattern)
- **Backend / Host:** ASP.NET Core Minimal APIs & Background Workers
- **Database / ORM:** SQLite with Entity Framework Core (WAL mode enabled)
- **Encryption:** `Microsoft.AspNetCore.DataProtection` persisted to volume (`/app/data/keys`)
- **Testing:** xUnit, FluentAssertions, NSubstitute, Microsoft.EntityFrameworkCore.Sqlite (In-memory)

## Architecture & SDD Rules
1. **Spec Compliance:** Always cross-reference `/specs/` before implementing any models, endpoints, or UI components.
2. **BFF Authentication:** Use cookie-based ASP.NET Core Identity authentication (`SameSite=Lax`, `HttpOnly`). Never store JWTs or tokens in browser `localStorage`.
3. **Strict Tenancy Isolation:** Feeds and feed URLs are strictly private and user-isolated. Never expose one user's feed data to another.
4. **Resilience & Polling:** Background ingestion uses native `BackgroundService` and `PeriodicTimer`. Never block the thread pool. Use bounded channels or controlled concurrency (`Parallel.ForEachAsync` with max degree of 4).
5. **Instapaper Ingestion:** Always target `https://www.instapaper.com/api/2/bookmarks`. Requests must be signed with global Consumer Key/Secret + decrypted user OAuth token/secret.
6. **Error Handling:** Follow the exponential backoff policy (attempts 1-3) before flagging articles as `Failed`. Feeds with failed articles report `Warning` status on dashboards.

## Common CLI Commands
- `dotnet build`
- `dotnet test`
- `dotnet run --project src/Server`
- `docker compose up --build -d`

## Project Structure
```
├── CLAUDE.md                   # AI Assistant instructions & architecture rules
├── README.md                   # Repository overview & setup
├── docker-compose.yml          # Production homelab deployment
├── specs/                      # Spec-Driven Development Specifications
│   ├── 01-architecture-srs.md  # System requirements & host architecture
│   ├── 02-data-models.md       # Entity models, SQLite & EF Core schema
│   ├── 03-api-contracts.md     # Minimal API endpoints & DTO contracts
│   ├── 04-ui-ux-blazor.md      # Blazor WASM screens, routes & state
│   ├── 05-testing-strategy.md  # Unit test suite & validation cases
│   └── 06-deployment-docker.md # Dockerfile, volume layouts & security
└── src/                        # Source implementation (Client, Server, Shared)
```
