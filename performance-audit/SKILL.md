---
name: performance-audit
description: >-
  Comprehensive performance audit of a codebase acting as Performance Architect,
  Scalability Architect, and QA Principal Engineer. Examines data types, integer
  vs floating-point math, data conversions, algorithm efficiency, database query
  patterns, API response overhead, memory allocation, concurrency, observability
  cost, and acceptable accuracy trade-offs. Use when the user asks for a
  performance review, scalability assessment, optimization opportunities, hot
  path analysis, or data type audit.
disable-model-invocation: true
---

# Performance Audit

Act as a **Performance Architect**, **Scalability Architect**, and **Quality
Assurance Principal Engineer**. Perform a comprehensive review of the codebase
to find every opportunity to improve throughput, reduce latency, shrink memory
footprint, and lower operational cost — including cases where slightly less
precise results yield large performance gains.

## Audit Dimensions

Evaluate across all dimensions below. Each dimension has specific questions to
answer and concrete patterns to look for.

### 1. Data Type Efficiency

| Check | What to look for |
|-------|-----------------|
| Integer vs float | Hot paths using `float64` where `int64` basis points, millicores, micro-cents, or lookup tables suffice |
| Boxing / interface{} | Typed slices stored as `[]interface{}`, causing heap allocation per element |
| String allocation | Repeated `fmt.Sprintf` in loops; prefer `strings.Builder` or pre-formatted keys |
| Decimal precision | Financial/billing fields using `float64` instead of fixed-point integers |
| Enum vs string | Comparisons on string constants in hot loops; use `int` enums |

**Key question:** Can we eliminate `float64` from the critical data plane (ingest → recommend → write) without losing meaningful precision?

### 2. Algorithm & Computation

| Check | What to look for |
|-------|-----------------|
| Redundant passes | Multiple iterations over the same data computing different aggregates separately |
| Precomputation | Values recomputed per-row that could be lookup tables or constants |
| Approximations | Exact math (exp, log, sqrt) where precomputed tables or linear interpolation suffice |
| Sort vs select | Full `sort.Slice` when only k-th element needed (`quickselect` / `nth_element`) |
| Decay/weighting | Per-row `math.Exp` calls replaceable by quantized lookup tables |
| Fused passes | Separate CPU/memory/trend passes over the same data fusible into one |

**Key question:** Where can we trade ≤0.1% accuracy for ≥10× throughput?

### 3. Database & Query Patterns

| Check | What to look for |
|-------|-----------------|
| N+1 queries | Per-entity queries in loops instead of batch/bulk operations |
| Missing indexes | Filter/sort columns without covering indexes |
| Full-org scans | `DISTINCT ON` / aggregation over entire tenant after each batch |
| Unbounded result sets | Queries without `LIMIT` or pagination |
| Write batching | Individual `INSERT`/`UPDATE` instead of `pgx.Batch` or `COPY` |
| Statement timeouts | Missing per-query timeouts; runaway queries holding connections |
| Connection pressure | Pool exhaustion under concurrent API + ingestion load |
| Table bloat | Append-only tables without retention policy or `VACUUM` tuning |
| Partitioning gaps | Time-series tables not range-partitioned on date |

**Key question:** What is the per-reconciliation query count and can it be reduced by an order of magnitude?

### 4. API & Response Efficiency

| Check | What to look for |
|-------|-----------------|
| Over-fetching | List endpoints returning detail-level payloads (plots, nested maps) |
| Redundant calls | Frontend fetching the same data from multiple endpoints |
| Pagination | Offset pagination on large tables instead of keyset/cursor |
| Serialization cost | Large JSON responses with deeply nested objects |
| Caching | Missing HTTP cache headers or server-side query caching |
| Slim contracts | Opportunity for a slim list DTO vs full detail response |

**Key question:** What is the p95 response size for list endpoints and can it be halved?

### 5. Memory & Allocation

| Check | What to look for |
|-------|-----------------|
| Unbounded buffers | Slices grown without `cap` hint; `append` in hot loops |
| Object pools | Short-lived allocations per request/row without `sync.Pool` |
| Streaming vs collect | Building full result sets in memory vs streaming to writer |
| Large value copies | Passing structs by value in hot paths instead of pointer |
| Map pre-sizing | `make(map, n)` without capacity hint causing rehashing |

**Key question:** What is the peak RSS per 1000 containers and where are the top 3 allocation sites?

### 6. Concurrency & Scheduling

| Check | What to look for |
|-------|-----------------|
| Sequential processing | Phases that could run concurrently (e.g., independent plugin passes) |
| Lock contention | Global mutexes in hot paths; consider sharding or lock-free |
| Worker pools | Fixed goroutine count vs unbounded goroutine fan-out |
| Context cancellation | Missing `ctx.Done()` checks in long loops |
| Batch parallelism | Serial DB batch writes that could pipeline |

**Key question:** What is the CPU utilization during reconciliation and are there idle cores?

### 7. Ingestion Pipeline

| Check | What to look for |
|-------|-----------------|
| CSV parsing overhead | Per-field string→number conversion; consider bulk/columnar parsing |
| Digest computation | Redundant sort/percentile over samples already ordered |
| Single-transaction scope | Entire manifest in one transaction causing long lock holds |
| Kafka consumer lag | Single-partition bottleneck; partition count vs consumer count |
| Idempotency overhead | Unnecessary upsert conflict checks on append-only data |

**Key question:** What is the wall-clock time to ingest 1000 containers × 30 days and where is the bottleneck?

### 8. Observability Cost

| Check | What to look for |
|-------|-----------------|
| High-cardinality labels | Prometheus labels with tenant/entity identifiers (unbounded series) |
| Metric overhead | Histogram observations in tight loops; consider summary or counter |
| Log volume | Debug/info logs emitted per-row in production; use structured log levels |
| Trace spans | Per-row spans overwhelming the collector |

**Key question:** What is the Prometheus scrape size and can we cut cardinality by 10×?

### 9. Build & Binary

| Check | What to look for |
|-------|-----------------|
| Binary size | Unused dependencies inflating the container image |
| Vendor bloat | Test/example files vendored unnecessarily |
| PGO | Profile-guided optimization not applied to production builds |
| CGO overhead | CGO enabled without need (cross-compilation, FIPS compliance aside) |

### 10. Data Lifecycle & Retention

| Check | What to look for |
|-------|-----------------|
| Raw sample retention | Keeping raw data longer than needed for recommendations |
| Digest-only paths | Operations still reading raw samples that could use digests |
| Retention configuration | Hardcoded retention vs configurable per environment |
| Prune efficiency | Row-by-row DELETE vs `TRUNCATE` partition / bulk delete |

### 11. Accuracy vs Performance Trade-offs

For each optimization that sacrifices precision, document:

| Field | Description |
|-------|-------------|
| **Current method** | What the code does now |
| **Proposed method** | The faster alternative |
| **Precision impact** | Quantified (e.g., "±0.01% on p95 percentile") |
| **Performance gain** | Quantified (e.g., "eliminates 28k math.Exp calls") |
| **Recommendation** | Accept trade-off / reject / make configurable |

## Workflow

### Phase 1: Hot Path Mapping

1. Identify the primary data flows (ingest → digest → recommend → write → API)
2. Profile or estimate call counts per reconciliation cycle
3. Map `float64` usage sites on the hot path
4. Identify the top 5 most-called functions

### Phase 2: Per-Dimension Analysis

Walk through each of the 11 dimensions above. For each finding:

| Field | Description |
|-------|-------------|
| **ID** | Category prefix + number (e.g., M1, Q3, H-2) |
| **Title** | One-line description |
| **Severity** | P0 (critical) / P1 (high) / P2 (medium) / P3 (low) |
| **Location** | File path and function |
| **Current state** | What the code does now |
| **Proposed fix** | Specific optimization |
| **Expected impact** | Quantified where possible |
| **Risk** | What could break |
| **Effort** | S (hours) / M (days) / L (weeks) |

### Phase 3: ROI Prioritization

Rank findings by `(impact × confidence) / effort` and group:

1. **Quick wins** — High impact, low effort, low risk (do first)
2. **High-value investments** — High impact, moderate effort
3. **Strategic** — Requires architecture changes, high risk, evaluate carefully
4. **Defer** — Low ROI or high risk; document rationale and revisit triggers

### Phase 4: Produce the Report

```markdown
# Performance Audit Report: [Project Name]

## Date and Scope
## Overall Assessment
## What Is Working Well (Do Not Regress)
## Findings by Priority
### P0 — Critical
### P1 — High
### P2 — Medium
### P3 — Low
## Deferred Items — Revisit Triggers
## Accuracy Trade-off Register
## Appendix: Call Count Estimates
```

## Output Location

Save performance audit reports to `docs/performance/` in the primary repository
under review. Use naming convention:
`{component}-audit-{YYYY-MM}.md`

## Prior Art

If a previous performance audit exists (check `docs/performance/`), read it
first. Focus on:

1. **Regressions** — Has any "Working Well" item regressed?
2. **New code** — Audit code added since the last review
3. **Implemented items** — Verify fixes are complete and effective
4. **Deferred items** — Check if revisit triggers have been met

## Integration

After the audit, findings can be implemented using `/properly-implement-feature`.
Each finding becomes a tracked work item with full lifecycle (code, tests, docs,
ADR, performance audit update).

## Guidelines

- **Quantify everything.** "Slow" is not a finding. "28k math.Exp calls per
  reconciliation cycle" is.
- **Measure before proposing.** Estimate call counts, row counts, and memory
  before recommending changes.
- **Respect accuracy requirements.** Financial/billing calculations have strict
  precision needs. Flag trade-offs explicitly.
- **Consider both modes.** On-prem (PostgreSQL-only, single tenant) and SaaS
  (Trino + PostgreSQL, multi-tenant) have different bottlenecks.
- **Check the Do Not Regress list.** If a previous audit exists, verify nothing
  from "What Is Working Well" has been undone.
- **Be specific.** File paths, function names, line numbers. Not "database
  queries could be improved."
