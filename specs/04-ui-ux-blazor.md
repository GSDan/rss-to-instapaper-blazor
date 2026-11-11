# 04 - UI & UX Specification (Blazor WASM)

## 1. Route & Navigation Map

| Route | Authorization | Purpose / Description |
|---|---|---|
| `/setup` | Anonymous (Unprovisioned only) | First-run setup wizard: Seeds admin user and global Instapaper API consumer keys. |
| `/login` | Anonymous | Cookie authentication form. |
| `/register` | Anonymous | Invite token acceptance and user account registration. |
| `/` | Authorize | High-level feed dashboard showing cards with health status, statistics, and sync schedules. |
| `/settings` | Authorize | Personal settings: Instapaper user token configuration, password change. |
| `/admin` | Authorize (Admin only) | Invite generation, user list, and global application configuration updates. |
| `/feeds/{id}` | Authorize | Feed drill-down: Per-article status table, error messages, and manual sync action. |

---

## 2. Component Design & Interactivity

### First-Run Setup Wizard (`/setup`)
- Redirects automatically to `/` if instance is already provisioned.
- Form inputs:
  - Admin Username
  - Admin Password (masked)
  - Instapaper Consumer Key
  - Instapaper Consumer Secret
- On completion, seeds SQLite, clears setup state, signs in admin, and redirects to dashboard.

### Main Feed Dashboard (`/`)
- Shows user's active feeds with masked URLs for privacy.
- Status Badges:
  - `Healthy` (Green)
  - `Warning` (Amber - indicating articles undergoing retry backoff)
  - `Error` (Red - consecutive poll failures or invalid credentials)
- Quick Stats: Total items synced, last poll time, next scheduled poll.
- Direct link to drill-down view (`/feeds/{id}`).

### Add Feed Modal with Live Backfill Preview
- Feed URL field with on-blur inspection (`POST /api/feeds/inspect`).
- Interactive backfill configuration:
  - Number input: `[ 5 ]`
  - Unit dropdown: `[ Days | Weeks | Months ]`
  - Reactive label: `Import last [ 5 ] [ Months ▾ ] (22 items found)`
- Content Mode Options:
  - Option 1: Send URL only (Instapaper scrapes webpage).
  - Option 2: Push article content directly from feed.
- Content Warning Banner:
  - Appears dynamically if feed items lack `<content:encoded>` or full HTML tags.
  - Checkbox: "Fall back to URL scrape if full content is missing" (default: checked).

### Feed Article Drill-Down (`/feeds/{id}`)
- Displays table of articles discovered for the feed:
  - Date published
  - Article title & link
  - Delivery mode (`Direct Content` vs `URL Scrape`)
  - Status badge (`Pushed`, `Pending`, `Retrying (x/3)`, `Failed`)
  - Error tooltip for failed/retrying articles
  - "Manual Sync" button for on-demand retry of specific items.
