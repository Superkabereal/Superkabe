<div align="center">
  <img src="https://www.superkabe.com/image/logo-v2.png" alt="Superkabe Logo" width="120" />

  <h1>Superkabe</h1>

  <p><strong>Native cold email sender + deliverability protection layer for B2B outbound teams.</strong></p>
  <p>Send campaigns from your own Gmail, Microsoft 365, or SMTP mailboxes <em>and/or</em> run as active middleware in front of Smartlead, Instantly, EmailBison, or Reply.io.<br/>One dashboard. One protection layer. Two execution paths.</p>

  <a href="https://superkabe.com">Website</a> &nbsp;|&nbsp;
  <a href="https://superkabe.hashnode.dev">Blog</a> &nbsp;|&nbsp;
  <a href="https://medium.com/@richardson_50826">Medium</a> &nbsp;|&nbsp;
  <a href="#getting-started">Getting Started</a> &nbsp;|&nbsp;
  <a href="#architecture">Architecture</a>

  <br/><br/>

  [![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
  [![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue.svg)](https://www.typescriptlang.org/)
  [![Node.js](https://img.shields.io/badge/Node.js-20+-green.svg)](https://nodejs.org/)
  [![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

</div>

---

## The Problem

Every cold email guide focuses on copy and subject lines. Nobody talks about what happens when your infrastructure breaks at scale.

A single bad lead list with 8% invalid emails spikes your bounce rate overnight. One domain gets blacklisted and the others sharing that IP range suffer too. A mailbox sending fine yesterday gets flagged because Gmail changed their filtering rules.

At scale, outbound isn't a marketing problem. It's an infrastructure problem.

**The cost:** A burned domain = ~$20,000 in lost pipeline revenue (10 missed meetings x $10K deal size x 20% close rate). Most teams burn 2-3 domains per quarter without realizing it.

## What Superkabe Does

Superkabe runs two lanes on the same tenant:

**Native Sequencer** — connect Gmail, Microsoft 365, or SMTP mailboxes directly. Build multi-step sequences with A/B variants, per-campaign schedules, daily limits, and send-time spreading. Full tracking, unified inbox, ESP-aware routing.

**Deliverability Protection Layer (DPL)** — active middleware that gates SMTP and API traffic between your data sources and sending platforms. Applies uniformly to native sends and to connected Smartlead / Instantly / EmailBison / Reply.io campaigns.

| Capability | What it does |
|---|---|
| **Native Sequencer** | Send via Gmail, Microsoft 365, or SMTP. Multi-step sequences with A/B variants, daily limits, and scheduled windows |
| **Unified Inbox** | Reply threading across every connected mailbox via IMAP polling (60s cadence) |
| **Tracking** | HMAC-signed open pixels, click wrappers, and unsubscribe links (180-day TTL, replay-safe) |
| **ESP-Aware Routing** | Distributes sends using 60% mailbox capacity + 40% 30-day per-ESP bounce/reply performance |
| **Email Validation** | Syntax, MX, disposable, catch-all detection, MillionVerifier API fallback (tier-gated) |
| **Domain Insight Cache** | Catch-all verdicts cached per-domain — 50k leads across 500 domains → ~500 API calls |
| **Real-Time Monitoring** | Ingests webhooks from Smartlead / Instantly / EmailBison / Reply.io, tracks bounce rates per mailbox every 60s |
| **Automated Kill Switch** | Auto-pauses mailboxes, domains, or campaigns when thresholds are breached |
| **Correlation Engine** | Pre-pause check: mailbox-level, domain-level, sibling, or provider-concentration failure |
| **5-Phase Healing Pipeline** | `PAUSED → QUARANTINE → RESTRICTED_SEND → WARM_RECOVERY → HEALTHY` — graduated recovery with warmup re-enablement via Zapmail |
| **Risk-Aware Routing** | GREEN/YELLOW/RED lead classification with per-mailbox risk caps (max 2 risky leads per 60 emails) |
| **Infrastructure Assessment** | SPF/DKIM/DMARC checks + 400+ DNSBL queries with tier weighting |
| **Multi-Platform Support** | Smartlead, Instantly, EmailBison, Reply.io via adapter pattern |
| **Slack Alerts** | Real-time notifications for pauses, blacklistings, recovery milestones (with OAuth install + slash commands) |
| **MCP Server** | 16 tools over stdio for AI agent integration (lead import, campaign CRUD, launch/pause, replies) |

## How It Works

```
Clay / CSV / API ingest
  → Email Validation (syntax, MX, disposable, catch-all, API)
    → Health Gate (GREEN / YELLOW / RED classification)
      → Routing + Execution Gate (persona + health + ESP perf)
        ├─▶ Native Sequencer (Gmail / Microsoft 365 / SMTP)
        └─▶ Smartlead / Instantly / EmailBison / Reply.io
          → Bounce + send + reply events back to Superkabe
            → Monitoring → Auto-Pause → 5-Phase Healing
```

The sending lane is whatever the campaign is configured for. The protection layer is shared — every send path goes through the same validation, gating, monitoring, and healing.

## Architecture

```
                         ┌──────────────┐
                         │    Slack     │ (real-time alerts)
                         └──────┬───────┘
                                │
┌─────────────┐   ┌────────────┴─────────────┐   ┌──────────────────┐
│  Clay       │──▶│    Superkabe             │──▶│  NATIVE:         │
│  CSV Upload │   │                          │   │  Gmail / M365 /  │
│  API v1     │   │  ▪ Email Validation      │   │  SMTP            │
└─────────────┘   │  ▪ Health Gate           │   │                  │
                  │  ▪ Risk-Aware Routing    │   │  CONNECTED:      │
                  │  ▪ Execution Gate        │──▶│  Smartlead       │
                  │  ▪ Monitoring (60s)      │   │  Instantly       │
                  │  ▪ Healing Pipeline      │   │  EmailBison      │
                  │  ▪ ESP-Aware Routing     │   │  Reply.io        │
                  │  ▪ IMAP Reply Polling    │◀──┤                  │
                  │  ▪ Tracking Endpoints    │   └──────────────────┘
                  └──────────────────────────┘  (webhooks + IMAP back in)
```

### Unified Campaign Model

Legacy (platform-synced) campaigns and native sequencer campaigns live in the **same `Campaign` table**, distinguished by `source_platform` (`smartlead`, `instantly`, `emailbison`, `replyio`, `sequencer`). Child entities (`SequenceStep`, `CampaignAccount`, `CampaignLead`) reference `Campaign` directly. Sequencer-only fields (schedule, daily limits, tracking, stop-on-reply) are nullable on legacy rows.

### Healing Pipeline

When a mailbox gets paused, it doesn't sit in limbo. It enters a 5-phase graduated recovery:

```
PAUSED (cooldown: 24h / 72h / 7d)
  → QUARANTINE (DNS health check — SPF, DKIM, DMARC, blacklist)
    → RESTRICTED_SEND (warmup 10/day, 15 clean sends required)
      → WARM_RECOVERY (50/day + ramp, 3+ days, <2% bounce)
        → HEALTHY (re-added to campaigns, maintenance warmup)
```

Each phase has explicit graduation criteria enforced by `stateTransitionService`. Relapse penalties escalate. Phase transitions use optimistic locking (`updateMany` + `recovery_phase` filter) to prevent race conditions across workers.

### State Authorities

Strict single-writer rule for each entity's status:

| Entity | Status authority |
|---|---|
| `Lead`, `Mailbox`, `Domain` | `entityStateService` |
| `Campaign` | `campaignHealthService` (infrastructure-driven, not bounce-rate) |
| Phase transitions | `healingService` with optimistic locking |
| Cooldowns + transition history | `stateTransitionService` |

All other services route through these — no direct `prisma.mailbox.update({ status })` elsewhere in the codebase.

### Platform Adapter Pattern

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

Four implementations: `SmartleadAdapter`, `InstantlyAdapter`, `EmailBisonAdapter`, `ReplyioAdapter`. Sequencer campaigns need no adapter — `sequencerEnrollmentService` handles lead enrollment natively. `platformRegistry.getAdapter()` throws for `sequencer`; `tryGetAdapter()` returns `null` for graceful skip.

## Tech Stack

| Layer | Technology |
|---|---|
| **Backend** | Node.js 20, Express 5, TypeScript |
| **Database** | PostgreSQL 16, Prisma 5 |
| **Job Processing** | BullMQ + Redis (distributed locking via `SET NX EX`) |
| **Frontend** | Next.js 16 (App Router), React 19.2, TailwindCSS 3.4 |
| **Editor** | TipTap (rich email bodies with variable insertion) |
| **Mailbox OAuth** | `googleapis`, `@azure/msal-node` for Gmail + Microsoft 365 |
| **IMAP** | `imapflow` (5 parallel accounts, 60s polling) |
| **Warmup** | Zapmail integration, phase-specific config |
| **Encryption** | AES-256-GCM (`salt:iv:authTag:encrypted`) for mailbox credentials |
| **Billing** | Polar (webhook-validated subscription events) |
| **Deployment** | Docker Compose (staging) / Railway (production) |
| **Alerts** | Slack (OAuth install, slash commands, Events API) |

### Backend Structure

```
backend/src/
├── routes/          # REST API routing
├── controllers/     # Request handling, validation
├── services/        # Core business logic
│   ├── emailValidationService.ts         # Validation pipeline + DomainInsight cache
│   ├── leadHealthService.ts              # GREEN/YELLOW/RED classification
│   ├── routingService.ts                 # Risk-aware lead routing
│   ├── monitoringService.ts              # Real-time health monitoring
│   ├── healingService.ts                 # 5-phase recovery pipeline
│   ├── warmupService.ts                  # Phase-specific warmup via Zapmail
│   ├── correlationService.ts             # Cross-entity failure correlation
│   ├── entityStateService.ts             # Lead/Mailbox/Domain status authority
│   ├── stateTransitionService.ts         # Transition validation + cooldowns
│   ├── campaignHealthService.ts          # Campaign.status authority
│   ├── loadBalancingService.ts           # Effective load share analysis
│   ├── infrastructureAssessmentService.ts # SPF/DKIM/DMARC + DNSBL
│   ├── dnsblService.ts                   # 400+ DNSBL queries, tier-weighted
│   ├── sendQueueService.ts               # Native sequencer dispatcher + BullMQ
│   ├── emailSendAdapters.ts              # SMTP/Gmail/Microsoft transporter cache
│   ├── gmailSendService.ts               # Gmail OAuth + send
│   ├── microsoftSendService.ts           # Microsoft 365 OAuth + send
│   ├── trackingService.ts                # Open pixel, click wrap, unsubscribe
│   ├── sequencerEnrollmentService.ts     # Native-lane lead enrollment
│   ├── mailboxProvisioningService.ts     # Shadow Mailbox+Domain for ConnectedAccount
│   ├── smartleadInfrastructureMutator.ts # Smartlead API write-back
│   └── complianceService.ts              # Retention: 90d/180d/365d/730d
├── adapters/        # Smartlead, Instantly, EmailBison, Reply.io + registry
├── workers/         # Lead processor 10s, send dispatcher 60s, IMAP 60s,
│                    # platform sync 20m, warmup 4h, ESP perf 6h, scoring 24h
└── types/           # MONITORING_THRESHOLDS, STATE_TRANSITIONS, enums
```

### MCP Server

Model Context Protocol server (`mcp-server/`) exposes 16 tools over stdio for AI agent integration, authenticated via `SUPERKABE_API_KEY`:

`get_account`, `import_leads`, `list_leads`, `get_lead`, `validate_leads`, `get_validation_results`, `create_campaign`, `list_campaigns`, `get_campaign`, `update_campaign`, `launch_campaign`, `pause_campaign`, `get_campaign_report`, `get_campaign_replies`, `send_reply`, `list_mailboxes`, `list_domains`.

## Getting Started

### Prerequisites

- Node.js 20+
- PostgreSQL 15+
- Redis 7+
- Docker + Docker Compose (for staging)

### Installation

```bash
# Clone the repository
git clone https://github.com/Superkabereal/Superkabe.git
cd Superkabe

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env
# Edit .env with your database URL, Redis URL, and API keys

# Run database migrations
npx prisma migrate deploy

# Start the development server
npm run dev
```

### Staging (Docker Compose)

```bash
./staging.sh up    # Postgres 16 :5433, backend :4000, frontend :3000
```

### Environment Variables

```bash
# Core
DATABASE_URL="postgresql://user:pass@localhost:5432/superkabe"
REDIS_URL="redis://localhost:6379"
ENCRYPTION_KEY="..."                 # AES-256-GCM key for mailbox credentials

# Connected platforms (any subset)
SMARTLEAD_API_KEY="..."
INSTANTLY_API_KEY="..."
EMAILBISON_API_KEY="..."
REPLYIO_API_KEY="..."

# Validation
MILLION_VERIFIER_API_KEY="..."       # Tier-gated: growth/scale/enterprise

# Native sequencer OAuth
GOOGLE_OAUTH_CLIENT_ID="..."
GOOGLE_OAUTH_CLIENT_SECRET="..."
MICROSOFT_OAUTH_CLIENT_ID="..."
MICROSOFT_OAUTH_CLIENT_SECRET="..."

# Integrations
SLACK_CLIENT_ID="..."                # For /slack OAuth install
SLACK_SIGNING_SECRET="..."
POLAR_WEBHOOK_SECRET="..."           # Billing webhook HMAC
ZAPMAIL_API_KEY="..."                # Warmup during recovery
```

## Subscription Tiers

| Tier | Validation credits | Mailbox limit | MillionVerifier API |
|---|---|---|---|
| **Trial** (14 days) | capped | capped | disabled |
| **Starter** | 10k/mo | small | disabled (internal only) |
| **Growth** | 50k/mo | medium | enabled at score < 60 |
| **Scale** | 100k/mo | large | enabled at score < 75 |
| **Enterprise** | custom | custom | enabled at score < 75 |

Billing events flow through Polar webhooks (HMAC-validated).

## Results

Representative outcomes from teams running Superkabe:

- Bounce rate: **1.2%** (down from 6–9%)
- Domains burned in 6 months: **0** (vs. 2–3/month before)
- Infrastructure management: **~2 hours/week** (down from ~15)
- Inbox placement: **91%** (up from 52%)

## Integrations

| Platform | Ingestion | Mutation | Status |
|---|---|---|---|
| **Gmail** (native) | OAuth + IMAP | SMTP send | Production |
| **Microsoft 365** (native) | OAuth + IMAP | Graph send | Production |
| **SMTP** (native) | IMAP | SMTP send | Production |
| **Smartlead** | Webhook | REST API | Production |
| **Instantly** | Webhook | REST API | Production |
| **EmailBison** | Webhook | REST API | Production |
| **Reply.io** | Webhook | REST API | Production |
| **Clay** | Webhook (HMAC) | — | Production |
| **Slack** | OAuth + Events | Webhook | Production |
| **Polar** | Webhook (HMAC) | — | Production (billing) |
| **MillionVerifier** | — | REST API | Production (tier-gated) |
| **Zapmail** | — | REST API | Production (warmup) |

## Public Endpoints (no auth)

Security-sensitive surface; all validated via HMAC or signed tokens:

- `/t/o/:token`, `/t/c/:token`, `/t/u/:token` — HMAC-signed tracking (180-day TTL, replay-safe, forged tokens silently fail closed)
- `/api/ingest/clay` — HMAC-SHA256 signature required
- `/api/monitor/{smartlead,instantly,emailbison,replyio}-webhook` — HMAC-SHA256 from `OrganizationSetting`
- `/api/billing/polar-webhook` — HMAC-validated
- `/api/auth/google`, `/sequencer/accounts/{google,microsoft}/callback` — OAuth callbacks
- `/slack/*` — Slack OAuth / commands / events

## Who Is This For

- **RevOps engineers** managing 50+ mailboxes across multiple domains
- **B2B growth agencies** running outbound for multiple clients
- **Technical founders** who can't afford to burn domains
- **GTM teams** scaling cold outreach past 10,000 emails/month
- **AI agent builders** who want programmatic campaign control (MCP server)

## Contributing

We welcome contributions. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## Blog Posts

- [From Domain Purchase to First Reply: Engineering Deliverability-First Outbound](https://superkabe.hashnode.dev/from-domain-purchase-to-first-reply-engineering-deliverability-first-outbound) — Hashnode
- [Why 90% of Cold Outreach Fails (It's Not Your Copy)](https://medium.com/@richardson_50826/why-90-of-cold-outreach-fails-its-not-your-copy-7fdbd48aa7d4) — Medium
- [Building an Outbound Email Engine That Doesn't Burn Your Domains](https://dev.to/richardson_euginsimon_2e/building-an-outbound-email-engine-that-doesnt-burn-your-domains-architecture-deep-dive-1dpg) — Dev.to

## License

[MIT](LICENSE)

---

<div align="center">
  <a href="https://superkabe.com"><strong>superkabe.com</strong></a>
</div>
