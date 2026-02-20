# SMB Sales Boost — Claude Skill

A Claude skill that lets subscribers query the SMB Sales Boost lead database using natural language. Search, filter, and export newly registered small and medium businesses across the United States with comprehensive company data including contact information, location, and AI-enriched categories.

## Installation

Place the `smb-sales-boost/` folder in your Claude skills directory:

```
/mnt/skills/user/smb-sales-boost/
├── SKILL.md          # Skill instructions for Claude
├── openapi.json      # Full OpenAPI 3.1 specification (reference)
└── README.md         # This file
```

## Requirements

- Active SMB Sales Boost subscription: **Pro**, **Platinum**, **Enterprise**, or **Agency**
- API key generated from Dashboard > API tab (keys start with `smbk_`)
- Base URL: `https://smbsalesboost.com/api/v1`

## Two Databases

SMB Sales Boost provides two separate lead databases:

- **Home Improvement** (`home_improvement`) — Contractor/home improvement businesses with star ratings, review counts, review snippets, phone numbers, profile URLs, and categories
- **Other** (`other`) — General newly registered businesses with registered URLs, URL registration dates, crawled URLs, descriptions, emails, phone numbers, and redirect status

Each has different filterable fields. Users can switch between databases (with a cooldown period).

## What Users Can Do

Once the skill is installed, users can interact with SMB Sales Boost in natural language:

- **Search leads:** "Find new plumbing businesses in Texas"
- **Filter by location:** "Show me businesses in Miami and Orlando"
- **Filter by rating:** "Find contractors in California with at least 4 stars" (home_improvement only)
- **Find fresh leads:** "Show me leads updated in the last week" (uses the Last Updated date — the best indicator of lead freshness)
- **Export data:** "Export these leads as a CSV"
- **Manage filter presets:** "Save this search as 'Texas Plumbers'"
- **Manage keyword lists:** "Create a keyword list for dental suppliers"
- **Set up email schedules:** "Email me new plumbing leads in Texas every day"
- **AI suggestions:** "What categories should I target for my dental supply company?"
- **AI keywords:** "Generate keywords for my business"
- **Manage blacklist:** "Exclude example.com from future exports"
- **Account info:** "What subscription plan am I on?"

## Understanding "Last Updated"

The `lastUpdated` field is the primary way to identify the freshest, most actionable leads:

- **Home Improvement leads:** Last Updated means a new phone number was detected
- **Other leads:** Last Updated means the primary phone number and/or primary email address was updated
- Also includes newly added records
- Many businesses launch a website before adding contact info, so the Last Updated date captures when that information first becomes available

## Example Conversation

```
User: I sell HVAC equipment. Find me newly registered HVAC businesses 
      in Florida and Georgia updated in the last 7 days.

Claude: [Calls GET /leads with positiveKeywords=["HVAC"], 
         stateInclude=FL,GA, lastUpdatedFrom=2026-02-13]

I found 342 HVAC-related businesses in Florida and Georgia updated 
in the last week. Here are the first 10:

| Company Name          | City        | State | Phone        | Last Updated |
|-----------------------|-------------|-------|--------------|--------------|
| Cool Breeze HVAC LLC  | Miami       | FL    | 305-555-0123 | 2026-02-18   |
| ...                   | ...         | ...   | ...          | ...          |

Would you like me to export all 342 results, narrow the search, 
or save this as a filter preset?
```

## Rate Limits

- Exports: 1 per 5 minutes, max 10,000 leads per export
- AI category suggestions: 5 requests per minute
- AI keyword generation: 5 requests per minute

## API Coverage

The skill covers all 30 SMB Sales Boost API endpoints:

| Feature | Endpoints | Operations |
|---------|-----------|------------|
| User Profile | `/me` | Get, Update |
| Lead Search | `/leads` | Search with filters |
| Lead Export | `/leads/export` | Export to CSV/JSON/XLSX |
| Filter Presets | `/filter-presets` | List, Create, Delete |
| Keyword Lists | `/keyword-lists` | List, Create, Update, Delete |
| Email Schedules | `/email-schedules` | List, Create, Update, Delete |
| Export Formats | `/export-formats` | List, Create, Get, Update, Delete, Set Default |
| Export History | `/export-history` | List, Download |
| Database Settings | `/settings/database`, `/settings/switch-database` | Get, Switch |
| AI Features | `/ai/suggest-categories`, `/ai/generate-keywords` | Suggest categories, Generate keywords |
| Export Blacklist | `/export-blacklist` | List, Add, Remove |
