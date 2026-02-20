---
name: smb-sales-boost
description: Query and manage leads from the SMB Sales Boost B2B lead database. Search newly registered businesses, filter by location/industry/keywords, export leads, manage filter presets, and use AI-powered category suggestions. Requires an active SMB Sales Boost subscription (Pro, Platinum, Enterprise, or Agency) and API key.
---

# SMB Sales Boost Skill

This skill enables natural language interaction with the SMB Sales Boost API — a B2B lead generation platform providing access to newly registered small and medium businesses across the United States.

## Setup

The user must provide their API key. Keys have a `smbk_` prefix and are generated from the Dashboard > API tab. The key is passed as a Bearer token in the Authorization header of every request.

**Base URL:** `https://smbsalesboost.com/api/v1`

**Important:** API access requires a Pro, Platinum, Enterprise, or Agency subscription plan.

## Authentication

All requests must include:
```
Authorization: Bearer <API_KEY>
```

If the user hasn't provided their API key yet, ask them for it before making any requests. Store it in a variable for reuse throughout the session.

## Rate Limits

- Exports: 1 export per 5 minutes
- AI category suggestions: 5 requests per minute
- AI keyword generation: 5 requests per minute

Rate limit headers are returned on every response: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`. If rate limited, check the `Retry-After` header for seconds to wait.

## Two Database Types

SMB Sales Boost has two separate databases:

1. **`home_improvement`** — Home improvement/contractor businesses with fields like star ratings, review counts, review snippets, phone numbers, profile URLs, and categories
2. **`other`** — General newly registered businesses with fields like registered URLs, URL registration dates, crawled URLs, descriptions, emails, phone numbers, and redirect status

Some filter parameters only work with one database type. The user's account has a default database setting. Always check which database the user wants to query.

---

## Core Endpoints

### 1. Search Leads — `GET /leads`

The primary endpoint. Translates natural language queries into filtered lead searches.

**Key Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `page` | integer | Page number (default: 1) |
| `limit` | integer | Results per page (max 1000, default 100) |
| `database` | string | `home_improvement` or `other` |
| `positiveKeywords` | JSON array string | Keywords to include (OR logic) |
| `negativeKeywords` | JSON array string | Keywords to exclude (AND logic) |
| `orColumns` | JSON array string | Column names to search keywords against |
| `stateInclude` | string | Comma-separated state codes: `CA,NY,TX` |
| `stateExclude` | string | Comma-separated state codes to exclude |
| `cityInclude` | JSON array string | Cities to include |
| `cityExclude` | JSON array string | Cities to exclude |
| `zipInclude` | JSON array string | ZIP codes to include |
| `zipExclude` | JSON array string | ZIP codes to exclude |
| `nameIncludeTerms` | JSON array string | Business name include terms |
| `nameExcludeTerms` | JSON array string | Business name exclude terms |
| `lastUpdatedFrom` | date string | Filter by Last Updated date (after this date) |
| `lastUpdatedTo` | date string | Filter by Last Updated date (before this date) |

**Understanding "Last Updated" — this is critical for finding the freshest leads:**
- **Home Improvement leads:** Last Updated means a new phone number was detected
- **Other leads:** Last Updated means the primary phone number and/or primary email address was updated
- Both databases also include newly added records in this date
- Many businesses launch a website before adding contact info, so the Last Updated date captures when that information first becomes available — making it the primary way to identify the most actionable leads
| `countryInclude` | JSON array string | Countries to include |
| `countryExclude` | JSON array string | Countries to exclude |
| `sortBy` | string | Field to sort by |
| `sortOrder` | string | `asc` or `desc` (default: `desc`) |

**Home Improvement Only:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `minStars` / `maxStars` | number | Star rating range |
| `minReviewCount` / `maxReviewCount` | integer | Review count range |
| `categoriesIncludeTerms` / `categoriesExcludeTerms` | JSON array string | Category filters |
| `reviewSnippetIncludeTerms` / `reviewSnippetExcludeTerms` | JSON array string | Review text filters |
| `profileUrlIncludeTerms` / `profileUrlExcludeTerms` | JSON array string | Profile URL filters |

**Other Database Only:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `urlIncludeTerms` / `urlExcludeTerms` | JSON array string | Registered URL filters |
| `crawledUrlIncludeTerms` / `crawledUrlExcludeTerms` | JSON array string | Crawled URL filters |
| `descriptionIncludeTerms` / `descriptionExcludeTerms` | JSON array string | Short description filters |
| `descriptionLongIncludeTerms` / `descriptionLongExcludeTerms` | JSON array string | Long description filters |
| `emailPrimaryInclude` / `emailPrimaryExclude` | JSON array string | Primary email filters |
| `phonePrimaryInclude` / `phonePrimaryExclude` | JSON array string | Primary phone filters |
| `redirectFilter` | string | `yes` or `no` — filter by redirect status |

**Important:** At least one positive filter is required (positiveKeywords or any column-specific include terms).

**Response includes:** `leads` array, `totalCount`, `page`, `limit`, `databaseType`

**Lead fields:** `id`, `companyName`, `state`, `city`, `zip`, `phone`, `email`, `categories`, `lastUpdated` (phone/email masked for free users). The `lastUpdated` field indicates when contact information was last detected or updated — this is the best indicator of lead freshness and actionability.

### 2. Export Leads — `POST /leads/export`

Export filtered leads as CSV, JSON, or XLSX files.

**Request body:**
```json
{
  "database": "home_improvement" | "other",
  "filters": { /* same filter params as GET /leads */ },
  "selectedIds": [1, 2, 3],  // alternative to filters
  "formatId": 123  // optional export format template ID
}
```

**Response:** `files` array (with base64-encoded data), `leadCount`, `exportId`, `databaseType`

Rate limited: 1 export per 5 minutes, max 10,000 leads per export.

### 3. Filter Presets — `/filter-presets`

- `GET /filter-presets` — List all saved presets
- `POST /filter-presets` — Create a preset (requires `name` and `filters` object)
- `DELETE /filter-presets/{id}` — Delete a preset

### 4. Keyword Lists — `/keyword-lists`

- `GET /keyword-lists` — List all keyword lists
- `POST /keyword-lists` — Create (requires `name`, optional `keywords` and `negativeKeywords` arrays)
- `PUT /keyword-lists/{id}` — Update
- `DELETE /keyword-lists/{id}` — Delete

### 5. Email Schedules — `/email-schedules`

- `GET /email-schedules` — List schedules
- `POST /email-schedules` — Create (requires `name`, `filterPresetId`, `intervalValue`, `intervalUnit`)
- `PATCH /email-schedules/{id}` — Update (supports `isActive` toggle)
- `DELETE /email-schedules/{id}` — Delete

### 6. Export Formats — `/export-formats`

- `GET /export-formats` — List custom export formats
- `POST /export-formats` — Create (requires `name`, supports `fileType`, `fieldMappings`, split settings)
- `GET /export-formats/{id}` — Get specific format
- `PATCH /export-formats/{id}` — Update
- `DELETE /export-formats/{id}` — Delete
- `POST /export-formats/{id}/set-default` — Set as default

### 7. Export History — `/export-history`

- `GET /export-history` — List past exports (optional `limit` param, default 50)
- `GET /export-history/{id}/download` — Re-download (expires after 7 days)

### 8. AI Features

**`POST /ai/suggest-categories`** — Get AI category suggestions based on company profile.

Required: `companyName`, `companyDescription`, `productService`
Optional: `companyWebsite`, `smbType`, `excludeCategories`

**`POST /ai/generate-keywords`** — Trigger async keyword generation based on profile and target categories.

### 9. Export Blacklist — `/export-blacklist`

- `GET /export-blacklist` — List blacklisted entries
- `POST /export-blacklist` — Add entry (single or batch via `entries` array)
- `DELETE /export-blacklist/{id}` — Remove entry

### 10. Account

- `GET /me` — Get user profile (subscription, settings, onboarding status)
- `PATCH /me` — Update profile (firstName, lastName, companyName, companyWebsite)
- `GET /settings/database` — Check current database type and switch availability
- `POST /settings/switch-database` — Switch between databases (has cooldown)

---

## Natural Language Translation Guide

When users make natural language requests, translate them into API calls:

| User Says | API Call |
|-----------|---------|
| "Find plumbing businesses in Texas" | `GET /leads?positiveKeywords=["plumbing"]&stateInclude=CA` |
| "Show me new restaurants in NYC" | `GET /leads?positiveKeywords=["restaurant"]&cityInclude=["New York"]` |
| "Get 50 leads with high ratings" | `GET /leads?limit=50&minStars=4` (home_improvement) |
| "Export all my filtered results" | `POST /leads/export` with current filters |
| "What categories should I target?" | `POST /ai/suggest-categories` |
| "Save this search for later" | `POST /filter-presets` |
| "Show my recent exports" | `GET /export-history` |
| "What plan am I on?" | `GET /me` |
| "Exclude these domains from exports" | `POST /export-blacklist` |

## Building API Requests

Use `curl` or equivalent HTTP requests. Example:

```bash
curl -s "https://smbsalesboost.com/api/v1/leads?positiveKeywords=%5B%22plumbing%22%5D&stateInclude=TX&limit=10" \
  -H "Authorization: Bearer smbk_xxxxxxxxxxxx" \
  -H "Content-Type: application/json"
```

**Remember:**
- JSON array parameters must be URL-encoded (e.g., `["plumbing"]` → `%5B%22plumbing%22%5D`)
- At least one positive filter is required for lead searches
- Check which database the user needs before applying database-specific filters
- Phone and email are masked for free-tier users
- Present results in a clean, readable table format
- When exporting, the response contains base64-encoded file data — decode and save as a file for the user

## Error Handling

| Status | Meaning |
|--------|---------|
| 400 | Bad request — check parameters |
| 401 | Invalid or missing API key |
| 403 | Active subscription required |
| 404 | Resource not found |
| 429 | Rate limited — check `Retry-After` header |
| 500 | Server error |

All errors return: `{ "error": "error_code", "message": "Human-readable message" }`
