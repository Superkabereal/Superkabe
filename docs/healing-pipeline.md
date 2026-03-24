# Healing Pipeline

Superkabe's 5-phase graduated recovery system for mailboxes and domains.

## Overview

When a mailbox gets auto-paused due to bounce threshold breach, it doesn't sit in limbo. It enters a structured recovery pipeline that gradually restores it to production.

```
PAUSED → QUARANTINE → RESTRICTED_SEND → WARM_RECOVERY → HEALTHY
```

## Phase Details

### Phase 0: PAUSED

**Duration:** Cooldown timer (exponential backoff)
- 1st offense: 24 hours
- 2nd offense: 72 hours
- 3rd+ offense: 7 days

**Activity:** No sends. Mailbox removed from all campaigns on the sending platform.

**Graduation:** Cooldown timer expires. Checked by `metricsWorker` every 60 seconds.

### Phase 1: QUARANTINE

**Duration:** Until DNS health passes.

**Activity:** No sends. System checks domain-level infrastructure:
- SPF record valid?
- DKIM record valid?
- Domain blacklisted?

**Graduation:** All DNS checks pass. If the domain has a broken SPF record, the mailbox stays here until it's fixed. No point warming up a mailbox on a broken domain.

**Checked by:** `warmupTrackingWorker` every 4 hours.

### Phase 2: RESTRICTED_SEND

**Duration:** Until clean sends requirement met.

**Activity:** Conservative warmup re-enabled.
- 10 warmup emails/day, flat volume (no ramp-up)
- 30% target engagement rate

**Graduation criteria:**
- 1st offense: 15 clean sends with zero hard bounces
- Repeat offender: 25 clean sends
- Adjusted by resilience score (low resilience = more sends required)

### Phase 3: WARM_RECOVERY

**Duration:** Minimum 3 days.

**Activity:** Increased warmup volume.
- 50 warmup emails/day
- +5 emails/day ramp-up
- 40% target engagement rate

**Graduation criteria:**
- 50+ clean sends (adjusted for resilience)
- 3+ days in phase
- Bounce rate under 2% throughout

### Phase 4: HEALTHY

**Activity:** Full production sending restored.
- Mailbox re-added to all campaigns it was removed from
- Maintenance warmup (10/day) continues in background
- Resilience score gets +10 bonus
- Relapse counter reset

## Relapse Handling

If a mailbox bounces during any recovery phase:

| Relapse Count | Action | Cooldown |
|---|---|---|
| 1st | Regress to QUARANTINE | 48 hours |
| 2nd | Full PAUSED | 72 hours |
| 3rd+ | Full PAUSED + manual review flag | 7 days |

Each relapse also:
- Resets clean sends counter to 0
- Applies -25 resilience score penalty
- Sends Slack alert

## Optimistic Locking

Phase transitions use optimistic locking to prevent race conditions:

```typescript
const result = await prisma.mailbox.updateMany({
  where: { id: mailboxId, recovery_phase: fromPhase },
  data: { recovery_phase: toPhase, phase_entered_at: new Date() }
});

if (result.count === 0) {
  // Another process already changed the phase — abort
  return;
}
```

## Workers

| Worker | Frequency | Responsibility |
|---|---|---|
| `metricsWorker` | Every 60s | PAUSED → QUARANTINE (cooldown expiry) |
| `warmupTrackingWorker` | Every 4h | QUARANTINE → RESTRICTED → WARM → HEALTHY |
