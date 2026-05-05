<div align="center">
  <h1>Open Service Protocol (OSP)</h1>
  <p><strong>Service discovery, provisioning, credentials, and lifecycle management for AI agents.</strong></p>
  <p><em>What MCP is to tool access, OSP is to service provisioning.</em></p>
  <p>
    <a href="https://osp.sh">Website</a> ·
    <a href="spec/osp-v1.0.md">Specification</a> ·
    <a href="schemas/">JSON Schemas</a> ·
    <a href="docs/getting-started.md">Getting Started</a> ·
    <a href="skills/">Provider Skills</a>
  </p>
  <p>
    <img src="https://img.shields.io/badge/spec-v1.1--draft-blue" alt="Spec Version">
    <img src="https://img.shields.io/badge/license-Apache%202.0-green" alt="License">
    <img src="https://img.shields.io/badge/status-active%20development-orange" alt="Status">
    <img src="https://img.shields.io/badge/SDKs-Rust%20%7C%20TypeScript%20%7C%20Python%20%7C%20Go-purple" alt="SDKs">
  </p>
</div>

---

## Why OSP Exists

AI agents are becoming autonomous operators of developer infrastructure. They need to spin up databases, deploy apps, configure auth, rotate credentials, and tear down resources without turning every provider integration into a one-off browser workflow or brittle API wrapper.

OSP defines that lifecycle as a protocol: provider discovery through `.well-known` manifests, provision requests, encrypted credential delivery, rotation, deprovisioning, events, conformance, and payment-rail-neutral settlement.

Current implementation: Rust core crates, TypeScript/Python/Go SDK surfaces, JSON Schemas, provider skills, conformance tests, an MCP server, and a static website.

Today, every service still has its own proprietary onboarding flow:

- **Stripe Projects** (launched March 2026) locks providers into Stripe's invite-only, single-rail ecosystem
- **Manual signup flows** force agents to open browsers, solve CAPTCHAs, verify emails — things agents can't do
- **Ad-hoc API wrapping** means every agent↔provider integration is custom, fragile, and non-standard

There is no open standard that lets an agent ask *"what services are available?"*, provision one, pay through any rail, receive encrypted credentials, and manage the lifecycle. **OSP fills this gap.**

### The Core Thesis

> Reusable credentials are the wrong primitive. Every service interaction should be a first-class protocol operation — discoverable, provisionable, rotatable, and auditable.

OSP treats service provisioning as a protocol problem, not a platform problem. Any provider can implement it. Any agent can consume it. Any payment rail can settle it.

## How It Works

```
Agent                          Provider (e.g. Supabase)
  |                                  |
  |  GET /.well-known/osp.json      |
  |--------------------------------->|  1. Discover: agent finds provider's manifest
  |  { offerings, tiers, pricing }   |
  |<---------------------------------|
  |                                  |
  |  POST /osp/v1/provision          |
  |  { offering, tier, public_key }  |  2. Provision: agent requests a resource
  |--------------------------------->|
  |  { resource_id, credentials }    |
  |<---------------------------------|  3. Credentials: Ed25519-encrypted, delivered securely
  |                                  |
  |  POST /osp/v1/rotate/{id}       |
  |--------------------------------->|  4. Manage: rotate, upgrade, deprovision via standard endpoints
  |  { new_credentials }             |
  |<---------------------------------|
```

**Provider publishes** `/.well-known/osp.json`:
```json
{
  "provider_id": "supabase.com",
  "display_name": "Supabase",
  "offerings": [{
    "offering_id": "supabase/postgres",
    "name": "Managed PostgreSQL",
    "category": "database",
    "tiers": [
      { "tier_id": "free", "name": "Free", "price": { "amount": "0", "currency": "USD" } },
      { "tier_id": "pro", "name": "Pro", "price": { "amount": "25", "currency": "USD", "interval": "P1M" } }
    ]
  }],
  "accepted_payment_methods": ["free", "sardis_wallet", "stripe_spt", "x402"]
}
```

**Agent provisions in one call:**
```bash
POST https://api.supabase.com/osp/v1/provision
{
  "offering_id": "supabase/postgres",
  "tier_id": "free",
  "project_name": "my-agent-app",
  "agent_public_key": "base64url_ed25519_public_key"
}
# Returns: resource_id, encrypted credentials, connection strings
```

## OSP vs. The Alternatives

| | Stripe Projects | Manual APIs | OSP |
|---|---|---|---|
| Provider onboarding | Invite-only | N/A | Self-registration via `.well-known` |
| Protocol | Proprietary | Per-provider | Open standard (Apache 2.0) |
| Payment rail | Stripe only | Per-provider | Any (Sardis, Stripe, x402, free) |
| Discovery | CLI catalog | Documentation | `/.well-known/osp.json` + registry |
| Credential security | Proprietary vault | Varies | Ed25519 encrypted bundles |
| Agent delegation | No | No | A2A protocol support |
| Cost controls | No | No | Budget guardrails, cost-in-PR |
| Identity lifecycle | Static keys | Static keys | Short-lived tokens, NHI inventory |

## Spec v1.1 Features

The OSP specification (~9,600 lines) covers the complete lifecycle:

**Core Protocol** — Discovery, provisioning, credential encryption, billing, deprovisioning, webhooks, conformance levels

**v1.1 Additions:**
- **A2A Agent Delegation** — Agent-to-agent delegated provisioning with task lifecycle tracking
- **Non-Human Identity Lifecycle** — Short-lived tokens, NHI inventory, orphan detection, OIDC/SPIFFE federation
- **FinOps Cost-as-Code** — Budget guardrails, cost-in-PR, anomaly detection, environment TTLs
- **Service Dependency Graph** — Impact analysis, health propagation, auto-generated architecture docs
- **Golden Paths + Scorecards** — Service maturity scoring, compliance checklists (SOC2/HIPAA/GDPR), guided remediation
- **Agent Observability** — OpenTelemetry tracing, audit logs, human-in-the-loop gates, cost-per-action tracking
- **MCP Alignment** — Combined `.well-known` discovery, Streamable HTTP transport bridge
- **Progressive Deployment** — Canary provisioning with promote/rollback lifecycle
- **Ephemeral Environments** — PR-triggered envs with TTL and shareable URLs
- **TypeScript Config** — `osp.config.ts` with Pulumi-style programmatic configuration
- **Provider Onboarding SDK** — Become OSP-compatible in hours, not weeks

## Repository Structure

```
osp/
├── spec/                              # Protocol specification (9,600+ lines)
│   └── osp-v1.0.md                   # Core spec document (v1.1 draft)
├── schemas/                           # JSON Schema definitions (draft 2020-12)
│   ├── service-manifest.schema.json
│   ├── provision-request.schema.json
│   ├── provision-response.schema.json
│   ├── credential-bundle.schema.json
│   ├── usage-report.schema.json
│   ├── webhook-event.schema.json
│   └── examples/                      # 14 example manifests (10 providers)
├── osp-core/                          # Rust core implementation
│   └── crates/
│       ├── osp-crypto/                # Ed25519, x25519, canonical JSON
│       ├── osp-manifest/              # Types, fetch, verify, validate, cache
│       ├── osp-vault/                 # Encrypted credential storage
│       ├── osp-cli/                   # CLI with 18 commands
│       ├── osp-provider/              # Adapter trait + 8 provider adapters
│       ├── osp-registry/              # axum server + SQLite registry
│       ├── osp-conformance/           # Test suite + badge generation
│       └── osp-sdk/                   # High-level Rust SDK
├── reference-implementation/
│   ├── typescript/                    # TypeScript SDK (v0.2.0, 139 tests)
│   │   └── src/                       # Client, MCP server, resolver, plugins
│   └── python/                        # Python SDK (v0.2.0, 171 tests)
│       └── src/osp/                   # Client, provider, resolver, integrations
├── osp-sdk-go/                        # Go SDK (142 tests)
├── sardis-integration/                # Sardis payment rail integration
│   └── src/                           # Payment flow, MCP extension, CLI bridge
├── skills/                            # LLM integration skills (10 providers)
├── website/                           # Next.js + Tailwind marketing site
├── .github/workflows/                 # CI/CD (schema validation, tests, deploy)
└── docs/                              # Guides and documentation
    ├── getting-started.md
    ├── for-providers.md
    ├── for-agents.md
    └── stripe-comparison.md
```

## SDKs

| Language | Package | Tests | Status |
|----------|---------|-------|--------|
| **Rust** | `osp-core` (8 crates) | 46 | Core implementation |
| **TypeScript** | `@osp/client`, `@osp/mcp-server` | 139 | Production-grade with MCP tools |
| **Python** | `osp-client` | 171 | Async client + FastAPI/Django integrations |
| **Go** | `osp-sdk-go` | 142 | Full client + provider + crypto |

### TypeScript Quick Start

```typescript
import { OSPClient } from '@osp/client';

const client = new OSPClient();
const manifest = await client.discover('https://supabase.com');
const result = await client.provision({
  offering_id: 'supabase/postgres',
  tier_id: 'free',
  project_name: 'my-app',
});
console.log(result.credentials_bundle.credentials.SUPABASE_URL);
```

### Python Quick Start

```python
from osp import OSPClient

async with OSPClient() as client:
    manifest = await client.discover("https://supabase.com")
    result = await client.provision(
        offering_id="supabase/postgres",
        tier_id="free",
        project_name="my-app",
    )
    print(result.credentials_bundle.credentials["SUPABASE_URL"])
```

### Go Quick Start

```go
client := osp.NewClient(osp.WithTimeout(30 * time.Second))
manifest, _ := client.Discover(ctx, "https://supabase.com")
result, _ := client.Provision(ctx, osp.ProvisionRequest{
    OfferingID:  "supabase/postgres",
    TierID:      "free",
    ProjectName: "my-app",
})
fmt.Println(result.CredentialsBundle.Credentials["SUPABASE_URL"])
```

## Provider Skills

OSP includes LLM integration skills for 10 providers — structured knowledge files that help AI agents use each service effectively:

Supabase | Neon | Vercel | Clerk | Upstash | Resend | Cloudflare | PostHog | Turso | Railway

Each skill includes: Quick Start, Credentials, Common Operations, Framework Guides, and Gotchas.

## Sardis Integration

OSP is payment-rail agnostic. [Sardis](https://sardis.sh) is the founding maintainer and provides the reference payment integration:

- **Payment Rail** — `sardis_wallet` payment method with escrow lifecycle
- **MCP Extension** — 9 OSP tools for Claude/GPT agents
- **CLI Bridge** — `sardis projects add` → OSP provision + Sardis payment

Other payment rails (Stripe SPT, x402, MPP, invoicing) are equally supported.

## Design Principles

1. **Open.** Apache 2.0. No gatekeeping, no invite lists, no approval queues.
2. **Payment-rail agnostic.** OSP does not privilege any payment system.
3. **Provider-neutral.** Any service can implement OSP. No marketplace cut.
4. **Machine-first.** Designed for agent↔provider interaction, not human UIs.
5. **Secure by default.** Ed25519 signatures, encrypted credentials, short-lived tokens.
6. **Observable.** OpenTelemetry traces, audit logs, cost tracking on every operation.

## Getting Started

- **For Providers**: [How to make your service OSP-compatible](docs/for-providers.md)
- **For Agent Developers**: [How to discover and provision services](docs/for-agents.md)
- **Full Specification**: [OSP v1.1 spec](spec/osp-v1.0.md)
- **Comparison**: [OSP vs Stripe Projects](docs/stripe-comparison.md)

## Contributing

OSP is developed in the open. We welcome contributions:

- **Protocol changes**: Submit a SIP (Service Improvement Proposal) as a GitHub Discussion
- **New schemas**: Submit a PR to `schemas/`
- **Provider adapters**: Submit a PR to `osp-core/crates/osp-provider/`
- **SDK improvements**: Submit a PR to `reference-implementation/` or `osp-sdk-go/`
- **New skills**: Submit a PR to `skills/`

See [CONTRIBUTING.md](CONTRIBUTING.md) for full guidelines.

## Founding Maintainers

- [Sardis](https://sardis.sh) — open-source financial authority layer for AI agents

## License

Apache 2.0 — see [LICENSE](LICENSE)
