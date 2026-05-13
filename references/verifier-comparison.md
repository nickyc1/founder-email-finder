# Email Verifier Comparison

A practical comparison of the major email verification services. Pricing and capabilities current as of late 2026.

## TL;DR

| Verifier | Best for | Free tier | Approximate cost |
|---|---|---|---|
| [Reoon](https://reoon.com) | Day-to-day batch verification, AppSumo LTD | 4,500/day for free | Lifetime deal $39-89 covers most operators |
| [Hunter](https://hunter.io) | Combined finder + verifier, smaller volumes | 25/month | $34-149/mo by volume |
| [ZeroBounce](https://www.zerobounce.net) | Enterprise-grade accuracy, deliverability scoring | 100 trial | $16 per 1,000 (1-100K), $0.0080-0.0035 per email above |
| [NeverBounce](https://neverbounce.com) | High-volume scrubs of lists 100K+ | 1,000 free | $8 per 1,000 (1-10K), drops to $3 at scale |
| [Bouncer](https://www.usebouncer.com) | Solid all-rounder, transparent pricing | 100 trial | $9 per 1,000 (1-10K) |
| [MillionVerifier](https://millionverifier.com) | Lowest-cost bulk option | 100 free | $4 per 1,000 (10K+) |

## What "accuracy" actually means

Verifiers report results in roughly the same buckets, but each one uses slightly different definitions:

| Status | What it usually means |
|---|---|
| `safe` / `valid` / `deliverable` | Mail server accepts; mailbox exists; low bounce risk |
| `role_account` | Generic mailbox (`info@`, `support@`, `team@`) — deliverable but not personal |
| `catch_all` / `accept_all` | Domain accepts everything; mailbox existence can't be confirmed |
| `unknown` | Verifier couldn't get a definitive answer |
| `invalid` / `undeliverable` | Mailbox doesn't exist, domain doesn't exist, or syntax broken |
| `disposable` | Temporary / throwaway domain (mailinator, etc.) |

Different verifiers categorize the same email differently. A `safe` on Reoon may show as `catch_all` on Hunter. There is no universal ground truth.

Two practical implications:

1. **Don't try to outsmart it.** Treat each verifier's categories as that verifier's opinion. Don't reclassify.
2. **For high-stakes lists (cold outreach), use two verifiers.** Reoon's `safe` plus Hunter's `safe` is more reliable than either alone.

## Pick by use case

### "I run a Google Sheet of 1K-5K prospects per month"

Use Reoon. The lifetime deal pays back in week 2. The API is simple. 4,500/day free tier is more than enough.

### "I need a finder + verifier in one"

Use Hunter. Their `domain-search` and `email-finder` endpoints feed verifications directly. Higher per-call cost, but you save the integration work.

### "I'm scrubbing a list of 50K-500K"

Use NeverBounce or MillionVerifier. Bulk pricing breaks toward $0.003-0.004 per email at high volume.

### "I'm running enterprise-scale outbound"

Use ZeroBounce or Bouncer. Their accuracy and deliverability scoring add value that's worth the price premium when an outreach campaign deliverability hit costs you weeks.

### "I want offline / on-prem verification"

Look at [DeBounce](https://debounce.io) — they have a Windows desktop app. Reoon also has a YellowPages desktop scraper companion that includes verification.

## API call patterns

Most verifiers follow the same pattern:

```
GET https://api.example.com/v1/verify?email=EMAIL&key=KEY
→ {
    "email": "alice@example.com",
    "status": "safe",
    "details": { ... provider-specific fields ... }
  }
```

Reoon's full URL: `https://emailverifier.reoon.com/api/v1/verify?email=EMAIL&key=KEY&mode=power`

Hunter's: `https://api.hunter.io/v2/email-verifier?email=EMAIL&api_key=KEY`

ZeroBounce's: `https://api.zerobounce.net/v2/validate?api_key=KEY&email=EMAIL&ip_address=`

NeverBounce's: `https://api.neverbounce.com/v4/single/check?key=KEY&email=EMAIL`

The skill defaults to Reoon but can fall back. Set the order in your account config.

## Rate limits

| Verifier | Per-second | Per-minute | Per-day |
|---|---|---|---|
| Reoon | 2-3 | ~60 | 4,500 (free) / unlimited paid |
| Hunter | 10 | 600 | depends on plan |
| ZeroBounce | 100 | depends on plan | depends on plan |
| NeverBounce | 50 | varies | depends on plan |

Default the skill to batch-of-20 with a 60-second wait between batches. This works for all of them and prevents 429s.

## Cost math at typical operator volume

For a marketing operator verifying ~500 emails/week (~26K/year):

| Verifier | Annual cost |
|---|---|
| Reoon lifetime | $89 once, then $0 |
| Hunter Pro | $49/mo × 12 = $588 |
| ZeroBounce pay-as-you-go | ~$208 (at $8/1K) |
| NeverBounce pay-as-you-go | ~$104 (at $4/1K, with the discounted tier) |
| MillionVerifier | ~$78 (at $3/1K bulk) |

Reoon's lifetime deal is the clear winner for this profile if it's available. Otherwise NeverBounce bulk pricing wins on per-email cost.

## Multi-provider waterfall

Best practice for high-quality outreach lists:

1. **Primary:** Reoon (`mode=power`)
2. **If status is `unknown` or `catch_all`:** Hunter for second opinion
3. **If both say uncertain:** ZeroBounce for tiebreaker
4. **Mark as risky** if all three disagree

Pseudocode:

```python
def verify(email):
    r1 = reoon.verify(email)
    if r1.status in ("safe", "invalid", "disposable"):
        return r1
    r2 = hunter.verify(email)
    if r2.status == r1.status:
        return r1  # consensus
    # disagreement — try a third
    r3 = zerobounce.verify(email)
    return decide(r1, r2, r3)
```

Cost increases linearly with how often you fall through. For most lists, ~80% of emails resolve at step 1.

## Common pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Hitting rate limits | 429s during a batch | Add the 60s wait between batches |
| Trusting `catch_all` as `safe` | Bounces creeping up over time | Treat `catch_all` as `risky`, not `safe` |
| Re-verifying the same email | Burning credits | Cache by `email -> status -> timestamp`, refresh weekly |
| Skipping the verifier entirely | Bounce rate destroying domain reputation | Always verify before send, period |
| Using verifier as a finder | High miss rate | Verifiers don't find emails — pair with RocketReach, Hunter Finder, etc. |

## When to skip verification

You can skip the verifier when:

- The email came directly from the customer (form submission, CRM sync from a paid account)
- You're sending to a list you've sent to within the last 14 days without bounces
- The volume is below ~5 emails per day (manual judgment is fine)

Verify anytime you're sending to a list that was acquired, scraped, or enriched.
