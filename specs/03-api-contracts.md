# 03 - API & Service Contracts Specification

All endpoints are hosted by the ASP.NET Core backend. Authentication is enforced via session cookie unless marked `[Anonymous]`. State-mutating routes validate anti-forgery tokens (`X-XSRF-TOKEN`).

## 1. System & Authentication Endpoints

### `POST /api/bootstrap` [Anonymous]
Provisions the initial instance. Active only when `AppSettings.IsProvisioned == false`.
```json
// Request
{
  "adminUsername": "admin",
  "adminPassword": "SuperSecretPassword123!",
  "instapaperConsumerKey": "key_xyz",
  "instapaperConsumerSecret": "secret_abc"
}
// Response: 200 OK or 403 Forbidden (if already provisioned)
```

### `POST /api/auth/login` [Anonymous]
```json
// Request
{
  "username": "dan",
  "password": "UserPassword123!"
}
// Response: 200 OK (with Set-Cookie) | 401 Unauthorized
```

### `POST /api/auth/logout` [Authorize]
Clears authentication cookie. Response: `200 OK`.

### `GET /api/auth/me` [Authorize]
```json
// Response: 200 OK
{
  "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "username": "dan",
  "role": "Admin",
  "hasInstapaperCredentials": true
}
```

---

## 2. User & Invite Administration Endpoints

### `POST /api/invites` [Authorize(Roles = "Admin")]
```json
// Request
{
  "expiresInDays": 7
}
// Response: 201 Created
{
  "inviteToken": "inv_a8f9d0124c944ebc",
  "inviteUrl": "/register?token=inv_a8f9d0124c944ebc",
  "expiresUtc": "2026-11-18T16:00:00Z"
}
```

### `POST /api/auth/register-with-token` [Anonymous]
```json
// Request
{
  "inviteToken": "inv_a8f9d0124c944ebc",
  "username": "newuser",
  "password": "Password987!"
}
// Response: 201 Created | 400 Bad Request
```

### `PUT /api/user/credentials` [Authorize]
```json
// Request
{
  "instapaperToken": "user_oauth_token",
  "instapaperTokenSecret": "user_oauth_secret"
}
// Response: 200 OK
```

---

## 3. Feed Management Endpoints

### `POST /api/feeds/inspect` [Authorize]
Inspects an RSS URL in real time prior to creation.
```json
// Request
{
  "feedUrl": "https://private-feed.example.com/rss?token=abc",
  "backfillValue": 3,
  "backfillUnit": "Months" // "Days", "Weeks", "Months"
}

// Response: 200 OK
{
  "title": "Private Security Research Feed",
  "totalItemsInFeed": 45,
  "itemsMatchingBackfill": 18,
  "hasFullContentBodies": false,
  "itemsMissingContentCount": 18,
  "warning": "This feed does not provide full HTML content in <content:encoded> nodes. Articles will be ingested as summaries unless URL fallback is enabled."
}
```

### `POST /api/feeds` [Authorize]
```json
// Request
{
  "feedUrl": "https://private-feed.example.com/rss?token=abc",
  "title": "Private Security Research Feed",
  "pollIntervalMinutes": 120,
  "preferFeedContent": true,
  "fallbackToUrlIfContentMissing": true,
  "backfillValue": 3,
  "backfillUnit": "Months"
}
// Response: 201 Created
```

### `GET /api/feeds` [Authorize]
```json
// Response: 200 OK
[
  {
    "id": "7cb77c08-e818-40ff-9076-78c6670868f7",
    "title": "Private Security Research Feed",
    "feedUrl": "https://private-feed.example.com/rss?token=abc",
    "pollIntervalMinutes": 120,
    "preferFeedContent": true,
    "fallbackToUrlIfContentMissing": true,
    "healthStatus": "Warning",
    "lastErrorMessage": "Article push failed (HTTP 429). Retrying in 15 minutes.",
    "lastPolledUtc": "2026-11-11T15:30:00Z",
    "totalArticlesTracked": 18,
    "failedArticlesCount": 1
  }
]
```

### `GET /api/feeds/{id}/articles` [Authorize]
```json
// Query: ?page=1&pageSize=20&statusFilter=Failed
// Response: 200 OK
{
  "items": [
    {
      "id": "9bb67f81-9b16-419b-a0da-b7b51b7524bb",
      "title": "Zero-Day Vulnerability Report",
      "articleUrl": "https://private-feed.example.com/posts/zero-day",
      "publishedUtc": "2026-11-10T08:00:00Z",
      "status": "Failed",
      "deliveryType": "UrlScrapeFallback",
      "retryCount": 3,
      "lastErrorMessage": "Instapaper API Rate limit exceeded (HTTP 429)"
    }
  ],
  "totalCount": 1,
  "page": 1,
  "pageSize": 20
}
```

### `POST /api/feeds/{feedId}/articles/{articleId}/sync` [Authorize]
Triggers an immediate manual sync for a specific article.
```json
// Response: 200 OK
{
  "status": "Pushed",
  "syncedAtUtc": "2026-11-11T16:15:00Z"
}
```
