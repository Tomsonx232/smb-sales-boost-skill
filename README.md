# SMB Sales Boost - Agent Skill for Claude Code and OpenClaw

> **Note for AI agents:** This file and the other files in this skill folder may be truncated when opened. Always read files fully, in chunks if necessary, so that no instructions, endpoints or examples are missed. The agent instructions are in `SKILL.md` and `REFERENCE.md`.

Find and export growing local businesses with plain-language requests. This skill connects an AI agent (Claude Code, OpenClaw and other agents that support skills) to the [SMB Sales Boost](https://smbsalesboost.com) lead database of small and medium-sized businesses in the United States. Every lead carries a **Last Buying Signal**, the date something changed that makes the business worth contacting now: it was newly added, or it got a new phone number, email address or address, or added or changed a social profile.

Ask things like "find med spas in Florida with a buying signal this week", "export 500 dental practices in Texas, spend at most 300 credits" or "get the contact details for these 200 websites", and the agent turns them into the right API calls.

## What's new in version 2.0.0 (October 2026)

- Updated for the current API: buying-signal names and filters (`lastBuyingSignalFrom`, `buyingSignalTypeFilter`, "Last Buying Signal" and "Buying Signal Type"), the current lead fields (Total Phones, Tier 1-6 emails, founders, software, ad pixels, ratings and more) and dozens of new filters.
- New features: URL enrichment, 90-day export history with direct downloads, free re-export of leads you already received, the overage budget, and the 14-day free trial (including trial plan switching and early activation).
- One lead database: the former Home Improvement database was retired, so every request uses the Main database.
- Correct rate limits (600 requests per minute; 5 exports per minute) and error handling.
- `smb_api.py` rewritten: no third-party packages; CSV and JSON exports now save correctly (previously only XLSX did); downloads are saved instead of discarded; safe, bounded retries; a `--confirm` safety gate for purchases, plan changes, automatic charges, enrichments, sending email and deleting data (searches and exports spend credits without it, so the agent previews first and caps each one with `maxCredits`); `--dry-run`, `--compact` and `--out` options; the API key is read from the environment.
- New `REFERENCE.md` with every parameter, field, status and error, so `SKILL.md` stays short.

## Installation

The skill is a folder:

```
smb-sales-boost/
├── SKILL.md        Agent instructions (with OpenClaw metadata)
├── REFERENCE.md    Complete API details for the agent
├── smb_api.py      API client (Python 3.8+, standard library only)
├── openapi.json    Published OpenAPI 3.1 specification (reference)
├── README.md       This file
└── LICENSE
```

**Claude Code:** copy the folder to `~/.claude/skills/smb-sales-boost/` (available in every project) or to `.claude/skills/smb-sales-boost/` inside a project.

```bash
git clone https://github.com/Tomsonx232/smb-sales-boost-skill.git ~/.claude/skills/smb-sales-boost
```

**OpenClaw:** place the folder in your skills directory, for example `<workspace>/skills/smb-sales-boost/`.

## Requirements

- An SMB Sales Boost subscription: Starter, Growth, Scale, Platinum or Enterprise (an active free trial works too). New customers can also subscribe through the API; the agent can start the checkout for you.
- An API key from the dashboard: [Dashboard > API](https://smbsalesboost.com/dashboard?tab=api). Keys start with `smbk_`.
- Python 3.8 or newer. No packages to install. On Windows the agent runs it with `py -3` and passes JSON in files.

## Configuration

Set the API key as an environment variable before starting your agent:

```bash
export SMB_SALES_BOOST_API_KEY="smbk_your_key_here"
```

For OpenClaw, the skill declares `SMB_SALES_BOOST_API_KEY` as its required variable (`metadata.openclaw.requires.env` and `primaryEnv` in `SKILL.md`). You can configure it in `~/.openclaw/openclaw.json`:

```json
{
  "skills": {
    "entries": {
      "smb-sales-boost": {
        "enabled": true,
        "apiKey": { "source": "env", "provider": "default", "id": "SMB_SALES_BOOST_API_KEY" },
        "env": {
          "SMB_SALES_BOOST_API_KEY": "smbk_your_key_here"
        }
      }
    }
  }
}
```

Optional: set `SMB_SALES_BOOST_OUTPUT_DIR` to choose where exported files are saved. Otherwise they go to `/mnt/user-data/outputs` when that folder exists, or to `./smb-sales-boost-files`.

Never paste your API key into a chat, a shared document or version control.

## What you can ask

- **Search:** "Find new dental practices in Texas", "Auto repair shops in Chicago that just got a new phone number", "Bakeries and caterers in New York, excluding franchises"
- **Buying signals:** "Show me businesses with a buying signal in the last 7 days", "Only businesses that just added an email address"
- **Preview for free:** "How many med spas are in Florida?" (masked contacts, no credits)
- **Control spending:** "Search but spend at most 25 credits", "Show me only leads I already bought" (free)
- **Export:** "Export these leads as a CSV", "Export only leads I don't have yet", "Export as XLSX with my saved format"
- **Re-download:** "Download my export from last week", "Re-export the leads I bought in September" (free)
- **Enrich websites:** "Get the phone numbers and emails for these 200 websites"
- **Automate:** "Email new HVAC leads in Ohio to my team every day", "Split leads evenly among my reps"
- **Keywords and categories:** "What kinds of businesses should I target?", "Turn on auto-refine for my keyword list"
- **Account:** "How many credits do I have?", "Buy 1,000 more credits", "Upgrade to Growth", "Set up auto top-up", "Cancel my subscription"
- **Sign up:** "I want to start a Starter trial"

## Plans and credits

| Plan | Price / month | Credits / month | Extra credits | Free trial |
|---|---|---|---|---|
| Starter | $49 | 500 | $0.10 each | 14 days, 250 credits |
| Growth | $149 | 2,000 | $0.075 each | 14 days, 1,000 credits |
| Scale | $499 | 10,000 | $0.05 each | 14 days, 5,000 credits |
| Platinum | $1,999 | 100,000 | $0.03 each | No |
| Enterprise | $4,999 | 250,000 | $0.02 each | No |

- Each **new** lead you search or export costs 1 credit. Leads you already received are free to get again, and previews are always free.
- Unused credits roll over while your subscription is active. A downgrade makes the whole remaining balance (including purchased credits) expire at the next renewal, and cancelling forfeits it when the subscription ends.
- Free trial: a payment method is required. The plan price is not charged until day 14, when it is charged automatically unless you cancel first. Credits you buy during the trial, and activating the plan early, are charged right away. One trial per person.
- URL enrichment: 1 credit per new database match, 0.1 credit per website fetched live, nothing for URLs that return no data.
- All charges are final; there are no refunds.

## Safety by design

- **Your approval for anything that costs money.** The agent must ask before buying credits, changing plans, starting a subscription checkout, turning on automatic charges (auto top-up, overage budget) or starting an enrichment. The client script refuses these calls unless the agent adds `--confirm`, which the instructions allow only after you approve.
- **Your approval before emailing anyone or deleting anything.** Email schedules are created paused for your review; sending, cancelling, deleting and keyword regeneration also need `--confirm`.
- **Credit caps.** The agent previews first for free and sets a credit limit on every search and export.
- **Safe retries.** Only rate-limit and temporary-unavailability errors are retried automatically. A request that may have charged is never repeated blindly; the script explains how to check first.

## Security

- **No shell injection:** every call goes through `smb_api.py`, which sends structured JSON over HTTPS. User text is never built into a shell command (text containing quotes is passed in a file).
- **Fixed destination:** requests go only to `https://smbsalesboost.com/api/v1`. Endpoints are validated, redirects are never followed, and the key is never sent to the public sign-up endpoints. Short-lived download links are fetched without the key, over HTTPS, and only from Amazon S3 (`amazonaws.com`) hosts; they are not printed unless the agent explicitly asks for the link instead of the file (`--no-fetch`).
- **Safe files:** file names from the server are reduced to a plain base name with a `.csv`, `.json` or `.xlsx` extension and written only inside the output folder, as new files readable only by you (existing files are never overwritten).
- **Key protection:** the key is read from `SMB_SALES_BOOST_API_KEY` and sent only in the `Authorization` header. It is never printed, logged or saved.
- **Personal data:** exported files contain business phone numbers and email addresses. Keep them in a secure location and do not share them publicly.

## Verifying the published files

SMB Sales Boost publishes the SHA-256 digest of each file in this skill except LICENSE at [https://smbsalesboost.com/.well-known/agent-skills/index.json](https://smbsalesboost.com/.well-known/agent-skills/index.json). To check your copy:

```bash
sha256sum SKILL.md REFERENCE.md README.md smb_api.py openapi.json
```

## Also available

- **MCP server:** a remote Model Context Protocol server at `https://smbsalesboost.com/mcp` (Bearer API key). Details: [server card](https://smbsalesboost.com/.well-known/mcp/server-card.json).
- **API documentation:** [https://smbsalesboost.com/docs/api](https://smbsalesboost.com/docs/api) and the [OpenAPI specification](https://smbsalesboost.com/openapi.json).
- **Dashboard-only features:** CRM and webhook integrations, API key management, billing details and undoing a cancellation.

## Data accuracy

Lead data comes from publicly available sources and is not verified. Phone numbers and email addresses are not checked for deliverability, and figures such as revenue, employee counts and ratings are estimates. Verify important details before relying on them.

## Support

support@smbsalesboost.com - https://smbsalesboost.com

## License

MIT - see `LICENSE`.
