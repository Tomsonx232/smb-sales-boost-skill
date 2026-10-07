# SMB Sales Boost API reference (for the smb-sales-boost skill)

> **Note for AI agents:** This file may be truncated when opened. Read it fully, in chunks if necessary. `SKILL.md` has the rules and workflows; this file has the complete details. Base URL: `https://smbsalesboost.com/api/v1`. Every endpoint needs `Authorization: Bearer smbk_...` except `POST /purchase` and `POST /claim-key`.

Contents: 1. Endpoints, 2. Lead search parameters, 3. Lead fields, 4. Exports, 5. Export history, 6. Lead export history, 7. Export formats, 8. Export blacklist, 9. Filter presets, 10. Keyword lists, 11. Email schedules, 12. AI endpoints, 13. Enrichments, 14. Account and billing, 15. Signing up, 16. Errors and headers.

## 1. Endpoints

Cost: "credits" means it spends credits; "money" means it charges or authorizes charges to the card. Limiter: every endpoint counts toward 600 requests per minute per key; "export" and "AI" are the extra shared 5-per-minute buckets.

| Method and path | Purpose | Cost | Limiter |
|---|---|---|---|
| `GET /me` | Profile, plan, credit balance, trial, auto top-up and overage settings | | |
| `PATCH /me` | Update name, company name, company website | | |
| `GET /settings/database` | Always reports the Main database (`other`) | | |
| `GET /leads` | Search leads, full contact details | credits | |
| `GET /leads/preview` | Search leads, contacts masked | free | |
| `GET /leads/other/schema-types` | List website schema types for `websiteSchemaFilter` | | |
| `POST /leads/export` | Export leads to a CSV, JSON or XLSX file | credits | export |
| `GET /export-history` | List export files (90-day window) | | |
| `GET /export-history/{id}/download` | Download an export file | | |
| `GET /export-history/{id}/download-url` | Short-lived direct download link for an export file | | |
| `GET /lead-export-history` | Every lead you received, with tracking ids | | |
| `GET /lead-export-history/ids` | All matching tracking ids | | |
| `POST /lead-export-history/refresh` | Replace stored copies with current data | free | export |
| `POST /lead-export-history/re-export` | Rebuild a file from stored copies | free | export |
| `POST /lead-export-history/refresh-and-export` | Refresh, then rebuild a file | free | export |
| `GET /export-formats`, `GET /export-formats/{id}` | List or get export formats | | |
| `POST /export-formats`, `PATCH /export-formats/{id}`, `DELETE /export-formats/{id}` | Create, update, delete | | |
| `POST /export-formats/{id}/set-default` | Make a format the default | | |
| `GET /export-blacklist`, `POST /export-blacklist`, `DELETE /export-blacklist/{id}` | Domains to keep out of exports | | |
| `GET /filter-presets`, `POST /filter-presets`, `DELETE /filter-presets/{id}` | Saved searches for email schedules | | |
| `GET /keyword-lists`, `POST /keyword-lists`, `PUT /keyword-lists/{id}`, `DELETE /keyword-lists/{id}` | Keyword lists | | |
| `GET /email-schedules`, `POST /email-schedules`, `PATCH /email-schedules/{id}`, `DELETE /email-schedules/{id}` | Scheduled lead emails | credits, sends email (an active schedule, the default on create, sends its first email within about 15 minutes) | |
| `POST /email-schedules/{id}/trigger` | Send a schedule now | credits, sends email | export |
| `POST /ai/suggest-categories` | Suggest customer categories | | AI |
| `POST /ai/generate-keywords` | Delete all keyword lists and regenerate them | | AI |
| `GET /ai/keyword-status` | Keyword generation jobs (last 24 hours) | | |
| `GET /ai/auto-refine/status` | Refinement progress | | |
| `POST /ai/auto-refine/enable`, `POST /ai/auto-refine/disable` | Start or stop refinement of a list | | AI (enable only) |
| `POST /enrichments` | Start a URL enrichment run | credits | |
| `GET /enrichments`, `GET /enrichments/{id}` | List runs, get one run | | |
| `GET /enrichments/{id}/download` | Short-lived link to the result CSV | | |
| `POST /purchase-credits` | Buy permanent credits with the card on file | money | |
| `GET /auto-top-up`, `PATCH /auto-top-up` | Automatic credit purchases | money (when enabled) | |
| `GET /overage-budget`, `PATCH /overage-budget` | Spend past zero, billed daily | money (when enabled) | |
| `POST /subscription/change-plan` | Upgrade, downgrade, trial switch or activation | money | |
| `POST /subscription/cancel` | Cancel at period end | | |
| `POST /purchase` | Subscription checkout, no API key needed | money | 5 per hour per IP |
| `POST /claim-key` | Older flow that cannot succeed; the key is emailed instead | | 30 per hour per IP |

## 2. Lead search parameters

Used by `GET /leads` and `GET /leads/preview` as query parameters, and inside the `filters` object of `POST /leads/export`, except `maxCredits`, `maxResults` and `excludePurchased`: in an export these go at the top level of the body, next to `filters` (inside `filters` they are silently ignored and the export is not capped). `page`, `limit` and `search` are not used by the export. Unknown parameters are ignored silently.

**Formats.** In a query string, list parameters are JSON-array strings (`["a","b"]`) except the comma-separated ones marked CS. In the export `filters` object, use real JSON arrays for every list (the export ignores JSON-array strings for most of them). The `smb_api.py` script converts both automatically when you pass real arrays.

### Paging, sorting and credit control

| Parameter | Format | Notes |
|---|---|---|
| `page` | integer | Default 1. |
| `limit` | integer | Default 100, clamped to 1-1000. On `GET /leads` this is also the most credits one call can spend. |
| `sortBy` | string | Default `lastBuyingSignal` (old name `lastUpdated` still accepted). Others: `wCompanyName`, `wOgrequestedUrl`, `wSimpleCrawledUrl`, `wProfileDate`, `wRedirectYn`, `wAddressCity`, `wAddressState`, `wAddressZip`, `wPhonePrimary`, `wEmailPrimary`, `wTimeScraped`, `wEmployeeCount`, `wFoundingDate`, `wIsChain`, `wPriceRange`, `wLegalName`, `wIndustryNaics`, `wDuns`, `wSlogan`, `wEmailPrimaryMx`, `wEmailDomainMx`, `wEmailSecurity`, `wAddressSource`, and the numeric `wNumLocations`, `wOpenPositions`, `wRatingValue`, `wReviewCount`. Unknown values fall back to the default. |
| `sortOrder` | `asc` or `desc` | Default `desc`. |
| `maxCredits` | integer, 0 or more | `GET /leads` (query; applies to that one call) and export (top-level body field, a JSON number). Most credits that call may spend. `0` returns only leads you already have. On `GET /leads` a non-number or negative value is ignored (no cap) and a decimal is cut to a whole number, so validate it. The export rejects a non-integer or negative value with `400`, but treats `null` as no cap. |
| `maxResults` | integer, 1 or more | `GET /leads` (query) and export (top-level body field). Most leads to return in this call: on `GET /leads` it trims the current page only, and trimmed leads are not carried to the next page; on export it caps the file. |
| `excludePurchased` | `true` | Preview (query) and export (top-level body boolean). Leaves out leads flagged `contactExported` (see section 3). On preview it is applied after paging, so a page can hold fewer than `limit` leads and `pagination.total` still counts them. `GET /leads` ignores it. |
| `database` | string | Ignored; always the Main database (`other`). |

### Keyword and text filters (these can start a search)

At least one of `positiveKeywords`, `nameIncludeTerms`, `urlIncludeTerms`, `crawledUrlIncludeTerms`, `descriptionIncludeTerms` must be present. Otherwise `GET /leads` and preview return no leads with `requiresKeywords: true`, and export returns `400`.

| Parameter | Matches | Notes |
|---|---|---|
| `positiveKeywords` | `orColumns` | OR across keywords and columns. Case-insensitive substring. `*` is a wildcard; on URL columns spaces are wildcards too. |
| `negativeKeywords` | `orColumns` | Drops a lead when any keyword matches any column (empty values are kept). |
| `orColumns` | | Columns for both keyword lists. Default `wOgrequestedUrl`, `wSimpleCrawledUrl`, `wCompanyName`, `aiCategoryEstimation`. Allowed: those plus `wProfileDescriptionShort`, `wAddressCity`, `wAddressState`, `wAddressZip`, `wPhonePrimary`, `wPhoneTotal`, `wEmailPrimary`, `wEmailTier1` to `wEmailTier6`. Searches stay fast only with the four defaults. |
| `nameIncludeTerms`, `nameExcludeTerms` | Company name | Include terms join the keyword OR group when `wCompanyName` is in `orColumns` (default); otherwise each is required. |
| `urlIncludeTerms`, `urlExcludeTerms` | Registered URL | Same rules. |
| `crawledUrlIncludeTerms`, `crawledUrlExcludeTerms` | Crawled URL | Same rules. |
| `descriptionIncludeTerms`, `descriptionExcludeTerms` | Short description | Not in the default `orColumns`, so each include term is required unless you add `wProfileDescriptionShort`. |
| `search` | Company name, both URLs, short description, city, primary phone | Plain substring; `*` is not a wildcard. `GET /leads` and preview only. Does not count as a positive filter. |

### Location

| Parameter | Format | Notes |
|---|---|---|
| `stateInclude`, `stateExclude` | CS | Uppercase two-letter codes, exact match. |
| `cityInclude`, `cityExclude` | list | Case-insensitive substring. |
| `zipInclude`, `zipExclude` | list | Exact match. |
| `streetInclude`, `streetExclude` | list | Substring. |
| `stateEmptyFilter`, `cityEmptyFilter`, `zipEmptyFilter`, `streetEmptyFilter` | `only_empty` / `exclude_empty` | Empty-value filters. |

### Dates and buying signals

Dates accept ISO 8601 (`2026-09-01`, `2026-09-01T12:00:00Z`) or relative `rel:<n><unit>` with units `h`, `d`, `w`, `m` (months), `y`. Only the Last Buying Signal dates are compared as real dates, and an invalid one makes the call fail with `500`, so check the format. `registrationDateFrom/To` and `timeScrapedFrom/To` are compared as text against the stored values (Time Scraped looks like `2026-09-01 14:30:00`), so use a plain `YYYY-MM-DD` date for them: a time of day, including the time a `rel:` value resolves to, is not applied on the boundary day (a `From` bound leaves out that whole day and a `To` bound keeps it), and an invalid value matches the wrong rows instead of failing.

| Parameter | Notes |
|---|---|
| `lastBuyingSignalFrom`, `lastBuyingSignalTo` | Last Buying Signal range (inclusive). Old names `lastUpdatedFrom`, `lastUpdatedTo` still work; the new name wins if both are sent. A date-only `To` means the start of that day. |
| `buyingSignalTypeFilter` | CS. Exact, case-sensitive types: `Newly Added`, `Phone Primary`, `Total Phones`, `Email Primary`, `Tier 1 Emails`, `Tier 2 Emails`, `Address Full`, `Instagram`, `LinkedIn`, `Facebook`, `YouTube`, `TikTok`, `X (Twitter)`, `Google Maps`, `Yelp`, `Pinterest`, `Other Social`. Old name `updateReasonFilter` still works. |
| `registrationDateFrom`, `registrationDateTo` | Domain registration date. |
| `timeScrapedFrom`, `timeScrapedTo` | When the website was last crawled. |

**What moves the Last Buying Signal:** the lead is first added (`Newly Added`), or a new non-empty value differs from the stored one in Phone Primary, Total Phones, Email Primary, Tier 1 Emails, Tier 2 Emails or Address Full, or a social profile is added or changed for a source (Instagram, LinkedIn, Facebook, YouTube, TikTok, X (Twitter), Google Maps, Yelp, Pinterest, Other Social). `Buying Signal Type` lists the reasons for the most recent change. Removing a value, or changes to other fields, do not move it.

### Contact filters

Term filters take a list whose items are strings or `{"v": "text", "m": "contains" | "starts" | "ends"}` (default contains). Matching is case-insensitive; `*` is a literal character here, not a wildcard. Phones are matched against the stored text, so formatting matters (for an area code use `{"v": "(512)", "m": "starts"}`).

| Include / exclude | Empty filter (`only_empty` / `exclude_empty`) |
|---|---|
| `phonePrimaryInclude`, `phonePrimaryExclude` | `phonePrimaryEmptyFilter` |
| `phoneTotalInclude`, `phoneTotalExclude` | `phoneTotalEmptyFilter` |
| `emailPrimaryInclude`, `emailPrimaryExclude` | `emailPrimaryEmptyFilter` |
| `emailTier1Include` ... `emailTier6Include`, `emailTier1Exclude` ... `emailTier6Exclude` | `emailTier1EmptyFilter` ... `emailTier6EmptyFilter` |
| | `descriptionEmptyFilter` (short description) |

### Company detail columns

For each key below: `<key>Include` and `<key>Exclude` (term lists as above) and `<key>EmptyFilter`. Keys: `employeeCount`, `foundingDate`, `isChain`, `priceRange`, `legalName`, `industryNaics`, `duns`, `founder`, `founderEmail`, `founderPhone`, `openingHours`, `areaServed`, `offers`, `brand`, `slogan`, `logoUrl`, `socialProfiles`, `platform`, `bookingOpsCrmSoftware`, `reputationChatSoftware`, `marketingEmailSoftware`, `adPixels`, `paymentFinancingSoftware`.

| Parameter | Format | Notes |
|---|---|---|
| `minNumLocations`, `maxNumLocations` | number | Number of locations. Leads without a value are excluded when a bound is set. |
| `minOpenPositions`, `maxOpenPositions` | number | Job postings found on the website. `openPositionsEmptyFilter`: empty = no careers page found. |
| `minRatingValue`, `maxRatingValue` | number | Review rating. |
| `minReviewCount`, `maxReviewCount` | number | Number of reviews. |
| `emailPrimaryMxValues`, `emailDomainMxValues` | list | Exact email provider domains, for example `["google.com","outlook.com"]`. |
| `emailSecurityValues` | list | Exact values such as `spf`, `dkim`, `dmarc`, `bimi`, `none`. |
| `addressSourceValues` | list | Exact address-source values. |
| `emailPrimaryMxEmptyFilter`, `emailDomainMxEmptyFilter`, `emailSecurityEmptyFilter`, `addressSourceEmptyFilter` | empty filter | |
| `websiteSchemaFilter` | CS | Substring match on website schema types; values from `GET /leads/other/schema-types` (a bare JSON array; it is cached and refreshed every 6 hours, but a call made shortly after a server restart or while the refresh runs can take up to about 2 minutes or time out, so retry later). |
| `redirectFilter` | `yes` / `no` | `no` includes leads with no value. |

## 3. Lead fields

Leads from `GET /leads` and preview use display-name keys, in this order: `id`, `Last Buying Signal`, `Last Updated` (deprecated duplicate of Last Buying Signal), `Buying Signal Type` (array or null), `Registered URL`, `Registration Date`, `Redirect` (`Yes` / `No`), `Crawled URL`, `Company Name`, `AI Categories` (array or null), `Address Full`, `Street`, `City`, `State`, `Zip`, `Address Source`, `Platform`, `Booking/Ops/CRM Software`, `Reputation/Chat Software`, `Marketing/Email Software`, `Ad Pixels`, `Payment/Financing Software`, `Employees`, `Number of Job Postings`, `Founding Date`, `Number of Locations`, `Is Chain?`, `Rating Value`, `Review Count`, `Price Range`, `Legal Name`, `NAICS Code`, `DUNS Number`, `Founder Name(s)`, `Founder Email(s)`, `Founder Phone(s)`, `Latitude`, `Longitude`, `Opening Hours`, `Area Served`, `Offers`, `Brand`, `Slogan`, `Logo URL`, `Social Profiles`, `Phone Primary`, `Total Phones`, `Email Primary`, `Email Primary Provider`, `Tier 1 Emails`, `Tier 2 Emails`, `Tier 3 Emails`, `Tier 4 Emails`, `Tier 5 Emails`, `Tier 6 Emails`, `Website Email Provider`, `Website Email Security`, `Website Schema`, `Description Short`, `Description Long`, `Time Scraped`.

- Values are strings or null unless noted. `Social Profiles` is a list written as text, like `['https://facebook.com/x']`.
- Preview masks phones and emails (`(512) ***-**42`, `hel*@domain.com`, list fields masked item by item), including founder emails and phones, except on leads you already have, which come back unmasked with `contactExported: true`. `contactExported: true` in a preview does not guarantee the lead is free in `GET /leads`.
- Response: `{"data": {"leads": [...], "pagination": {"page", "limit", "total", "pages"}, "databaseType": "other", "creditsUsed", "creditsRemaining", "maxResults"?, "maxCredits"?}}`. Preview adds `"preview": true` and has no credit fields. With no positive filter: `leads: []`, `requiresKeywords: true` and a `message`.
- Headers: `X-Credits-Capped: true` when your balance cut the results; `X-Lead-Query-Path` is `sidecar` (fast), `cached` (fast, served from a 20-second cache) or `trigram` (slow, up to about 110 seconds).
- Charging (credit plans): on each page, new leads come first and are cut to the smaller of `maxCredits` and your spendable balance; leads you already have fill the rest of the page free; the page is then cut to `maxResults`. Each returned lead counts as received, and connected CRM and webhook integrations receive it.

## 4. Exports: `POST /leads/export`

| Body field | Type | Notes |
|---|---|---|
| `filters` | object | Any parameters from section 2 except `search`, `page`, `limit`, `database`, `maxCredits`, `maxResults` and `excludePurchased` (send those three as top-level body fields; inside `filters` they are silently ignored, so a `maxCredits` placed there does not cap spending). Lists must be real arrays (`stateInclude` and `stateExclude` also accept a comma string without spaces, such as `"FL,GA"`: the export does not trim the pieces, so `"FL, GA"` silently ignores GA; `buyingSignalTypeFilter` and `websiteSchemaFilter` also accept a comma string; `positiveKeywords`, `negativeKeywords` and `orColumns` also accept a JSON-array string). Needs a positive filter unless `selectedIds` is used. Up to 100,000 leads. |
| `selectedIds` | integer array | Export these lead ids instead of a filter (filters are then ignored). Practical limit about 12,000 ids (100 KB request body). |
| `maxCredits` | integer, 0 or more (a JSON number) | Most credits to spend. Without it the export can use the whole balance plus any overage budget. |
| `maxResults` | integer, 1 or more | Most leads in the file. |
| `maxLeads` | integer, 1 or more | Like `maxResults`, but the overflow (`overflowCount` in the response) is kept in your lead reservoir while it has room and merged into your next `maxLeads` export. The reservoir is one pool per account, shared with email schedules, up to your plan's monthly credit allowance; leads that do not fit are not kept. |
| `excludePurchased` | boolean `true` | Only leads you do not have yet. |
| `formatId` | integer | Export format to use (section 7). A wrong id silently uses your default format, or the built-in CSV. |
| `inline` | boolean | Default `true` (file contents in the response). `false` (or `?inline=false`) returns `exportId` and `downloadUrl` per file instead. |

- File type and columns come from the export format: the one named by `formatId`, otherwise your account's default format (section 7; set in the dashboard or with `set-default`), otherwise the built-in CSV with the default columns below. Before a large export without `formatId`, check `GET /export-formats` for a format with `isDefault: true` and `fileType: "xlsx"` (see the next point).
- XLSX: at most 20,000 rows in one file. An export that would put more in any one file (a single file, a split part, or the `includeTotal` file, which holds every lead) returns `400 export_too_large` after the credits were charged. For large XLSX exports the format must keep every file at 20,000 rows or less: `splitFiles: true`, `splitMode: "max_rows"`, `maxRowsPerFile` of 20,000 or less and `includeTotal: false`. To recover, do not simply repeat the request (an export picks new leads first, so a repeat can charge for different leads): re-run it with a CSV or such a split format, `maxCredits: 0` and no `excludePurchased`, which returns the leads you already have at no cost, or re-export them from the lead export history (section 6).
- Response: `{"data": {"files": [{"fileName", "fileType", "data", "size"}], "leadCount", "exportId", "databaseType": "other", "creditsUsed", "creditsRemaining", "maxLeads"?, "overflowCount"?, "maxResults"?, "maxCredits"?}}`. `files[].data` is plain text for CSV and JSON and base64 for XLSX. With `inline: false`, files carry `exportId` and `downloadUrl` (`/api/v1/export-history/{id}/download`, needs the API key) instead of `data`; a file that could not be stored keeps its `data`.
- File names: `leads-main-YYYY-MM-DD.csv` (or `leads-other-...`); split files add `_part1`, `_part2` and an optional `_total`.
- Default columns (59): `Last Buying Signal`, `Buying Signal Type`, `Registered URL`, `Registration Date`, `Redirect`, `Crawled URL`, `Company Name`, `Categories (AI Enrichment)`, `Phone Primary`, `Total Phones`, `Email Primary`, `Email Primary Provider`, `Tier 1 Emails` to `Tier 6 Emails`, `Website Email Provider`, `Website Email Security`, `Website Schema`, `Description Short`, `Description Long`, `Address Full`, `Street`, `City`, `State`, `Zip`, `Address Source`, `Platform`, `Booking/Ops/CRM Software`, `Reputation/Chat Software`, `Marketing/Email Software`, `Ad Pixels`, `Payment/Financing Software`, `Employees`, `Number of Job Postings`, `Founding Date`, `Number of Locations`, `Is Chain?`, `Rating Value`, `Review Count`, `Price Range`, `Legal Name`, `NAICS Code`, `DUNS Number`, `Founder Name(s)`, `Founder Email(s)`, `Founder Phone(s)`, `Latitude`, `Longitude`, `Opening Hours`, `Area Served`, `Offers`, `Brand`, `Slogan`, `Logo URL`, `Social Profiles`, `Time Scraped`.
- Cells are limited to 32,767 characters in CSV and XLSX (Excel's limit); JSON is not limited. Array values are written as text like `['a', 'b']`.
- The export blacklist (section 8) is applied to exports and previews.
- Every file is also saved in export history for 90 days.

## 5. Export history

- `GET /export-history?limit=N` (0-200, default 50; newest 200 files only): items `{id, fileName, fileType, leadCount, fileSize, formatName, databaseType, filters, createdAt, expiresAt, downloadAvailable, unavailableReason}`. `unavailableReason` is `expired`, `not_stored` or null. Stored `filters` use the internal names (`updateReasonFilter`, `lastUpdatedFrom`).
- `GET /export-history/{id}/download`: the raw file (not JSON) with `Content-Disposition: attachment`.
- `GET /export-history/{id}/download-url`: `{"data": {"downloadUrl", "expiresInSeconds", "fileName", "fileType", "fileSize", "leadCount", "fileExpiresAt"}}`. The link works for at most 15 minutes and must be fetched without the `Authorization` header. Treat it like a password.
- Errors for both: `404 not_found`, `410 {"error": "gone", "reason": "expired" | "not_stored"}` (re-export the leads free, section 6), `503` with `Retry-After: 30` (storage temporarily unavailable).

## 6. Lead export history

Every lead you received (dashboard, API, email schedules, CRM pushes, enrichments) has one tracking row, with a stored copy of the lead as you received it.

- `GET /lead-export-history`: query `limit` (1-200, default 50), `offset`, `dateFrom`, `dateTo` (inclusive), `exportMethod` (`dashboard`, `api`, `email_schedule`, `crm_push`, `enrichment`). Response `{"data": [{"id", "leadId", "exportedAt", "exportMethod", "leadSnapshot", "lastChangedFields", "lead", ...}], "totalCount", "limit", "offset"}`. `id` is the tracking id. Snapshots use internal camelCase field names. Receiving the same lead again updates its row.
- `GET /lead-export-history/ids`: same filters, no paging: `{"data": [ids], "totalCount"}`.
- `POST /lead-export-history/re-export` with `{"trackingIds": [...], "formatId"?}`: free; returns one file (raw, not JSON) built from the stored copies. 1-5,000 ids. `formatId` must be a JSON number; split settings are ignored.
- `POST /lead-export-history/refresh` with `{"trackingIds": [...]}` or `{"filters": {"dateFrom", "dateTo", "exportMethod"}}`: free; overwrites the stored copies with current data and reports `changed` and `changedFields` per id. At most 5,000 rows.
- `POST /lead-export-history/refresh-and-export`: refresh, then return one file like re-export; it also clears the change markers.
- All three POSTs share the 5-per-minute export limit.

## 7. Export formats

- Create (`POST /export-formats`): `name` (1-100 characters), `fileType` (`csv`, `json`, `xlsx`) and `fieldMappings` (at least one `{"sourceField", "exportName", "type": "field" | "spacer"}`) are required. Optional: `startRow` (1-100), `includeHeaders` (default true), `splitFiles` (default false), `splitMode` (`max_rows`, `equal_parts`, `parts_max_rows`), `maxRowsPerFile` (10-100,000; for XLSX keep it at 20,000 or less), `numberOfParts` (2-100), `includeTotal` (for XLSX it must stay `false` above 20,000 leads, since the total file holds every lead), `combinedAssigneeColumnName`, `combinedFileNameColumnName` (1-50 characters, for email-schedule combined files). Use whole numbers.
- `PATCH` updates any of those fields; `POST /export-formats/{id}/set-default` makes it the default.
- `sourceField` keys: `lastUpdated` (Last Buying Signal), `lastUpdatedReason` (Buying Signal Type), `wOgrequestedUrl`, `wProfileDate`, `wRedirectYn`, `wSimpleCrawledUrl`, `wCompanyName`, `aiCategoryEstimation`, `wPhonePrimary`, `wPhoneTotal`, `wEmailPrimary`, `wEmailPrimaryMx`, `wEmailTier1` to `wEmailTier6`, `wEmailDomainMx`, `wEmailSecurity`, `wProfileCategory`, `wProfileDescriptionShort`, `wProfileDescriptionLong`, `wAddressFull`, `wAddressStreet`, `wAddressCity`, `wAddressState`, `wAddressZip`, `wAddressSource`, `wPlatform`, `wBookingOpsCrmSoftware`, `wReputationChatSoftware`, `wMarketingEmailSoftware`, `wAdPixels`, `wPaymentFinancingSoftware`, `wEmployeeCount`, `wOpenPositions`, `wFoundingDate`, `wNumLocations`, `wIsChain`, `wRatingValue`, `wReviewCount`, `wPriceRange`, `wLegalName`, `wIndustryNaics`, `wDuns`, `wFounder`, `wFounderEmail`, `wFounderPhone`, `wGeoLat`, `wGeoLong`, `wOpeningHours`, `wAreaServed`, `wOffers`, `wBrand`, `wSlogan`, `wLogoUrl`, `wSocialProfiles`, `wTimeScraped`. An unknown key produces an empty column.

## 8. Export blacklist

- `POST /export-blacklist` with `{"value": "example.com", "entryType": "domain"}`, or `{"entries": [...]}` for several (one invalid entry rejects the batch). `value` is 1-500 characters, stored as sent.
- A lead is left out of exports and previews when its registered or crawled URL contains the value (or the value contains the URL), or its primary email's domain equals the value. Because of the second test, once the blacklist has any domain entry, a lead whose Registered URL or Crawled URL is empty is also left out. `GET /leads`, email schedules, re-export and refresh-and-export do not apply the blacklist.
- `GET /export-blacklist` lists entries; `DELETE /export-blacklist/{id}` removes one.

## 9. Filter presets

Presets are used only by email schedules. `POST /filter-presets` body: `{"name": "TX dentists", "filters": {"hash": "<search string>"}}`. The search string is URL-query style (a leading `#` is optional). Keys the schedule engine reads:

| Key | Meaning | Format |
|---|---|---|
| `ni` / `ne` | Company name include / exclude | JSON array, URL-encoded |
| `ui` / `ue` | Registered URL include / exclude | JSON array |
| `cli` / `cle` | Crawled URL include / exclude | JSON array |
| `di` / `de` | Short description include / exclude | JSON array |
| `nkl`, `ukl`, `ckl`, `dkl` | Keyword list ids; positive lists become keywords, negative lists become exclusions | comma-separated ids |
| `si` / `se` | States include / exclude | comma-separated codes, no spaces |
| `ci` / `ce`, `zi` / `ze` | City / ZIP include / exclude | JSON array |
| `rf` | Redirect filter | `yes` / `no` |
| `orc` | Keyword columns (as `orColumns`) | comma-separated |
| `q` | Plain substring search | text |
| `urf` | Buying signal types | comma-separated |

A hash has no key for `positiveKeywords`: `ni`, `ui`, `cli` and `di` each match only their own column and none of them searches AI Categories, while keywords from a positive keyword list in `nkl` are searched like `positiveKeywords` (including AI Categories). So to reproduce a `positiveKeywords` search, create a positive keyword list and put its id in `nkl`. A positive filter is required: a non-empty `ni`, `ui`, `cli` or `di`, or the id of one of your own positive keyword lists (with at least one keyword) in `nkl`/`ukl`/`ckl`/`dkl`. A negative list alone does not count, ids that are not yours are dropped, and a preset without a positive filter silently sends nothing. Example in Python: `"#" + urllib.parse.urlencode({"ni": json.dumps(["*dental*", "*dentist*"]), "si": "TX,OK"})`. Other dashboard filters are not applied by schedules. Presets cannot be edited: create a new one, point the schedule at it with `PATCH` (add `"isActive": false` to the same call if the user should review the new search before it sends), then delete the old one (deleting a preset first deletes its schedules).

## 10. Keyword lists

- `POST /keyword-lists`: `{"name", "type": "positive" | "negative", "keywords": ["*dental*", ...], "sourceCategories"?: ["Dentists"]}`. `name` is trimmed and must be 1-255 characters, `keywords` a non-empty array of strings, `sourceCategories` an array of strings or null. Other fields are ignored; invalid input returns `400 validation_error`.
- `PUT /keyword-lists/{id}`: any of `name`, `type`, `keywords`, `sourceCategories`.
- `GET /keyword-lists` returns every list with its refinement state (`autoRefineEnabled`, `refinementStatus`: null, `running`, `paused`, `completed`, `failed`; `scoresPerRound`, `pairedListId`). `DELETE` does not delete the paired list.
- Lists are not applied to API searches automatically; pass their `keywords` as `positiveKeywords` or `negativeKeywords`.
- A running refinement rewrites the list's `keywords`, and email schedules whose preset names the list in `nkl`/`ukl`/`ckl`/`dkl` use the new keywords on their next send. Deleting a list (or `POST /ai/generate-keywords`, which deletes them all) silently removes it from those presets: a deleted negative list stops excluding leads, so the schedule can send and charge for more leads, and a preset whose only positive filter was a deleted list stops sending. Regenerated lists get new ids, so point the schedule at a new preset that uses them.

## 11. Email schedules

| Field | Required | Notes |
|---|---|---|
| `name` | yes | 1-100 characters |
| `filterPresetId` | yes | JSON number of your preset |
| `intervalValue` | yes | Whole number 1-720 |
| `intervalUnit` | yes | `hours` or `days` |
| `recipients` | yes | `[{"email": "a@b.com", "splitParts"?: [1]}]`, 1-50 |
| `isActive` | no | Default `true`. Create with `false`, then activate with `PATCH`. |
| `exportFormatId` | no | JSON number or null (default format) |
| `maxLeadsPerEmail` | no | 1-100,000 or null. New leads past the cap, and new leads your credits could not cover, are kept in your lead reservoir (one pool per account, shared by your schedules and API exports, up to your plan's monthly credit allowance; when it is full, more leads are not kept) and sent with a later email. On credit plans, leads you already received that do not fit are not kept. |
| `distributionMode` | no | `full_copy` (default) or `split_evenly` |
| `fullCopyRecipients` | no | Plain email strings, up to 50; receive everything in `split_evenly` mode when at least 2 `recipients` are active (with only 1 active recipient the schedule sends as `full_copy` and these addresses get nothing) |
| `combinedFileEnabled`, `combinedRecipients` | no | A merged file for split formats, sent to plain email strings (up to 50) |
| `combinedIncludeAssignee`, `combinedIncludeFileName`, `combinedColumnPosition` (`beginning` / `end`), `combinedAssigneeColumnName`, `combinedFileNameColumnName` | no | Combined-file columns |

- Each run sends leads whose Last Buying Signal is newer than the last send (the first run sends every match, up to 10,000), puts new leads first, caps them at your spendable credits and `maxLeadsPerEmail`, sends the email, then charges 1 credit per new lead. New leads cut by either cap are kept in the lead reservoir (one pool per account, shared by your schedules and API exports, up to your plan's monthly credit allowance; when it is full, more leads are not kept) and are added to a later email of the schedule, where they cost 1 credit each. On credit plans, leads you already received that do not fit are not kept. A lead that shares a primary phone or email with one you already received counts as received and is free. The schedule pauses only when your spendable balance is 0. There is no time-of-day setting: the schedule is checked every 15 minutes and sends once the interval has passed since the previous send.
- `pauseReason`: `null` (running), `user` (paused by the user) or `insufficient_credits` (paused automatically; resumes when credits are added, the overage budget is enabled, or a trial switches to a larger plan). `PATCH {"isActive": true}` with no credits returns `200` but leaves the schedule paused with `insufficient_credits`.
- `POST /email-schedules/{id}/trigger` ignores the interval and sends now. `400` if the schedule is paused. A `200` does not prove an email went out (for example, no new leads); compare `lastSent` and `totalSentCount`. A `500` is not safe to repeat.
- Deleting a schedule also deletes its list of recipients who unsubscribed.

## 12. AI endpoints

- `POST /ai/suggest-categories`: required `companyName`, `companyDescription`, `productService`; optional `companyWebsite` (the server reads the site), `excludeCategories` (up to 50). Each text at most 2,000 characters. Response `{"data": {"categories": [...], "websiteAnalysisFailed": false}}`. Not saved.
- `POST /ai/generate-keywords`: no body. Cancels pending jobs, deletes ALL keyword lists, then (only when `GET /me` shows non-empty `targetCategories`) generates one positive and one negative list per category with refinement turned on. Answers `{"data": {"message": "Keyword generation started", "status": "pending"}}` even when nothing will be generated.
- `GET /ai/keyword-status`: `{"data": {"hasJobs", "jobs": [{"id", "status": "pending" | "completed" | "failed" | "cancelled", ...}]}}` for the last 24 hours.
- `GET /ai/auto-refine/status` (optional `?listId=`): per list `listId`, `name`, `type`, `autoRefineEnabled`, `refinementStatus`, `currentRound`, `maxRounds`, `scoresPerRound` (`round`, `score` 1-10, `reasoning`), `lastRefinedAt`, `pairedListId`.
- `POST /ai/auto-refine/enable` / `disable` with `{"listId": <number>}`: applies to the list and its pair. Enabling resets the score history; at most 3 refinements run at once across all customers, and a start that could not run is not queued, so check the status and enable again later if `refinementStatus` stays null.

## 13. Enrichments

- `POST /enrichments` body: `urls` (array of strings; up to 100,000 after duplicates are removed), `maxCredits` (whole number of at least 1; when omitted the run can spend up to your current credit balance but never the overage budget, while a `maxCredits` above your balance lets it draw on the overage budget, which is billed to the card), `databaseMatchPercent` (0-100, default 100: the share of `maxCredits` that may go to new database matches; unused budget flows to live fetches, never the reverse), `contactTypes` (`phone`, `email`, `either`; the minimum a result must carry to be delivered and charged), `emailResults` (boolean; default from the account; the email links to the dashboard). Unknown option names are ignored; `null` values are rejected. Response `202 {"data": {"id", "status": "queued", "urlCount", "duplicatesRemoved", "skippedInvalid"}}`. `400 invalid_request` for bad options or no usable URLs.
- URLs are matched by host name (path, `www.` and protocol ignored for matching; the full URL is used for live fetches). Private and local addresses are refused.
- `GET /enrichments?limit=&offset=` (limit 1-100, default 25) and `GET /enrichments/{id}`: `{"id", "status", "createdAt", "deliveredAt", "options", "urls": {"submitted", "duplicatesRemoved", "skippedInvalid"}, "database": {"matched", "delivered", "charged", "free", "skippedOverBudget", "noRequestedContact"}, "live": {"sentToScraper", "delivered", "failed", "noRequestedContact", "skippedOverBudget"}, "credits": {"databaseCharged", "liveCharged", "total", "final"}, "result": {"rowCount", "ready"}, "error"}`. `error` can be set on a delivered run; it is informational.
- Run status: `queued`, `matching`, `scraping`, `merging`, `delivering`, then `completed`, `partial` (delivered, but a credit limit stopped it early) or `failed` (no file).
- Charging: database matches are charged within seconds of submitting; live fetches are charged when the file is published (`credits.final` becomes true). Live fetch tenths of a credit accumulate across runs and are deducted in whole credits.
- `GET /enrichments/{id}/download`: `{"data": {"downloadUrl", "expiresInSeconds": 300}}`, a link to fetch without the API key. `409 not_ready` while the run is in progress. Download within 90 days.
- Result CSV: one row per usable URL (duplicates and unparseable URLs are only counted), columns `Submitted URL`, `Status`, `Source` (`database`, `live_scrape`, `none`), `Credits Charged`, then the 59 lead columns. Status values: `matched_paid` (1 credit), `matched_free`, `scraped` (0.1 credit), `scrape_failed`, `skipped_db_budget`, `skipped_scrape_budget`, `matched_no_contact`, `scraped_no_contact`, `skipped_invalid` (all free except the two noted).

## 14. Account and billing

**`GET /me`** fields: `id`, `email`, `firstName`, `lastName`, `companyName`, `companyWebsite`, `smbType` (always `other`), `subscriptionPlan`, `subscriptionStatus`, `isSubscribed`, `freeTrialEligible`, `trial` (`isTrialing`, `startedAt`, `endsAt`, `plan`, `outcome`, `convertedAt`, `firstChargeAmountCents` (list price), `freeTrialUsedAt`, `daysRemaining`), `freeLeadsUsed`, `totalLeadsExported`, `monthlyLeadsExported`, `onboardingCompleted`, `targetCategories`, `targetLocations`, `monthlyCredits`, `monthlyCreditsUsed`, `monthlyCreditsRemaining`, `permanentCredits`, `temporaryCreditsRemaining`, `totalCreditsRemaining`, `creditOverageRate` (cents per credit), `autoTopUp` (`enabled`, `triggerType`, `triggerAmount`, `purchaseType`, `purchaseAmount`, `capType`, `capAmount`), `overageBudget` (`enabled`, `budgetType`, `budgetAmount`, `creditsUsedThisCycle`, `amountUsedThisCycleCents`, `pendingChargeCents`, `billingSuspended`). During a trial `monthlyCredits` is the trial allowance.

**`PATCH /me`:** `firstName`, `lastName` (50 characters each), `companyName` (100), `companyWebsite` (500, http or https; `acme.com` is stored as `https://acme.com/`). `null` clears a field.

**`POST /purchase-credits`:** `creditCount` (whole number, 100 up to 5x the monthly allowance: Starter 2,500, Growth 10,000, Scale 50,000, Platinum 500,000, Enterprise 1,250,000) or `dollarAmount` (at least 1; can buy fewer than 100 credits). Optional `Idempotency-Key` header or `idempotencyKey` (8-64 letters, digits, `_`, `-`): a retry with the same key within 24 hours is not charged again (Stripe keeps keys for about a day); after that, check `permanentCredits` in `GET /me` before buying again. Response `{"data": {"success", "creditsAdded", "totalPermanentCredits", "amountCharged", "amountChargedCents", "pricePerCredit", "stripePaymentIntentId"}}`. Errors: `400 no_payment_method`, `402 payment_failed` or `authentication_required` (use a new key after the card is fixed), `403`, `409 plan_change_in_progress` (nothing charged; retry shortly). Credits bought while a paid-plan downgrade is pending (not a trial plan switch) expire at the next renewal with the rest of the balance. New credits also restart email schedules paused for insufficient credits.

**Auto top-up (`PATCH /auto-top-up`):** replaces the whole configuration. Enabling needs `triggerType` and `purchaseType` (`credits` or `dollars`), `triggerAmount` and `purchaseAmount` (purchase at least 100 credits or $1, at most the 5x limit), optional `capType` (`credits`, `dollars` or null) with `capAmount` (a rolling 30-day cap). It compares only the permanent balance with the trigger, and does not run during a trial or while a downgrade is pending. `{"enabled": false}` clears it. `GET` adds `monthlyUsage` and `pricePerCredit`.

**Overage budget (`PATCH /overage-budget`):** `{"enabled": true, "budgetType": "dollars" | "credits", "budgetAmount": N}` (`budgetType` defaults to `dollars` when omitted; at most the dollar or credit equivalent of 5x the monthly allowance per billing cycle). Enabling it also restarts email schedules paused for insufficient credits. Usage past zero is charged to the card once a day (amounts under $0.50 roll to the next day); a failed charge pauses overage use, and 5 failures turn it off. `{"enabled": false}` stops new usage; anything already used is still charged. `GET` returns `enabled`, `budgetType`, `budgetAmount`, `cycleUsage`, `pendingCharge`, `headroomCredits`, `billingSuspended`, `pricePerCredit`.

**`POST /subscription/change-plan`:** `{"targetPlan": "starter" | "growth" | "scale", "mode"?: "switch_trial" | "activate_now"}`. Only between Starter, Growth and Scale. While a downgrade is pending, moves are judged against the plan the downgrade came from, not the current plan. `GET /me` does not show a pending downgrade, so ask the user.

| Situation | Result | Response `changeType` |
|---|---|---|
| Upgrade: a plan above the current one or, while a downgrade is pending, the plan the downgrade came from or a higher one | Full new price charged now, billing cycle restarts, all unused credits kept, new allowance granted | `upgrade` (with `newMonthlyCredits`, `permanentCreditsAdded`) |
| Downgrade: a lower plan or, while a downgrade is pending, any plan below the one the downgrade came from (even one above the current plan, for example Scale, then Starter, then Growth) | No charge now; at the next renewal the whole balance expires, then the new plan's allowance is granted | `downgrade` |
| Trial, `switch_trial` (default) | No charge; trial end date unchanged; allowance = 50% of the new plan minus credits already used | `trial_switch` (with `newCredits`, `creditsRemaining`, `trialEndsAt`) |
| Trial, `activate_now` | Trial ends, the target plan's full price is charged now, full allowance granted, the target plan's remaining trial allowance is added to permanent credits | `trial_activated` (with `newCredits`, `permanentCreditsAdded`, `amountChargedCents`; `alreadyProcessed: true` means it was already done) |

Errors: `400` (same plan, `already_active`, Platinum/Enterprise/legacy plan), `402 payment_declined` (trial activation; `trialStillActive: true`, nothing changed) or `payment_failed` (upgrade), and `409` codes where nothing changed: `plan_change_in_progress` (retry in a moment), `plan_state_changed` (re-read `GET /me`, then retry), `trial_conversion_pending` (wait `retryAfterSeconds`, then re-read: the trial will have converted), `trial_state_changed` (re-read state), `trial_purchase_rate_lock` (credits bought during the trial block moving to a plan with a higher per-credit price; keep the plan, activate it, or use those credits first), `renewal_in_progress` (follow the message), `activation_in_progress` (retry after a refresh). A `500` or timeout may have charged: check `GET /me` before retrying.

**`POST /subscription/cancel`:** no body. `{"data": {"message", "cancelAtPeriodEnd": true, "currentPeriodEnd" (Unix seconds), "status", "isTrial", "trialEndsAt"}}`. Calling it again is harmless. There is no API to undo it.

## 15. Signing up: `POST /purchase`

No API key. Body `{"email", "plan": "starter" | "growth" | "scale" | "platinum" | "enterprise", "trial"?: false}`: Starter, Growth and Scale start a 14-day free trial by default when eligible; only the JSON boolean `false` skips it (the plan price is then charged at checkout and `reason` is `not_requested`). Response `201 {"data": {"checkoutUrl", "expiresAt", "plan", "trial": {"eligible", "days", "credits", "endsAt", "note", "reason"?}, "message"}}`. `reason` when not eligible: `not_requested`, `plan_not_eligible`, `trial_already_used`, `currently_subscribed`. Errors: `400 invalid_email` / `invalid_plan`, `403 forbidden` / `disposable_email`, `409 already_subscribed`, `410 plan_retired`. The API key is emailed after checkout (an address that already has an account first gets an email to confirm the purchase). `POST /claim-key` cannot succeed because no claim token is issued; do not use it.

## 16. Errors and headers

Error body: `{"error": "<code>", "message": "..."}`, plus `details` for `validation_error` and `invalid_request`, `creditsRemaining` for `insufficient_credits`, `reason` for `gone`, `retryAfterSeconds` for `trial_conversion_pending`, `declineCode` and `trialStillActive` for `payment_declined`. A rate-limit 429 has `"error": "Rate limit exceeded"` and `retryAfter`; the lead-search capacity 429 has `"error": "lead_query_capacity_busy"`. Malformed JSON gives `400 bad_request`; a body over 100 KB (32 MB for enrichments) gives `413 payload_too_large`.

Safe to retry: every `429` and `503` (after `Retry-After`), every `GET` except `GET /leads`, `PATCH` calls, `POST /subscription/cancel`, and `POST /purchase-credits` with the same idempotency key within 24 hours. Not safe to repeat blindly after a `500`, a timeout or a `524`: `GET /leads`, `POST /leads/export`, `POST /enrichments`, `POST /email-schedules/{id}/trigger`, `POST /subscription/change-plan`, and creating presets, lists or schedules (a retry can create a duplicate; list first).

Response headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (ISO timestamp), `Retry-After` (seconds), `X-Credits-Capped`, `X-Lead-Query-Path`.
