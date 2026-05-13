---
name: founder-email-finder
description: Find and verify founder emails for prospects in a Google Sheet, and write only confidence-approved emails into the target column. Use when asked to enrich leads, find founder emails, verify emails, or run Clay-style lead-email workflows across spreadsheet rows.
---

# Founder Email Finder

A rate-limit-safe, confidence-scored email enrichment workflow.

## Safety defaults

- Start in **dry-run** unless the user explicitly approves writes
- Never write to columns other than `Founder Email` unless explicitly asked
- Never claim a verified email without verifier evidence
- Never send outbound email

## Required config

| Variable | Purpose |
|---|---|
| `SPREADSHEET_ID` | Target Google Sheet |
| `WORKSHEET_NAME` or `WORKSHEET_GID` | Worksheet within the sheet |
| `REOON_API_KEY` | Email verifier (Reoon) |

Optional provider keys (recommended for better hit-rate):

| Variable | Provider |
|---|---|
| `ROCKETREACH_API_KEY` | RocketReach — primary people-data source |
| `HUNTER_API_KEY` | Hunter — secondary fallback |
| `DROPCONTACT_API_KEY` | Dropcontact — secondary fallback |

## Default enrichment order

1. RocketReach (best direct-hit quality, paid credits)
2. Hunter free plan (fallback)
3. Dropcontact free / trial (fallback)
4. Website crawl + pattern inference

## Expected columns

- `Website`
- `Founder Name`
- `Founder Email` (target)
- `Founder LinkedIn` (optional, strong identity signal)

If header names differ, map nearest names and print the mapping before the run.

## Modes

### Dry-run (default)

- Discover candidates
- Verify candidates
- Output proposed actions only

### Write mode (explicit approval required)

- Run same logic
- Write approved result row-by-row
- Checkpoint every batch

## Discovery stack (best-first, not guess-first)

Use sources in this order per row:

1. **Existing high-confidence signals**
   - Already-present founder email in notes/source columns
   - Existing founder domain hints

2. **Provider lookup (best accuracy first)**
   - RocketReach (or equivalent people-data provider) by name + company domain
   - Secondary provider fallback if first provider misses

3. **Website extraction**
   - Crawl: `/`, `/contact`, `/about`, `/team`, `/impressum`, `/privacy`
   - Extract visible emails and `mailto:` links

4. **Pattern generation (last resort)**
   - `first@domain`, `first.last@domain`, `firstlast@domain`, `f.last@domain`, `f_last@domain`
   - Use only when identity confidence is high (name + domain match)

## Verification policy (Reoon)

Verify every candidate before write:

```bash
curl -s "https://emailverifier.reoon.com/api/v1/verify?email=EMAIL&key=$REOON_API_KEY&mode=power"
```

Status policy:

| Reoon status | Action |
|---|---|
| `safe` | approve |
| `role_account` | approve, mark as team inbox in run log |
| `catch_all` | approve only if no better candidate AND user allowed fallback writes |
| `unknown` | same as `catch_all` |
| `invalid`, `disposable` | reject |

## Confidence scoring

Score each candidate with weighted evidence:

- `+4` verifier `safe`
- `+2` verifier `role_account`
- `+3` exact domain match to website
- `+3` local-part matches founder name
- `+2` found in provider result with same company
- `+1` found on official site page
- `-3` generic mailbox (`info@`, `support@`, `contact@`) when a founder email is expected
- `-4` mismatch between founder, company, and domain

Write only the highest-confidence candidate passing the verification policy.

## Throughput + rate-limit handling

Deterministic micro-batches to avoid hangs and API throttling:

- Batch size: 20 rows
- Per-row timeout: 20s discovery, 15s verification
- Reoon concurrency: 1-2 requests max
- Backoff on 429: exponential (15s, 30s, 60s, 120s)
- Checkpoint after each row write and after each batch summary

Operational note: Reoon's free quota is ~4,500/day. Keep throttling enabled even with quota headroom to prevent burst 429 responses.

If 429 persists for 5+ minutes, pause verifier calls and continue with discovery-only queue.

## Write rules

- Write only `Founder Email` in write mode
- Preserve stronger existing values
- Never overwrite a non-empty founder email unless the user explicitly requests overwrite

## Run output

For each processed row:

```
Row | Product | Candidate | Source | Verifier Status | Confidence | Action
```

End-of-batch totals:

- written
- flagged (`catch_all` / `unknown`)
- rejected
- manual review
- no candidate
- rate-limited skips

Unresolved rows return with reason codes:

- `NO_SOURCE_EMAIL`
- `PROVIDER_MISS`
- `IDENTITY_AMBIGUOUS`
- `VERIFIER_RATE_LIMIT`
- `VERIFIER_REJECTED`

## Stop conditions

Stop and report if:

- missing required sheet/verifier config
- sheet auth fails
- header mapping unresolved
- 8+ consecutive network failures to same provider/domain

## Implementation tips

- Cache `domain → known pattern` table from prior wins; reuse before brute-forcing
- Persist per-row audit trail so retries resume cleanly
- Split pipeline into two queues: (1) discovery, (2) verification/write
- Always run a small dry-run on 5-10 rows before flipping to write mode

This structure delivers higher hit-rate and lower cost than pure guessing + verify loops.

## Related skills

| Skill | Relationship |
|---|---|
| [`paid-ads-context`](https://github.com/nickyc1/paid-ads-context) | Reads section 1 (ICP) to filter or score prospects before enrichment |
| [`customer-research`](https://github.com/nickyc1/customer-research) | Provides the interview recruit lists this skill enriches |
| [`ad-creative`](https://github.com/nickyc1/ad-creative) | Indirect — enriched prospects fuel outbound list-building that complements paid ads |
| [`n8n-recipes`](https://github.com/nickyc1/n8n-recipes) | The `webhook-lead-enrich-and-route` recipe uses the same waterfall logic embedded in an n8n workflow |
