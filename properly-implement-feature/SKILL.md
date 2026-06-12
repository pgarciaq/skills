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

### 1. Implement the feature

Write the code for the feature/fix described in the user's request. Follow
existing code patterns, style, and architecture conventions.

### 2. Run existing tests

Run the relevant test suite to ensure nothing is broken by the change.

### 3. Evaluate and execute additional tasks

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

### 4. Produce the completion report

After all tasks are done, present a summary table:

```
| Task | Status | Notes |
|------|--------|-------|
| Code implementation | Done | ... |
| Existing tests pass | Done | ... |
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
