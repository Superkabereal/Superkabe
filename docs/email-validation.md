# Email Validation Layer

Superkabe validates every incoming email before it reaches a sending lane (native sequencer or connected platform). This prevents bad leads from spiking bounce rate and triggering the healing pipeline unnecessarily.

## Pipeline

```
Lead arrives (Clay webhook, CSV upload, or API v1 ingest)
  → Internal validation (free, <50ms)
    → DomainInsight cache lookup (per-domain catch-all / MX memoization)
      → API validation if needed (paid, tier-gated)
        → Health Gate classification (GREEN / YELLOW / RED)
          → Routing (persona + health + ESP perf) → campaign
```

All incoming paths are signed:
- `/api/ingest/clay` — HMAC-SHA256
- `/api/validation` — authenticated CSV upload (stored as `ValidationBatch` + `ValidationLead` rows)
- `/api/ingest` — API v1 with API key

## Internal Checks (always run)

| Check | What it does | Impact on score |
|---|---|---|
| Syntax validation | RFC 5322 compliance | Invalid = score 0, blocked |
| MX record lookup | Domain has a mail server? | No MX = score 0, blocked |
| Disposable domain | Checked against ~30k known disposable providers | Match = score 0, blocked |
| Catch-all detection | Domain accepts any address? | Catch-all = −30 to score |

## API Verification (conditional, tier-gated)

Uses MillionVerifier's API for SMTP-level probing. Only triggered when:
- The organization's subscription tier allows API calls, **and**
- The internal score is below that tier's threshold (or the domain is catch-all / unknown).

### Tier-Based API Usage

| Tier | API enabled? | Threshold | Typical API usage |
|---|---|---|---|
| **Trial** (14d) | ❌ | — | 0% |
| **Starter** | ❌ | — | 0% (internal only) |
| **Growth** | ✅ | score < 60 | ~20–30% of leads |
| **Scale** | ✅ | score < 75 | ~35–40% of leads |
| **Enterprise** | ✅ | score < 75 | ~35–40% of leads (custom credits) |

Credits are deducted per API call and tracked against the org's monthly validation allowance.

## DomainInsight Caching

The biggest cost saver. Catch-all, MX, and known-bad verdicts are cached on the `DomainInsight` table — if `@bigcorp.com` is catch-all, we check ONE email, cache the result, and skip API checks for every subsequent lead at that domain.

For a customer with 50k leads across 500 domains, this turns 50k potential API calls into ~500.

Response caching is also Redis-backed (`utils/responseCache.ts`) with 15-second TTL, per-org scoped, `scanStream`-based invalidation (never `KEYS`).

## Scoring

| Score | Classification | Routing behavior |
|---|---|---|
| 80–100 | **GREEN** | Normal routing, any eligible mailbox |
| 50–79 | **YELLOW** | Routed with risk caps (max 2 risky per 60 emails per mailbox) |
| 0–49 | **RED** | Blocked. Never reaches any sending lane. |

Classification is done by `leadHealthService`; routing enforcement is in `routingService`.

## Data Model

- `Lead` — carries the current validation score and Health Gate classification.
- `ValidationBatch` + `ValidationLead` — CSV batches; each row tracks per-email status.
- `ValidationAttempt` — full attempt history (method, provider, latency, verdict), retained for accuracy analysis over time.
- `DomainInsight` — per-domain cached verdicts (catch-all, MX, disposable, last-checked).

Existing leads in the platform are **not** retroactively validated — this layer only applies to newly ingested leads. Operators can manually re-validate a batch from the dashboard if needed.
