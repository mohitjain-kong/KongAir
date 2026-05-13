# KongAir APIOps Demo Guide

## Overview

KongAir is a fictional airline used to demonstrate **APIOps** — treating API configuration as code, with Git as the single source of truth. Changes to OpenAPI specs or Kong configurations flow through automated CI/CD pipelines into Kong Konnect with zero manual steps.

---

## Repository Structure

### The 4 Microservices

| Service | Path | Port | Deck Tag |
|---------|------|------|----------|
| Flights API | `flight-data/flights/` | 8080 | `flight-data` |
| Routes API | `flight-data/routes/` | 8081 | `route-data` |
| Bookings API | `sales/bookings/` | 8082 | `sales` |
| Customer API | `sales/customer/` | 8083 | `sales` |

Each service contains:
- `openapi.yaml` — the API contract (source of truth)
- `kong/patches.yaml` — Kong-specific overrides (upstream URLs, plugins, auth)

Additional layers:
- `experience/kong/experience-service.yaml` — BFF/aggregation layer config
- `platform/kong/` — global platform config: consumers, plugins, vaults, ACME

---

## The Pipeline — 3 GitHub Actions Workflows

### Workflow 1: `stage-changes-for-kong.yaml`

**Trigger:** Push to `main` (or `workflow/**`) when OAS or Kong config files change (excludes `PRD/`)

**Purpose:** Convert developer changes into Kong gateway configuration and open a staging PR for review.

```
Developer edits openapi.yaml or patches.yaml
              ↓
         Push to main
              ↓
     [has-changes] job
  dorny/paths-filter checks if
  relevant files actually changed
              ↓
      [oas-break] job
  oasdiff checks for breaking
  API changes → GitHub Issue if found
              ↓
     [oas-to-kong] job
  Convert → Patch → Tag → Merge → Lint
  Opens PR: "Stage Kong Gateway Configuration"
```

**deck commands:**

```bash
# Convert OAS to Kong service/route config
deck file openapi2kong -s flight-data/flights/openapi.yaml

# Apply team-specific patches (upstream URLs, plugins)
deck file patch flight-data/flights/kong/patches.yaml

# Tag services and routes for selective sync
deck file add-tags \
  --selector "$.services[*]" \
  --selector "$.services[*].routes[*]" \
  flight-data \
  -o .github/artifacts/kong/flight-data-flights-kong.yaml

# Merge all team configs into one file
deck file merge .github/artifacts/kong/*-kong.yaml \
  -o .github/artifacts/kong/kong-combined.yaml

# Merge experience layer
deck file merge \
  .github/artifacts/kong/kong-combined.yaml \
  experience/kong/experience-service.yaml \
  -o .github/artifacts/kong/kong-combined.yaml

# Merge platform config (consumers, plugins, vaults) + patch + tag
deck file merge \
  .github/artifacts/kong/kong-combined.yaml \
  platform/kong/platform-kong-base.yaml \
  platform/kong/consumers/* \
  platform/kong/plugins/* \
  platform/kong/vaults/* | \
deck file patch platform/kong/patches.yaml | \
deck file add-tags \
  -o platform/kong/.generated/kong.yaml \
  "platform-repo-managed"

# Lint the final merged config
deck file lint -s platform/kong/.generated/kong.yaml platform/kong/lint-rulesets.yaml
```

**Output:** PR opened with updated `platform/kong/.generated/kong.yaml`

---

### Workflow 2: `stage-kong-for-PRD.yaml`

**Trigger:** Push to `main` when `platform/kong/.generated/kong.yaml` changes (i.e., after Workflow 1's PR merges)

**Purpose:** Validate config against Konnect and show a diff of what will change — gives platform team a gate before production.

```
Staging PR merged to main
              ↓
   [stage-for-prd] job (environment: prd)
              ↓
   Copy generated config to PRD/kong/kong.yaml
              ↓
   deck gateway validate  ← catches incompatible config BEFORE diff
              ↓
   deck gateway diff      ← shows exactly what will change in Konnect
              ↓
   Opens PR: "Stage Kong Gateway Configuration for PRD"
   (targets main, includes diff output for review)
```

**deck commands:**

```bash
# Copy generated config to PRD folder
cp platform/kong/.generated/kong.yaml PRD/kong/kong.yaml

# Validate config is compatible with Konnect (catches issues early)
deck gateway validate \
  --konnect-control-plane-name "${{ vars.KONNECT_CP_NAME }}" \
  --konnect-token ${{ secrets.KONNECT_PAT }} \
  --konnect-addr "${{ vars.KONNECT_ADDR }}" \
  PRD/kong/kong.yaml

# Show what will change in Konnect (dry run — no changes applied)
deck gateway diff --select-tag platform-repo-managed \
  --konnect-control-plane-name "${{ vars.KONNECT_CP_NAME }}" \
  --konnect-token ${{ secrets.KONNECT_PAT }} \
  --konnect-addr "${{ vars.KONNECT_ADDR }}" \
  PRD/kong/kong.yaml
```

**Output:** PR opened with `PRD/kong/kong.yaml` — ops/platform team reviews the diff and approves

---

### Workflow 3: `deploy-kong-PRD.yaml`

**Trigger:** Push to `main` when `PRD/kong/kong.yaml` changes (i.e., after Workflow 2's PRD PR merges)

**Purpose:** Apply changes to Konnect and publish API specs to both the Dev Portal and the Konnect Catalog.

```
PRD PR merged to main
              ↓
   [deploy-kong] job (environment: prd)
              ↓
   deck gateway validate  ← final safety check
              ↓
   deck gateway sync      ← applies changes to Konnect
              ↓
   Publish specs to Dev Portal (v2 API Products)
              ↓
   Publish specs to Konnect Catalog (v3 APIs)
```

**deck commands:**

```bash
# Validate again before touching production
deck gateway validate \
  --konnect-control-plane-name "${{ vars.KONNECT_CP_NAME }}" \
  --konnect-token ${{ secrets.KONNECT_PAT }} \
  --konnect-addr "${{ vars.KONNECT_ADDR }}" \
  PRD/kong/kong.yaml

# Sync ONLY resources tagged platform-repo-managed (selective sync — safe)
deck gateway sync --select-tag platform-repo-managed \
  --konnect-control-plane-name "${{ vars.KONNECT_CP_NAME }}" \
  --konnect-token ${{ secrets.KONNECT_PAT }} \
  --konnect-addr "${{ vars.KONNECT_ADDR }}" \
  PRD/kong/kong.yaml
```

---

## End-to-End Flow

```
Developer edits openapi.yaml or patches.yaml
              ↓
         Push to main
              ↓
  ┌─── Workflow 1: stage-changes-for-kong ───┐
  │  • Breaking change detection (oasdiff)   │
  │  • openapi2kong conversion               │
  │  • patch → add-tags → merge → lint       │
  │  • PR opened for staging review          │
  └──────────────────────────────────────────┘
              ↓ (PR merged)
  ┌─── Workflow 2: stage-kong-for-PRD ───────┐
  │  • deck validate (catches bad config)    │
  │  • deck diff (shows Konnect delta)       │
  │  • PRD PR opened for ops review          │
  └──────────────────────────────────────────┘
              ↓ (PRD PR merged)
  ┌─── Workflow 3: deploy-kong-PRD ──────────┐
  │  • deck validate (final safety net)      │
  │  • deck sync (pushes to Konnect)         │
  │  • Publish to Dev Portal (v2)            │
  │  • Publish to Konnect Catalog (v3)       │
  └──────────────────────────────────────────┘
```

---

## Konnect Publishing — Two Separate Systems

After syncing the gateway config, specs are published to two distinct Konnect systems serving different audiences.

### Dev Portal — Consumer-Facing (v2 API Products)

| Property | Detail |
|----------|--------|
| Purpose | External developers browse, discover, and subscribe to APIs |
| Konnect API | `/v2/api-products` |
| GitHub Variable | `KONNECT_PORTAL_ID` |
| UI Location | Konnect → Dev Portal |
| Audience | External API consumers |
| Spec format | base64 encoded |

**API call sequence:**
```bash
# 1. Find or create the API Product
GET  /v2/api-products
POST /v2/api-products
     body: { "name": "KongAir Flights", "portal_ids": ["<PORTAL_ID>"] }

# 2. Find or create a Product Version
GET  /v2/api-products/{id}/product-versions?filter[name]=1.0.0
POST /v2/api-products/{id}/product-versions
     body: { "name": "1.0.0", "publish_status": "published" }

# 3. Upload or update the OpenAPI spec
GET  /v2/api-products/{id}/product-versions/{vid}/specifications
POST /v2/.../specifications
     body: { "name": "oas.yaml", "content": "<base64-encoded-yaml>" }
PATCH /v2/.../specifications/{sid}   ← if spec already exists
     body: { "name": "oas.yaml", "content": "<base64-encoded-yaml>" }
```

---

### Konnect Catalog — Internal Governance (v3 APIs)

| Property | Detail |
|----------|--------|
| Purpose | Internal API registry for platform teams — governance, discoverability |
| Konnect API | `/v3/apis` |
| GitHub Variable | `KONNECT_PORTAL_ID_V3` |
| UI Location | Konnect → API Catalog |
| Audience | Internal platform/architecture teams |
| Spec format | Raw YAML string (via `jq -Rs '.'`) |

**API call sequence:**
```bash
# 1. Find or create the Catalog API entry
GET  /v3/apis
POST /v3/apis
     body: { "name": "KongAir Flights", "version": "1.0.0", "description": "..." }

# 2. Upload a spec version
POST /v3/apis/{id}/versions
     body: { "spec": { "content": "<raw-yaml-string>" } }

# 3. Publish to the v3 portal
PUT  /v3/apis/{id}/publications/{PORTAL_ID_V3}
     body: { "visibility": "public" }
```

---

### Side-by-Side Comparison

| | Dev Portal (v2) | Konnect Catalog (v3) |
|--|--|--|
| Audience | External consumers | Internal platform teams |
| API base path | `/v2/api-products` | `/v3/apis` |
| Spec format | base64 encoded | Raw YAML string |
| Portal linking | `portal_ids: [...]` on product | `PUT .../publications/{portalId}` |
| UI location | Dev Portal tab | Konnect → API Catalog |
| GitHub variable | `KONNECT_PORTAL_ID` | `KONNECT_PORTAL_ID_V3` |
| Use case | API marketplace / subscriptions | API governance / inventory |

---

## GitHub Environment Variables (`prd` environment)

| Variable | Type | Example Value | Purpose |
|----------|------|---------------|---------|
| `KONNECT_ADDR` | var | `https://eu.api.konghq.com` | Konnect region endpoint |
| `KONNECT_CP_NAME` | var | `FCA Control Plane` | Control plane name (quote if spaces) |
| `KONNECT_PORTAL_ID` | var | `c35e4220-...` | v2 classic Dev Portal ID |
| `KONNECT_PORTAL_ID_V3` | var | `30ab00aa-...` | v3 Catalog portal ID |
| `DEPLOY_TARGET` | var | `KONNECT` | Deployment target (blank defaults to Konnect) |
| `KONNECT_PAT` | **secret** | `kpat_...` | Personal Access Token |

---

## deck Command Reference

| Command | What it does |
|---------|-------------|
| `deck file openapi2kong` | Converts an OpenAPI spec into Kong service/route config |
| `deck file patch` | Applies a patch file to merge Kong-specific config (auth, plugins, upstreams) |
| `deck file add-tags` | Tags services/routes with a label for selective sync |
| `deck file merge` | Combines multiple Kong config files into one |
| `deck file lint` | Validates config against custom ruleset (style, security, naming) |
| `deck gateway validate` | Validates config is compatible with target Konnect CP (online check) |
| `deck gateway diff` | Shows what would change in Konnect without applying anything |
| `deck gateway sync --select-tag` | Applies changes to Konnect, only touching resources with the given tag |

### Why `--select-tag platform-repo-managed`?

Konnect may have resources created manually (via UI or Admin API). Using `--select-tag` ensures deck **only manages resources it owns** — it will not delete or overwrite anything that wasn't created by this pipeline.

### Why `validate` before `diff` and before `sync`?

`deck gateway diff` and `sync` will fail if the config contains Konnect-incompatible settings (e.g. `storage: kong` on the ACME plugin — only `redis` is supported on Konnect). Running `validate` first gives a clear error message early, before wasting time on a sync attempt.

---

## Key Issues Encountered and Fixed

| Issue | Root Cause | Fix Applied |
|-------|-----------|-------------|
| `upload-artifact@v3` deprecated | Action version outdated | Updated to `v4` |
| `env.KONNECT_CP_NAME` resolving empty | GitHub Env vars require `vars.` context, not `env.` | Changed all references to `vars.KONNECT_CP_NAME` |
| "FCA Control Plane" breaking CLI args | Spaces in CP name not quoted | Quoted `"${{ vars.KONNECT_CP_NAME }}"` in all deck commands |
| Both Workflow 1 and 2 firing simultaneously | Workflow 2 had no path filter — fired on every push to main | Added `paths: platform/kong/.generated/kong.yaml` to Workflow 2 |
| PRD PR targeting wrong branch | `create-pull-request` defaulted to current branch as base | Added `base: main` to the create-pull-request step |
| `deck gateway validate` missing | Validate step not in pipeline | Added before `deck diff` (Workflow 2) and before `deck sync` (Workflow 3) |
| ACME `storage: kong` invalid on Konnect | Konnect does not support local Kong storage | Changed to `storage: redis` with `storage_config.redis` config |
| Duplicate API products on each pipeline run | URL filter (`filter[name]=KongAir Flights`) fails with spaces | Fixed with client-side jq: `select(.name == $name)` |
| Wrong portal ID for v3 Catalog | v2 portal ID used in v3 publication call | Introduced `KONNECT_PORTAL_ID_V3` variable for v3 catalog portal |

---

## Demo Narrative

> "A developer changes an OpenAPI spec — maybe a new endpoint, a changed parameter. That single Git push triggers automated breaking-change detection, Kong config generation, linting, and a staging PR. A platform engineer reviews the `deck diff` output showing exactly what will change in the gateway — no surprises, no manual config. On approval, the config syncs to Konnect. Simultaneously, the updated API specs are published to the **Dev Portal** for external consumers to discover and subscribe, and to the **Konnect Catalog** for internal governance — all from a single Git merge, zero manual steps."

---

## Files to Clean Up Before Demo

- [ ] Delete `delete-products.sh` (temporary test script)
- [ ] Delete `test-catalog.sh` (temporary test script)
- [ ] Verify `KONNECT_PORTAL_ID_V3` is set in GitHub `prd` environment
- [ ] Confirm API products are visible in Dev Portal
- [ ] Confirm APIs are visible in Konnect Catalog
