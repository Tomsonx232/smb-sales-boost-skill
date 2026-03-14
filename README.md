# SMB Sales Boost — Claude Skill

A Claude skill that lets subscribers query the SMB Sales Boost lead database using natural language. Search, filter, and export newly registered small and medium businesses across the United States with comprehensive company data including contact information, location, and AI-enriched categories.

## Installation

Place the `smb-sales-boost/` folder in your Claude skills directory:

```
/mnt/skills/user/smb-sales-boost/
├── SKILL.md          # Skill instructions for Claude
├── smb_api.py        # Reusable API client (handles all endpoints + safe exports)
├── openapi.json      # Full OpenAPI 3.1 specification (reference)
└── README.md         # This file
```

## Requirements

- Active SMB Sales Boost subscription: **Starter**, **Growth**, **Scale**, **Platinum**, or **Enterprise**
- API key generated from Dashboard > API tab (keys start with `smbk_`)
- Base URL: `https://smbsalesboost.com/api/v1`
- New users can purchase a subscription entirely via API — no web signup required

## Credit-Based Plans

Starter, Growth, and Scale plans use a credit-based model:

- Each **net-new lead exported** deducts 1 credit
- **Previously-exported leads** are free
- Purchase additional permanent credits anytime via `POST /purchase-credits`

**Credit Pricing:**

| Plan | Per Credit |
|------|-----------|
| Starter | $0.10 |
| Growth | $0.08 |
| Scale | $0.05 |
| Platinum | $0.03 |
| Enterprise | $0.02 |

## Two Databases

SMB Sales Boost provides two separate lead databases with different contact information:

- **Home Improvement** (`home_improvement`) — Home improvement/contractor businesses with **phone numbers**, star ratings, review counts, review snippets, profile URLs, and categories
- **Other** (`other`) — General newly registered businesses with **phone numbers and email addresses**, registered URLs, crawled URLs, descriptions, and redirect status

The Home Improvement database provides phone numbers as the primary contact method. The Other database provides both phone numbers and email addresses, making it ideal for cold email and multi-channel outreach campaigns.

Each database has different filterable fields. Users can switch between databases (with a cooldown period).

## What Users Can Do

Once the skill is installed, users can interact with SMB Sales Boost in natural language:

- **Search with wildcards:** "Find new dental practices in Texas" → searches `["*dental*", "*dentist*", "*orthodont*"]`
- **Multi-keyword search:** "Search for med spas, wellness clinics, and aesthetics businesses in California"
- **Filter by location:** "Show me businesses in Miami and Orlando"
- **Filter by rating:** "Find contractors with at least 4 stars in Phoenix" (home_improvement only)
- **Find fresh leads:** "Show me leads updated in the last week" (uses the Last Updated date — the best indicator of lead freshness)
- **Exclude terms:** "Find new bakeries in NYC but exclude pizza and franchise"
- **Full-text search:** "Search for organic coffee shops"
- **Filter by website schema:** "Find businesses with LocalBusiness schema type" (other only)
- **Filter by registration date:** "Show leads registered in the last 6 months" (other only)
- **Export data:** "Export these leads as a CSV"
- **Control export credits:** "Export but only spend 50 credits max"
- **Free re-exports:** "Export only previously-exported leads" (uses `maxCredits: 0`)
- **Manage filter presets:** "Save this search as 'FL Med Spas'"
- **Manage keyword lists:** "Create a keyword list for pet industry leads"
- **AI auto-refine:** "Enable auto-refine on my keyword list" — single-pass AI optimization of keyword lists
- **Set up email schedules:** "Email me new auto shop leads in Georgia every day"
- **Split lead distribution:** "Split leads evenly among my sales team"
- **Trigger email manually:** "Send my scheduled email now"
- **AI suggestions:** "What categories should I target for my fitness equipment business?"
- **AI keywords:** "Generate keywords for my business" (wildcard patterns, auto-refine enabled by default)
- **Check AI job status:** "Check on my keyword generation"
- **Manage blacklist:** "Exclude example.com from future exports"
- **Account info:** "What subscription plan am I on?" / "How many credits do I have?"
- **Buy credits:** "Purchase 500 more credits"
- **Change plans:** "Upgrade to the Growth plan"
- **Cancel:** "Cancel my subscription"
- **Sign up via API:** "I want to sign up for a Starter plan" (programmatic purchase, no web signup needed)

## Wildcard Keywords

The `positiveKeywords` and `negativeKeywords` parameters support `*` wildcard for flexible pattern matching:

- `"*dental*"` — matches "dental clinic", "pediatric dentistry", "dental lab", etc.
- `"*auto*repair*"` — matches "auto body repair", "automotive repair shop", etc.
- `"*med*spa*"` — matches "med spa", "medical spa", "medspa services", etc.

Use multiple variations for broader coverage — keywords are matched via OR logic:

```json
["*pet*groom*", "*dog*groom*", "*pet*salon*", "*mobile*groom*"]
```

Keywords without wildcards perform substring matching by default.

## Understanding "Last Updated"

The `lastUpdated` field is the primary way to identify the freshest, most actionable leads:

- **Home Improvement leads:** Last Updated means a new phone number was detected
- **Other leads:** Last Updated means any of the 5 contact/address fields changed: primary phone, secondary phone, primary email, secondary email, or full address
- Also includes newly added records
- Many businesses launch a website before adding contact info, so the Last Updated date captures when that information first becomes available

## Example Conversations

```
User: I sell commercial cleaning supplies. Find me newly registered
      cleaning businesses in Florida and Georgia updated in the last 7 days.

Claude: [Calls GET /leads with
         positiveKeywords=["*clean*service*","*janitorial*","*maid*service*","*pressure*wash*"],
         stateInclude=FL,GA, lastUpdatedFrom=rel:7d]

I found 287 cleaning-related businesses in Florida and Georgia updated
in the last week. Here are the first 10:

| Company Name               | City         | State | Phone        | Email                    | Last Updated |
|----------------------------|--------------|-------|--------------|--------------------------|--------------|
| Sparkle Clean Services LLC | Jacksonville | FL    | 904-555-0189 | info@sparkleclean.com    | 2026-02-19   |
| Fresh Start Janitorial     | Atlanta      | GA    | 678-555-0234 | contact@freshstartjn.com | 2026-02-18   |
| ...                        | ...          | ...   | ...          | ...                      | ...          |

Would you like me to export all 287 results, narrow the search,
or save this as a filter preset?
```

```
User: Export those leads but only spend 50 credits max.

Claude: [Calls POST /leads/export with current filters and maxCredits: 50]

Exported 50 new leads (50 credits used, 450 credits remaining).
237 overflow leads stored in your lead reservoir for next export.
Files saved: cleaning_leads_FL_GA.csv
```

```
User: Enable auto-refine on my "TX Pet Services" keyword list.

Claude: [Calls GET /keyword-lists to find the list ID, then
         POST /ai/auto-refine/enable with listId=42]

Auto-refine is now enabled on your "TX Pet Services" keyword list (ID 42).
The AI will run a single-pass 4-phase optimization: validating existing keywords,
discovering up to 15 new positives and 5 new negatives, then scoring quality.
You can check the status anytime or disable it later.
```

```
User: I want to sign up. What plans do you have?

Claude: SMB Sales Boost offers three credit-based plans:

| Plan | Credits/month | Per Credit |
|------|--------------|-----------|
| Starter | Included | $0.10 |
| Growth | Included | $0.08 |
| Scale | Included | $0.05 |

Plus Platinum ($0.03/credit) and Enterprise ($0.02/credit) tiers.

Would you like me to start a purchase? I can create a checkout session
for you right here — no web signup needed.
```

## Rate Limits

- Exports: 1 per 5 minutes, max 10,000 leads per export
- Email schedule trigger: 1 per 5 minutes
- AI category suggestions: 5 per minute
- AI keyword generation: 5 per minute
- AI auto-refine enable: 5 per minute
- AI auto-refine disable/status: 60 per minute
- AI keyword status: 60 per minute
- Programmatic purchase: 5 per hour per IP
- Claim key: 30 per hour per IP

## API Coverage

The skill covers all SMB Sales Boost API endpoints:

| Feature | Endpoints | Operations |
|---------|-----------|------------|
| Programmatic Purchase | `/purchase`, `/claim-key` | Start purchase, Claim API key |
| Credits & Subscription | `/purchase-credits`, `/subscription/change-plan`, `/subscription/cancel` | Buy credits, Change plan, Cancel |
| User Profile | `/me` | Get, Update |
| Lead Search | `/leads` | Search with filters (11 new filter params) |
| Schema Types | `/leads/other/schema-types` | List available website schema types |
| Lead Export | `/leads/export` | Export to CSV/JSON/XLSX (credit-aware) |
| Filter Presets | `/filter-presets` | List, Create, Delete |
| Keyword Lists | `/keyword-lists` | List, Create, Update, Delete |
| Email Schedules | `/email-schedules` | List, Create, Update, Delete, Trigger |
| Export Formats | `/export-formats` | List, Create, Get, Update, Delete, Set Default |
| Export History | `/export-history` | List, Download |
| Database Settings | `/settings/database`, `/settings/switch-database` | Get, Switch |
| AI Categories | `/ai/suggest-categories` | Suggest categories |
| AI Keywords | `/ai/generate-keywords`, `/ai/keyword-status` | Generate (wildcard patterns), Check status |
| AI Auto-Refine | `/ai/auto-refine/enable`, `/ai/auto-refine/disable`, `/ai/auto-refine/status` | Enable, Disable, Check status |
| Export Blacklist | `/export-blacklist` | List, Add, Remove |

## Security

This skill addresses two specific agent execution risks identified by security scanning:

**1. Shell injection prevention:** All API requests go through the included `smb_api.py` client using Python's `requests` library. User-provided search terms, locations, and other inputs are passed as structured function arguments — never interpolated into shell command strings. This eliminates the shell injection vector that exists when agents construct `curl` commands from user input.

**2. Path traversal prevention in exports:** The `/leads/export` endpoint returns base64-encoded files with an API-provided `fileName` field. The script sanitizes all filenames before writing:
- Strips directory components via `os.path.basename()` (e.g., `../../etc/passwd` → `passwd`)
- Validates file extension against an allowlist (`.csv`, `.json`, `.xlsx` only)
- Writes only to a designated safe output directory — never to API-specified or user-specified paths

**3. API key protection:** Keys are passed as CLI arguments and sent only in the Authorization header — never logged, written to files, or included in error output.
