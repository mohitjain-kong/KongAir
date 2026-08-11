# decK Tagging — Version Control, Governance & Production Rollback

## A Reference Whitepaper for Kong APIOps Practitioners

---

## Table of Contents

1. [What Are Tags in decK?](#1-what-are-tags-in-deck)
2. [How Tags Work Internally](#2-how-tags-work-internally)
3. [The Core Patterns](#3-the-core-patterns)
   - 3.1 [Ownership Tags](#31-ownership-tags--platform-repo-managed)
   - 3.2 [Version Tags](#32-version-tags--version010)
   - 3.3 [Environment Tags](#33-environment-tags--envproduction)
   - 3.4 [Domain / Team Tags](#34-domain--team-tags)
4. [Selective Sync — The Safety Mechanism](#4-selective-sync--the-safety-mechanism)
5. [Version Control with Git + Tags](#5-version-control-with-git--tags)
6. [Production Rollback Playbook](#6-production-rollback-playbook)
7. [Multi-Stage Pipeline Promotion](#7-multi-stage-pipeline-promotion)
8. [Multi-Team in One Control Plane](#8-multi-team-in-one-control-plane)
9. [Audit and Observability](#9-audit-and-observability)
10. [Tag Governance — Naming Conventions](#10-tag-governance--naming-conventions)
11. [KongAir Reference Implementation](#11-kongair-reference-implementation)
12. [Command Reference](#12-command-reference)
13. [Common Mistakes and How to Avoid Them](#13-common-mistakes-and-how-to-avoid-them)

---

## 1. What Are Tags in decK?

Tags in Kong Gateway are **arbitrary string labels** that can be attached to almost any resource — services, routes, plugins, consumers, upstreams, certificates, and vaults.

They are:
- **Stored in Konnect** alongside the resource (not just local metadata)
- **Queryable** — you can filter, dump, and sync by tag
- **Composable** — a resource can have multiple tags simultaneously
- **The foundation of safe, automated GitOps pipelines**

Tags appear in your Kong state file like this:

```yaml
services:
  - name: flights-service
    host: flights.internal
    port: 8080
    tags:
      - flight-data
      - version:0.1.0
      - env:production
      - platform-repo-managed
    routes:
      - name: get-flights
        paths:
          - /flights
        tags:
          - flight-data
          - version:0.1.0
          - env:production
          - platform-repo-managed
```

---

## 2. How Tags Work Internally

When decK syncs a state file to Konnect, it compares the **desired state** (your YAML file) against the **actual state** (what's in Konnect). Without any tag scoping, decK treats itself as the owner of **everything** in the control plane — resources in Konnect that are not in your file will be deleted.

Tags change this contract:

```
deck gateway sync --select-tag <tag> kong.yaml
```

With `--select-tag`, decK:

1. Fetches only resources from Konnect that have `<tag>` attached
2. Compares those resources against your state file
3. Creates / updates / deletes **only within that tagged set**
4. Leaves everything else in Konnect completely untouched

This makes `--select-tag` the single most important safety feature in decK for production use.

---

## 3. The Core Patterns

### 3.1 Ownership Tags — `platform-repo-managed`

**Purpose:** Declare that a resource is owned and managed exclusively by this Git pipeline. Anything with this tag will be reconciled on every sync; anything without it is invisible to decK.

```yaml
# Applied at the final merge step in CI
deck file add-tags \
  -o PRD/kong/kong.yaml \
  "platform-repo-managed"
```

```bash
# Sync: only touch resources this pipeline owns
deck gateway sync --select-tag platform-repo-managed PRD/kong/kong.yaml

# Diff: only show changes for pipeline-owned resources
deck gateway diff --select-tag platform-repo-managed PRD/kong/kong.yaml
```

**What it protects against:**
- A manually created route in the Konnect UI getting deleted because it's not in your YAML
- Another team's resources getting overwritten by your sync
- Accidental full-wipe when a new engineer runs `deck gateway sync` without understanding the environment

**Rule:** Every resource managed by an automated pipeline should carry an ownership tag. No exceptions.

---

### 3.2 Version Tags — `version:0.1.0`

**Purpose:** Record the exact API version that is deployed to the gateway for every service and route. The version is sourced directly from the OpenAPI spec's `info.version` field, so the tag is always in sync with the contract.

```bash
# In CI — extract version from OAS, apply as tag
VERSION=$(yq '.info.version' flight-data/flights/openapi.yaml)

deck file openapi2kong -s flight-data/flights/openapi.yaml | \
  deck file patch flight-data/flights/kong/patches.yaml | \
  deck file add-tags \
    --selector "$.services[*]" \
    --selector "$.services[*].routes[*]" \
    flight-data "version:$VERSION" \
    -o .github/artifacts/kong/flight-data-flights-kong.yaml
```

**What you can do with version tags:**

```bash
# Dump everything currently tagged as v0.1.0 — what's live?
deck gateway dump --select-tag version:0.1.0

# Confirm which version of the Flights API is in Konnect right now
deck gateway dump --select-tag flight-data | yq '.services[].tags'
# → ["flight-data", "version:0.1.0", "env:production", "platform-repo-managed"]

# Find all services NOT yet on v0.2.0 (still on old version)
deck gateway dump | yq '.services[] | select(.tags[] | test("version:") | not)'
```

**Why this matters for rollback:** When you need to roll back from `version:0.2.0` to `version:0.1.0`, the Git history of `PRD/kong/kong.yaml` is your source of truth. The version tag on the live resource tells you what's deployed. Together they give you a complete audit trail without looking at Konnect logs.

---

### 3.3 Environment Tags — `env:production`

**Purpose:** Declare which environment a resource belongs to. In a multi-stage pipeline this is how you distinguish staging config from production config in the same repository.

```bash
# Staging pipeline
deck file add-tags "platform-repo-managed" "env:staging" \
  -o staging/kong/kong.yaml

# Production pipeline
deck file add-tags "platform-repo-managed" "env:production" \
  -o PRD/kong/kong.yaml
```

**Multi-stage promotion flow:**

```
openapi.yaml edited
       ↓
 CI runs on workflow/**
       ↓
 Generates staging/kong/kong.yaml  → syncs to STAGING CP (env:staging)
       ↓
 PR review + approval
       ↓
 Generates PRD/kong/kong.yaml      → syncs to PRD CP (env:production)
```

**Querying by environment:**

```bash
# What's the difference between staging and prod right now?
deck gateway dump --konnect-control-plane-name "Staging CP" \
  --select-tag env:staging > staging-current.yaml

deck gateway dump --konnect-control-plane-name "PRD CP" \
  --select-tag env:production > prod-current.yaml

diff staging-current.yaml prod-current.yaml
```

---

### 3.4 Domain / Team Tags

**Purpose:** Identify which team or domain owns a resource. Enables independent team deployments into the same control plane.

```yaml
# Flights team resources
tags:
  - flight-data      ← domain tag
  - version:0.1.0
  - env:production
  - platform-repo-managed

# Sales team resources
tags:
  - sales            ← domain tag
  - version:1.3.0
  - env:production
  - platform-repo-managed
```

```bash
# Flights team syncs independently — won't touch Sales resources
deck gateway sync --select-tag flight-data flights-kong.yaml

# Sales team syncs independently — won't touch Flights resources
deck gateway sync --select-tag sales sales-kong.yaml
```

---

## 4. Selective Sync — The Safety Mechanism

This is the single most important concept when running decK in production.

### Without `--select-tag` (dangerous)

```bash
deck gateway sync PRD/kong/kong.yaml
```

decK fetches **all** resources from Konnect and reconciles against your file. Resources in Konnect that are not in your file are **deleted**. This is correct only when one pipeline owns 100% of the control plane.

### With `--select-tag` (safe for shared environments)

```bash
deck gateway sync --select-tag platform-repo-managed PRD/kong/kong.yaml
```

decK fetches only resources tagged `platform-repo-managed`. Everything else is invisible. Only resources within that scope are created, updated, or deleted.

### Combining multiple tags (AND logic)

```bash
# Only sync resources that have BOTH tags
deck gateway sync \
  --select-tag platform-repo-managed \
  --select-tag env:production \
  PRD/kong/kong.yaml
```

### Visual illustration

```
Konnect Control Plane
┌─────────────────────────────────────────────────────┐
│                                                     │
│  ┌──── platform-repo-managed ──────────────────┐   │
│  │  flights-service  (flight-data, v:0.1.0)    │   │
│  │  bookings-service (sales, v:1.3.0)          │   │
│  │  global-rate-limit plugin                   │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  manually-created-route  ← NOT tagged, invisible    │
│  ui-consumer             ← NOT tagged, invisible    │
│                                                     │
└─────────────────────────────────────────────────────┘

deck gateway sync --select-tag platform-repo-managed kong.yaml
→ Touches ONLY the resources inside the box
→ manually-created-route and ui-consumer are never touched
```

---

## 5. Version Control with Git + Tags

Tags and Git together create a complete version control system for your gateway configuration.

### The State File Pattern

Every time the CI pipeline runs, it:
1. Generates `PRD/kong/kong.yaml` from source OAS + patches
2. Commits it back to the workflow branch with `[skip ci]`
3. This file is the **exact state of what was synced** — including all tags

```
Git History of PRD/kong/kong.yaml
─────────────────────────────────
commit 931bbc0  chore: update PRD/kong/kong.yaml [skip ci]
                → services tagged: version:0.2.0, env:production

commit 33cf248  chore: update PRD/kong/kong.yaml [skip ci]
                → services tagged: version:0.1.0, env:production

commit 5a6e288  chore: update PRD/kong/kong.yaml [skip ci]
                → services tagged: version:0.1.0, env:production
```

**At any point in time you can answer:**
- What was deployed on a specific date? → `git show <sha>:PRD/kong/kong.yaml`
- What changed between two deploys? → `git diff <sha1> <sha2> -- PRD/kong/kong.yaml`
- Who approved the change that broke production? → `git log --follow PRD/kong/kong.yaml`

### Linking OAS versions to gateway state

Because the version tag is extracted from `openapi.yaml` at build time, the Git history of both files tells the full story:

```bash
# When was v0.2.0 of Flights first deployed?
git log --all --oneline -- flight-data/flights/openapi.yaml
# 8bc4e22  bump Flights API to v0.2.0   ← spec change
# 931bbc0  chore: update PRD/kong/kong.yaml [skip ci]  ← gateway updated
```

---

## 6. Production Rollback Playbook

### Scenario: v0.2.0 deployed and broke production

**Time budget: under 5 minutes.**

#### Step 1 — Identify the last known-good state file

```bash
git log --oneline PRD/kong/kong.yaml
# 931bbc0  chore: update PRD/kong/kong.yaml [skip ci]   ← CURRENT (v0.2.0, broken)
# 33cf248  chore: update PRD/kong/kong.yaml [skip ci]   ← PREVIOUS (v0.1.0, good)
# 5a6e288  chore: update PRD/kong/kong.yaml [skip ci]   ← older
```

#### Step 2 — Extract the good state file

```bash
git show 33cf248:PRD/kong/kong.yaml > rollback.yaml
```

#### Step 3 — Preview what will change (optional but recommended)

```bash
deck gateway diff \
  --select-tag platform-repo-managed \
  --konnect-control-plane-name "PRD CP" \
  --konnect-token $KONNECT_PAT \
  --konnect-addr $KONNECT_ADDR \
  rollback.yaml
```

You should see changes from v0.2.0 → v0.1.0. Verify the diff looks right before applying.

#### Step 4 — Apply the rollback

```bash
deck gateway sync \
  --select-tag platform-repo-managed \
  --konnect-control-plane-name "PRD CP" \
  --konnect-token $KONNECT_PAT \
  --konnect-addr $KONNECT_ADDR \
  rollback.yaml
```

#### Step 5 — Confirm

```bash
deck gateway dump --select-tag platform-repo-managed | yq '.services[].tags'
# Should show version:0.1.0 on all services
```

#### Step 6 — Update Git to reflect rollback

```bash
cp rollback.yaml PRD/kong/kong.yaml
git add PRD/kong/kong.yaml
git commit -m "revert: rollback gateway to v0.1.0 (hotfix for broken v0.2.0)"
git push origin main
```

**Why this is safe:**
- `--select-tag platform-repo-managed` ensures only pipeline-owned resources are touched
- Manually created resources in Konnect are completely unaffected
- The rollback file already has the correct `version:0.1.0` tags from when it was originally generated
- Git history now shows the revert as a discrete, auditable commit

---

## 7. Multi-Stage Pipeline Promotion

### Architecture

```
Source (openapi.yaml + patches.yaml)
            │
            ▼
    ┌─── CI Workflow ───┐
    │                   │
    │  openapi2kong     │
    │  patch            │
    │  add-tags         │
    │  merge            │
    │  lint             │
    │  validate         │
    │  diff             │
    └───────────────────┘
            │
     ┌──────┴──────┐
     ▼             ▼
 staging/       PRD/kong/
 kong/           kong.yaml
 kong.yaml       (env:production)
 (env:staging)
     │             │
     ▼             ▼
 STAGING CP     PRD CP
 (auto deploy)  (PR approval required)
```

### Tag-based promotion script

```bash
#!/bin/bash
# promote.sh — take current staging config and promote to prod tags

# Pull current staging state
deck gateway dump \
  --konnect-control-plane-name "Staging CP" \
  --select-tag env:staging \
  > staging-current.yaml

# Re-tag for production
# (remove env:staging, add env:production)
cat staging-current.yaml | \
  yq 'del(.. | select(type == "!!seq") | .[] | select(. == "env:staging"))' | \
  yq '(.. | select(type == "!!seq") | select(. == ["env:staging"])) |= . + ["env:production"]' \
  > prod-ready.yaml

# Diff against prod before applying
deck gateway diff \
  --konnect-control-plane-name "PRD CP" \
  --select-tag platform-repo-managed \
  prod-ready.yaml

# Apply to prod
deck gateway sync \
  --konnect-control-plane-name "PRD CP" \
  --select-tag platform-repo-managed \
  prod-ready.yaml
```

---

## 8. Multi-Team in One Control Plane

### Problem

Large organisations often have multiple teams — platform, payments, identity, content — all using the same Kong Konnect control plane. Without tagging, any team's sync could accidentally delete another team's resources.

### Solution: Domain Tags + Team-Scoped Syncs

Each team owns their resources with a domain tag. CI pipelines for each team sync only their domain.

```
Konnect PRD Control Plane
├── flights-service        [flight-data, version:0.1.0, platform-repo-managed]
├── routes-service         [route-data, version:0.1.0, platform-repo-managed]
├── bookings-service       [sales, version:1.3.0, platform-repo-managed]
├── customer-service       [sales, version:1.3.0, platform-repo-managed]
├── experience-service     [experience, platform-repo-managed]
└── global-rate-limiter    [platform-repo-managed]  ← platform team owns this
```

**Flights team CI:**
```bash
deck gateway sync --select-tag flight-data flights-kong.yaml
# Only creates/updates/deletes flight-data tagged resources
```

**Sales team CI:**
```bash
deck gateway sync --select-tag sales sales-kong.yaml
# Only creates/updates/deletes sales tagged resources
```

**Platform team CI (manages shared plugins, consumers, vaults):**
```bash
deck gateway sync --select-tag platform-repo-managed full-kong.yaml
# Manages everything with the ownership tag
```

### Access control alignment

In Konnect, RBAC roles can be scoped to match the tag boundaries:

| Team | Tag | Konnect Role |
|------|-----|--------------|
| Flights | `flight-data` | flight-data-admin |
| Sales | `sales` | sales-admin |
| Platform | `platform-repo-managed` | control-plane-admin |

---

## 9. Audit and Observability

### What's deployed right now?

```bash
# Full dump of all pipeline-managed resources
deck gateway dump \
  --select-tag platform-repo-managed \
  --konnect-control-plane-name "PRD CP" \
  --konnect-token $KONNECT_PAT \
  --konnect-addr $KONNECT_ADDR \
  > current-state.yaml

# Which versions are live?
yq '.services[] | .name + ": " + (.tags[] | select(test("version:")))' current-state.yaml
# flights-service: version:0.1.0
# routes-service: version:0.1.0
# bookings-service: version:1.3.0
# customer-service: version:0.2.0
```

### Has production drifted from Git?

```bash
# Compare what's in Git vs what's in Konnect
deck gateway diff \
  --select-tag platform-repo-managed \
  --konnect-control-plane-name "PRD CP" \
  --konnect-token $KONNECT_PAT \
  --konnect-addr $KONNECT_ADDR \
  PRD/kong/kong.yaml

# If output is empty → Git and Konnect are in sync (good)
# If output shows changes → Konnect has drifted (manual changes detected)
```

**Run this as a scheduled check:**

```yaml
# GitHub Actions scheduled drift detection
name: Drift Detection
on:
  schedule:
    - cron: '0 9 * * *'   # Daily at 9am UTC

jobs:
  drift-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: kong/setup-deck@v1
        with:
          deck-version: '1.58.0'
          wrapper: false
      - name: Check for drift
        run: |
          DIFF=$(deck gateway diff \
            --select-tag platform-repo-managed \
            --konnect-control-plane-name "${{ vars.KONNECT_CP_NAME }}" \
            --konnect-token ${{ secrets.KONNECT_PAT }} \
            --konnect-addr "${{ vars.KONNECT_ADDR }}" \
            PRD/kong/kong.yaml)

          if [ -n "$DIFF" ]; then
            echo "::warning::Configuration drift detected!"
            echo "$DIFF"
            # Optionally: post to Slack, open GitHub Issue, etc.
          else
            echo "No drift. Konnect matches Git."
          fi
```

---

## 10. Tag Governance — Naming Conventions

### Recommended Schema

| Category | Pattern | Examples |
|----------|---------|---------|
| Ownership | `<pipeline>-managed` | `platform-repo-managed`, `sales-repo-managed` |
| Version | `version:<semver>` | `version:0.1.0`, `version:2.3.1` |
| Environment | `env:<name>` | `env:production`, `env:staging`, `env:dev` |
| Domain | `<domain-name>` | `flight-data`, `sales`, `experience`, `identity` |
| Region | `region:<code>` | `region:eu`, `region:us-east` |
| Criticality | `tier:<level>` | `tier:critical`, `tier:standard` |

### Rules

1. **All pipeline-managed resources must have an ownership tag.** If it doesn't have one, decK can't safely scope its sync.

2. **Version tags come from the OAS `info.version` field.** Never manually set version tags — always extract from source via `yq`.

3. **One environment per resource.** A resource should not have both `env:staging` and `env:production`. If you need to test in both, create separate resources.

4. **Tags are additive, not hierarchical.** `flight-data` + `version:0.1.0` + `env:production` are three independent labels, not a tree. Any of them can be used as a `--select-tag` filter independently.

5. **Do not use tags as runtime config.** Tags are for operations and governance, not for plugin logic or routing decisions. Use route `headers`, `paths`, or consumer `groups` for runtime behaviour.

---

## 11. KongAir Reference Implementation

### How KongAir applies all four tag patterns

**CI Workflow (`ci-kong.yaml`):**

```yaml
# Step 1: Domain + Version tags per service
- name: Convert Flights API to Kong
  run: |
    VERSION=$(yq '.info.version' flight-data/flights/openapi.yaml)
    deck file openapi2kong -s flight-data/flights/openapi.yaml | \
      deck file patch flight-data/flights/kong/patches.yaml | \
      deck file add-tags \
        --selector "$.services[*]" \
        --selector "$.services[*].routes[*]" \
        flight-data "version:$VERSION" \
        -o .github/artifacts/kong/flight-data-flights-kong.yaml

# Step 2: Ownership + Environment tags at platform merge
- name: Platform additions, patches and tag
  run: |
    deck file merge ... | \
    deck file patch platform/kong/patches.yaml | \
    deck file add-tags \
      -o PRD/kong/kong.yaml \
      "platform-repo-managed" "env:production"
```

**Resulting tags on a Flights route in Konnect:**

```yaml
tags:
  - flight-data          # domain — Flights team owns this
  - version:0.1.0        # version — from openapi.yaml info.version
  - platform-repo-managed # ownership — CI pipeline manages this
  - env:production       # environment — this is in PRD
```

**Deploy Workflow (`deploy-kong-PRD.yaml`):**

```bash
# Scoped sync — only touch what the pipeline owns
deck gateway sync \
  --select-tag platform-repo-managed \
  --konnect-control-plane-name "${{ vars.KONNECT_CP_NAME }}" \
  --konnect-token ${{ secrets.KONNECT_PAT }} \
  --konnect-addr "${{ vars.KONNECT_ADDR }}" \
  PRD/kong/kong.yaml
```

---

## 12. Command Reference

### Applying Tags

```bash
# Tag all services and their routes
deck file add-tags \
  --selector "$.services[*]" \
  --selector "$.services[*].routes[*]" \
  my-tag \
  -o output.yaml \
  input.yaml

# Tag with multiple labels at once
deck file add-tags \
  --selector "$.services[*]" \
  --selector "$.services[*].routes[*]" \
  domain-tag "version:$VERSION" "env:production" \
  -o output.yaml

# Tag at pipeline end (after merge)
deck file merge input1.yaml input2.yaml | \
  deck file add-tags "platform-repo-managed" "env:production" \
  -o final.yaml
```

### Syncing by Tag

```bash
# Sync only pipeline-owned resources
deck gateway sync --select-tag platform-repo-managed kong.yaml

# Sync only a specific domain
deck gateway sync --select-tag flight-data kong.yaml

# Sync resources with multiple required tags (AND)
deck gateway sync \
  --select-tag platform-repo-managed \
  --select-tag env:production \
  kong.yaml
```

### Diffing by Tag

```bash
# What would change if I deploy?
deck gateway diff --select-tag platform-repo-managed kong.yaml

# Capture diff output for a PR body
DIFF=$(deck gateway diff --select-tag platform-repo-managed kong.yaml)
```

### Dumping by Tag

```bash
# Export only resources with a specific tag
deck gateway dump --select-tag flight-data > flight-data-current.yaml

# Export and check versions
deck gateway dump --select-tag platform-repo-managed | \
  yq '.services[] | .name + ": " + (.tags[] | select(test("version:")))'
```

### Validating Before Sync

```bash
# Validate config is Konnect-compatible before any sync
deck gateway validate \
  --konnect-control-plane-name "PRD CP" \
  --konnect-token $KONNECT_PAT \
  --konnect-addr $KONNECT_ADDR \
  PRD/kong/kong.yaml

# Always run validate before diff, diff before sync
# validate → diff → sync
```

---

## 13. Common Mistakes and How to Avoid Them

### Mistake 1: Syncing without `--select-tag` in a shared control plane

```bash
# WRONG — will delete everything in Konnect not in your file
deck gateway sync kong.yaml

# RIGHT — only touch what you own
deck gateway sync --select-tag platform-repo-managed kong.yaml
```

**Why it matters:** Without scoping, `deck gateway sync` is equivalent to saying "the gateway should contain exactly what's in this file — nothing more, nothing less." In a shared Konnect control plane with manually created resources or other team pipelines, this deletes everything it doesn't know about.

---

### Mistake 2: Using `filter[name]` for URL-based version lookup

```bash
# WRONG — URL encoding breaks with spaces or special chars
curl "$KONNECT_ADDR/v2/api-products/$ID/product-versions?filter[name]=$VERSION"

# RIGHT — client-side filter with jq
curl "$KONNECT_ADDR/v2/api-products/$ID/product-versions" | \
  jq -r --arg v "$VERSION" '.data[] | select(.name == $v) | .id'
```

---

### Mistake 3: Using `.name` instead of `.version` for v3 Catalog versions

```bash
# WRONG — v3 Catalog version objects have no .name field
jq '.data[] | select(.name == $v) | .id'

# RIGHT — the field is .version
jq -r --arg v "$VERSION" '.data[] | select(.version == $v) | .id'
```

---

### Mistake 4: POSTing a new version when one already exists (409 Conflict)

```bash
# WRONG — fails with 409 on every run after the first
curl -X POST "$KONNECT_ADDR/v3/apis/$API_ID/versions" ...

# RIGHT — upsert: check first, then PATCH or POST
VERSION_ID=$(curl -s "$KONNECT_ADDR/v3/apis/$API_ID/versions" \
  -H "Authorization: Bearer $TOKEN" | \
  jq -r --arg v "$VERSION" '.data[] | select(.version == $v) | .id // empty')

if [ -z "$VERSION_ID" ]; then
  # First run — create
  curl -X POST "$KONNECT_ADDR/v3/apis/$API_ID/versions" ...
else
  # Subsequent runs — update in place
  curl -X PATCH "$KONNECT_ADDR/v3/apis/$API_ID/versions/$VERSION_ID" ...
fi
```

---

### Mistake 5: CI commit triggering CI again (infinite loop)

```bash
# WRONG — CI commits back to the branch, which re-triggers CI, forever
git commit -m "chore: update PRD/kong/kong.yaml"

# RIGHT — [skip ci] tells GitHub Actions to ignore this push
git commit -m "chore: update PRD/kong/kong.yaml [skip ci]"
```

---

### Mistake 6: Hardcoding version tags

```yaml
# WRONG — version in tag gets out of sync with actual OAS version
deck file add-tags "version:0.1.0"

# RIGHT — always extract from the OAS spec dynamically
VERSION=$(yq '.info.version' openapi.yaml)
deck file add-tags "version:$VERSION"
```

---

## Summary

| Concept | Tag Pattern | decK Flag | Use Case |
|---------|------------|-----------|---------|
| Ownership | `platform-repo-managed` | `--select-tag` | Safe sync in shared CPs |
| Versioning | `version:0.1.0` | `--select-tag` | Audit, rollback, drift detection |
| Environment | `env:production` | `--select-tag` | Multi-stage promotion |
| Domain | `flight-data`, `sales` | `--select-tag` | Multi-team independence |
| Combined | all of the above | multiple `--select-tag` flags | AND-filter for precise scoping |

**The golden rule:** `--select-tag` is not optional in production. It is the boundary that makes automated GitOps safe in shared environments.

---

*Generated for the KongAir APIOps reference implementation. Verified against decK 1.58.0 and Kong Konnect.*
