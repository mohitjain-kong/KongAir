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

## The Pipeline — 2 GitHub Actions Workflows

### Workflow 1: `ci-kong.yaml`

**Trigger:** Push to `workflow/**` when OAS or Kong config files change

**Purpose:** Convert OAS to Kong config, validate and diff against Konnect, commit `PRD/kong/kong.yaml` back to the workflow branch, and open a PR to `main` with the gateway diff in the PR body for review.

```
Developer edits openapi.yaml or patches.yaml
              ↓
    Push to workflow/finastra-demo
              ↓
     [has-changes] job
  dorny/paths-filter checks if
  relevant files actually changed
              ↓
  [build-validate-stage] job (environment: prd)
  Convert → Patch → Tag → Merge → Lint
              ↓
  deck gateway validate  ← catches incompatible config early
              ↓
  deck gateway diff      ← captures what will change in Konnect
              ↓
  Commits PRD/kong/kong.yaml to workflow branch [skip ci]
              ↓
  Opens/updates PR: workflow/finastra-demo → main
  (gateway diff shown in PR body for reviewer)
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
  -o PRD/kong/kong.yaml \
  "platform-repo-managed"

# Lint the final merged config
deck file lint -s PRD/kong/kong.yaml platform/kong/lint-rulesets.yaml

# Validate config is compatible with Konnect (online check)
deck gateway validate \
  --konnect-control-plane-name "${{ vars.KONNECT_CP_NAME }}" \
  --konnect-token ${{ secrets.KONNECT_PAT }} \
  --konnect-addr "${{ vars.KONNECT_ADDR }}" \
  PRD/kong/kong.yaml

# Diff against Konnect — output captured into PR body
deck gateway diff --select-tag platform-repo-managed \
  --konnect-control-plane-name "${{ vars.KONNECT_CP_NAME }}" \
  --konnect-token ${{ secrets.KONNECT_PAT }} \
  --konnect-addr "${{ vars.KONNECT_ADDR }}" \
  PRD/kong/kong.yaml
```

**Output:**
- `PRD/kong/kong.yaml` committed back to `workflow/finastra-demo` with `[skip ci]`
- PR opened from `workflow/finastra-demo` → `main` with the full gateway diff in the body
- Workflow branch always holds the latest generated state file

---

### Workflow 2: `deploy-kong-PRD.yaml`

**Trigger:** Push to `main` when `PRD/kong/kong.yaml` or any `openapi.yaml` changes (i.e., after Workflow 1's PR merges)

**Purpose:** Apply changes to Konnect and publish API specs to both the Dev Portal and the Konnect Catalog.

```
PR merged to main (workflow/finastra-demo → main)
              ↓
   [deploy-kong] job (environment: prd)
              ↓
   deck gateway sync      ← applies changes to Konnect
              ↓
   Publish specs to Dev Portal (v2 API Products)
              ↓
   Publish specs to Konnect Catalog (v3 APIs)
```

**deck commands:**

```bash
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
    Push to workflow/finastra-demo
              ↓
  ┌─── Workflow 1: ci-kong.yaml ──────────────────────┐
  │  • openapi2kong conversion                        │
  │  • patch → add-tags → merge → lint                │
  │  • deck validate (catches bad config)             │
  │  • deck diff (captures Konnect delta)             │
  │  • Commits PRD/kong/kong.yaml [skip ci]           │
  │  • Opens PR to main with diff in body             │
  └───────────────────────────────────────────────────┘
              ↓ (reviewer sees diff in PR, approves and merges)
  ┌─── Workflow 2: deploy-kong-PRD.yaml ──────────────┐
  │  • deck sync (pushes to Konnect)                  │
  │  • Publish to Dev Portal (v2)                     │
  │  • Publish to Konnect Catalog (v3)                │
  └───────────────────────────────────────────────────┘
```

> **Key design decision:** `workflow/finastra-demo` is the working branch. All changes — source OAS, patches, and the generated `PRD/kong/kong.yaml` — live here. The PR from this branch to `main` is the production gate. `main` always reflects what is deployed.

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

# 2. Find existing version or create new (upsert logic)
GET  /v3/apis/{id}/versions
     → find by: .data[] | select(.version == "1.0.0") | .id

# If version exists → PATCH (update spec content in place)
PATCH /v3/apis/{id}/versions/{version_id}
     body: { "spec": { "content": "<raw-yaml-string>" } }

# If version does not exist → POST (first run only)
POST /v3/apis/{id}/versions
     body: { "spec": { "content": "<raw-yaml-string>" } }

# 3. Publish to the v3 portal
PUT  /v3/apis/{id}/publications/{PORTAL_ID_V3}
     body: { "visibility": "public" }
```

> **Why PATCH instead of POST?** The v3 Catalog treats version strings as unique per API. `POST /versions` with the same version string returns `409 Conflict` on every subsequent run. The fix is to `GET` existing versions, match on the `version` field (not `name`), and `PATCH` the existing entry.

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
| `KONNECT_CP_NAME` | var | `finastra Control Plane` | Control plane name (quote if spaces) |
| `KONNECT_PORTAL_ID` | var | `c35e4220-...` | v2 classic Dev Portal ID |
| `KONNECT_PORTAL_ID_V3` | var | `30ab00aa-...` | v3 Catalog portal ID |
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

### Why `validate` before `diff`?

`deck gateway diff` will fail if the config contains Konnect-incompatible settings (e.g. `storage: kong` on the ACME plugin — only `redis` is supported on Konnect). Running `validate` first gives a clear error message early, before wasting time on a diff or sync attempt.

### Why `[skip ci]` on the generated commit?

The CI workflow commits `PRD/kong/kong.yaml` back to the `workflow/finastra-demo` branch. Without `[skip ci]`, this push would trigger the CI again — causing an infinite loop. The `[skip ci]` tag in the commit message tells GitHub Actions to skip the workflow for that specific push.

---

## Key Issues Encountered and Fixed

| Issue | Root Cause | Fix Applied |
|-------|-----------|-------------|
| `upload-artifact@v3` deprecated | Action version outdated | Updated to `v4` |
| `env.KONNECT_CP_NAME` resolving empty | GitHub Env vars require `vars.` context, not `env.` | Changed all references to `vars.KONNECT_CP_NAME` |
| "finastra Control Plane" breaking CLI args | Spaces in CP name not quoted | Quoted `"${{ vars.KONNECT_CP_NAME }}"` in all deck commands |
| Both old workflows firing simultaneously | Workflow 2 had no path filter — fired on every push to main | Resolved by consolidating to single CI workflow on `workflow/**` only |
| `deck gateway validate` missing | Validate step not in pipeline | Added before `deck diff` in CI workflow |
| ACME `storage: kong` invalid on Konnect | Konnect does not support local Kong storage | Changed to `storage: redis` with `storage_config.redis` config |
| Duplicate API products on each pipeline run | URL filter (`filter[name]=KongAir Flights`) fails with spaces | Fixed with client-side jq: `select(.name == $name)` |
| Wrong portal ID for v3 Catalog | v2 portal ID used in v3 publication call | Introduced `KONNECT_PORTAL_ID_V3` variable for v3 catalog portal |
| Catalog spec not updating — `409 Conflict` on every run | `POST /v3/apis/{id}/versions` fails if version string already exists; also used `.name` field which doesn't exist (correct field is `.version`) | Upsert logic: `GET` versions → `select(.version == $v)` → `PATCH` if exists, `POST` if not |
| OAS `info.title` changes not triggering portal republish | Deploy workflow only watched `PRD/kong/kong.yaml` — title changes don't alter gateway config so pipeline never ran | Added all `openapi.yaml` paths to deploy workflow trigger paths |
| 3-workflow pipeline was complex and confusing | Intermediate staging PRs created extra steps with no clear owner | Consolidated to 2 workflows: CI on `workflow/**` + deploy on `main` |

---

## Demo Narrative

> "A developer changes an OpenAPI spec — maybe a new endpoint, a changed parameter. That single push to the workflow branch triggers Kong config generation, linting, validation against Konnect, and a diff of exactly what will change in the gateway. The diff is embedded directly in the PR body — the approver can review the precise gateway delta before merging. On approval, the config syncs to Konnect. Simultaneously, the updated API specs are published to the **Dev Portal** for external consumers to discover and subscribe, and to the **Konnect Catalog** for internal governance — all from a single Git merge, zero manual steps."

---

## Files to Clean Up Before Demo

- [ ] Delete `delete-products.sh` (temporary test script)
- [ ] Delete `test-catalog.sh` (temporary test script)
- [ ] Verify `KONNECT_PORTAL_ID_V3` is set in GitHub `prd` environment
- [ ] Confirm API products are visible in Dev Portal
- [ ] Confirm APIs are visible in Konnect Catalog
