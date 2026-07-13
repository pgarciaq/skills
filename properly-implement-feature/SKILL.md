---
name: properly-implement-feature
description: >-
  Implement a feature or fix end-to-end, including code, tests, API spec,
  documentation, and tooling updates. Use when implementing features, fixes,
  or changes that may affect APIs, tests, or documentation — not just code.
---

# Properly Implement Feature

Implementing a feature is more than writing code. This skill ensures every
deliverable is evaluated and either completed or explicitly dismissed.

## Workflow

### 1. Verify placement before implementing (UI features)

When the feature involves adding a new UI element (tab, page, navigation
item, data table), locate an existing element of the same type FIRST:

1. Find a sibling element (e.g., for "add Node tab" → find the "Namespace" tab)
2. Read the file that renders it — note which app/package it belongs to
3. Verify your new element should go in the same app (host vs remote in
   Module Federation, parent vs child component, etc.)
4. If the element crosses app boundaries (e.g., data logic in one app,
   navigation in another), plan changes in BOTH apps before writing code

**Never assume the element belongs in the app where you found the data
logic.** In monorepos with Module Federation, navigation lives in the
host app while data components live in the remote app.

Skip this step for features that do not involve UI elements.

#### ROS Optimizations: term/engine projection (mandatory for new recommendation tabs)

When adding or changing a **recommendations list tab** or **detail/breakdown page**
in `koku-ui-ros` (Container, Namespace, Node, or future entity types), follow the
same projection pattern as the existing tabs. Do not ship a list without term and
engine dropdowns unless the product explicitly excludes that entity type.

**List page checklist (koku-ui-ros):**

1. Use a dedicated `*Table` + `*Toolbar` pair under
   `apps/koku-ui-ros/src/routes/optimizations/optimizationsTable/` — not legacy
   `OptimizationsTable` / `OptimizationsToolbar` alone.
2. Include `OptimizationsProjectionToolbar` in the list toolbar (term + engine
   `PerspectiveSelect` controls).
3. Wire URL state with `useUrlState({ prefix: '<entity>_', ... })` — existing
   prefixes: `ctr_` (containers), `ns_` (namespaces), `node_` (nodes). Pick a
   new unique prefix for new tabs.
4. Pass projection to list API calls via `withRosListProjection()` from
   `api/ros/rosListParams.ts`. Defaults: `short_term` + `cost`.
5. Render list cells from the selected term/engine — never hardcode
   `short_term` or a single engine in column formatters.
6. **Container tab pitfall:** `optimizationsDetails.tsx` must render
   `OptimizationsContainersTable` (with projection toolbar), not legacy
   `OptimizationsTable`, when the namespace feature toggle is off.

**Detail/breakdown checklist (koku-ui-ros):**

1. Do **not** add term/engine dropdowns on detail or breakdown pages.
2. Read projection from URL/navigation state (`useBreakdownProjection` or
   equivalent) and pass `filter[term]` / `filter[engine]` on detail fetches via
   `encodeRosDetailFetchQuery()`.
3. When navigating list → detail, preserve term/engine in the URL or location
   state so breakdown matches the list selection.

**Backend checklist (ros-ocp-backend, when list/detail APIs change):**

1. Accept `filter[term]` and `filter[engine]` on list **and** detail endpoints.
2. Exclude list rows with no data for the selected term+engine when both filters
   are explicit (see `list_projection_filter.go`).
3. Resolve `order_by` variation aliases from term/engine when applicable.
4. Update `openapi.json` and handler integration tests.

Reference implementations: Container/Namespace/Node tables and toolbars in
`koku-ui-ros`; backend helpers in `ros-ocp-backend/internal/api/queryparams/projection.go`.
See also `koku-ui/apps/koku-ui-ros/AGENTS.md` and workspace rule
`koku-ecosystem.mdc` (Optimizations projection section).

### 2. GitHub Issue Tracking

Every feature or fix must have a corresponding GitHub issue on
`https://github.com/pgarciaq/ros-ocp-backend/issues`.

**Workflow:**

1. **Check if an issue already exists** for the feature/fix. If not, create one
   with a clear problem statement (what's wrong or what's needed).
2. **Never overwrite the issue description.** The original description is the
   problem statement and must remain unchanged.
3. **Add implementation details as comments**, not edits to the body. Each
   comment documents what was done:
   - Design decisions made (and why)
   - Endpoint spec (path, params, response shape)
   - Files created/modified
   - Frontend requirements (if applicable)
   - Empty states, error handling, edge cases

This keeps clear separation between "what's the problem" (description) and
"how we solved it" (comments), and preserves history.

**Commands:**
```bash
# Create issue (problem statement only)
gh issue create --repo pgarciaq/ros-ocp-backend --title "..." --body "..."

# Add implementation comment (NEVER use gh issue edit --body)
gh issue comment <NUMBER> --repo pgarciaq/ros-ocp-backend --body "..."
# Or from a file:
gh issue comment <NUMBER> --repo pgarciaq/ros-ocp-backend --body-file path/to/file.md
```

### 3. Implement the feature

Write the code for the feature/fix described in the user's request. Follow
existing code patterns, style, and architecture conventions.

### 3. Run existing tests

Run the relevant test suite to ensure nothing is broken by the change.

### 3.5. Build and deploy to test cluster (UI features)

When the feature targets a UI deployed on a test cluster (e.g., UXSNO),
build and deploy BEFORE verifying state transitions:

1. Commit and push all code changes
2. **Verify architecture** on build host and cluster — do not assume SNO means
   aarch64 (UXSNO is amd64; Apollo SNO is arm64):
   ```bash
   uname -m
   oc get nodes -o custom-columns=NAME:.metadata.name,ARCH:.status.nodeInfo.architecture
   ```
   Match image arch to cluster (`amd64` → native x86_64 build; `arm64` → aarch64
   or `--platform linux/arm64`). See AGENTS.md and
   `cost-onprem-chart/.cursor/rules/aarch64-sno-deployment.mdc` §0.
3. Pull on the build host (hypervisor or local)
4. Build using the **repo's existing Containerfile** — check AGENTS.md for
   the exact path and command. **Never create a new Containerfile.**
   Use `--platform` only when cross-building for arm64 from x86_64.
5. Push to the cluster registry with a **unique** tag — never reuse tags
   (`imagePullPolicy: IfNotPresent` silently keeps old images)
6. Update the deployment with `oc set image` using the correct **container
   name** — multi-container pods have specific names (e.g., `app` not `ui`).
   Check AGENTS.md or `oc get deploy -o yaml` for the container name.
7. Wait for rollout to complete

**Critical mistakes to avoid:**
- Building `--platform linux/arm64` for an amd64 cluster (e.g. UXSNO)
- Using `npm run build` instead of `build:onprem` for federated modules
  (generates cloud-only artifacts that fail at runtime)
- Using `Dockerfile` when the file is named `Containerfile` (or vice versa)
- Using the wrong base image (e.g., plain `nginx` instead of UBI nginx —
  causes permission errors on OpenShift)
- Guessing the image structure instead of reading the existing Containerfile

Skip this step only when no test cluster is involved (e.g., local dev server
testing only).

### 4. Verify state transitions (UI features)

When the feature involves a data table, list view, pagination, or
filter/sort controls, verify the following transitions BEFORE moving on.
Each row represents a real production bug pattern.

| # | Transition | What to verify |
|---|-----------|----------------|
| 1 | Page 1 → Page 2 → Page 3 | Count stays constant; no duplicate rows across pages |
| 2 | Sort ASC → Sort DESC | NULL values at bottom (not top) for DESC |
| 3 | Sort by numeric field | Numeric order, not lexicographic ("35" before "6") |
| 4 | Apply filter → results | Correct filtered count; pagination resets to page 1 |
| 5 | Apply filter → remove filter | Page recovers; no crash; no stale data |
| 6 | Paginate → remove filter | Returns to page 1 of unfiltered results |
| 7 | API returns empty results | No crash; shows empty state; no stale data |
| 8 | API returns error | Shows error state; page recoverable after retry |
| 9 | Sort by field with NULLs → paginate | Cursor correctly handles NULL sort values |

**How to verify:** If a live environment is available, test via API calls
(curl/httpie) to confirm backend behavior. If not, trace through the code
mentally and confirm each guard exists. Add unit tests for non-obvious
paths (NULL cursor handling, empty results, filter removal).

Skip this step only for features that do not involve data display,
pagination, or filtering.

### 5. Evaluate and execute additional tasks

For **each** task below, determine whether it applies. If it does, do it.
If it doesn't, note why in the final report.

| # | Task | When it applies |
|---|------|-----------------|
| 1 | **Write more unit tests** | New code paths, edge cases, or behaviors not covered by existing tests |
| 2 | **Write more E2E tests** | New user-facing flows, API contracts, or integration points that E2E tests should verify |
| 3 | **Update the OpenAPI spec** | New/changed API endpoints, request/response fields, status codes, or headers |
| 4 | **Update the API cheat sheet** | New endpoints or significant behavior changes that operators/developers reference |
| 5 | **Update the Bruno collection** | New endpoints or request patterns worth having a ready-made example for |
| 6 | **Update internal docs** | Architecture docs, runbooks, monitoring guides, configuration docs, known issues |
| 7 | **Update public website / docs-site** | Customer-facing documentation that mirrors internal docs |
| 8 | **Write an ADR** | Architectural or design decisions with trade-offs, alternative approaches considered, or non-obvious rationale that future developers need to understand |
| 9 | **Update the CHANGELOG** | Any user-visible change, new feature, fix, deprecation, or performance improvement |
| 10 | **Update the ADR index** | When a new ADR is created — update both `docs/architecture/adr/README.md` and `docs-site/architecture/adrs.md` |
| 11 | **Update the performance audit** | Only when the user explicitly asks to update the audit report, or when closing out a dedicated performance optimization phase |

### 6. Produce the completion report

After all tasks are done, present a summary table:

```
| Task | Status | Notes |
|------|--------|-------|
| UI placement verified | Done / Dismissed | reason (e.g., "no UI element added") |
| Code implementation | Done | ... |
| Existing tests pass | Done | ... |
| Build and deploy | Done / Dismissed | reason (e.g., "no test cluster") |
| State transitions | Done / Dismissed | reason (e.g., "no UI data view") |
| New unit tests | Done / Dismissed | reason |
| New E2E tests | Done / Dismissed | reason |
| OpenAPI spec | Done / Dismissed | reason |
| API cheat sheet | Done / Dismissed | reason |
| Bruno collection | Done / Dismissed | reason |
| Internal docs | Done / Dismissed | reason |
| Public website | Done / Dismissed | reason |
| ADR | Done / Dismissed | reason |
| CHANGELOG | Done / Dismissed | reason |
| ADR index | Done / Dismissed | reason |
| Performance audit | Done / Dismissed | reason |
```

## Decision guidelines

- **Dismiss** a task only when you can articulate a clear reason (e.g., "no new
  API fields were added" or "no Bruno collection exists in this repo").
- **Never** dismiss a task just because it's extra work.
- When in doubt, **do it** — false negatives (missing updates) are worse than
  false positives (unnecessary but harmless updates).
- If the OpenAPI spec or other generated file is corrupted/truncated, report
  the issue rather than silently skipping it.

## Locating project artifacts

These paths vary by project. Search the workspace for:

| Artifact | Common locations |
|----------|-----------------|
| OpenAPI spec | `openapi.json`, `openapi.yaml`, `docs/openapi.*` |
| Bruno collection | `bruno/`, `**/bruno/` in workspace repos |
| API cheat sheet | `costmgmt-api-cheatsheet/`, adjacent repo |
| Internal docs | `docs/`, `docs/operations/`, `docs/architecture/` |
| Public website | `docs-site/`, `website/`, `public/docs/` |
| E2E tests | `tests/e2e/`, `testing/`, adjacent `cost-onprem-chart/tests/` |
| ADRs | `docs/architecture/adr/`, `docs/adr/` |
| ADR index (internal) | `docs/architecture/adr/README.md`, `docs/adr/README.md` |
| ADR index (public) | `docs-site/architecture/adrs.md` |
| Changelog | `CHANGELOG.md` at repo root |
| Performance audit | `docs/performance/` |

## Example dismissal reasons

- "No new API endpoints or response fields were added" (OpenAPI, Bruno, cheat sheet)
- "No Bruno collection exists in this repository" (Bruno)
- "Change is internal/backend-only with no user-facing behavior change" (E2E)
- "The docs-site mirrors internal docs and both were updated" (public website)
- "Existing E2E tests already cover this flow at sufficient granularity" (E2E)
- "No architectural decision or trade-off was involved — straightforward bugfix" (ADR)
- "No user-visible change — internal refactoring only" (CHANGELOG)
- "No new ADR was created" (ADR index)
- "Performance audit updates are done periodically, not per-feature" (performance audit)

## ADR guidelines

An ADR (Architecture Decision Record) should be written when:
- A design decision involved choosing between alternatives with meaningful trade-offs
- The rationale might not be obvious to future developers reading the code
- The decision constrains future work or closes off alternative approaches
- Performance, accuracy, or complexity trade-offs were made

An ADR should NOT be written for:
- Routine bugfixes with obvious causes
- Simple refactoring with no behavioral change
- Following established patterns without deviation

When writing an ADR:
- Follow the existing numbering convention (check the latest ADR number and increment)
- Follow the existing format in the repo (read a recent ADR for style)
- Always update BOTH the internal ADR index AND the public ADR page
