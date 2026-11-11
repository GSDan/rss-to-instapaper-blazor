# 01 - System Requirements & Architectural Specification (SRS / SAD)

## 1. Architectural Overview & Hosting Constraints
The application is structured as a single-container, self-hosted appliance implementing the **Backend-for-Frontend (BFF)** pattern. An ASP.NET Core host serves a Blazor WebAssembly (WASM) Single Page Application, exposes authenticated Minimal APIs, executes asynchronous background feed orchestration, and manages local persistence.

```
+-----------------------------------------------------------------------------------+
| Host Container (Docker: mcr.microsoft.com/dotnet/aspnet:9.0)                      |
|                                                                                   |
|  +---------------------------+       +-----------------------------------------+  |
|  | Blazor WebAssembly (WASM) | <---> | ASP.NET Core Host                       |  |
|  | SPA Frontend              |       | - BFF Anti-Forgery / SameSite Cookies   |  |
|  +---------------------------+       | - Minimal APIs                          |  |
|                                      | - Feed Ingestion (BackgroundService)    |  |
|                                      | - Instapaper API v2 Client              |  |
|                                      +-----------------------------------------+  |
|                                                           |                       |
|                                           +---------------+---------------+       |
|                                           |                               |       |
|                                           v                               v       |
|                            +----------------------------+   +-------------------+ |
|                            | EF Core SQLite Provider    |   | Data Protection   | |
|                            | (/app/data/app.db)         |   | (/app/data/keys)  | |
|                            +----------------------------+   +-------------------+ |
+-------------------------------------------|-------------------------------|-------+
                                            v                               v
                         Mounted Host Volume (/var/lib/docker/volumes/... -> /app/data)
```

### Container & Filesystem Topography
- **Container Base Image:** `mcr.microsoft.com/dotnet/aspnet:9.0` (Alpine or Chiseled Ubuntu).
- **Mounted Volume Directory:** Single volume mount target at `/app/data`.
  - `/app/data/app.db`: SQLite database file (`PRAGMA journal_mode=WAL;`).
  - `/app/data/keys`: Ephemeral key repository used by `Microsoft.AspNetCore.DataProtection` to ensure encryption keys survive container destruction, recreation, and updates.
- **Network Boundary:** Ingress via reverse proxy (Caddy, Traefik, Nginx) over internal Docker bridge networks; listens internally on port `8080` (HTTP).

---

## 2. Authentication, Authorization & Security Architecture

### First-Run State Detection & Bootstrapping
1. On application startup, a middleware/filter evaluates `SystemStateService.IsProvisionedAsync()`.
2. If no administrative user exists in SQLite, the system enters an unprovisioned state:
   - All API routes return `428 Precondition Required` except for `/api/bootstrap/*`.
   - Blazor WASM automatically routes any incoming navigation to `/setup` (Interactive First-Run Wizard).
3. The wizard accepts:
   - Global Instapaper **Consumer Key** and **Consumer Secret**.
   - Admin **Username** and **Password** (Argon2id/PBKDF2 via ASP.NET Core Identity).
4. Upon successful submission, the system writes global credentials, seeds the primary admin account, and sets `IsProvisioned = true`. Subsequent calls to `/api/bootstrap` return `403 Forbidden`.

### Session Management & Anti-Forgery
- **BFF Cookie Flow:** Authentication utilizes ASP.NET Core Identity cookie authentication (`Identity.Application`).
- **Cookie Flags:** `HttpOnly = true`, `Secure = SameAsRequest` (allows local LAN HTTP or reverse-proxy HTTPS via `X-Forwarded-Proto`), `SameSite = SameSiteMode.Lax`.
- **Anti-Forgery:** State-changing endpoints (`POST`, `PUT`, `DELETE`) require a valid request token via `X-XSRF-TOKEN` header validated by ASP.NET Core AntiForgery middleware.

### Token Encryption at Rest
- User-specific Instapaper OAuth tokens and secrets are stored in the database as encrypted strings.
- Encryption is implemented via `IDataProtector` from `Microsoft.AspNetCore.DataProtection`:
  ```csharp
  services.AddDataProtection()
      .PersistKeysToFileSystem(new DirectoryInfo(@"/app/data/keys"))
      .SetApplicationName("RssToInstapaperApp");
  ```
- Domain service `ITokenEncryptionService` encapsulates `Protect()` and `Unprotect()` operations before persisting or fetching tokens.

---

## 3. Background Ingestion & Scheduling Engine

### Concurrency & Engine Lifecycle
- Ingestion is implemented via a native `BackgroundService` (`FeedPollingBackgroundService`).
- Uses internal `PeriodicTimer` (evaluating every 60 seconds) to query SQLite for active feeds where `NextPollUtc <= DateTime.UtcNow`.
- Feeds are dispatched across a bounded worker channel (`Channel<Guid>`) using `Parallel.ForEachAsync` (default concurrency: 4 workers).

### Instapaper API v2 Ingestion Specifications
- **Base URL:** `https://www.instapaper.com/api/2/bookmarks`
- **OAuth 1.0a Signing:** Requests are signed using the global **Consumer Key**, **Consumer Secret**, and the feed owner's decrypted **User Token** and **User Secret**.
- **Payload Mapping:**
  - `url`: Canonical article link from RSS `<link>` node.
  - `title`: Article title.
  - `content`: Article HTML payload (populated only when feed configuration specifies direct content ingest and valid markup exists).
  - `resolve_final_url`: `1` (ensures Instapaper follows redirects).
