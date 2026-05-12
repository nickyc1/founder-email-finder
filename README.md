# founder-email-finder

A [Claude Code](https://claude.com/claude-code) skill that enriches a Google Sheet of prospects with verified founder emails.

The opinion: don't guess. Use the best paid people-data source first, fall back to website crawls, only use pattern inference as a last resort. Verify every candidate before writing. Track confidence per row.

## Why this exists

Generic email-finding tools either:

1. Spray thousands of patterns, verify them all, and dump everything that doesn't bounce into your sheet (bad data, fast). Or
2. Cost $399/month (Clay) for what's mostly a verification waterfall

This is a clean, opinionated waterfall you can run in your own infra with API keys you control. It hits 60-80% verified-founder coverage on most prospect lists with the full provider stack, 30-50% on free-tier providers alone.

## What it does

For each row in your Google Sheet:

1. **Discover** candidate emails via a best-first waterfall:
   - RocketReach (or equivalent people-data API)
   - Hunter / Dropcontact fallback
   - Website crawl (`/`, `/contact`, `/about`, `/team`)
   - Pattern generation (last resort, only with high identity confidence)

2. **Verify** every candidate through [Reoon](https://reoon.com/) (`safe` / `role_account` / `catch_all` / `unknown` / `invalid` / `disposable`)

3. **Score** confidence with weighted evidence (domain match, name match, verifier status, provider corroboration)

4. **Write** only the highest-confidence candidate that passes verification policy — into the `Founder Email` column, nothing else

Default mode is dry-run. Write mode requires explicit approval.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- A Google Sheets MCP configured (or any sheet read/write tool the agent can use)
- A Reoon API key (verifier — required)
- Optional: RocketReach, Hunter, Dropcontact API keys (any combination)

## Install

```bash
git clone https://github.com/nickyc1/founder-email-finder.git ~/.claude/skills/founder-email-finder
```

Set your provider API keys as environment variables or in your secrets manager. Make sure the agent has read/write access to the target Google Sheet.

Restart Claude Code. The skill is available.

## Usage

In Claude Code:

```
Use founder-email-finder to enrich SPREADSHEET_ID=1abcd...
WORKSHEET_NAME=Prospects.
Dry-run only — show me what would change.
```

```
Same sheet, write mode now. Use the same dry-run logic, write only confidence ≥ 5.
```

The skill walks you through the dry-run summary before any writes. You approve, it writes the row, it checkpoints, it moves on.

## Confidence policy

Each candidate gets scored:

| Evidence | Δ |
|---|---|
| Verifier says `safe` | +4 |
| Verifier says `role_account` | +2 |
| Exact domain match to website | +3 |
| Local-part matches founder name | +3 |
| Found in provider result with same company | +2 |
| Found on an official site page | +1 |
| Generic mailbox (`info@`, `support@`, `contact@`) when founder expected | -3 |
| Mismatch between founder / company / domain | -4 |

Only the highest-confidence candidate passing the verifier policy gets written. Configurable threshold (default: 5+).

## Stakeholder safety

- Never overwrites a non-empty `Founder Email` unless the user explicitly says so
- Never writes to columns other than `Founder Email`
- Never sends outbound email — this is enrichment only
- Per-row audit trail so retries resume cleanly

## Rate limiting

- Batch size: 20 rows
- Reoon concurrency: 1-2 requests max
- 429 backoff: 15s → 30s → 60s → 120s
- If 429 persists 5+ min, pause verifier and queue discovery-only

## Repo structure

```
founder-email-finder/
├── SKILL.md                              # the skill prompt Claude Code reads
└── README.md
```

This skill is pure prompt — no Python scripts. The agent uses your Google Sheets MCP and direct HTTP calls to Reoon/RocketReach/Hunter/Dropcontact.

## License

MIT — see [LICENSE](LICENSE).

Built by [Nick Christensen](https://github.com/nickyc1).
