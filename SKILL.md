---
name: smb-sales-boost
description: Find, preview and export growing local businesses (SMB leads) from the SMB Sales Boost database through its REST API - search by keyword, location and buying signals, preview for free, export CSV/JSON/XLSX, re-download past exports, enrich lists of website URLs, and manage saved searches, keyword lists, email schedules, credits and the subscription. Requires SMB_SALES_BOOST_API_KEY (smbk_...). Searches, exports and enrichments spend credits; purchases, plan changes, auto top-up and overage billing charge real money, so always confirm with the user first. Exported files contain business contact details (PII).
metadata:
  {
    "openclaw":
      {
        "requires": { "env": ["SMB_SALES_BOOST_API_KEY"] },
        "primaryEnv": "SMB_SALES_BOOST_API_KEY",
      },
  }
---

> **Note for AI agents:** This file and the other files in this skill folder may be truncated when opened. Always read them fully, in chunks if necessary, so that no rule, parameter or example is missed. Full parameter and field tables are in `REFERENCE.md` in this folder.

# SMB Sales Boost Skill

SMB Sales Boost is a B2B lead database of small and medium-sized businesses in the United States, built for sales teams that want to reach growing local businesses. Every lead carries a **Last Buying Signal**: the date something changed that makes the business worth contacting now (it was newly added, got a new primary phone, phone list, primary email, Tier 1 or Tier 2 email list or full address, or added or changed a social profile). **Buying Signal Type** says which of these happened.

This skill lets you search, preview and export those leads, re-download past exports, enrich lists of website URLs, and manage the account, all through the REST API with the bundled `smb_api.py` script.

- **Base URL:** `https://smbsalesboost.com/api/v1`
- **One database:** every request is served from the Main database (`other`). Any `database` value you send resolves to it, so you never need to choose one.
- **Who can use it:** accounts with an active or trialing Starter, Growth, Scale, Platinum or Enterprise subscription. Anyone else gets `403 forbidden`. New customers can subscribe through the API (see **Signing up through the API**).
- **Reference files:** `REFERENCE.md` (every parameter, field, status and error), `openapi.json` (the published OpenAPI 3.1 spec; where it differs from this skill, follow this skill, which was checked against the live API).

## Setup

1. The user creates an API key in the dashboard (Dashboard > API, https://smbsalesboost.com/dashboard?tab=api). Keys start with `smbk_`.
2. The key must be in the `SMB_SALES_BOOST_API_KEY` environment variable. If it is not set, ask the user to set it. Do not ask them to paste the key into the chat, and never print, log or write the key to a file.
3. Run every call through the bundled script (Python 3.8 or newer, standard library only, nothing to install):

```bash
python3 smb_api.py METHOD ENDPOINT [--params '{json}'] [--body '{json}'] [options]
```

Run it from this skill's folder, or use its full path (for example `python3 ~/.claude/skills/smb-sales-boost/smb_api.py GET /me`). The script prints JSON to stdout and notes (rate-limit headers, warnings) to stderr. Do not hand-build `curl` commands; the script encodes parameters correctly, saves files safely and enforces the confirmation rules below.

**Quoting user text safely:** `--params` and `--body` take JSON inside single quotes. If any value contains a single quote (for example a company name like `Joe's Bakery`), write the JSON to a file with your file tool and pass `--params-file PATH` or `--body-file PATH` instead. Never paste user text into a shell command unquoted.

**Windows:** run the script with `py -3 smb_api.py` (or `python smb_api.py`) and always pass JSON through `--params-file` or `--body-file`. Command Prompt does not treat single quotes as quotes, and Windows PowerShell 5.1 removes the inner double quotes, so inline JSON fails there. The script reads UTF-8 files (with or without a byte-order mark) and UTF-16 files that start with a byte-order mark.

## Rules for every session

1. **Real money: confirm first.** These calls charge, or authorize charges to, the card on file: `POST /purchase` (subscription checkout), `POST /purchase-credits`, `POST /subscription/change-plan` (an upgrade or an early trial activation charges immediately), and `PATCH /auto-top-up` or `PATCH /overage-budget` with `enabled: true` (automatic future charges). Before any of them, tell the user exactly what will happen and what it costs (amount and date), and wait for an explicit yes. **All charges are final; there are no refunds.** The script refuses these calls unless you add `--confirm`; add it only after the user has approved that exact action. Adding credits (a purchase, an upgrade, an early trial activation or a trial switch to a larger plan) or enabling the overage budget also restarts every email schedule that was paused for insufficient credits (with the overage budget, their leads can be billed to the card), so check `GET /email-schedules` and tell the user first.
2. **Credits: preview first, cap every spend.** `GET /leads`, `POST /leads/export` and `POST /enrichments` spend credits. Run `GET /leads/preview` first (it is free) to check the match count and quality, keep `limit` small, and always pass `maxCredits`. After `GET /leads` or `POST /leads/export`, tell the user `creditsUsed` and the remaining balance: `creditsRemaining`, or `totalCreditsRemaining` from `GET /me` when the page came back empty (the field then shows 0). `POST /enrichments` returns neither: report `credits.total` from `GET /enrichments/{id}` (final once `credits.final` is true) and the balance from `GET /me`.
3. **Email to real people: confirm first.** An active email schedule emails real recipients and spends credits, and `POST /email-schedules/{id}/trigger` sends immediately. Create schedules with `"isActive": false`, review them with the user, then activate. The script requires `--confirm` for creating an active schedule, activating one, triggering one, and changing the recipients, preset or `maxLeadsPerEmail` of a schedule that is not paused in the same call. Editing a keyword list (`PUT /keyword-lists/{id}`) or turning on auto-refine also changes what the schedules whose preset uses that list send, so check `GET /filter-presets` and pause those schedules first.
4. **Destructive calls: confirm first.** `POST /ai/generate-keywords` deletes all of the user's keyword lists before generating new ones. Every `DELETE` is permanent, and deleting a filter preset also deletes the email schedules that use it. `POST /subscription/cancel` cancels the subscription. The script requires `--confirm` for all of these.
5. **Never blindly repeat a call that may have charged.** If `GET /leads`, `POST /leads/export`, `POST /enrichments`, a schedule trigger, a plan change or a credit purchase fails with a 5xx, a timeout or a dropped connection, it may still have completed. Follow the `advice` field the script prints (check `GET /me`, `GET /export-history`, `GET /enrichments` or `GET /email-schedules` first). The script retries `429` and `503` responses by itself; those are always safe because nothing was done.
6. **Protect personal data.** Results and files contain business phone numbers and email addresses. Keep files in the output folder, do not paste full lead lists into public channels, and only show the user what they asked for.
7. **Stay on documented endpoints.** Use only the endpoints in this skill and `REFERENCE.md`. Integrations, API key management, billing details and undoing a cancellation are dashboard-only (see **Not available through the API**).
8. **Data accuracy.** Contact details come from public sources and are not verified; revenue, employee and similar figures are estimates. Say so if the user is about to rely on them.

## Plans and credits

Every plan is credit-based. One credit buys one new lead.

| Plan | Price / month | Monthly credits | Extra credits | Max credits per purchase | 14-day free trial |
|---|---|---|---|---|---|
| Starter | $49 | 500 | $0.10 each | 2,500 | Yes, 250 trial credits |
| Growth | $149 | 2,000 | $0.075 each | 10,000 | Yes, 1,000 trial credits |
| Scale | $499 | 10,000 | $0.05 each | 50,000 | Yes, 5,000 trial credits |
| Platinum | $1,999 | 100,000 | $0.03 each | 500,000 | No |
| Enterprise | $4,999 | 250,000 | $0.02 each | 1,250,000 | No |

How charging works:

- **1 credit per new lead** returned by `GET /leads` or exported by `POST /leads/export`.
- **Free:** leads you have already received through any channel (dashboard, API, email schedules, enrichments), and new leads that share a primary phone or primary email with a lead you already received. Previews are always free.
- **Automatic cap:** on every charged call, new leads come first and are cut to `min(maxCredits, your spendable balance)`; already-received leads fill the remaining slots for free. If your balance was the limit, the response has the header `X-Credits-Capped: true` (the script prints a note). Leads cut by a cap are dropped, not moved to the next page (exception: on `POST /leads/export` with `maxLeads`, they go to the reservoir and are added to your next `maxLeads` export, where new leads cost credits).
- **Spending order:** temporary (promotional) credits, then the monthly allowance, then permanent credits (rolled-over and purchased), then the optional overage budget (billed to the card daily).
- **Unused monthly credits roll over** into the permanent balance while the subscription stays active. Downgrading makes the whole remaining balance expire at the next renewal; cancelling forfeits it when the subscription ends.
- **Auto top-up** (if the user enabled it) buys credits automatically when the permanent balance drops below a threshold, which charges the card.
- `GET /me` is the authoritative balance: `totalCreditsRemaining` (temporary + monthly remaining + permanent), plus `monthlyCredits`, `monthlyCreditsUsed`, `monthlyCreditsRemaining`, `permanentCredits`, `temporaryCreditsRemaining` and `creditOverageRate` (cents per credit).

**Free trial:** Starter, Growth and Scale start with a 14-day free trial. A payment method is required and nothing is charged until day 14 (the user's bank may show a temporary authorization that disappears within a few days); on day 14 the plan price is charged automatically unless the subscription is cancelled first. Trial credits are 50% of the plan's allowance and the full allowance is granted when the trial converts. One trial per person, checked by email address and payment method. During a trial the API works normally, except that auto top-up and the overage budget cannot be enabled. While `GET /me` shows `trial.isTrialing: true`, the plan price has not been charged yet (credits bought with `POST /purchase-credits` during the trial are charged immediately); `trial.firstChargeAmountCents` (the plan's list price; a discount on the account can make the actual charge lower) is charged on `trial.endsAt` unless the user cancels first.

## Finding leads: the standard workflow

1. **Translate the request into filters.** Use several wildcard keyword variations (they are OR'ed): "dentists in Texas" becomes `positiveKeywords: ["*dental*", "*dentist*", "*orthodont*"]` and `stateInclude: ["TX"]`.
2. **Preview (free):** `GET /leads/preview`. Read `data.pagination.total` for the match count and look at a few results (contacts are masked).
3. **Agree on the spend:** tell the user how many leads match and that each new lead costs 1 credit.
4. **Get the leads:** either `GET /leads` (results come back in the response, one page at a time) or `POST /leads/export` (a CSV, JSON or XLSX file; better for more than a few dozen leads). Always pass `maxCredits`. `GET /leads` does not apply the user's export blacklist (preview and export do), so leads from blacklisted domains can come back and are charged like any other new lead. If the user keeps an export blacklist, use `POST /leads/export`.
5. **Report:** show results in a short table and state `creditsUsed` and the balance (`creditsRemaining`, or `GET /me` if the page was empty).

```bash
# 1-2. Preview: free, contacts masked
python3 smb_api.py GET /leads/preview --params '{"positiveKeywords":["*dental*","*dentist*","*orthodont*"],"stateInclude":["TX"],"limit":10}' --compact

# 4a. Search: returns up to 25 leads, never spends more than 25 credits
python3 smb_api.py GET /leads --params '{"positiveKeywords":["*dental*","*dentist*","*orthodont*"],"stateInclude":["TX"],"limit":25,"maxCredits":25}' --compact --out tx-dentists

# 4b. Export: a CSV file of up to 500 leads, never more than 500 credits
python3 smb_api.py POST /leads/export --body '{"filters":{"positiveKeywords":["*dental*","*dentist*","*orthodont*"],"stateInclude":["TX"]},"maxResults":500,"maxCredits":500}'
```

`--compact` prints only the key fields of each lead; `--out NAME` also saves the full response as `NAME.json` in the output folder. Use both for larger searches so the full records do not flood the conversation.

## Search parameters

`GET /leads` and `GET /leads/preview` take the same filters as query parameters. With the script, give lists as real JSON arrays; it encodes each parameter in the format the API expects. The most useful ones:

| Parameter | What it does |
|---|---|
| `positiveKeywords` | Keywords to include (OR). Matched as case-insensitive substrings against Registered URL, Crawled URL, Company Name and AI Categories. `*` is a wildcard: `*auto*repair*` matches "auto body repair" and "automotive repair". In URL columns, spaces also act as wildcards. |
| `negativeKeywords` | Keywords to exclude: a lead is dropped if any of them matches any of those columns. |
| `orColumns` | Columns the keywords search. Default `["wOgrequestedUrl","wSimpleCrawledUrl","wCompanyName","aiCategoryEstimation"]`. Adding `"wProfileDescriptionShort"` also searches the short description, but it moves keyword searches onto the slow path (up to about 110 seconds); keyword searches stay fast only while every `orColumns` entry is one of the four defaults. To search AI categories only, use `["aiCategoryEstimation"]`. |
| `nameIncludeTerms` / `nameExcludeTerms` | Company-name terms. While `wCompanyName` is in `orColumns` (the default), include terms are OR'ed with `positiveKeywords`, so adding them to a keyword search widens it instead of narrowing it; each include term becomes required only when `wCompanyName` is left out of `orColumns`. Exclude terms always remove matching leads. |
| `urlIncludeTerms` / `urlExcludeTerms`, `crawledUrlIncludeTerms` / `crawledUrlExcludeTerms` | Website URL terms. Same rule as company-name terms, with `wOgrequestedUrl` (Registered URL) and `wSimpleCrawledUrl` (Crawled URL) in `orColumns`. |
| `descriptionIncludeTerms` / `descriptionExcludeTerms` | Short-description terms. Each include term is required (AND) unless `wProfileDescriptionShort` is in `orColumns`. |
| `stateInclude` / `stateExclude` | Two-letter uppercase state codes, exact match: `["TX","OK"]`. |
| `cityInclude` / `cityExclude` | City names, substring match: `["Austin"]`. |
| `zipInclude` / `zipExclude` | ZIP codes, exact match. |
| `lastBuyingSignalFrom` / `lastBuyingSignalTo` | Last Buying Signal date range, inclusive: ISO 8601 (`2026-09-01`, `2026-09-30T23:59:59Z`) or relative (`rel:7d`; units `h`, `d`, `w`, `m` = months, `y`). A date-only `To` means the start of that day, so to include all of September 30 send `2026-09-30T23:59:59Z`. An invalid date fails with `500`, so check the format. |
| `buyingSignalTypeFilter` | Only leads whose latest buying signal is one of these types (exact, case-sensitive): `Newly Added`, `Phone Primary`, `Total Phones`, `Email Primary`, `Tier 1 Emails`, `Tier 2 Emails`, `Address Full`, `Instagram`, `LinkedIn`, `Facebook`, `YouTube`, `TikTok`, `X (Twitter)`, `Google Maps`, `Yelp`, `Pinterest`, `Other Social`. |
| `phonePrimaryEmptyFilter`, `emailPrimaryEmptyFilter` | `exclude_empty` keeps only leads that have that contact detail (a lead without contacts still costs a credit), `only_empty` the opposite. |
| `minRatingValue` / `maxRatingValue`, `minReviewCount` / `maxReviewCount`, `minNumLocations` / `maxNumLocations` | Numeric ranges. |
| `registrationDateFrom` / `registrationDateTo` | Domain registration date as a plain `YYYY-MM-DD`, for newly launched businesses (it is compared as text, so a `rel:` value in `registrationDateFrom` leaves out the boundary day; see `REFERENCE.md`). |
| `websiteSchemaFilter` | Website schema types, for example `["LocalBusiness","Dentist"]`. `GET /leads/other/schema-types` lists the values (the first call can be slow). |
| `redirectFilter` | `yes` or `no`: whether the registered domain redirects elsewhere. |
| `search` | Plain substring search over company name, both URLs, short description, city and primary phone (not a wildcard search). |
| `sortBy` / `sortOrder` | Default `lastBuyingSignal` / `desc` (newest buying signals first). |
| `page` / `limit` | Default 1 / 100; `limit` is at most 1000. |
| `maxCredits` / `maxResults` | `GET /leads` (and the `POST /leads/export` body); preview ignores both. On `GET /leads` they cap the credits spent and the leads returned by that one call, and each page is a separate call. `maxCredits: 0` returns only leads you already have, free. |
| `excludePurchased` | `GET /leads/preview` (and the `POST /leads/export` body); `GET /leads` ignores it. `true` hides leads flagged `contactExported`. On preview it is applied after paging, so a page can hold fewer than `limit` leads and `pagination.total` still counts the hidden ones. |

**At least one positive filter is required:** `positiveKeywords`, `nameIncludeTerms`, `urlIncludeTerms`, `crawledUrlIncludeTerms` or `descriptionIncludeTerms`. Without one, `GET /leads` and preview return no leads, charge nothing and set `requiresKeywords: true`, and `POST /leads/export` returns `400 bad_request`. Location, date, signal and other filters only narrow a search; they cannot start one. Never send an empty keyword (`[""]`): it matches everything.

There are many more filters (Tier 1-6 emails, Total Phones, founders, software and ad-pixel columns, email provider and security, employee counts, NAICS, open positions and more). See `REFERENCE.md` for the complete list and formats. Unknown or misspelled parameters are ignored silently, which makes a search broader, so double-check names.

**Speed:** keyword, company-name, URL, state and city filters with the default sort answer in about a second. Other filters can take up to about 110 seconds (the script prints a note when a query took the slow path). Previewing first warms a 20-second cache, so an identical `GET /leads` right after it is fast.

### Results

```json
{"data": {
  "leads": [ { "id": 2841123, "Company Name": "Bright Smile Dental", "City": "Austin", "State": "TX",
               "Phone Primary": "(512) 555-0142", "Email Primary": "hello@brightsmiledentaltx.com",
               "Last Buying Signal": "2026-10-03T09:14:52.118Z", "Buying Signal Type": ["Phone Primary"], "...": "..." } ],
  "pagination": { "page": 1, "limit": 25, "total": 1834, "pages": 74 },
  "databaseType": "other", "creditsUsed": 18, "creditsRemaining": 4982, "maxCredits": 25 }}
```

- Each lead has 61 fields with display names (`Company Name`, `Phone Primary`, `Total Phones`, `Email Primary`, `Tier 1 Emails`, `AI Categories`, `Registered URL`, `Description Short`, `Founder Name(s)` and more; full list in `REFERENCE.md`). `Last Updated` is a deprecated duplicate of `Last Buying Signal`.
- `pagination.total` is the full match count before credit caps.
- `creditsRemaining` does not include overage headroom and shows `0` when a page is empty; check `GET /me` for the real balance.
- **Preview** masks contacts (`(512) ***-**42`, `hel*@brightsmiledentaltx.com`) except on leads you already have, which come back unmasked with `"contactExported": true`. Preview also flags a lead this way when one of its Total Phones or Tier 1-6 emails matches the primary phone or primary email of a lead you received before; `GET /leads` and exports still charge 1 credit for such a lead, so `contactExported: true` does not guarantee the lead is free. Preview never charges.

## Exporting files

`POST /leads/export` creates a file from a search and saves it to the output folder automatically.

```bash
python3 smb_api.py POST /leads/export --body '{"filters":{"positiveKeywords":["*med*spa*","*aesthet*","*botox*"],"stateInclude":["FL"],"lastBuyingSignalFrom":"rel:30d"},"maxResults":1000,"maxCredits":300}'
```

- **Body:** `filters` (the same filters as search, as real JSON arrays, except `search`, `page` and `limit`: the export ignores those three without an error, so a `search` term that narrowed your preview does not narrow the export, and the export can then match and charge for more leads; and never put `maxCredits`, `maxResults` or `excludePurchased` inside `filters`, where they are ignored), plus optional top-level `maxCredits`, `maxResults`, `maxLeads`, `excludePurchased`, `formatId` and `inline`. Use `selectedIds` (lead ids) instead of `filters` to export specific leads. The script converts list filters given as strings or numbers into real arrays (the export ignores other forms of most list filters), moves a misplaced `maxCredits`, `maxResults`, `maxLeads`, `excludePurchased`, `formatId` or `inline` out of `filters`, and refuses `filters.search`, `filters.page` and `filters.limit`.
- **Without `maxCredits` an export can spend your entire balance** (plus any overage budget). Always set it.
- `maxResults` caps the total leads in the file. `maxLeads` also caps it, but keeps the leads that did not fit (including those cut by `maxCredits`) in a "reservoir" that is merged into your next `maxLeads` export whatever its filters are, and new leads among them cost 1 credit each. The reservoir is one pool per account, shared with email schedules and capped at the plan's monthly credit allowance; leads that do not fit are not kept. Prefer `maxResults` unless the user wants that carry-over.
- `excludePurchased: true` exports only leads you do not have yet.
- **Formats:** without `formatId` the export uses your default export format (the one with `isDefault: true` in `GET /export-formats`, which can be XLSX), or the built-in CSV when you have none; a wrong `formatId` also falls back to it. JSON and XLSX come from a saved export format. There is no file-type option in the request, so to be sure of CSV, pass the id of a CSV format.
- **XLSX limit:** at most 20,000 rows in any one file (a single file, a split part, or the `includeTotal` file), and the `400 export_too_large` error arrives after the credits were charged. For bigger exports use a CSV format, or an XLSX format with `splitFiles: true`, `splitMode: "max_rows"`, `maxRowsPerFile` of 20,000 or less and `includeTotal: false`. If it happens, do not simply repeat the request (an export always picks new leads first, so a repeat can charge for different leads): re-run it with a CSV format, `maxCredits: 0` and no `excludePurchased`, which returns only leads you already have at no cost, or re-export the charged leads from the lead export history (below).
- **Size:** up to 100,000 leads per export. 5 export requests per minute.
- **Response:** the script saves each file and prints `leadCount`, `creditsUsed`, `creditsRemaining`, `exportId` and the saved paths. The default file contains 59 columns, including `Last Buying Signal` and `Buying Signal Type`.
- `inline: false` (or `--params '{"inline":false}'`) returns download links instead of file contents; the script then downloads the files itself. Use it for very large exports.

**Re-downloading:** every export file is kept for 90 days.

```bash
python3 smb_api.py GET /export-history --params '{"limit":10}'     # newest first: id, fileName, leadCount, expiresAt, downloadAvailable
python3 smb_api.py GET /export-history/412/download                 # saves the file
```

After 90 days (or for files exported before 26 September 2026, or whose upload failed) the download answers `410 gone`. The leads can still be re-exported **free** from the lead-level history:

```bash
python3 smb_api.py GET /lead-export-history --params '{"limit":50}'                    # every lead you received, with its tracking id
python3 smb_api.py POST /lead-export-history/re-export --body '{"trackingIds":[55123,55124]}'
```

`re-export` rebuilds a file from the stored copies (up to 5,000 tracking ids, no credits). `POST /lead-export-history/refresh` and `/refresh-and-export` first replace the stored copies with the current data (free, but the original copies are overwritten). See `REFERENCE.md`.

## Enriching a list of websites

`POST /enrichments` looks up a list of website URLs. URLs already in the database return the full lead record; others are fetched live from the website.

**Pricing:** 1 credit for a URL found in the database that you have not bought before; 0.1 credit for a URL fetched live; 0 for a URL you already have, one that returned no data, or one without the contact details you asked for. A URL found in the database is never fetched live. Submitted URLs and results are never added to the database. There is no pricing dialog through the API, so state these prices and agree on `maxCredits` with the user first (the script requires `--confirm`).

```bash
python3 smb_api.py POST /enrichments --body-file urls.json --confirm
# urls.json: {"urls": ["https://brightsmiledentaltx.com/contact", "acme-plumbing.com"], "maxCredits": 50, "contactTypes": "either", "emailResults": false}

python3 smb_api.py GET /enrichments/412             # poll: status queued, matching, scraping, merging, delivering, then completed / partial / failed
python3 smb_api.py GET /enrichments/412/download    # once completed or partial: saves the CSV (one row per usable URL with a Status column; duplicates and unparseable URLs are only counted)
```

- Up to 100,000 URLs per run (after removing duplicates). Send full URLs; a deep link can carry better contact data. Put the most important URLs first, because a credit limit trims from the end of the list.
- Options (exact names, top level of the body; a misspelled name is ignored and removes your limit): `maxCredits` (whole number of at least 1; when omitted the run can spend up to your current credit balance but never the overage budget, while a `maxCredits` above your balance lets it draw on the overage budget, which is billed to the card), `databaseMatchPercent` (0-100, default 100), `contactTypes` (`phone`, `email` or `either`, the default), `emailResults` (default: the account setting; set `false` to skip the notification email). Never send `null` for an option.
- Runs are asynchronous. Poll `GET /enrichments/{id}` every 30-60 seconds at first, then every few minutes; large lists can take hours. `partial` means the file is ready but a credit limit stopped the run early. `failed` never produces a file.
- Each `POST` creates a new run and there is no undo. If a submit fails or times out, check `GET /enrichments` before sending it again.

## Saved searches, keyword lists and email schedules

**Filter presets** exist to drive email schedules. `POST /filter-presets` takes `{"name": "...", "filters": {"hash": "..."}}`, where `hash` uses the dashboard's search format (`ni` company-name terms, `ui` URL terms, `si` states, `ci` cities and so on; see `REFERENCE.md`). A preset without a positive filter in that format (a company-name, URL or description term, or one of the user's own positive keyword lists) never sends anything. A hash has no key for `positiveKeywords`: `ni`, `ui`, `cli` and `di` each match only their own column and none of them searches AI Categories, while keywords from a positive keyword list in `nkl` are searched like `positiveKeywords` (Registered URL, Crawled URL, Company Name and AI Categories). To make a schedule send the same leads as a `positiveKeywords` search you previewed, first create a positive keyword list with those keywords (`POST /keyword-lists`), then put its id in `nkl` together with the location keys (for example `"#" + urllib.parse.urlencode({"nkl": "42", "si": "OH"})`); a negative list in `nkl` works like `negativeKeywords`. Presets cannot be edited, and deleting one also deletes its email schedules.

**Keyword lists** (`/keyword-lists`) store keyword sets. Create with `{"name": "...", "type": "positive" or "negative", "keywords": ["*dental*"]}`. Lists are not applied to API searches automatically: read a list's `keywords` and pass them as `positiveKeywords` or `negativeKeywords`.

**Email schedules** (`/email-schedules`) email new matching leads on an interval and spend credits for new leads. Required: `name`, `filterPresetId` (number), `intervalValue` (whole number 1-720), `intervalUnit` (`hours` or `days`) and `recipients` (`[{"email": "rep@company.com"}]`, up to 50). Always create with `"isActive": false`, review with the user, then activate with `PATCH /email-schedules/{id}` and `{"isActive": true}` (needs `--confirm`). An active schedule sends its first email within about 15 minutes, including every lead that currently matches (up to 10,000). Options include `maxLeadsPerEmail` (new leads over this cap, and new leads your credits could not cover, are kept in the account's shared lead reservoir while it has room and sent with a later email; on credit plans, already-received leads over the cap are not kept), `distributionMode` (`full_copy` or `split_evenly`; `fullCopyRecipients` only receive mail when at least 2 recipients are active) and combined-file settings (`REFERENCE.md`). There is no time-of-day setting: a daily schedule sends about 24 hours after the previous send, so the time drifts. `POST /email-schedules/{id}/trigger` sends right away. A `200` does not prove an email went out; compare `lastSent` and `totalSentCount` in `GET /email-schedules` before and after.

## AI helpers

- `POST /ai/suggest-categories` with `companyName`, `companyDescription` and `productService` (optional `companyWebsite`, `excludeCategories`) returns 4-12 suggested customer categories for the user's business. Nothing is saved. Turn the suggestions into keywords yourself (for example "Dentists" becomes `["*dental*","*dentist*"]`).
- `POST /ai/generate-keywords` **deletes every existing keyword list** and regenerates lists from the account's target categories, which can only be set in the dashboard. If `GET /me` shows an empty `targetCategories`, it deletes the lists and generates nothing. Filter presets that point at keyword lists by id (`nkl`, `ukl`, `ckl` or `dkl` in the hash, which dashboard-saved presets also use) silently lose those keywords and exclusions when the lists are deleted, and the same applies to `DELETE /keyword-lists/{id}`: their email schedules can stop sending, or start emailing a broader set of leads and spending credits on them. Check `GET /filter-presets` first and pause any affected schedules. Only use it when the user explicitly wants all lists replaced (needs `--confirm`). Check progress with `GET /ai/keyword-status`.
- `POST /ai/auto-refine/enable` / `disable` with `{"listId": 42}` turn on or off the AI refinement of a keyword list and its paired list; `GET /ai/auto-refine/status?listId=42` shows progress. Enabling resets the list's score history, and the refinement rewrites the list's keywords, which changes what any email schedule using the list sends (pause those schedules first if the user wants to review). A `200` does not guarantee the run started: if `autoRefineEnabled` is true but `refinementStatus` stays `null`, call enable again a few minutes later.

## Account, credits and billing

- `GET /me`: profile, plan, `subscriptionStatus`, the credit balance fields, the `trial` object, and the `autoTopUp` and `overageBudget` settings. `PATCH /me` updates `firstName`, `lastName`, `companyName` and `companyWebsite`.
- `POST /purchase-credits` (needs `--confirm`): buy permanent credits with the card on file, charged immediately. Send `creditCount` (at least 100, at most 5x the plan's monthly credits) or `dollarAmount` (at least 1). Price per credit is the plan's rate in the table above. The script sends an idempotency key and prints it; if the call times out, re-run it within 24 hours with `--idempotency-key` set to that value, so it is not charged twice (after 24 hours, check `permanentCredits` in `GET /me` before buying again). Works during a trial, and is charged immediately. If the user moved to a lower plan in the current paid billing period (not a plan switch during a free trial), that downgrade is still pending and credits bought now expire at the next renewal together with the rest of the balance (auto top-up pauses for this reason, a manual purchase does not). `GET /me` does not show a pending downgrade, so ask the user before buying, and say so when you confirm the price.
- `GET /auto-top-up`, `PATCH /auto-top-up`: automatic credit purchases when the permanent balance falls below a threshold. Enabling requires `triggerType`, `triggerAmount`, `purchaseType` and `purchaseAmount` (and optional `capType`, `capAmount`), replaces the whole configuration (send every field you want to keep) and needs `--confirm`. `{"enabled": false}` turns it off and always works. Not available during a trial.
- `GET /overage-budget`, `PATCH /overage-budget`: lets usage continue after the balance reaches zero, billed to the card once a day at the plan's rate, up to a per-cycle budget (`{"enabled": true, "budgetType": "dollars", "budgetAmount": 50}`; `budgetType` defaults to dollars). Needs `--confirm` to enable. Not available during a trial.
- `POST /subscription/change-plan` (needs `--confirm`) with `{"targetPlan": "starter" | "growth" | "scale"}`:
  - **Upgrade:** charges the new plan's full price now, restarts the billing cycle, keeps every unused credit and grants the new allowance immediately.
  - **Downgrade:** no charge now. At the next renewal the entire remaining balance (unused, rolled-over and purchased credits) expires before the lower allowance is granted. Warn the user. Moving back to the original plan (or higher) before renewal cancels the pending downgrade and keeps the credits, but it is billed like an upgrade: that plan's full price is charged now and the billing cycle restarts. Confirm the amount with the user first. While a downgrade is pending, a move to a plan between the current plan and the original one (for example Scale, then Starter, then Growth) is not an upgrade: nothing is charged and the whole balance still expires at renewal. `GET /me` does not show whether a downgrade is pending, so ask the user whether they moved to a lower plan in this billing period before you describe the cost.
  - **During a free trial**, `mode` decides: `switch_trial` (the default) changes the trial plan with no charge and keeps the trial end date (credits already used stay used); `activate_now` ends the trial, charges the target plan's full price now and grants its full allowance. A declined card returns `402 payment_declined` with `trialStillActive: true`: nothing changed and the trial continues.
  - Platinum and Enterprise changes go through support. A `409` means nothing changed; show its `message` (see `REFERENCE.md` for the codes).
- `POST /subscription/cancel` (needs `--confirm`): cancels at the end of the current billing period, and access continues until then. During a trial the card is never charged, but the free trial is used up. It can only be undone in the dashboard before the period ends. Remaining credits are forfeited when the subscription ends. A confirmation email is sent.

## Signing up through the API

New customers can subscribe without the website:

```bash
python3 smb_api.py POST /purchase --body '{"email":"owner@company.com","plan":"growth"}' --confirm
```

- Plans: `starter`, `growth`, `scale` (14-day free trial by default), `platinum`, `enterprise` (no trial, charged at checkout). `"trial": false` skips the trial and charges at checkout.
- The response has a Stripe `checkoutUrl` (valid 24 hours) and a `trial` block (`eligible`, `days`, `credits`, `endsAt`, `note`, and `reason` when not eligible). Before the user opens the link, tell them the plan price and exactly when it is charged: at checkout, or automatically when the trial ends.
- After checkout **the API key is emailed** to that address. If the address already has an account, the purchase must first be confirmed from an email sent to it, and the key follows. Do not use `POST /claim-key`; it belongs to an older flow and cannot succeed.
- An address with an active or trialing subscription gets `409 already_subscribed` (a trialing account should use `change-plan` instead). Each call creates a new checkout; never open two. Limited to 5 per hour.

## Rate limits, errors and retries

| Limit | Applies to |
|---|---|
| 600 requests per minute per API key | Every authenticated endpoint |
| 5 per minute per API key, shared | `POST /leads/export`, the three `POST /lead-export-history/...` calls and `POST /email-schedules/{id}/trigger` |
| 5 per minute per API key, shared | `POST /ai/suggest-categories`, `POST /ai/generate-keywords`, `POST /ai/auto-refine/enable` |
| 10 lead searches at a time per account | `GET /leads`, `GET /leads/preview`, `POST /leads/export` (more returns `429 lead_query_capacity_busy`) |
| 5 per hour / 30 per hour per IP | `POST /purchase` / `POST /claim-key` |

Failed requests count toward the limits. Responses carry `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (an ISO timestamp) and, on a 429 or 503, `Retry-After` (seconds).

| Status | Meaning | What to do |
|---|---|---|
| 400 | Bad request (`bad_request`, `validation_error` with `details`, `invalid_request`); `export_too_large` comes after the credits were charged | Fix the request; for `export_too_large` follow **XLSX limit** under **Exporting files** and do not repeat the export |
| 401 | Missing, invalid or revoked API key | Ask the user to check `SMB_SALES_BOOST_API_KEY` |
| 402 | `insufficient_credits` (rare: credits ran out between check and charge), or a card problem (`payment_failed`, `payment_declined`, `authentication_required`) | Read `error`; for a card problem the user updates their card in the dashboard |
| 403 | No active subscription, or a feature not available on this account or during a trial | Explain; do not retry |
| 404 | Unknown endpoint or id | Check the id |
| 409 | Conflict (`plan_change_in_progress`, `already_subscribed`, `not_ready` and others); nothing changed | Show the `message`; retry only when it says so |
| 410 | `gone`: export file expired or not stored; `plan_retired` from `POST /purchase` | For `gone`, re-export free via `/lead-export-history/re-export`; for `plan_retired`, pick a current plan |
| 429 | Rate limit or `lead_query_capacity_busy` | Wait `Retry-After` (the script retries by itself, up to 60 s) |
| 500 | Server error | On charging calls do NOT repeat blindly (see Rule 5) |
| 503 | `service_unavailable`, temporary | Retry after `Retry-After` (the script does this) |
| 524 | Network edge timeout after about 100 seconds; the server may still finish | Treat like a 500 on charging calls |

Errors look like `{"error": "code", "message": "Human-readable text"}`; the script adds `httpStatus`, and `advice` where a failure may have had an effect.

## Natural language examples

| User says | Call |
|---|---|
| "Find new dental practices in Texas" | Preview, then `GET /leads` with `positiveKeywords: ["*dental*","*dentist*","*orthodont*"]`, `stateInclude: ["TX"]`, `limit`, `maxCredits` |
| "Med spas in Florida with a buying signal this week" | `positiveKeywords: ["*med*spa*","*medical*spa*","*aesthet*","*botox*"]`, `stateInclude: ["FL"]`, `lastBuyingSignalFrom: "rel:7d"` |
| "Auto repair shops in Chicago that just got a new phone number" | `positiveKeywords: ["*auto*repair*","*mechanic*","*body*shop*"]`, `cityInclude: ["Chicago"]`, `buyingSignalTypeFilter: ["Phone Primary"]` |
| "Pet groomers in California, no boarding or kennels" | `positiveKeywords: ["*pet*groom*","*dog*groom*"]`, `negativeKeywords: ["*boarding*","*kennel*"]`, `stateInclude: ["CA"]` |
| "Only leads with an email address" | add `emailPrimaryEmptyFilter: "exclude_empty"` |
| "Well-reviewed restaurants in Atlanta" | `positiveKeywords: ["*restaurant*","*grill*","*bistro*"]`, `cityInclude: ["Atlanta"]`, `minRatingValue: 4.5`, `minReviewCount: 50` |
| "Businesses that registered their domain in the last 3 months" | a keyword filter plus `registrationDateFrom` set to the date 3 months ago as `YYYY-MM-DD` |
| "How many bakeries are in New York?" | `GET /leads/preview`, read `data.pagination.total` |
| "Show me only leads I already bought" | `GET /lead-export-history` (every lead you received, paged with `limit` and `offset`, free). To limit it to one search, run `GET /leads` with that search's filters and `maxCredits: 0` on every page, because each page only keeps the already-received leads among that page's matches |
| "Export the results but spend at most 50 credits" | `POST /leads/export` with `maxCredits: 50` |
| "Export only leads I don't have yet" | `POST /leads/export` with `excludePurchased: true` and a `maxCredits` agreed with the user (every lead in this export costs 1 credit) |
| "Download my export from last week" | `GET /export-history`, then `GET /export-history/{id}/download` |
| "Get me the contact details for these 200 websites" | `POST /enrichments` (state the prices, agree on `maxCredits`, `--confirm`) |
| "Email me new HVAC leads in Ohio every morning" | Create a positive keyword list with the HVAC keywords, then a preset whose hash has `nkl` set to that list's id and `si=OH`, create the schedule with `intervalValue: 1`, `intervalUnit: "days"` and `isActive: false`, review, then activate. There is no time-of-day setting: the first email goes out within about 15 minutes of activation and later ones about a day after the previous send, so the time drifts. Tell the user this instead of promising a fixed time |
| "How many credits do I have?" | `GET /me`, `totalCreditsRemaining` |
| "Buy 1,000 more credits" | Confirm the price, then `POST /purchase-credits` with `creditCount: 1000` and `--confirm` |
| "Upgrade to Scale" | Confirm price and timing, then `POST /subscription/change-plan` with `targetPlan: "scale"` and `--confirm` |
| "I want to sign up for Growth" | Explain the trial and charge date, then `POST /purchase` with `--confirm` |

## smb_api.py reference

| Option | Purpose |
|---|---|
| `--params JSON` / `--params-file PATH` | Query parameters (any method). Lists as JSON arrays. `-` reads stdin. |
| `--body JSON` / `--body-file PATH` | JSON body for POST, PATCH and PUT. |
| `--confirm` | Required for calls that charge money, authorize future charges, send email, spend credits on enrichment, cancel or delete. Add only after the user approves. |
| `--dry-run` | Show the request (URL, body, whether `--confirm` is needed) without sending it. |
| `--compact` | Print only key fields of each lead. |
| `--out NAME` | Also save the full JSON response as `NAME.json` in the output folder. |
| `--output-dir DIR` | Where files are saved. Default: `$SMB_SALES_BOOST_OUTPUT_DIR`, else `/mnt/user-data/outputs` if it exists, else `./smb-sales-boost-files`. |
| `--idempotency-key KEY` | Reuse the key from a timed-out `POST /purchase-credits` (within 24 hours). |
| `--timeout SEC` | Network timeout (default 180). |
| `--no-retry` / `--no-fetch` | Do not retry 429/503 / print a download link instead of downloading. |

File handling: exports, `/export-history/{id}/download`, `/export-history/{id}/download-url`, the lead-export-history files and `/enrichments/{id}/download` are saved automatically as new files (never overwriting; mode 600), and the script prints the paths. Short-lived download links are used directly and not printed, unless you pass `--no-fetch` (the printed link then works without an API key for a few minutes, so do not share or log it). Exit codes: 0 success, 1 API error (or an unexpected client error), 2 usage error or confirmation required (nothing was sent), 3 network error or timeout, 4 local file error (the script creates the output folder before sending, so a folder problem stops it before anything is charged).

## Security

- **No shell injection:** the script sends structured JSON over HTTPS with Python's standard library; user text never becomes part of a shell command when you use `--params-file` / `--body-file` for text with quotes.
- **Fixed destination:** requests go only to `https://smbsalesboost.com/api/v1`; endpoints are validated (no `..`, no other hosts), redirects are never followed, and the key is never sent to `/purchase` or `/claim-key`. Download links are fetched without the key, over HTTPS, and only from Amazon S3 (`amazonaws.com`) hosts.
- **Safe files:** file names from the server are reduced to a plain base name, limited to `.csv`, `.json` and `.xlsx`, and written only inside the output folder as new files readable only by the user.
- **Key handling:** read from `SMB_SALES_BOOST_API_KEY` and sent only in the `Authorization` header. It is never printed, logged or written to a file. Passing it as the first argument still works but exposes it to other local processes; prefer the environment variable.
- **Confirmation gate:** money, email, deletion and enrichment calls are refused without `--confirm`.

## Not available through the API

These are managed in the dashboard (https://smbsalesboost.com/dashboard): creating or revoking API keys; CRM and webhook integrations (HubSpot, Salesforce, Pipedrive, Zapier, n8n, Make, Pipedream, Clay, custom webhooks), although leads you search or export through the API are still pushed to any integration the user has connected; updating the card or viewing invoices; undoing a cancellation; target categories for keyword generation; the default "email enrichment results" setting. A remote MCP server is also available at `https://smbsalesboost.com/mcp` (details: https://smbsalesboost.com/.well-known/mcp/server-card.json).
