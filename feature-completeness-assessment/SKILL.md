---
name: feature-completeness-assessment
description: >-
  Assess the implementation completeness of a feature across code, tests, docs,
  and operational readiness. Classifies each aspect as "Fully implemented",
  "MVP / Placeholder", or "Not implemented". Use when the user asks for a
  feature completeness report, implementation status, readiness assessment,
  or wants to know what's done vs what's missing for a feature.
disable-model-invocation: true
---

# Feature Completeness Assessment

Assess the implementation completeness of a specific feature. The first
parameter is the feature to assess.

## Classification Levels

| Level | Definition |
|-------|-----------|
| **Fully implemented** | Production-ready. Code complete, tested, documented, operational concerns addressed. No known gaps. |
| **MVP / Placeholder** | Functional but incomplete. Works for the happy path but missing edge cases, tests, docs, or operational hardening. May have TODOs, hardcoded values, or incomplete error handling. |
| **Not implemented** | Missing entirely, or stub/interface exists but no functional code behind it. |

## Assessment Dimensions

Evaluate the feature across ALL of these dimensions:

| Dimension | What to check |
|-----------|--------------|
| **Core logic** | Does the feature work? Are all code paths implemented? Edge cases handled? |
| **Data model** | Are schemas/migrations complete? Indexes present? Constraints enforced? |
| **API surface** | Endpoints registered? Request validation? Response shape correct? Error codes? |
| **Configuration** | Config knobs exist? Defaults sensible? Documented in .env.example? |
| **Unit tests** | Critical paths tested? Edge cases? Error conditions? Mocks appropriate? |
| **Integration tests** | End-to-end flow tested? Database interactions? External service interactions? |
| **E2E tests** | User-facing flows verified? Deployment-level validation? |
| **OpenAPI spec** | Endpoints documented? Schemas accurate? Examples provided? |
| **Internal docs** | Architecture docs? Runbooks? Configuration guides? ADRs? |
| **Public docs** | Customer-facing documentation? User guides? Migration notes? |
| **Observability** | Metrics? Structured logging? Alerting guidance? Dashboard support? |
| **Error handling** | Graceful degradation? User-facing error messages? Recovery paths? |
| **Security** | Auth/authz enforced? Input validated? Injection prevented? |
| **Performance** | Pagination? Caching? Query optimization? Load tested? |
| **Operational** | Health checks aware? Graceful shutdown? Migration path? Rollback plan? |

## Workflow

### 1. Identify the feature scope

From the user's description, determine:
- What the feature IS (functional requirements)
- Where it lives in the codebase (packages, files, modules)
- What its boundaries are (what's adjacent but separate)

### 2. Investigate each dimension

For each dimension, search the codebase and classify as Fully/MVP/Not implemented.
Be specific — cite files, line numbers, and concrete evidence.

### 3. Produce the report

```markdown
# Feature Completeness: [Feature Name]

## Summary

| Status | Count |
|--------|-------|
| Fully implemented | N |
| MVP / Placeholder | N |
| Not implemented | N |

## Overall Readiness: [Production-ready / Needs hardening / Early stage]

## Detailed Assessment

| Dimension | Status | Evidence / Notes |
|-----------|--------|-----------------|
| Core logic | Fully implemented | `internal/engine/foo.go` — all paths covered |
| Data model | Fully implemented | Migration 0142, indexes on X, Y |
| API surface | MVP / Placeholder | Endpoint exists but missing 422 response |
| Unit tests | MVP / Placeholder | Happy path tested, no edge case coverage |
| Integration tests | Not implemented | No test file exists |
| ... | ... | ... |

## Gaps and Recommendations

### MVP items to complete (priority order)
1. [Specific gap] — [what's needed] — [effort estimate]
2. ...

### Not implemented items
1. [What's missing] — [why it matters] — [effort estimate]
2. ...

## Strengths
- [What's done well — acknowledge good implementation]
```

## Guidelines

- **Be evidence-based.** Every classification must cite specific code, files, or absence thereof.
- **MVP is not a pejorative.** It means "works but not hardened" — appropriate for early features.
- **Consider the feature's maturity stage.** A brand-new feature being MVP is expected; a 6-month-old feature still at MVP is a concern.
- **Check git blame/log.** Recent changes may indicate active development vs abandoned code.
- **Look for TODOs and FIXMEs.** They often mark acknowledged placeholder implementations.
- **Don't conflate "not needed" with "not implemented."** If a dimension genuinely doesn't apply to this feature, mark it as "N/A" with a brief explanation, not "Not implemented."
- **Be actionable.** The gaps section should tell the team exactly what to do next.

## Example Assessment (abbreviated)

```
Feature: GPU Time-Slicing Recommendations

| Dimension | Status | Evidence |
|-----------|--------|----------|
| Core logic | Fully implemented | Classification tree in gpu_timeslicing.go |
| Data model | Fully implemented | Migration 0138, partitioned table |
| API surface | Fully implemented | List + detail + CSV endpoints |
| Configuration | Fully implemented | ROS_ENABLED_PLUGINS includes timeslicing |
| Unit tests | MVP / Placeholder | Happy path only, no edge cases for MIG exclusion |
| Integration tests | Fully implemented | handlers_gpu_timeslicing_integration_test.go |
| E2E tests | Not implemented | No cost-onprem-chart test for GPU |
| OpenAPI spec | MVP / Placeholder | Missing `after` param documentation |
| Internal docs | Fully implemented | docs/features-gpu-timeslicing.md |
| Public docs | Fully implemented | docs-site/features/gpu-timeslicing.md |
| Observability | MVP / Placeholder | Metrics exist but no runbook section |
| Error handling | Fully implemented | Unsupported order_by returns 400 |
| Security | Fully implemented | RBAC enforced, input validated |
| Performance | Fully implemented | SQL pagination, no in-memory fallback |
| Operational | Fully implemented | Plugin disable kills routes cleanly |

Overall: Production-ready (minor gaps in E2E and OpenAPI docs)
```
