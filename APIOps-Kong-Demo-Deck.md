# APIOps w/ Kong
### KongAir Demo

---

## The Challenge

- Service interfaces are as critical as the code itself
- Exposing interfaces (APIs) directly to clients creates operational risks and development burden on engineering
- API Gateways provide flexible facades to your services and provide key layers for security, control, and observability

**The challenge: Deliver functional applications and their APIs to production consistently, safely, and reliably.**

---

## What is APIOps?

> APIOps is the process of applying principles of DevOps and GitOps to API and microservice delivery lifecycles.

**Git as the single source of truth — for both gateway config and API documentation.**

---

## API First

- API specifications are the source of truth for service behaviors and documentation
- API specifications can also drive API Gateway runtime behavior
- We use Kong's declarative management tool, **decK**, to bridge these concepts

```
OpenAPI Spec  ──▶  decK  ──▶  Kong Gateway Config
```

> https://docs.konghq.com/deck/latest/

---

## Extensions

- Kong policies can be enabled by providing extensions within the OpenAPI specification directly
- Kong defaults can be modified and plugins can be configured at global and operation levels

```yaml
# openapi.yaml
paths:
  /flights:
    get:
      x-kong-plugin-rate-limiting:
        config:
          minute: 100
```

---

## Transformations

- OpenAPI Specifications describe the service contract — key for the client
- OpenAPI Specifications and Kong Gateway configurations are not fully compatible
- Gateway configuration may require aspects not applicable to the OAS specification or relevant to the client
- **Patch files** allow additions or modifications to generated configuration

---

## Patch Transformation Example

Override the service hostname with the internally routable host:

```yaml
# kong/patches.yaml
- selectors:
    - "$.services[*]"
  values:
    host: flights.internal.svc.cluster.local
    port: 8080
    protocol: http
```

```bash
deck file openapi2kong -s openapi.yaml | \
  deck file patch kong/patches.yaml | \
  deck file add-tags --selector "$.services[*]" flight-data
```

---

## Federated APIOps

- Engineering organizations with non-trivial team structures or many services may require more advanced operational tools
- **Federated APIOps** means empowering development teams while maintaining governance capabilities
- APIOps workflows can range from simple, to very complex multi-repository pipelines
- With Kong, this can be done with a declarative approach using APIOps commands built directly into decK

---

## go-apiops

> https://github.com/Kong/go-apiops

- Open Source GoLang library that houses the APIOps commands built into deck
- Contains documentation and example OAS files

**Key deck file commands:**

| Command | Purpose |
|---------|---------|
| `deck file openapi2kong` | Convert OAS → Kong config |
| `deck file patch` | Apply team overrides |
| `deck file add-tags` | Tag for selective sync |
| `deck file merge` | Combine multiple configs |
| `deck file lint` | Validate against rulesets |

---

## KongAir

> https://github.com/kong/kongair

- Simplified example repository that mimics a micro-service based organization wishing to use federated APIOps
- Multiple services based on OpenAPI specifications that drive code generation, gateway configuration, and a developer portal
- CI/CD pipelines using decK to automate API delivery to Kong Konnect

**4 microservices: Flights · Routes · Bookings · Customer**

---

## KongAir — Repository Structure

```
KongAir/
├── flight-data/
│   ├── flights/
│   │   ├── openapi.yaml        ← API contract (source of truth)
│   │   └── kong/patches.yaml   ← Kong-specific overrides
│   └── routes/
│       ├── openapi.yaml
│       └── kong/patches.yaml
├── sales/
│   ├── bookings/
│   └── customer/
├── experience/kong/            ← BFF aggregation layer
├── platform/kong/              ← Global: consumers, plugins, vaults
└── PRD/kong/kong.yaml          ← Generated: deployed to Konnect
```

---

## The APIOps Pipeline — 2 Workflows

```
Developer edits openapi.yaml
         ↓
  Push to workflow branch
         ↓
┌────────────────────────────────────────┐
│        ci-kong.yaml                    │
│                                        │
│  openapi2kong → patch → add-tags       │
│       → merge → lint                   │
│         ↓                              │
│  deck gateway validate  ← online check │
│         ↓                              │
│  deck gateway diff  ← captures delta   │
│         ↓                              │
│  Commit PRD/kong/kong.yaml [skip ci]   │
│         ↓                              │
│  Open PR → main (diff in PR body)      │
└────────────────────────────────────────┘
         ↓  Reviewer approves & merges
┌────────────────────────────────────────┐
│        deploy-kong-PRD.yaml            │
│                                        │
│  deck gateway sync                     │
│  Publish → Dev Portal (v2)             │
│  Publish → Konnect Catalog (v3)        │
└────────────────────────────────────────┘
```

---

## Pipeline Step 1 — Build & Validate

Each team owns their OAS + patches. The pipeline assembles everything:

```bash
# Each team: OAS → Kong config
deck file openapi2kong -s flight-data/flights/openapi.yaml | \
  deck file patch flight-data/flights/kong/patches.yaml | \
  deck file add-tags --selector "$.services[*]" flight-data

# Platform team: merge all + add global config + lint
deck file merge .github/artifacts/kong/*-kong.yaml | \
  deck file merge platform/kong/plugins/* platform/kong/vaults/* | \
  deck file patch platform/kong/patches.yaml | \
  deck file add-tags "platform-repo-managed" \
  -o PRD/kong/kong.yaml

deck file lint -s PRD/kong/kong.yaml platform/kong/lint-rulesets.yaml
```

---

## Pipeline Step 2 — Gate Before Production

```bash
# Validate config is Konnect-compatible (online check)
# Catches issues like unsupported storage backends, missing fields
deck gateway validate \
  --konnect-control-plane-name "FCA Control Plane" \
  --konnect-token $KONNECT_PAT \
  PRD/kong/kong.yaml

# Show exactly what will change — no changes applied
deck gateway diff --select-tag platform-repo-managed \
  --konnect-control-plane-name "FCA Control Plane" \
  --konnect-token $KONNECT_PAT \
  PRD/kong/kong.yaml
```

The **diff is embedded in the PR body** — the approver sees the exact gateway delta before clicking merge.

---

## Pipeline Step 2 — PR with Diff

```diff
Merging this PR will apply the following changes to Kong Gateway in PRD:

updating service flights-service  {
   "tags": [
-    "flight-data1",       ← old tag (typo)
+    "flight-data",        ← corrected
     "platform-repo-managed"
   ]
}

Summary:
  Created: 0
  Updated: 3
  Deleted: 0
```

**No surprises. Full visibility before production.**

---

## Pipeline Step 3 — Sync to Konnect

```bash
# Selective sync — only touches resources this pipeline owns
deck gateway sync --select-tag platform-repo-managed \
  --konnect-control-plane-name "FCA Control Plane" \
  --konnect-token $KONNECT_PAT \
  PRD/kong/kong.yaml

# Summary:
#   Created: 0
#   Updated: 3
#   Deleted: 0
```

`--select-tag` ensures deck **never touches manually created resources** in Konnect — it only manages what it owns.

---

## Beyond Gateway Config — API Publishing

After syncing the gateway, the same pipeline publishes specs to **two separate Konnect systems**:

```
Merge to main
      ↓
deck gateway sync     → Kong Gateway routes & plugins updated
      ↓
Publish to Dev Portal → External consumers browse & subscribe
      ↓
Publish to Catalog    → Internal governance & discoverability
```

One merge. Three outcomes.

---

## Dev Portal — Consumer-Facing (v2 API Products)

**Who:** External developers browsing and subscribing to APIs

```
Konnect → Dev Portal → KongAir Flights → 1.0.0 → Spec
```

**How the pipeline publishes:**

```bash
# Find or create API Product (linked to portal)
POST /v2/api-products
     { "name": "KongAir Flights", "portal_ids": ["<PORTAL_ID>"] }

# Find or create Product Version
POST /v2/api-products/{id}/product-versions
     { "name": "1.0.0", "publish_status": "published" }

# Upload or update spec (base64 encoded)
POST /v2/.../specifications
     { "name": "oas.yaml", "content": "<base64>" }
PATCH /v2/.../specifications/{id}    ← idempotent on re-runs
```

**Result:** Developers see endpoints, schemas, and can register applications.

---

## Konnect Catalog — Internal Governance (v3 APIs)

**Who:** Platform and architecture teams — internal API registry

```
Konnect → API Catalog → KongAir Flights → Version history → Spec
```

**How the pipeline publishes:**

```bash
# Find or create Catalog API entry
POST /v3/apis
     { "name": "KongAir Flights", "version": "1.0.0" }

# Upsert spec version (PATCH if exists, POST if new)
GET  /v3/apis/{id}/versions
     → select(.version == "1.0.0") | .id   ← find existing

PATCH /v3/apis/{id}/versions/{version_id}
     { "spec": { "content": "<raw-yaml>" } }

# Publish to internal portal
PUT  /v3/apis/{id}/publications/{PORTAL_ID_V3}
     { "visibility": "public" }
```

> **Key lesson:** v3 Catalog version strings are unique — always PATCH, never POST twice.

---

## Two Publishing Systems — Side by Side

| | Dev Portal (v2) | Konnect Catalog (v3) |
|--|--|--|
| **Audience** | External consumers | Internal platform teams |
| **Purpose** | API marketplace | API governance |
| **Spec format** | base64 encoded | Raw YAML string |
| **Portal link** | `portal_ids` on product create | `PUT .../publications/{portalId}` |
| **UI location** | Konnect → Dev Portal | Konnect → API Catalog |
| **GitHub var** | `KONNECT_PORTAL_ID` | `KONNECT_PORTAL_ID_V3` |

---

## End-to-End Demo Flow

| Step | Action | Result |
|------|--------|--------|
| 1 | Edit `openapi.yaml` | Source of truth updated |
| 2 | Push to `workflow/fca-demo` | CI triggers automatically |
| 3 | CI: convert + lint + validate | Config verified before touching prod |
| 4 | CI: `deck diff` | Exact gateway changes captured |
| 5 | CI: commit + open PR | Diff visible in PR body for review |
| 6 | Reviewer merges PR | Production gate passed |
| 7 | Deploy: `deck sync` | Gateway updated in Konnect |
| 8 | Deploy: publish specs | Dev Portal + Catalog both updated |

**One spec change. Two PRs. Zero manual steps after merge.**

---

## What APIOps with Kong Gives You

| Without APIOps | With APIOps + Kong |
|---|---|
| Manual gateway config | Generated from OAS spec |
| Config drift between teams | Single merged config, one source of truth |
| No visibility before deploy | `deck diff` in every PR body |
| Separate portal updates | Auto-published on every merge |
| Silent failures | Validate catches issues before diff/sync |
| Risk of touching wrong resources | `--select-tag` isolates pipeline-owned resources |

---

## Key decK Commands Used

```bash
deck file openapi2kong   # OAS → Kong services/routes
deck file patch          # Apply team overrides
deck file add-tags       # Tag for selective sync
deck file merge          # Combine multi-team configs
deck file lint           # Validate against custom rulesets

deck gateway validate    # Online: catch Konnect-incompatible config
deck gateway diff        # Show delta without applying
deck gateway sync        # Apply changes (tagged resources only)
```

---

## Thank You

**KongAir APIOps Demo**

- Repository: `github.com/mohitjain-kong/KongAir`
- decK docs: `docs.konghq.com/deck/latest/`
- Konnect: `cloud.konghq.com`

> *"A developer changes a spec. A reviewer sees the exact gateway diff. One merge deploys the config, publishes to the Dev Portal, and registers in the Catalog. Zero manual steps."*
