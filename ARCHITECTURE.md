# DeployClaw — Platform Architecture

> **Version:** 1.0 · **Date:** 2026-02-17 · **Author:** Automus Prime  
> **Goal:** Deploy morning of Feb 17. Security-first multi-tenant agent hosting.

---

## 1. Overview

DeployClaw is a managed platform for deploying, securing, and monitoring OpenClaw AI agents. Each customer gets isolated agent instances with encrypted credential storage, audit logging, permission boundaries, and 2FA-protected dashboards.

**Core Differentiators vs SimpleClaw:**
- Security-first (isolated containers, encrypted vaults, audit trails)
- Multi-tenant with zero cross-contamination
- Enterprise-ready from day one
- 20% cheaper pricing

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      DEPLOYCLAW                          │
│                                                          │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │  Landing  │    │  Dashboard   │    │   Admin API   │  │
│  │  (Static) │    │  (Next.js)   │    │  (Node/Express│  │
│  │           │    │  + Auth      │    │   or Hono)    │  │
│  └──────────┘    └──────┬───────┘    └───────┬───────┘  │
│                         │                     │          │
│                    ┌────┴─────────────────────┴────┐     │
│                    │        Core API Layer          │     │
│                    │   (Auth · Billing · Provision)  │     │
│                    └────────────┬───────────────────┘     │
│                                │                          │
│         ┌──────────────────────┼──────────────────────┐  │
│         │                      │                      │  │
│  ┌──────▼──────┐  ┌───────────▼──────┐  ┌───────────▼┐  │
│  │  Credential │  │   Orchestrator   │  │  Audit Log │  │
│  │  Vault      │  │   (Container     │  │  Service   │  │
│  │  (encrypted │  │    lifecycle)    │  │  (append-  │  │
│  │   at rest)  │  │                  │  │   only)    │  │
│  └─────────────┘  └────────┬─────────┘  └────────────┘  │
│                            │                              │
│         ┌──────────────────┼──────────────────────┐      │
│         │                  │                      │      │
│  ┌──────▼──────┐  ┌───────▼──────┐  ┌────────────▼─┐   │
│  │  Agent A    │  │  Agent B     │  │  Agent C     │   │
│  │  (isolated  │  │  (isolated   │  │  (isolated   │   │
│  │   container)│  │   container) │  │   container) │   │
│  │  Client X   │  │  Client X    │  │  Client Y    │   │
│  └─────────────┘  └──────────────┘  └──────────────┘   │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| **Landing page** | Static HTML/CSS (existing) | Already built, fast, zero deps |
| **Dashboard** | Next.js 14 (App Router) | SSR, auth built-in, React ecosystem |
| **API** | Hono on Node.js | Lightweight, fast, TypeScript-first |
| **Database** | PostgreSQL (Neon free tier → Supabase) | Relational, proven, free tier available |
| **Auth** | Better Auth or Lucia | Self-hosted, 2FA support, no vendor lock |
| **Credential Vault** | AES-256-GCM encrypted in DB + env master key | Simple, secure, no external dependency |
| **Container Orchestration** | Docker Compose → Kubernetes (later) | Start simple, scale when needed |
| **Hosting** | Hetzner CX22 ($5.40/mo) or Railway | Cheap, EU+US, good Docker support |
| **Monitoring** | Custom health checks + Uptime Kuma | Self-hosted, free |
| **Audit Logs** | Append-only PostgreSQL table | Immutable, queryable |
| **Payments** | Stripe | Industry standard |

---

## 4. Security Architecture (Non-Negotiable)

### 4.1 Isolation
- **Each agent runs in its own Docker container** with:
  - Separate network namespace
  - No shared volumes between clients
  - Resource limits (CPU, RAM, disk)
  - Read-only filesystem where possible
- **Network policies**: Agents can only reach their authorized external services
- **No agent can access another client's container, data, or credentials**

### 4.2 Credential Encryption
```
┌─────────────────────────────────────────┐
│  Master Key (env var, never in DB)       │
│  ↓                                       │
│  Per-Client Key (derived via HKDF)       │
│  ↓                                       │
│  AES-256-GCM encryption of each secret   │
│  ↓                                       │
│  Stored: { iv, ciphertext, tag, keyId }  │
└─────────────────────────────────────────┘
```
- API keys, OAuth tokens, passwords encrypted at rest
- Master key rotatable without re-encrypting everything (key versioning)
- Decryption only happens in-memory, never logged

### 4.3 Audit Logging
Every external action by any agent is logged:
```sql
CREATE TABLE audit_logs (
  id          BIGSERIAL PRIMARY KEY,
  agent_id    UUID NOT NULL REFERENCES agents(id),
  client_id   UUID NOT NULL REFERENCES clients(id),
  action_type VARCHAR(50) NOT NULL,  -- 'email_sent', 'api_call', 'file_access', etc.
  target      TEXT,                   -- recipient, URL, file path
  metadata    JSONB,                  -- action-specific details
  ip_address  INET,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Append-only: no UPDATE or DELETE permissions granted
-- Retention: 90 days default, configurable per client
```

### 4.4 Permission Boundaries
```typescript
interface AgentPermissions {
  // Scopes the agent is authorized to use
  scopes: (
    | 'email:read' | 'email:send'
    | 'calendar:read' | 'calendar:write'
    | 'files:read' | 'files:write'
    | 'api:outbound'
    | 'messaging:send'
  )[];
  
  // Rate limits per scope
  rateLimits: {
    [scope: string]: {
      maxPerHour: number;
      maxPerDay: number;
    };
  };
  
  // Allowed external domains (outbound network)
  allowedDomains: string[];
  
  // Max token spend per day (cost control)
  maxDailyTokenSpend: number;
}
```
- Agents cannot escalate beyond authorized scopes
- Rate limits prevent runaway agents
- Domain allowlists prevent data exfiltration
- Token spend caps prevent cost surprises

### 4.5 Dashboard Auth (2FA Mandatory)
- Email + password login (bcrypt hashed)
- **TOTP 2FA required** for all accounts (no SMS-only option)
- Session tokens: HTTP-only, Secure, SameSite=Strict cookies
- CSRF protection on all mutations
- IP-based rate limiting on login (5 attempts / 15 min)
- All deploy/modify/delete operations require active 2FA session

---

## 5. Database Schema (MVP)

```sql
-- Clients (companies/individuals)
CREATE TABLE clients (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        VARCHAR(255) NOT NULL,
  email       VARCHAR(255) UNIQUE NOT NULL,
  plan        VARCHAR(20) DEFAULT 'starter',
  stripe_id   VARCHAR(255),
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Users (dashboard access)
CREATE TABLE users (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_id    UUID NOT NULL REFERENCES clients(id),
  email        VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  totp_secret  VARCHAR(255),        -- encrypted
  totp_enabled BOOLEAN DEFAULT FALSE,
  role         VARCHAR(20) DEFAULT 'member', -- admin, member
  created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Agents (deployed OpenClaw instances)
CREATE TABLE agents (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_id    UUID NOT NULL REFERENCES clients(id),
  name         VARCHAR(255) NOT NULL,
  status       VARCHAR(20) DEFAULT 'provisioning', -- provisioning, running, stopped, error
  container_id VARCHAR(255),
  config       JSONB,               -- OpenClaw config (encrypted sensitive fields)
  permissions  JSONB NOT NULL,       -- AgentPermissions object
  host         VARCHAR(255),         -- server/node the container runs on
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  updated_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Encrypted credentials per agent
CREATE TABLE agent_credentials (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id    UUID NOT NULL REFERENCES agents(id),
  name        VARCHAR(100) NOT NULL, -- 'openai_key', 'gmail_oauth', etc.
  encrypted   JSONB NOT NULL,        -- { iv, ciphertext, tag, keyVersion }
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(agent_id, name)
);

-- Audit logs (append-only)
CREATE TABLE audit_logs (
  id          BIGSERIAL PRIMARY KEY,
  agent_id    UUID NOT NULL REFERENCES agents(id),
  client_id   UUID NOT NULL REFERENCES clients(id),
  action_type VARCHAR(50) NOT NULL,
  target      TEXT,
  metadata    JSONB,
  ip_address  INET,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_agents_client ON agents(client_id);
CREATE INDEX idx_audit_agent ON audit_logs(agent_id, created_at DESC);
CREATE INDEX idx_audit_client ON audit_logs(client_id, created_at DESC);
CREATE INDEX idx_audit_type ON audit_logs(action_type, created_at DESC);
```

---

## 6. API Routes (MVP)

```
Auth:
  POST   /api/auth/register        → Create account
  POST   /api/auth/login            → Login (returns session)
  POST   /api/auth/2fa/setup        → Generate TOTP secret + QR
  POST   /api/auth/2fa/verify       → Verify TOTP code
  POST   /api/auth/logout           → Destroy session

Dashboard:
  GET    /api/agents                 → List client's agents
  POST   /api/agents                 → Deploy new agent (requires 2FA)
  GET    /api/agents/:id             → Agent details + status
  PATCH  /api/agents/:id             → Update config (requires 2FA)
  DELETE /api/agents/:id             → Stop + remove agent (requires 2FA)
  POST   /api/agents/:id/restart     → Restart agent

Credentials:
  GET    /api/agents/:id/credentials       → List credential names (no values)
  POST   /api/agents/:id/credentials       → Add encrypted credential
  DELETE /api/agents/:id/credentials/:name → Remove credential

Monitoring:
  GET    /api/agents/:id/health      → Health check
  GET    /api/agents/:id/logs        → Recent audit logs
  GET    /api/agents/:id/metrics     → CPU/RAM/token usage

Billing:
  GET    /api/billing                → Current plan + usage
  POST   /api/billing/checkout       → Stripe checkout session
  POST   /api/billing/webhook        → Stripe webhook handler
```

---

## 7. Deployment Plan

### Phase 1: MVP (Deploy Morning Feb 17) ✅
**What ships:**
- Landing page (existing, polished)
- Waitlist/contact form (Tally or custom)
- Architecture doc (this document) for investor/client conversations

### Phase 2: Dashboard Alpha (Week 1-2)
- Next.js dashboard with auth + 2FA
- Agent CRUD (provision/start/stop)
- Credential vault
- Basic audit log viewer
- Deploy on Hetzner CX22

### Phase 3: Container Orchestration (Week 3-4)
- Docker-based agent provisioning
- Automated OpenClaw install per container
- Health monitoring + auto-restart
- Stripe billing integration

### Phase 4: Production Hardening (Week 5-6)
- Network isolation policies
- Rate limiting + abuse prevention
- Backup + disaster recovery
- Documentation + onboarding flow
- Security audit

### Phase 5: Scale (Month 2+)
- Multi-region deployment
- Kubernetes migration (if demand warrants)
- API for programmatic agent management
- White-label option for agencies

---

## 8. Hosting & Cost Model

### Infrastructure (per server)
| Resource | Spec | Cost |
|----------|------|------|
| Hetzner CX22 | 2 vCPU, 4GB RAM, 40GB | $5.40/mo |
| Hetzner CX32 | 4 vCPU, 8GB RAM, 80GB | $9.60/mo |
| PostgreSQL (Neon) | Free tier → $19/mo | $0-19/mo |
| Domain + SSL | deployclaw.com | ~$12/yr |

### Agent Density
- **PicoClaw (Go)**: ~10MB RAM → 300+ agents per CX22
- **OpenClaw (Node)**: ~150-300MB RAM → 10-20 agents per CX22
- **Hybrid**: PicoClaw for lightweight, OpenClaw for full-featured

### Unit Economics
| Plan | Price | Infra Cost | Gross Margin |
|------|-------|------------|-------------|
| Starter (1 agent) | $1,248 setup | ~$3 server | 99%+ (labor is the cost) |
| Care Standard | $1,960/mo | ~$15/mo infra | ~99% (support labor) |
| Care Plus (5 agents) | $3,900/mo | ~$30/mo infra | ~99% |
| Enterprise | $7,800+/mo | Custom | 85-95% |

---

## 9. Competitive Positioning

| Feature | DeployClaw | SimpleClaw | DIY |
|---------|-----------|------------|-----|
| One-click deploy | ✅ (managed) | ✅ | ❌ |
| Security hardening | ✅ Built-in | ❌ | Manual |
| Isolated containers | ✅ | ❌ | Manual |
| Encrypted credential vault | ✅ | ❌ | Manual |
| Audit logs | ✅ | ❌ | Manual |
| 2FA dashboard | ✅ | ❌ | N/A |
| Multi-agent orchestration | ✅ | ❌ | Manual |
| Managed monitoring | ✅ | ❌ | Manual |
| Price | 20% below market | Market rate | Free (+ time) |
| Enterprise-ready | ✅ Day 1 | ❌ | Depends |

---

## 10. Morning Deploy Checklist (Feb 17)

- [ ] Register `deployclaw.com` domain (or confirm available)
- [ ] Deploy landing page to Railway/Vercel/Netlify
- [ ] Add Tally waitlist form (or Cal.com booking link) to "Book a Free Call" buttons
- [ ] Set up Google Analytics or Plausible
- [ ] Create `deployclaw` GitHub org + repo
- [ ] Push this architecture doc to repo
- [ ] Share link with CrypWalk for review
- [ ] Post on Moltbook / social media

---

*Built by Automus Prime. Security is the moat.*
