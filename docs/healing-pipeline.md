# Healing Pipeline

Superkabe's 5-phase graduated recovery system for mailboxes and domains.

## Overview

When a mailbox gets auto-paused due to bounce threshold breach, it doesn't sit in limbo. It enters a structured recovery pipeline that gradually restores it to production.

```
PAUSED → QUARANTINE → RESTRICTED_SEND → WARM_RECOVERY → HEALTHY
```

The pipeline applies uniformly to native-sequencer mailboxes and to mailboxes on connected platforms (Smartlead, Instantly, EmailBison, Reply.io).

## State Authorities

Phase transitions are enforced by a strict single-writer model:

| Concern | Service |
|---|---|
| `Mailbox` / `Domain` / `Lead` status | `entityStateService` |
| `recovery_phase` transitions + optimistic locking | `healingService` |
| Transition validation, cooldown math, history rows | `stateTransitionService` |
| Manual-override tracking (doubles cooldowns) | `operatorProtectionService` |
| Pre-pause correlation check | `correlationService` |

No other service writes `status` or `recovery_phase` directly.

## Phase Details

### Phase 0: PAUSED

**Duration:** cooldown timer (exponential backoff)
- 1st offense: 24 hours
- 2nd offense: 72 hours
- 3rd+ offense: 7 days
- Manual-override tier: all cooldowns doubled

**Activity:** no sends. On connected platforms, the mailbox is removed from every campaign via `adapter.removeMailboxFromCampaign`. On the native sequencer, `sendQueueService` skips the mailbox on dispatcher scan.

**Graduation:** cooldown timer expires. Checked by the metrics worker every 60 seconds.

### Phase 1: QUARANTINE

**Duration:** until DNS health passes.

**Activity:** no sends. System runs domain-level infrastructure checks through `infrastructureAssessmentService`:
- SPF record valid?
- DKIM record valid?
- DMARC record valid?
- Domain present on any of ~400 DNSBL lists (`dnsblService`, tier-weighted, `Semaphore(50)`)?

**Graduation:** all checks pass. A broken SPF keeps the mailbox here until the operator fixes it — no point warming on a broken domain.

**Checked by:** `warmupTrackingWorker` every 4 hours.

### Phase 2: RESTRICTED_SEND

**Duration:** until clean-sends requirement met.

**Activity:** conservative warmup via Zapmail.
- 10 warmup emails/day, flat volume (no ramp)
- 30% target engagement rate

**Graduation criteria:**
- 1st offense: 15 clean sends with zero hard bounces
- Repeat offender: 25 clean sends
- Adjusted by resilience score (low resilience → more sends required)

### Phase 3: WARM_RECOVERY

**Duration:** minimum 3 days.

**Activity:** increased warmup volume.
- 50 warmup emails/day
- +5 emails/day ramp
- 40% target engagement rate

**Graduation criteria:**
- 50+ clean sends (adjusted for resilience)
- 3+ days in phase
- Bounce rate under 2% throughout

### Phase 4: HEALTHY

**Activity:** full production sending restored.
- Mailbox re-added to every campaign it was removed from (connected lane) or re-enabled in the native dispatcher (sequencer lane)
- Maintenance warmup (10/day) continues in background
- Resilience score: +10 bonus
- Relapse counter reset

## Relapse Handling

If a mailbox bounces during any recovery phase:

| Relapse count | Action | Cooldown |
|---|---|---|
| 1st | Regress to `QUARANTINE` | 48 hours |
| 2nd | Full `PAUSED` | 72 hours |
| 3rd+ | Full `PAUSED` + manual review flag | 7 days |

Each relapse also:
- Resets clean-sends counter to 0
- Applies −25 resilience score penalty
- Sends a Slack alert

## Optimistic Locking

Phase transitions use optimistic locking to prevent race conditions across the metrics worker, the warmup-tracking worker, and any operator-initiated transition:

```typescript
const result = await prisma.mailbox.updateMany({
  where: { id: mailboxId, recovery_phase: fromPhase },
  data: { recovery_phase: toPhase, phase_entered_at: new Date() }
});

if (result.count === 0) {
  // Another process already changed the phase — abort silently
  return;
}
```

Every transition also writes a `StateTransition` row (retained 180 days) for audit.

## Workers

| Worker | Frequency | Responsibility |
|---|---|---|
| `metricsWorker` | 60s | `PAUSED → QUARANTINE` (cooldown expiry), counter rollups |
| `warmupTrackingWorker` | 4h | `QUARANTINE → RESTRICTED → WARM → HEALTHY` graduation checks |
| `espPerformanceWorker` | 6h | 30-day per-ESP bounce/reply aggregation used by ESP-aware routing |
| `smartleadSyncWorker` / platform sync | 20m | Pull connected-platform state for cross-check |

## Pre-Pause Correlation Check

Before any `entityStateService` pause, `correlationService` classifies the failure:

- **Mailbox-level** — isolated → pause the mailbox only
- **Domain-level** — multiple siblings on the same domain failing → pause the domain
- **Provider-concentration** — spike across the same ESP → pause at the campaign/provider scope

This prevents the system from paper-cutting individual mailboxes when the real failure is upstream (e.g., an expiring DKIM key affecting a whole domain).
