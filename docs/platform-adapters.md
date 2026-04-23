# Platform Adapters

Superkabe supports five send lanes — one **native** (the built-in sequencer) and four **connected** (third-party platforms). The connected lanes are uniform behind an adapter pattern so the monitoring and healing engine can treat them identically.

## Source Platforms

The `SourcePlatform` enum (defined in `backend/src/types/index.ts`):

| Value | Lane | Needs adapter? |
|---|---|---|
| `sequencer` | Native (Gmail / Microsoft 365 / SMTP) | No — handled natively |
| `smartlead` | Connected | Yes |
| `instantly` | Connected | Yes |
| `emailbison` | Connected | Yes |
| `replyio` | Connected | Yes |

Both lanes share the same `Campaign` table, distinguished by `source_platform`. Sequencer-specific columns (`schedule_timezone`, `daily_limit`, `stop_on_reply`, `tracking_domain`, etc.) are nullable and only populated for `source_platform = 'sequencer'` rows.

## Interface

```typescript
interface PlatformAdapter {
  platform: SourcePlatform;
  pauseCampaign(orgId: string, campaignId: string): Promise<void>;
  resumeCampaign(orgId: string, campaignId: string): Promise<void>;
  addMailboxToCampaign(orgId: string, campaignId: string, mailboxId: string): Promise<void>;
  removeMailboxFromCampaign(orgId: string, campaignId: string, mailboxId: string): Promise<void>;
  pushLeadToCampaign(orgId: string, campaignId: string, lead: Lead): Promise<void>;
}
```

## Registry

Adapters are retrieved through `platformRegistry`:

```typescript
platformRegistry.getAdapter(platform)      // throws for 'sequencer'
platformRegistry.tryGetAdapter(platform)   // returns null for 'sequencer' (graceful skip)
```

The native sequencer intentionally has **no adapter**. Lead enrollment on the sequencer lane is handled by `sequencerEnrollmentService.enrollLeadInSequencerCampaign(orgId, campaignId, lead)` — the sequencer-lane equivalent of `adapter.pushLeadToCampaign`. The lead processor (`processor.ts`) branches on `source_platform` and calls the right code path.

## Supported Connected Platforms

### Smartlead

- **Pause:** `POST /campaigns/{id}/status` with `{ status: 'PAUSED' }`
- **Resume:** `POST /campaigns/{id}/status` with `{ status: 'START' }` (not `ACTIVE`)
- **Remove mailbox:** `DELETE /campaigns/{id}/email-accounts` with account ID in body
- **Rate limit:** 10 req/2s, bucket 10, queue 5000 — breaker 15 failures / 60s / 2 half-opens
- **Sync worker:** `smartleadSyncWorker.ts` runs every 20 minutes

### Instantly

- **Pause:** `POST /campaigns/{id}/pause`
- **Resume:** `POST /campaigns/{id}/activate`
- **Rate limit:** 10 req/2s, bucket 10, queue 5000 — breaker 15 / 60s / 2

### EmailBison

- **Pause:** `PATCH /api/campaigns/{id}/pause`
- **Resume:** `PATCH /api/campaigns/{id}/resume`
- **Rate limit:** 5 req/2s, bucket 5, queue 3000 — breaker 10 / 45s / 2

### Reply.io

- **Integration:** REST mutations + webhook ingestion
- **Rate limit:** 10 req/2s, bucket 10, queue 5000 — breaker 15 / 60s / 2

### Retry Policy (all connected platforms)

- `429` → infinite retry (never trips the breaker)
- `5xx` / `ECONNRESET` → max 5 retries with backoff
- `404` / `400` → ignored (no retry, no breaker trip)

## Native Sequencer (no adapter)

The sequencer lane runs through:

- `sendQueueService.ts` — 60s dispatcher + BullMQ worker. Queries `Campaign` rows where `source_platform = 'sequencer' AND status = 'active'`. Per-org priority, ESP-aware routing (60% mailbox capacity + 40% 30-day per-ESP performance), jitter spreading, transaction-wrapped `SendEvent` + `CampaignLead` + counter updates.
- `emailSendAdapters.ts` — unified SMTP/Gmail/Microsoft transporter abstraction with a 10-minute transporter cache.
- `gmailSendService.ts` / `microsoftSendService.ts` — OAuth for **mailbox** connection (distinct from user signup OAuth).
- `trackingService.ts` — open pixel, click wrap, unsubscribe footer via `utils/trackingToken.ts` (HMAC-SHA256, 180-day TTL).
- `warmupService.ts` — phase-specific warmup config delegated to Zapmail.

## Adding a New Connected Platform

1. Add a value to the `SourcePlatform` enum in `backend/src/types/index.ts`.
2. Create `backend/src/adapters/yourPlatformAdapter.ts` implementing all 5 `PlatformAdapter` methods.
3. Configure rate limiter + circuit breaker in `backend/src/utils/` (each platform gets its own instance).
4. Register in `backend/src/adapters/platformRegistry.ts`.
5. Add webhook handler under `backend/src/controllers/monitorController.ts` with HMAC signature validation (secret stored in `OrganizationSetting`).
6. Wire the sync worker entry if the platform needs a periodic pull.

The monitoring system, healing pipeline, correlation engine, and state machine all work identically regardless of which adapter is in play — the adapter layer is a thin translation to/from each vendor's API contract.
