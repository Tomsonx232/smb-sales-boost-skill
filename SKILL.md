---
name: smb-sales-boost
description: Query and manage leads from the SMB Sales Boost B2B lead database. Search newly registered businesses, filter by location/industry/keywords, export leads, manage filter presets, and use AI-powered category suggestions. Requires an active SMB Sales Boost subscription (Starter, Growth, Scale, Platinum, or Enterprise) and API key.
---

# SMB Sales Boost Skill

This skill enables natural language interaction with the SMB Sales Boost API — a B2B lead generation platform providing access to newly registered small and medium businesses across the United States.

## Setup

The user must provide their API key. Keys have a `smbk_` prefix and are generated from the Dashboard > API tab. The key is passed as a Bearer token in the Authorization header of every request.

**Base URL:** `https://smbsalesboost.com/api/v1`

**Important:** API access requires a Starter, Growth, Scale, Platinum, or Enterprise subscription plan. New users can purchase a subscription entirely via API using the Programmatic Purchase endpoints (no web signup required).

## Authentication

All requests must include:
```
Authorization: Bearer <API_KEY>
```

If the user hasn't provided their API key yet, ask them for it before making any requests. Store it in a variable for reuse throughout the session.

## Credit-Based System

Starter, Growth, and Scale plans use a **credit-based model** for lead exports:

- Each **net-new lead exported** deducts 1 credit
- **Previously-exported leads** are free (do not consume credits)
- Platinum and Enterprise plans are not credit-limited

**Credit Pricing (per credit):**
| Plan | Cost per Credit |
|------|----------------|
| Starter | $0.10 |
| Growth | $0.08 |
| Scale | $0.05 |
| Platinum | $0.03 |
| Enterprise | $0.02 |

Users can purchase additional permanent credits via `POST /purchase-credits`.

## Rate Limits

- Exports: 1 per 5 minutes
- Email schedule trigger: 1 per 5 minutes
- AI category suggestions: 5 per minute
- AI keyword generation: 5 per minute
- AI auto-refine enable: 5 per minute
- AI auto-refine disable: 60 per minute
- AI auto-refine status: 60 per minute
- AI keyword status: 60 per minute
- Programmatic purchase: 5 per hour per IP
- Claim key: 30 per hour per IP

Rate limit headers are returned on every response: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`. If rate limited, check the `Retry-After` header for seconds to wait.

## Two Database Types

SMB Sales Boost has two separate databases with different contact information available:

1. **`home_improvement`** — Home improvement/contractor businesses with **phone numbers**, star ratings, review counts, review snippets, profile URLs, and categories
2. **`other`** — General newly registered businesses with **phone numbers and email addresses**, registered URLs, crawled URLs, short/long descriptions, and redirect status

The Home Improvement database provides phone numbers as the primary contact method. The Other database provides both phone numbers and email addresses, making it ideal for cold email and multi-channel outreach campaigns.

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
| `positiveKeywords` | JSON array string | Keywords to include (OR logic). Supports `*` wildcard for pattern matching (e.g., `["*dental*", "*ortho*"]`). Without wildcards, performs substring matching by default. |
| `negativeKeywords` | JSON array string | Keywords to exclude (AND logic). Also supports `*` wildcard (e.g., `["*franchise*"]`). |
| `orColumns` | JSON array string | Column names to search keywords against |
| `search` | string | Full-text search across all fields |
| `stateInclude` | string | Comma-separated state codes: `CA,NY,TX` |
| `stateExclude` | string | Comma-separated state codes to exclude |
| `cityInclude` | JSON array string | Cities to include |
| `cityExclude` | JSON array string | Cities to exclude |
| `zipInclude` | JSON array string | ZIP codes to include |
| `zipExclude` | JSON array string | ZIP codes to exclude |
| `nameIncludeTerms` | JSON array string | Business name include terms |
| `nameExcludeTerms` | JSON array string | Business name exclude terms |
| `lastUpdatedFrom` | date string | Filter by Last Updated date (after this date). Supports ISO 8601 or relative format (e.g., `rel:7d`, `rel:6m`). |
| `lastUpdatedTo` | date string | Filter by Last Updated date (before this date) |
| `updateReasonFilter` | string | Comma-separated update reasons to filter by (e.g., "Newly Added", "Phone Primary") |

**Understanding "Last Updated" — this is critical for finding the freshest leads:**
- **Home Improvement leads:** Last Updated means a new phone number was detected
- **Other leads:** Last Updated means any of the 5 contact/address fields changed: primary phone, secondary phone, primary email, secondary email, or full address
- Both databases also include newly added records in this date
- Many businesses launch a website before adding contact info, so the Last Updated date captures when that information first becomes available — making it the primary way to identify the most actionable leads

| Parameter | Type | Description |
|-----------|------|-------------|
| `countryInclude` | JSON array string | Countries to include |
| `countryExclude` | JSON array string | Countries to exclude |
| `sortBy` | string | Field to sort by |
| `sortOrder` | string | `asc` or `desc` (default: `desc`) |

**Wildcard Keyword Tips:**
- Use `*` to match any characters: `"*dental*"` matches "dental clinic", "pediatric dentistry", etc.
- Combine wildcards for compound terms: `"*auto*repair*"` matches "auto body repair", "automotive repair shop", etc.
- Use multiple keyword variations for broader coverage: `["*dental*", "*dentist*", "*orthodont*"]`
- Keywords without wildcards still perform substring matching by default

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
| `emailSecondaryInclude` / `emailSecondaryExclude` | JSON array string | Secondary email filters |
| `phonePrimaryInclude` / `phonePrimaryExclude` | JSON array string | Primary phone filters |
| `phoneSecondaryInclude` / `phoneSecondaryExclude` | JSON array string | Secondary phone filters |
| `redirectFilter` | string | `yes` or `no` — filter by redirect status |
| `registrationDateFrom` / `registrationDateTo` | date string | Filter by domain registration date (ISO 8601 or relative format e.g., `rel:6m`) |
| `timeScrapedFrom` / `timeScrapedTo` | date string | Filter by when leads were scraped (ISO 8601 or relative format e.g., `rel:30d`) |
| `websiteSchemaFilter` | string | Comma-separated website schema types (e.g., `LocalBusiness,Organization`). Use `GET /leads/other/schema-types` for available values. |

**Important:** At least one positive filter is required (positiveKeywords or any column-specific include terms).

**Response includes:** `leads` array, `totalCount`, `page`, `limit`, `databaseType`

**Lead fields:** `id`, `companyName`, `state`, `city`, `zip`, `phone`, `email`, `categories`, `lastUpdated` (phone/email masked for free users). The `lastUpdated` field indicates when contact information was last detected or updated — this is the best indicator of lead freshness and actionability.

### 2. Website Schema Types — `GET /leads/other/schema-types`

Returns a sorted list of all distinct website schema types found in the Other leads database. Use these values with the `websiteSchemaFilter` parameter on `GET /leads`.

### 3. Export Leads — `POST /leads/export`

Export filtered leads as CSV, JSON, or XLSX files.

**Request body:**
```json
{
  "database": "home_improvement" | "other",
  "filters": { /* same filter params as GET /leads */ },
  "selectedIds": [1, 2, 3],  // alternative to filters
  "formatId": 123,  // optional export format template ID
  "maxLeads": 500,  // optional: cap leads per export, overflow stored in reservoir
  "maxResults": 1000,  // optional: total leads (new + previously-exported)
  "maxCredits": 100  // optional: credit spending cap (0 = only previously-exported leads)
}
```

**Credit system (Starter/Growth/Scale plans):**
- Each net-new lead exported deducts 1 credit
- Previously-exported leads are included for free
- Use `maxCredits` to control spending, `maxLeads` to limit volume
- Set `maxCredits: 0` to only receive previously-exported leads at no cost

**Response:** `files` array (with base64-encoded data), `leadCount`, `exportId`, `databaseType`, `creditsUsed`, `creditsRemaining`, `overflowCount`

**Error 402 Payment Required:** Returned when credit-plan users have insufficient credits.

Rate limited: 1 export per 5 minutes, max 10,000 leads per export.

### 4. Filter Presets — `/filter-presets`

- `GET /filter-presets` — List all saved presets
- `POST /filter-presets` — Create a preset (requires `name` and `filters` object)
- `DELETE /filter-presets/{id}` — Delete a preset

### 5. Keyword Lists — `/keyword-lists`

Keyword lists now support typed lists (positive or negative) with paired list management and source categories.

- `GET /keyword-lists` — List all keyword lists
- `POST /keyword-lists` — Create (requires `name`, optional `keywords` array, `sourceCategories` array max 3)
- `PUT /keyword-lists/{id}` — Update
- `DELETE /keyword-lists/{id}` — Delete

**Keyword list properties:** `name`, `keywords` (wildcard patterns e.g., `*dentist*`), `type` (positive/negative), `pairedListId` (linked positive/negative pair), `sourceCategories` (max 3), `autoRefineEnabled`, `refinementStatus` (running/completed/paused)

### 6. Email Schedules — `/email-schedules`

Email schedules now support distribution modes and lead reservoirs.

- `GET /email-schedules` — List schedules
- `POST /email-schedules` — Create (requires `name`, `filterPresetId`, `intervalValue`, `intervalUnit`, `recipients` min 1)
- `PATCH /email-schedules/{id}` — Update (supports `isActive` toggle)
- `DELETE /email-schedules/{id}` — Delete
- `POST /email-schedules/{id}/trigger` — Manually trigger an active schedule to send immediately (rate limited: 1 per 5 minutes)

**Distribution modes:**
- `full_copy` (default): All leads sent to every recipient
- `split_evenly`: Leads divided evenly among recipients. Optional `fullCopyRecipients` array for people who should receive the full list (e.g., managers)

**Lead reservoir:** Set `maxLeadsPerEmail` to cap leads per delivery. Overflow is stored and included in the next scheduled email.

### 7. Export Formats — `/export-formats`

- `GET /export-formats` — List custom export formats
- `POST /export-formats` — Create (requires `name`, supports `fileType`, `fieldMappings`, split settings)
- `GET /export-formats/{id}` — Get specific format
- `PATCH /export-formats/{id}` — Update
- `DELETE /export-formats/{id}` — Delete
- `POST /export-formats/{id}/set-default` — Set as default

### 8. Export History — `/export-history`

- `GET /export-history` — List past exports (optional `limit` param, default 50)
- `GET /export-history/{id}/download` — Re-download (expires after 7 days)

### 9. AI Features

**`POST /ai/suggest-categories`** — Get AI category suggestions based on company profile.

Required: `companyName`, `companyDescription`, `productService`
Optional: `companyWebsite`, `smbType`, `excludeCategories`

**`POST /ai/generate-keywords`** — Trigger async keyword generation based on your company profile and target categories (up to 3 per list). Keywords are generated as wildcard patterns and saved to keyword lists with auto-refine enabled by default. Use `/ai/keyword-status` to check progress.

**`GET /ai/keyword-status`** — Check the status of keyword generation jobs. Use this to poll for completion after triggering keyword generation.

**AI Auto-Refine** — Single-pass 4-phase optimization that automatically refines keyword lists using AI:

- Phase 1: Validates positive keywords (50% threshold, up to 2 variation attempts)
- Phase 1B: Discovers up to 15 new positive keywords in a single AI call
- Phase 2: Validates negative keywords (40% threshold)
- Phase 2B: Discovers up to 5 new negative keywords from ~80 sample leads
- Final quality score (1-10, median of 3) determines if a retry pass is needed
- Auto-refine turns off when complete
- Uses `sourceCategories` (max 3 per list) for accurate AI scoring

Endpoints:

- `POST /ai/auto-refine/enable` — Enable auto-refine for a keyword list (requires `listId`)
- `POST /ai/auto-refine/disable` — Disable auto-refine for a keyword list (requires `listId`)
- `GET /ai/auto-refine/status` — Check auto-refine status (optional `listId` query param to filter by specific list)

### 10. Export Blacklist — `/export-blacklist`

- `GET /export-blacklist` — List blacklisted entries
- `POST /export-blacklist` — Add entry (single or batch via `entries` array)
- `DELETE /export-blacklist/{id}` — Remove entry

### 11. Account

- `GET /me` — Get user profile (subscription plan, settings, onboarding status, credit balance)
- `PATCH /me` — Update profile (firstName, lastName, companyName, companyWebsite)
- `GET /settings/database` — Check current database type and switch availability
- `POST /settings/switch-database` — Switch between databases (has cooldown)

### 12. Programmatic Purchase — Buy a subscription via API

No web signup required. New users can purchase and get an API key entirely via API:

1. `POST /purchase` — Create a Stripe Checkout session. Provide `email` and `plan` (starter, growth, scale, platinum, or enterprise). Returns a `checkoutUrl` and `claimToken`.
2. Direct the user to complete payment at the checkout URL.
3. `POST /claim-key` — After payment, provide `email` and `claimToken` to retrieve the API key. If payment is still pending, returns status `pending` — poll every 5-10 seconds.

### 13. Credits & Subscription Management

- `POST /purchase-credits` — Purchase additional permanent credits. Provide either `creditCount` (min 100) or `dollarAmount` (min $1). Uses saved payment method (Stripe off-session charge).
- `POST /subscription/change-plan` — Upgrade or downgrade between starter, growth, and scale. On upgrade, unused monthly credits convert to permanent credits. Downgrades take effect at renewal.
- `POST /subscription/cancel` — Cancel subscription at end of current billing period. Access continues until period ends.

---

## Natural Language Translation Guide

When users make natural language requests, translate them into API calls. Use multiple wildcard keyword variations to cast a wider net — keywords are matched via OR logic so more variations means better coverage:

| User Says | API Call |
|-----------|---------|
| "Find new dental practices in Texas" | `GET /leads?positiveKeywords=["*dental*","*dentist*","*orthodont*"]&stateInclude=TX` |
| "Search for med spas and aesthetics businesses in Florida" | `GET /leads?positiveKeywords=["*med*spa*","*medical*spa*","*aesthet*","*botox*","*medspa*"]&stateInclude=FL` |
| "Show me auto repair shops in Chicago updated this week" | `GET /leads?positiveKeywords=["*auto*repair*","*body*shop*","*mechanic*","*oil*change*","*brake*"]&cityInclude=["Chicago"]&lastUpdatedFrom=rel:7d` |
| "Find pet grooming businesses in California, exclude boarding" | `GET /leads?positiveKeywords=["*pet*groom*","*dog*groom*","*pet*salon*"]&negativeKeywords=["*boarding*","*kennel*"]&stateInclude=CA` |
| "Get bakeries and catering companies in New York" | `GET /leads?positiveKeywords=["*bakery*","*bake*shop*","*cater*","*pastry*","*cake*"]&stateInclude=NY` |
| "Find fitness studios in Georgia and North Carolina" | `GET /leads?positiveKeywords=["*fitness*","*gym*","*yoga*","*pilates*","*crossfit*"]&stateInclude=GA,NC` |
| "Get 50 leads with high ratings" | `GET /leads?limit=50&minStars=4` (home_improvement only) |
| "Find businesses with LocalBusiness schema type" | `GET /leads?websiteSchemaFilter=LocalBusiness` (other only) |
| "Show leads registered in the last 6 months" | `GET /leads?registrationDateFrom=rel:6m` (other only) |
| "Export all my filtered results" | `POST /leads/export` with current filters |
| "Export but only spend 50 credits max" | `POST /leads/export` with `maxCredits: 50` |
| "Export only previously-exported leads (free)" | `POST /leads/export` with `maxCredits: 0` |
| "What categories should I target?" | `POST /ai/suggest-categories` |
| "Save this search as 'FL Med Spas'" | `POST /filter-presets` |
| "Show my recent exports" | `GET /export-history` |
| "What plan am I on?" | `GET /me` |
| "How many credits do I have left?" | `GET /me` |
| "Buy 500 more credits" | `POST /purchase-credits` with `creditCount: 500` |
| "Upgrade to the Growth plan" | `POST /subscription/change-plan` with `targetPlan: "growth"` |
| "Cancel my subscription" | `POST /subscription/cancel` |
| "Exclude these domains from exports" | `POST /export-blacklist` |
| "Enable auto-refine on my keyword list" | `POST /ai/auto-refine/enable` with `listId` |
| "Check on my keyword generation" | `GET /ai/keyword-status` |
| "Send my scheduled email now" | `POST /email-schedules/{id}/trigger` |
| "Split leads evenly among my sales team" | `POST /email-schedules` with `distributionMode: "split_evenly"` |
| "I want to sign up for a Starter plan" | `POST /purchase` with `plan: "starter"` |

## Building API Requests

Use the included `smb_api.py` script for all API calls. It handles authentication, URL encoding, response parsing, and safe file export in a single reusable file. **Do not use shell commands like `curl`** — constructing shell commands from user-provided input risks shell injection vulnerabilities.

### Usage

```bash
python smb_api.py <API_KEY> <METHOD> <ENDPOINT> [--params '{"key":"value"}'] [--body '{"key":"value"}'] [--output-dir /path/to/dir]
```

### Examples

```bash
# Search for med spas in Florida using wildcard keywords (OR logic)
python smb_api.py smbk_xxx GET /leads --params '{"positiveKeywords":"[\"*med*spa*\",\"*medical*spa*\",\"*aesthet*\",\"*botox*\",\"*medspa*\"]","stateInclude":"FL","limit":"25"}'

# Find auto shops in multiple states, exclude franchises
python smb_api.py smbk_xxx GET /leads --params '{"positiveKeywords":"[\"*auto*repair*\",\"*body*shop*\",\"*mechanic*\",\"*tire*\",\"*oil*change*\"]","negativeKeywords":"[\"*franchise*\",\"*jiffy*\"]","stateInclude":"GA,FL,NC,SC,TN","limit":"50"}'

# Search for recently updated dental leads in Texas
python smb_api.py smbk_xxx GET /leads --params '{"positiveKeywords":"[\"*dental*\",\"*dentist*\",\"*orthodont*\",\"*oral*surg*\"]","stateInclude":"TX","lastUpdatedFrom":"rel:7d"}'

# Full-text search across all fields
python smb_api.py smbk_xxx GET /leads --params '{"search":"organic coffee","limit":"25"}'

# Filter by website schema type (other database only)
python smb_api.py smbk_xxx GET /leads --params '{"websiteSchemaFilter":"LocalBusiness","stateInclude":"CA","limit":"25"}'

# Get available website schema types
python smb_api.py smbk_xxx GET /leads/other/schema-types

# Get account info (includes credit balance)
python smb_api.py smbk_xxx GET /me

# Export with credit controls
python smb_api.py smbk_xxx POST /leads/export --body '{"database":"other","filters":{"positiveKeywords":["*pet*groom*","*veterinar*","*dog*train*"],"stateInclude":"CA,OR,WA"},"maxCredits":100}'

# Export only previously-exported leads (free, no credits used)
python smb_api.py smbk_xxx POST /leads/export --body '{"database":"other","filters":{"positiveKeywords":["*dental*"],"stateInclude":"TX"},"maxCredits":0}'

# Purchase additional credits
python smb_api.py smbk_xxx POST /purchase-credits --body '{"creditCount":500}'

# Purchase credits by dollar amount
python smb_api.py smbk_xxx POST /purchase-credits --body '{"dollarAmount":50}'

# Change subscription plan
python smb_api.py smbk_xxx POST /subscription/change-plan --body '{"targetPlan":"growth"}'

# Cancel subscription
python smb_api.py smbk_xxx POST /subscription/cancel

# Start a programmatic purchase (no auth needed, but script still requires a placeholder key)
python smb_api.py none POST /purchase --body '{"email":"user@example.com","plan":"starter"}'

# Claim API key after payment
python smb_api.py none POST /claim-key --body '{"email":"user@example.com","claimToken":"tok_abc123"}'

# AI category suggestions
python smb_api.py smbk_xxx POST /ai/suggest-categories --body '{"companyName":"FitPro Supply","companyDescription":"Commercial fitness equipment distributor","productService":"Gym equipment, treadmills, weight systems"}'

# Create a filter preset
python smb_api.py smbk_xxx POST /filter-presets --body '{"name":"NY Bakeries","filters":{"positiveKeywords":["*bakery*","*bake*shop*","*cater*","*pastry*"],"stateInclude":"NY"}}'

# Create email schedule with split distribution
python smb_api.py smbk_xxx POST /email-schedules --body '{"name":"Daily TX Leads","filterPresetId":5,"intervalValue":1,"intervalUnit":"days","recipients":["rep1@co.com","rep2@co.com"],"distributionMode":"split_evenly","fullCopyRecipients":["manager@co.com"],"maxLeadsPerEmail":50}'

# Enable AI auto-refine on a keyword list
python smb_api.py smbk_xxx POST /ai/auto-refine/enable --body '{"listId":42}'

# Check auto-refine status for a specific list
python smb_api.py smbk_xxx GET /ai/auto-refine/status --params '{"listId":"42"}'

# Check keyword generation job status
python smb_api.py smbk_xxx GET /ai/keyword-status

# Manually trigger an email schedule
python smb_api.py smbk_xxx POST /email-schedules/15/trigger

# Delete a filter preset
python smb_api.py smbk_xxx DELETE /filter-presets/42
```

The script outputs JSON to stdout and rate limit headers to stderr. For export requests, files are automatically saved with sanitized filenames.

**Remember:**
- Use multiple wildcard keyword variations to cast a wider net (e.g., `["*dental*", "*dentist*", "*orthodont*"]` not just `["dental"]`) — keywords are matched via OR logic
- Use `*` for flexible pattern matching: `"*auto*repair*"` matches "auto body repair", "automotive repair shop", etc.
- JSON array parameters should be serialized as strings inside the `--params` JSON
- At least one positive filter is required for lead searches
- Check which database the user needs before applying database-specific filters
- Home Improvement database provides phone numbers; Other database provides phone numbers and email addresses
- Phone and email are masked for free-tier users
- Present results in a clean, readable table format
- For credit-plan users, mention credits used/remaining after exports
- The `POST /purchase` and `POST /claim-key` endpoints do not require authentication (no API key needed)

## Security

This skill addresses two specific agent execution risks: **shell injection** from constructing CLI commands with user input, and **arbitrary file writes** from unsanitized API-provided filenames.

**Shell injection prevention:** The `smb_api.py` script uses Python's `requests` library for all HTTP calls. User-provided search terms, locations, and other inputs are passed as structured function arguments — never interpolated into shell command strings. This eliminates the shell injection vector that exists when agents construct `curl` commands from user input.

**Path traversal prevention in exports:** The `/leads/export` endpoint returns base64-encoded files with an API-provided `fileName` field. A malicious or corrupted filename (e.g., `../../etc/passwd`) could write files to arbitrary locations. The script enforces three safeguards:
1. **Basename extraction:** `os.path.basename()` strips all directory components — `../../etc/passwd` becomes `passwd`
2. **Extension validation:** Only `.csv`, `.json`, and `.xlsx` extensions are allowed; anything else defaults to `.csv`
3. **Scoped output directory:** Files are written only to the designated output directory (`/mnt/user-data/outputs/` by default), never to user-specified or API-specified paths

**API key handling:** The key is passed as a CLI argument and sent only in the Authorization header. It is never logged, written to files, or included in error output.

## Error Handling

| Status | Meaning |
|--------|---------|
| 400 | Bad request — check parameters |
| 401 | Invalid or missing API key |
| 402 | Insufficient credits (credit-plan users) — check credit balance with `GET /me` |
| 403 | Active subscription required |
| 404 | Resource not found |
| 429 | Rate limited — check `Retry-After` header |
| 500 | Server error |

All errors return: `{ "error": "error_code", "message": "Human-readable message" }`
