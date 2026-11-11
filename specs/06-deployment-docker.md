# 06 - Deployment & Docker Specification

## 1. Docker Compose Configuration
```yaml
services:
  rss-to-instapaper:
    image: ghcr.io/dan/rss-to-instapaper:latest
    container_name: rss-to-instapaper
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ASPNETCORE_URLS=http://+:8080
      - DOTNET_RUNNING_IN_CONTAINER=true
    volumes:
      - ./data:/app/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/healthz"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
```

## 2. Directory Layout on Host Mount (`./data:/app/data`)
```
/app/data/
├── app.db               # SQLite database file
├── app.db-wal           # SQLite Write-Ahead Log
├── app.db-shm           # SQLite Shared Memory
└── keys/                # DataProtection XML key repository
    └── key-xxxxxxxx.xml
```

## 3. Reverse Proxy Configuration Example (Caddy)
```caddyfile
rss.home.internal {
    reverse_proxy rss-to-instapaper:8080 {
        header_up X-Forwarded-Proto {scheme}
    }
}
```
