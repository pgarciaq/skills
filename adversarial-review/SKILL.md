---
name: adversarial-review
description: >-
  Perform a comprehensive adversarial due diligence review covering security,
  correctness, auditability, operational robustness, performance, design quality,
  maintainability, and governance. Use when the user asks for a security audit,
  architecture review, due diligence, adversarial review, or codebase health
  assessment.
disable-model-invocation: true
---

# Adversarial Due Diligence Review

Perform a comprehensive adversarial review of a codebase, acting as a hostile
but fair auditor. The goal is to surface real risks, not to nitpick style.

## Dimensions

Evaluate across all eight dimensions:

| Dimension | What to look for |
|-----------|-----------------|
| **Security** | Auth bypasses, injection, SSRF, secrets in code, missing input validation, privilege escalation, unprotected endpoints |
| **Correctness** | Race conditions, data loss paths, silent failures, incorrect error handling, off-by-one, state corruption |
| **Auditability** | Missing structured logging, no request correlation, insufficient metrics, gaps in observability |
| **Operational robustness** | Missing health checks, no graceful shutdown, unbounded queues, no backpressure, missing circuit breakers |
| **Performance** | O(n) where O(1) possible, unbounded memory, missing pagination, N+1 queries, full table scans |
| **Design quality** | God objects, leaky abstractions, missing separation of concerns, tight coupling, inconsistent patterns |
| **Maintainability** | Dead code, missing docs, unclear naming, no tests for critical paths, high cyclomatic complexity |
| **Governance** | Missing ADRs, no changelog discipline, no CI enforcement, dependency hygiene, license compliance |

## Workflow

### Phase 1: Reconnaissance

1. Map the repository structure (entry points, key modules, config, tests, docs)
2. Identify the tech stack, frameworks, and deployment model
3. Understand the data flow (ingestion → processing → storage → API)
4. Note the authentication/authorization model
5. Check for existing audits, security docs, or known issues

### Phase 2: Systematic Analysis

For each dimension, perform targeted investigation:

- **Security:** Trace all external inputs to their consumption points. Check auth middleware coverage. Look for hardcoded secrets, permissive CORS, missing rate limits.
- **Correctness:** Identify concurrent code paths. Check error propagation. Look for silent swallows (`_ = err`), panics in library code, unvalidated assumptions.
- **Auditability:** Verify structured logging exists on critical paths. Check metrics coverage. Look for log levels that hide important information.
- **Operational:** Check shutdown handling, health endpoints, connection pool management, retry/backoff strategies, timeout configuration.
- **Performance:** Identify hot paths (list endpoints, aggregation queries). Check for in-memory pagination, unbounded allocations, missing indexes.
- **Design:** Look for God packages, circular dependencies, abstraction leaks, inconsistent error handling patterns.
- **Maintainability:** Check test coverage of critical paths, documentation freshness, dead code accumulation, TODO/FIXME density.
- **Governance:** Check for ADRs, changelog maintenance, CI pipeline coverage, dependency update strategy, vulnerability scanning.

### Phase 3: Finding Classification

For each finding, document:

| Field | Description |
|-------|-------------|
| **Number** | Sequential identifier |
| **Title** | One-line summary |
| **Severity** | Critical / High / Medium / Low / Informational |
| **Dimension** | Which of the 8 dimensions |
| **Location** | File path and line range |
| **Description** | What the issue is (2-3 sentences) |
| **Risk** | What could go wrong (concrete scenario) |
| **Recommendation** | How to fix it (specific, actionable) |
| **Effort** | S (hours) / M (days) / L (weeks) |

### Phase 4: Prioritization

Rank findings by compound risk (severity × likelihood × blast radius), then group into:

1. **Immediate** — Security or data-loss risks exploitable today
2. **Short-term** — Correctness/performance issues that degrade under load
3. **Medium-term** — Operational and design issues that slow development
4. **Backlog** — Governance and maintainability improvements

### Phase 5: Produce the Report

Structure the output document as:

```markdown
# Adversarial Due Diligence Review — [Project Name]

## Version & Date
Version: X.Y | Date: YYYY-MM-DD | Reviewer: AI-assisted

## Executive Summary
[2-3 paragraph overview: scope, key risks, overall assessment]

## Scorecard

| Dimension | Rating | Key gap |
|-----------|--------|---------|
| Security | ★★★★☆ | ... |
| ... | | |

## Findings Status Summary
[Table with ALL findings: #, title, severity, status]

## Findings Detail
[Full detail for each finding per Phase 3 format]

## Priority Remediation Order
[Ordered list with effort estimates]

## Accepted Risks
[Findings explicitly accepted with rationale]

## Current State
[Summary: total findings, resolved, accepted, open]
```

## Severity Definitions

| Level | Definition |
|-------|-----------|
| **Critical** | Exploitable today with high blast radius (data breach, full compromise, data loss) |
| **High** | Exploitable with moderate effort or significant correctness issue in production |
| **Medium** | Real risk requiring specific conditions or moderate impact |
| **Low** | Minor risk, defense-in-depth improvement, or correctness edge case |
| **Informational** | Best practice gap, technical debt, or governance improvement |

## Guidelines

- **Be adversarial, not hostile.** Find real risks, not style preferences.
- **Be specific.** "SQL injection in handlers_gpu.go:142 via unsanitized order_by" not "potential injection risks."
- **Be actionable.** Every finding must have a concrete recommendation.
- **Acknowledge strengths.** Note what's done well — it builds credibility.
- **Don't duplicate.** If a prior review exists, focus on NEW findings or verify prior fixes.
- **Consider the deployment model.** On-prem vs SaaS changes the threat model.
- **Check for regressions.** If fixes were applied, verify they're complete and not bypassed elsewhere.

## Incremental Reviews

When updating an existing review (not starting fresh):

1. Read the existing review document first
2. Check git history since last review for new code/features
3. Verify previously resolved findings are still resolved
4. Focus effort on new code paths and changed functionality
5. Bump the version number and date
6. Keep the historical record intact — append, don't rewrite

## Integration with Remediation

After the review, findings can be addressed using the `/properly-implement-feature`
skill. Each finding becomes a feature/fix request with full lifecycle tracking
(code, tests, docs, API spec, etc.).
