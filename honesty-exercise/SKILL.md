---
name: honesty-exercise
description: Perform a comprehensive alignment audit of a feature or plugin across requirements, implementation, documentation, API spec, cheat sheet, Bruno collection, unit tests, E2E tests, and IQE tests. Use when the user says "honesty exercise" or asks to audit alignment of a feature, plugin, or recommendation type.
disable-model-invocation: true
---

#  Honesty Exercise

A rigorous cross-source alignment audit for a feature or plugin. The user specifies the target (e.g., "container recommendations", "snapshot staleness", "PVC right-sizing"). The exercise produces a discrepancy matrix, fixes all misalignments, and reports honestly on what works, what's broken, and what's missing.

## Core Principles

1. **Requirements are authoritative.** When implementation disagrees with requirements, flag it — don't silently adapt tests/docs to match buggy code.
2. **Never fake it.** No vacuous tests, no weakened assertions, no `try/except: pass`, no `skipTest()` to hide failures.
3. **Fix everything you find.** Don't just report — implement the fix in code, docs, tests, cheatsheet, Bruno, OpenAPI.
4. **Be honest about gaps.** If something is missing or broken, say so clearly. Don't hide problems.
5. **Never lose existing content.** When fixing or updating a documentation page, **always merge** — never rewrite from scratch, never gut existing sections. Read the current page first, then ADD or CORRECT content. If a page has Quick Facts, mermaid diagrams, API examples, troubleshooting tables, or any other structured content, **keep all of it**. The only acceptable removals are factually incorrect statements being replaced with correct ones. Line count should stay the same or increase, never decrease significantly.

## Workflow

### Phase 1: Discovery

Locate all sources for the target feature:


| Source                     | Where to look                                                               |
| -------------------------- | --------------------------------------------------------------------------- |
| Requirements/design        | `ros-ocp-backend/docs/` (feature specs, `requirements.md`)                  |
| Go implementation          | `ros-ocp-backend/internal/` (engine, api, model, plugins, notifications)    |
| Internal architecture docs | `ros-ocp-backend/docs/architecture/`                                        |
| Public website docs        | `ros-ocp-backend/docs-site/` (plugin-reference, features, planned-features) |
| API cheat sheet            | `costmgmt-api-cheatsheet/costmgmt-api-cheatsheet.adoc`                      |
| Bruno collection           | `costmgmt-api-cheatsheet/bruno/Optimizations/`                              |
| Unit tests                 | `ros-ocp-backend/internal/` (`*_test.go`)                                   |
| E2E tests                  | `cost-onprem-chart/tests/suites/ros/`                                       |
| IQE tests                  | `iqe-ros-ocp-plugin/`                                                       |
| OpenAPI spec               | `ros-ocp-backend/openapi.json`                                              |
| Notification catalog       | `ros-ocp-backend/internal/notifications/catalog.go`                         |
| Koku backend               | `koku/` — Masu effective_rates, tag tables, listener, ingestion pipeline    |
| Koku-UI frontend           | `koku-ui/` — pages consuming ROS API (optimizations, recommendations)       |


#### Koku backend dependencies

ROS depends on Koku for:

- **Cost data**: Masu `effective_rates` endpoint provides cluster cost rates for savings calculations
- **Tag sync**: `reporting_ocptags_values` tables in tenant schema (used in `ROS_TAGS_SOURCE=db` mode)
- **Ingestion pipeline**: Koku Listener processes operator uploads, stores data in S3, ROS processor reads from S3
- **Provider/source management**: `api_provider`, `api_sources` tables
- **Shared database**: On-prem mode shares PostgreSQL between Koku and ROS

When auditing a feature, check if it depends on Koku internals. If the feature uses cost rates, tag data, or data ingestion, verify the Koku side works correctly too.

#### Koku-UI frontend impact

ROS API is consumed by the koku-ui frontend (Optimizations section). When auditing:

- **Response shape changes** (e.g., MoneyAmount migration, new fields) — verify the UI still renders correctly or has been updated
- **New endpoints** — check if UI components exist or are planned
- **Removed/renamed fields** — breaking changes need UI coordination
- Key UI files: `koku-ui/apps/koku-ui-hccm/src/` (routes/optimizations/, api/ros*)
- On-prem UI: `koku-ui/apps/koku-ui-onprem/`

### Phase 2: Audit

For each source, document:

1. **Endpoints** — what API paths are described/tested?
2. **Filters** — what query parameters (filter, order_by, group_by) are supported?
3. **Pagination** — offset, keyset, or both? Correct cursor shape?
4. **Response shape** — fields, nesting, types, `MoneyAmount` usage
5. **CSV export** — supported? What columns?
6. **Notification codes** — which codes, correct severity, correct plugin association?
7. **Classifications/recommendation types** — what enum values?
8. `**meta.currency`** — present on list endpoints returning monetary amounts?
9. `**confidence_level**` — present where meaningful?
10. **Data generation** — does NISE produce the right data? What YAML template?

### Phase 3: Produce Alignment Matrix

```
| Aspect | Requirements | Go Code | Internal Docs | Public Docs | Cheatsheet | Bruno | Unit Tests | E2E Tests | IQE Tests | OpenAPI |
|--------|-------------|---------|---------------|-------------|------------|-------|------------|-----------|-----------|---------|
```

Mark each cell: ✅ (aligned), ⚠️ (partially), ❌ (wrong/missing), — (N/A)

### Phase 4: Fix Discrepancies

For each discrepancy:

1. State what's wrong
2. Identify authoritative source (requirements > code > docs > tests)
3. Implement the fix immediately
4. If requirements and code disagree, **ask the user** before changing either

**CRITICAL — Documentation edits:**

- Before editing any existing docs page, **read the entire current file first**
- Fix inaccuracies by editing specific lines — do NOT rewrite the whole file
- If a page has Quick Facts, diagrams, examples, or tables — those stay
- Adding new sections is fine; removing/replacing existing sections is NOT
- If you think a section is outdated, update it in place — don't delete it
- After editing, compare line count: if it dropped significantly, you lost content — undo and redo properly

### Phase 5: Verify

1. Run affected Go tests: `go test ./internal/... -run <Pattern> -v`
2. Build: `go build ./...`
3. If SNO cluster is available, rebuild image + deploy + run E2E tests
4. Commit and push all fixes

### Phase 6: Report

Produce honest summary:

- What works end-to-end
- What was broken and is now fixed
- What remains genuinely missing (planned/future work)
- Any design questions or inconsistencies that need user decision

## Checklist of Common Issues

These are patterns that have been found repeatedly across plugins:

- [ ] **Docs preservation:** Before editing any docs-site page, note its line count. After editing, verify line count stayed the same or increased. NEVER reduce a page by more than 10 lines without explicit user approval.
- [ ] `filter[term]` accepts both `short_term` and `short` (NormalizeRecommendationTermFilter)
- [ ] Keyset pagination with proper tie-breaker (not just sort value)
- [ ] CSV export includes all meaningful JSON fields
- [ ] `meta.currency` on every list endpoint with monetary amounts
- [ ] `MoneyAmount` (not raw float/int) for all monetary API fields
- [ ] Storage uses `BIGINT` cents (not REAL/NUMERIC dollars)
- [ ] Notification codes in catalog.go match the correct plugin
- [ ] E2E tests use `get_fresh_token()` for auth (not password grant)
- [ ] E2E tests `pytest.skip` when no data (not silent pass)
- [ ] Bruno requests use correct field names and params
- [ ] Cheatsheet examples match actual API response shape
- [ ] Public docs cross-links resolve to existing files
- [ ] OpenAPI spec matches actual handler behavior (params, response schema)
- [ ] Unit tests don't weaken assertions or skip to hide failures

## SNO Cluster Access

If E2E testing is needed:

- SSH: `ssh -o StrictHostKeyChecking=no root@dell-r730-031.bkr.lab.eng.rdu2.dc.redhat.com`
- KUBECONFIG: `/root/.kcli/clusters/sno/auth/kubeconfig`
- Namespace: `cost-onprem`
- Keycloak realm: `cost-management`
- Build + deploy: `podman build`, `podman push`, `oc set image`

## Repos and Branches


| Repo                    | Path                                  | Branch                                |
| ----------------------- | ------------------------------------- | ------------------------------------- |
| ros-ocp-backend         | `~/dev/koku/ros-ocp-backend/`         | `pgarciaq-rosocp-superpowers-phase12` |
| costmgmt-api-cheatsheet | `~/dev/koku/costmgmt-api-cheatsheet/` | `pgarciaq-rosocp-superpowers-phase12` |
| cost-onprem-chart       | `~/dev/koku/cost-onprem-chart/`       | `pgarciaq-rosocp-superpowers-phase12` |
| iqe-ros-ocp-plugin      | `~/dev/koku/iqe-ros-ocp-plugin/`      | `pgarciaq-rosocp-superpowers-phase12` |
| nise                    | `~/dev/koku/nise/`                    | `pgarciaq-rosocp-superpowers-phase12` |


