# Email Validation Layer

Superkabe validates every incoming email before it reaches your sending platform. This prevents bad leads from spiking your bounce rate.

## Pipeline

```
Lead arrives (Clay webhook or CSV upload)
  → Internal validation (free, <50ms)
    → API validation if needed (paid, conditional)
      → Health Gate classification (GREEN/YELLOW/RED)
        → Routing to campaign
```

## Internal Checks (always run)

| Check | What it does | Impact on score |
|---|---|---|
| Syntax validation | RFC 5322 compliance | Invalid = score 0, blocked |
| MX record lookup | Domain has a mail server? | No MX = score 0, blocked |
| Disposable domain | Checked against ~30k known disposable providers | Match = score 0, blocked |
| Catch-all detection | Domain accepts any address? | Catch-all = -30 to score |

## API Verification (conditional)

Uses MillionVerifier's API for SMTP-level probing. Only triggered when:
- Internal score is below confidence threshold
- Organization's tier allows API calls

### Tier-Based API Usage

| Tier | When API is called | Estimated API usage |
|---|---|---|
| Starter (10k leads) | Never — internal only | 0% |
| Growth (50k leads) | Score < 60 or catch-all detected | ~20-30% of leads |
| Scale (100k leads) | Score < 75 or unknown or catch-all | ~35-40% of leads |

## Domain Insight Caching

The biggest cost saver. If `@bigcorp.com` is a catch-all domain, we check ONE email, cache the result, and skip API checks for every subsequent lead at that domain.

For a customer with 50k leads across 500 domains, this turns 50k potential API calls into ~500.

## Scoring

| Score | Classification | Routing |
|---|---|---|
| 80-100 | GREEN | Normal routing, any mailbox |
| 50-79 | YELLOW | Routed with risk caps (max 2 risky per 60 emails per mailbox) |
| 0-49 | RED | Blocked. Never reaches sending platform. |

## Data Model

Validation results are stored on the Lead record. Full attempt history is stored in `ValidationAttempt` (separate model) for accuracy analysis over time.

Existing leads in the platform are not retroactively validated — this layer only applies to newly ingested leads.
