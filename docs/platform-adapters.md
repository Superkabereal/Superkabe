# Platform Adapters

Superkabe uses an adapter pattern to support multiple email sending platforms through a single interface.

## Interface

```typescript
interface PlatformAdapter {
  platform: string;
  pauseCampaign(orgId: string, campaignId: string): Promise<void>;
  resumeCampaign(orgId: string, campaignId: string): Promise<void>;
  addMailboxToCampaign(orgId: string, campaignId: string, mailboxId: string): Promise<void>;
  removeMailboxFromCampaign(orgId: string, campaignId: string, mailboxId: string): Promise<void>;
}
```

## Supported Platforms

### Smartlead

- **Pause:** `POST /campaigns/{id}/status` with `{ status: 'PAUSED' }`
- **Resume:** `POST /campaigns/{id}/status` with `{ status: 'START' }` (not `ACTIVE`)
- **Remove mailbox:** `DELETE /campaigns/{id}/email-accounts` with account ID in body
- **Rate limit:** 10 requests per 2 seconds

### Instantly

- **Pause:** `POST /campaigns/{id}/pause`
- **Resume:** `POST /campaigns/{id}/activate`
- **Remove/add mailbox:** Standard REST endpoints
- **Rate limit:** More generous than Smartlead

### EmailBison

- **Pause:** `PATCH /api/campaigns/{id}/pause`
- **Resume:** `PATCH /api/campaigns/{id}/resume`
- **Rate limit:** Standard

## Adding a New Platform

1. Create `src/adapters/yourPlatformAdapter.ts`
2. Implement all 4 methods of the `PlatformAdapter` interface
3. Add rate limiting inside the adapter (each platform has different limits)
4. Register in `src/adapters/platformRegistry.ts`
5. Add webhook parsing in a new controller

The monitoring system, healing pipeline, and all other services work identically regardless of which adapter is being used.
