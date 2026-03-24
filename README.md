<div align="center">
  <img src="https://www.superkabe.com/image/logo-v2.png" alt="Superkabe Logo" width="120" />

  <h1>Superkabe</h1>

  <p><strong>Email deliverability protection for B2B outbound teams.</strong></p>
  <p>Active middleware that sits between your enrichment data and sending platforms.<br/>Monitors, gates, and auto-heals your email infrastructure in real time.</p>

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

Superkabe is a **Deliverability Protection Layer (DPL)** — active middleware that physically gates SMTP and API traffic between your data sources and sending platforms.

| Capability | What it does |
|---|---|
| **Email Validation** | Validates emails before they reach your sender — syntax, MX, disposable, catch-all detection, MillionVerifier API fallback |
| **Real-Time Monitoring** | Ingests webhooks from Smartlead/Instantly/EmailBison, tracks bounce rates per mailbox every 60 seconds |
| **Automated Kill Switch** | Auto-pauses mailboxes, domains, or campaigns when bounce thresholds are breached |
| **Correlation Engine** | Detects if failures are mailbox-level, domain-level, or campaign-level — pauses the right thing |
| **5-Phase Healing Pipeline** | `PAUSED → QUARANTINE → RESTRICTED_SEND → WARM_RECOVERY → HEALTHY` — graduated recovery with warmup re-enablement |
| **Risk-Aware Routing** | GREEN/YELLOW/RED lead classification with per-mailbox risk caps (max 2 risky leads per 60 emails) |
| **Load Balancing** | Distributes sends based on mailbox health, not round-robin |
| **Multi-Platform Support** | Smartlead, Instantly, EmailBison via adapter pattern — add a new platform by implementing 4 methods |
| **Slack Alerts** | Real-time notifications for pauses, blacklistings, recovery milestones |

## How It Works

```
Clay / CSV Upload
  → Email Validation (syntax, MX, disposable, catch-all, API)
    → Health Gate (GREEN / YELLOW / RED classification)
      → Risk-Aware Routing (persona + health score)
        → Smartlead / Instantly / EmailBison (sending)
          → Bounce webhooks back to Superkabe
            → Monitoring → Auto-Pause → Healing Pipeline
```

The sending platforms are dumb pipes. The data source is an external feed. All the intelligence sits in the control plane.

## Architecture

```
                         ┌──────────────┐
                         │    Slack     │ (real-time alerts)
                         └──────┬───────┘
                                │
┌─────────────┐   ┌────────────┴─────────────┐   ┌──────────────────┐
│  Clay       │──▶│    Superkabe (DPL)       │──▶│  Smartlead       │
│  CSV Upload │   │                          │   │  Instantly       │
└─────────────┘   │  ▪ Email Validation      │   │  EmailBison      │
                  │  ▪ Health Gate            │   │  (sending)       │
                  │  ▪ Risk-Aware Routing     │   └────────┬─────────┘
                  │  ▪ Monitoring (60s cycle) │            │
                  │  ▪ Healing Pipeline       │◀───────────┘
                  │  ▪ Load Balancing         │  (bounce webhooks)
                  └──────────────────────────┘
```

### Healing Pipeline

When a mailbox gets paused, it doesn't sit in limbo. It enters a 5-phase graduated recovery:

```
PAUSED (cooldown: 24h/72h/7d)
  → QUARANTINE (DNS health check — SPF, DKIM, blacklist)
    → RESTRICTED_SEND (warmup 10/day, 15 clean sends required)
      → WARM_RECOVERY (50/day + ramp, 3+ days, <2% bounce)
        → HEALTHY (re-added to campaigns, maintenance warmup)
```

Each phase has explicit graduation criteria. Relapse penalties escalate: 1st → quarantine, 2nd → full pause 72h, 3rd+ → 7 day pause + manual review.

### Platform Adapter Pattern

```typescript
interface PlatformAdapter {
  pauseCampaign(orgId: string, campaignId: string): Promise<void>;
  resumeCampaign(orgId: string, campaignId: string): Promise<void>;
  addMailboxToCampaign(orgId: string, campaignId: string, mailboxId: string): Promise<void>;
  removeMailboxFromCampaign(orgId: string, campaignId: string, mailboxId: string): Promise<void>;
}
```

Three implementations: `SmartleadAdapter`, `InstantlyAdapter`, `EmailBisonAdapter`. Adding a new sending platform = implement 4 methods.

## Tech Stack

| Layer | Technology |
|---|---|
| **Backend** | Node.js + Express, TypeScript |
| **Database** | PostgreSQL + Prisma ORM |
| **Job Processing** | BullMQ + Redis |
| **Frontend** | Next.js 14 (App Router), React Server Components |
| **Styling** | TailwindCSS + Framer Motion |
| **Deployment** | Railway |
| **Alerts** | Slack webhook integration |

### Backend Structure

```
src/
├── routes/          # REST API routing
├── controllers/     # Request handling, validation
├── services/        # Core business logic
│   ├── emailValidationService.ts      # Email validation pipeline
│   ├── leadHealthService.ts           # GREEN/YELLOW/RED classification
│   ├── routingService.ts              # Risk-aware lead routing
│   ├── monitoringService.ts           # Real-time health monitoring
│   ├── healingService.ts              # 5-phase recovery pipeline
│   ├── warmupService.ts               # Warmup management during recovery
│   ├── correlationService.ts          # Cross-entity failure correlation
│   ├── entityStateService.ts          # State machine (single authority)
│   ├── stateTransitionService.ts      # Transition validation + cooldowns
│   ├── loadBalancingService.ts        # Health-aware send distribution
│   └── smartleadInfrastructureMutator.ts  # Platform API mutations
├── adapters/        # Platform adapters (Smartlead, Instantly, EmailBison)
├── workers/         # Background workers (metrics, warmup tracking)
└── types/           # TypeScript types, state machines, enums
```

## Getting Started

### Prerequisites

- Node.js 20+
- PostgreSQL 15+
- Redis 7+

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

### Environment Variables

```bash
DATABASE_URL="postgresql://user:pass@localhost:5432/superkabe"
REDIS_URL="redis://localhost:6379"
SMARTLEAD_API_KEY="your-smartlead-api-key"
INSTANTLY_API_KEY="your-instantly-api-key"
MILLION_VERIFIER_API_KEY="your-millionverifier-api-key"  # Optional, for API validation tier
SLACK_WEBHOOK_URL="https://hooks.slack.com/services/..."  # Optional
```

## Results

Before Superkabe:
- Bounce rate: **6-9%** (should be under 2%)
- Domains burned per month: **2-3**
- Infrastructure firefighting: **~15 hours/week**

After Superkabe:
- Bounce rate: **1.2%**
- Domains burned in 6 months: **0**
- Infrastructure management: **~2 hours/week**
- Inbox placement: **91%** (up from 52%)

## Integrations

| Platform | Ingestion | Mutation | Status |
|---|---|---|---|
| **Smartlead** | Webhook | REST API | Production |
| **Instantly** | Webhook | REST API | Production |
| **EmailBison** | Webhook | REST API | Production |
| **Clay** | Webhook | — | Production |
| **Slack** | — | Webhook | Production |

## Who Is This For

- **RevOps engineers** managing 50+ mailboxes across multiple domains
- **B2B growth agencies** running outbound for multiple clients
- **Technical founders** who can't afford to burn domains
- **GTM teams** scaling cold outreach past 10,000 emails/month

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
