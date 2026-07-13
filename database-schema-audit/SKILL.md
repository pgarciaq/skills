---
name: database-schema-audit
description: >-
  Audit a database schema against production-readiness criteria. Use when
  reviewing models, migrations, or schema PRs, when onboarding to a new
  codebase, or when evaluating whether an AI-generated schema is safe for
  production. Covers indexes, constraints, data types, migrations,
  normalization, soft delete, naming, multi-tenancy, and read/write patterns.
---

# Database Schema Audit

Audit a database schema against the 9-point production-readiness checklist
derived from common failure patterns in AI-generated and hastily designed
schemas. Each item is rated PASS / PARTIAL / FAIL with specific evidence.

## When to Use This

- Reviewing a PR that adds or modifies models, tables, or migrations
- Onboarding to a new codebase and assessing schema maturity
- Evaluating an AI-generated schema before adopting it
- Periodic health check on an existing schema
- Before a major feature that will add significant new tables

## Workflow

### Step 0: Determine Audit Scope

**Full audit** (onboarding, periodic health check, evaluating a new codebase):
Examine all model files, all migration directories, and representative SQL
templates. Count migrations, estimate table count, assess overall maturity.

**PR-scope audit** (reviewing a PR that adds or modifies models/migrations):
Focus the checklist on the changed files, but spot-check existing patterns for
consistency. A new table that uses `FLOAT` for money is a FAIL even if the
rest of the schema uses `NUMERIC`. A new table that follows all existing
conventions is a PASS without re-auditing the entire history.

### Step 1: Identify the Schema Source

Determine where schema definitions live. Common patterns:

| Framework | Where to look |
|-----------|---------------|
| Django | `models.py` files, `migrations/` directories |
| GORM (Go) | Model structs with `gorm:` tags, migration `.sql` files |
| SQLAlchemy | Model classes, Alembic `versions/` |
| Rails | `db/schema.rb`, `db/migrate/` |
| Raw SQL | `CREATE TABLE` statements, migration scripts |
| Prisma | `schema.prisma` |

Search for model definitions, migration directories, and any SQL template
directories.

**Sanity check:** Verify the database engine is appropriate for the workload.
SQLite or localStorage for multi-user production is a FAIL before the
checklist even starts.

### Step 2: Run the 9-Point Checklist

Evaluate each item. For every item, provide a rating and cite specific
evidence (file paths, line numbers, concrete examples).

**Do not guess. Read the actual code.**

---

#### 1. Indexes Beyond Primary Key

**What to check:**
- Are there indexes on columns used in WHERE, JOIN, ORDER BY, GROUP BY?
- Are there composite indexes matching common multi-column query patterns?
- Are there partial indexes for low-cardinality filter columns (e.g., `status`, `type`)?
- Are there GIN indexes on JSONB/array columns?
- Are there covering indexes (INCLUDE) for index-only scans?
- Is there a strategy for index creation on large tables (CONCURRENTLY)?

**Red flags:**
- Only auto-generated PK indexes
- Indexes on every column (over-indexing wastes write performance)
- No indexes on foreign key columns (common in Django where FKs don't auto-index)
- Missing indexes on partitioned table partition keys

**Rating guide:**
- PASS: Indexes clearly designed for query patterns, composite/partial where appropriate
- PARTIAL: Some indexes exist but gaps in coverage or no partial/composite indexes
- FAIL: Only PK indexes or no explicit index definitions

---

#### 2. Foreign Keys and Constraints

**What to check:**
- Are foreign keys defined with explicit `ON DELETE` behavior?
- Are there UNIQUE constraints on natural keys?
- Are there CHECK constraints for domain validation (e.g., `end_date >= start_date`)?
- Is `ON DELETE CASCADE` vs `PROTECT` vs `SET_NULL` intentionally chosen?
- Are there `unique_together` / composite unique constraints where business logic demands them?

**Red flags:**
- ID columns with no FK constraint ("loosely coupled" tables)
- No `ON DELETE` specified (database default varies)
- Missing UNIQUE constraint where business logic requires uniqueness
- No CHECK constraints anywhere

**Rating guide:**
- PASS: FKs with explicit ON DELETE, UNIQUE on natural keys, CHECK where appropriate
- PARTIAL: FKs exist but ON DELETE is inconsistent or missing CHECKs
- FAIL: No FKs, bare ID columns, no constraints beyond PK

---

#### 3. Data Types

**What to check:**
- Are monetary values stored as `Decimal` / `NUMERIC` (not `FLOAT` / `REAL`)?
- Are UUIDs stored as native UUID type (not `VARCHAR(36)`)?
- Are booleans actual boolean fields (not `INT` 0/1 or `VARCHAR` "true"/"false")?
- Are timestamps timezone-aware (`TIMESTAMPTZ`, not bare `TIMESTAMP`)?
- Are string lengths domain-appropriate (not `VARCHAR(255)` for everything)?
- Are JSONB fields used instead of TEXT for structured data?

**Red flags:**
- `VARCHAR(255)` as the default string type everywhere
- `FLOAT` or `REAL` for money (rounding errors)
- `VARCHAR(36)` for UUIDs (wastes space, no native validation)
- Bare `TIMESTAMP` without timezone (ambiguous in multi-region deployments)
- Booleans stored as strings

**Rating guide:**
- PASS: Types match domains, Decimal for money, native UUIDs, TIMESTAMPTZ
- PARTIAL: Mostly correct but some legacy FLOAT/VARCHAR misuse
- FAIL: VARCHAR(255) everywhere, FLOAT for money, no type discipline

---

#### 4. Migration Strategy

**What to check:**
- Is there a migration framework (Django migrations, golang-migrate, Alembic, Flyway, Liquibase)?
- Are migrations numbered sequentially with no gaps?
- Are both up and down migrations provided?
- Is there a documented strategy for risky operations (adding NOT NULL to large tables, index creation on large tables)?
- Are there migration squashes/consolidations for long-lived projects?
- Is there CI validation of migrations?

**Red flags:**
- Only a `CREATE TABLE` dump with no migration tool
- Gaps in migration numbering
- No down/reverse migrations
- No documentation for migration procedures
- Migrations that would lock large tables without warning

**Rating guide:**
- PASS: Migration framework, sequential numbering, documented risky-op strategy
- PARTIAL: Migration framework exists but incomplete (missing downs, no squash strategy)
- FAIL: No migration framework, only raw DDL scripts

---

#### 5. Normalization

**What to check:**
- Is the core data model properly normalized (3NF or intentional deviations)?
- Is denormalization intentional and documented (e.g., summary tables for read performance)?
- Are there "god tables" with 50+ columns mixing unrelated concerns?
- Is there a clear distinction between write-side (normalized) and read-side (denormalized) tables?
- Are junction tables properly structured for many-to-many relationships?

**Red flags:**
- A single table with 100+ columns containing everything
- Comma-separated values in a column instead of a junction table
- Duplicated data across tables with no sync strategy
- Over-normalized schemas requiring 7+ JOINs for basic queries

**Rating guide:**
- PASS: Proper normalization with intentional, documented denormalization
- PARTIAL: Generally normalized but some unintentional duplication or god tables
- FAIL: One god table, CSV-in-columns, or severe over/under-normalization

---

#### 6. Data Lifecycle

The real question is not "do you have soft delete" but "what happens when data
needs to be removed, and can you recover from mistakes?"

**What to check:**
- Is there a deliberate data removal strategy (soft delete, partition drops, TTL, archival)?
- Are there audit columns (`created_at`, `updated_at`, `created_by`)?
- Is there an audit log table for sensitive operations?
- For hard-delete designs: is it intentional and appropriate for the domain?
- Is there a data retention / archival strategy (partition-based retention, time-bucketed purge)?
- Can you answer "what happened to record X?" after it's gone?

**Red flags:**
- Hard DELETE everywhere with no recovery path and no audit trail
- No `created_at` / `updated_at` on any table
- No audit trail for operations that compliance requires tracking
- Soft delete implemented inconsistently (some tables yes, some no)
- No retention strategy on tables that grow unboundedly

**Acceptable hard-delete patterns:**
- Append-only time-series data with partition-based retention (drop old partitions)
- Ephemeral/derived data that can be recomputed from upstream sources
- Data pipelines where records are regularly replaced during ingestion
- Stale flags + history tables as an alternative to soft delete

**Rating guide:**
- PASS: Deliberate lifecycle strategy (soft delete, partition retention, or audit trail)
- PARTIAL: Some audit columns exist but inconsistent coverage or no retention plan
- FAIL: No timestamps, no audit trail, hard delete on compliance-sensitive data

---

#### 7. Naming Conventions

**What to check:**
- Is casing consistent throughout (all `snake_case`, all `camelCase`, etc.)?
- Are table names consistently singular or plural?
- Do junction/map tables follow a naming pattern?
- Do index names follow a convention (`idx_table_column`)?
- Do FK constraint names follow a convention?
- Are column names descriptive (not `data`, `value`, `type` without context)?

**Red flags:**
- Mixed casing: `userId`, `user_id`, `UserID` in the same schema
- Singular and plural table names mixed: `user` and `orders`
- Index names like `idx_1`, `idx_2` with no semantic meaning
- Column names that require reading the code to understand

**Rating guide:**
- PASS: Consistent conventions for tables, columns, indexes, constraints
- PARTIAL: Mostly consistent with some historical inconsistencies
- FAIL: No discernible convention, mixed casing, ambiguous names

---

#### 8. Multi-Tenancy

**What to check:**
- Is there a tenant isolation strategy (schema-per-tenant, row-level, shared)?
- Is tenant filtering enforced at the query layer (not just application code)?
- Are there safeguards against cross-tenant data leaks?
- Is tenant context required before accessing tenant-scoped data?
- For single-tenant apps: is this intentional and documented?

**Red flags:**
- No tenant concept in a multi-user application
- Tenant isolation depends entirely on application-level WHERE clauses
- No RLS (Row Level Security) or schema isolation
- Cross-tenant queries possible without explicit authorization

**Context-dependent:** Single-tenant is fine for:
- Internal tools, personal projects, single-organization deployments
- On-prem software where each deployment is a single tenant

**Rating guide:**
- PASS: Explicit tenant isolation strategy, enforced at DB or middleware layer
- PARTIAL: Tenant column exists but isolation depends on application discipline
- FAIL: Multi-user app with no tenant concept or isolation
- N/A: Genuinely single-tenant application

---

#### 9. Read/Write Pattern Awareness

**What to check:**
- Is there evidence the schema was designed around query patterns (not just entities)?
- Are there partitioning strategies based on access patterns (time-range, tenant)?
- Are there pre-computed summary/aggregate tables for expensive reads?
- Are there materialized views or cache tables?
- Is there keyset/cursor pagination support (vs OFFSET-based)?
- Is there autovacuum/fillfactor tuning for high-churn tables?

**Red flags:**
- Single table serving both OLTP writes and analytical reads
- No partitioning on tables expected to grow to millions of rows
- OFFSET-based pagination on large tables
- No summary tables despite complex aggregation queries in the API
- Schema "optimized for nothing -- just store the data somewhere"
- N+1 query patterns in the application layer (schema enables them by lacking JOINable structure)
- Unbounded queries with no LIMIT or pagination strategy

**Also check the query layer:** The schema can have perfect indexes and
partitioning, but if the application fires N+1 queries, uses `SELECT *`, or
does unbounded `COUNT(*)` on large tables, the schema design is undermined.
Spot-check the ORM usage or query builder code for these patterns.

**Rating guide:**
- PASS: Schema clearly designed around access patterns, partitioning, pre-aggregation
- PARTIAL: Some optimization exists but gaps (e.g., no partitioning on large tables)
- FAIL: Entity-only design with no consideration for query performance

---

### Step 3: Produce the Audit Report

Structure the output as follows:

```markdown
## Database Schema Audit: [project name]

**Schema source:** [framework, location of model definitions]
**Migration count:** [number of migrations]
**Estimated table count:** [number of tables]

### Results

| # | Item | Rating | Key Finding |
|---|------|--------|-------------|
| 1 | Indexes | PASS/PARTIAL/FAIL | [one-line summary] |
| 2 | FK / Constraints | PASS/PARTIAL/FAIL | [one-line summary] |
| 3 | Data Types | PASS/PARTIAL/FAIL | [one-line summary] |
| 4 | Migration Plan | PASS/PARTIAL/FAIL | [one-line summary] |
| 5 | Normalization | PASS/PARTIAL/FAIL | [one-line summary] |
| 6 | Soft Delete / Audit | PASS/PARTIAL/FAIL | [one-line summary] |
| 7 | Naming | PASS/PARTIAL/FAIL | [one-line summary] |
| 8 | Multi-Tenancy | PASS/PARTIAL/FAIL | [one-line summary] |
| 9 | Read/Write Patterns | PASS/PARTIAL/FAIL | [one-line summary] |

**Overall: X PASS, Y PARTIAL, Z FAIL**

### Detailed Findings

[For each non-PASS item, provide:]
- What the gap is
- Specific evidence (file:line references)
- Recommended fix (if applicable)
- Whether the gap is intentional and acceptable

### Actionable Recommendations

[Prioritized list of concrete improvements, if any]
```

## Scoring Guide

| Score | Meaning |
|-------|---------|
| 9 PASS | Production-grade schema, well-engineered |
| 7-8 PASS | Solid schema with minor gaps, likely intentional trade-offs |
| 5-6 PASS | Functional but needs attention before scaling |
| 3-4 PASS | Significant gaps, high risk at scale |
| 0-2 PASS | Prototype-grade, not production-ready |

**Hard blockers regardless of total score:** A FAIL on items 2 (FK/Constraints),
8 (Multi-Tenancy), or 9 (Read/Write Patterns) is a hard blocker for production
readiness. A schema with 8 PASS and a FAIL on multi-tenancy is worse than one
with 5 PASS where the FAILs are naming and soft delete.

**Prioritize recommendations by blast radius:** data loss and security first,
performance second, maintainability third.

## Anti-Patterns

| Anti-Pattern | Why It's Wrong |
|---|---|
| Rating everything PASS without evidence | Read the actual models. Cite files and lines. |
| Failing items that are intentional trade-offs | Hard delete in a time-series pipeline is fine. Document why. |
| Auditing only one model file | Check ALL model files, not just the first one found. |
| Ignoring the migration directory | Migration discipline is as important as model design. |
| Applying RDBMS rules to document stores | This checklist is for relational databases. NoSQL has different criteria. |
| Penalizing denormalization in read-optimized tables | Denormalization is correct when it serves a specific query pattern. |

## Framework Quick Reference

| Framework | Indexes | FKs / Constraints | Types | Migrations |
|---|---|---|---|---|
| Django | `Meta.indexes`, `Meta.constraints` | `on_delete=` on `ForeignKey` | `DecimalField` vs `FloatField`, `UUIDField` | `<app>/migrations/`, `auto_now`/`auto_now_add` |
| GORM | `gorm:"index"`, `gorm:"uniqueIndex"` | `gorm:"constraint:OnDelete:CASCADE"` | `gorm:"type:numeric(10,4)"` | `.sql` files; watch for `AutoMigrate` in prod |
| SQLAlchemy | `Index()`, `UniqueConstraint()` | `ForeignKey(ondelete=)` | `Column(Numeric)` vs `Float` | Alembic `versions/` |
| Prisma | `@@index`, `@@unique` | `@relation(onDelete:)` | `Decimal` vs `Float` | `prisma migrate` history |
| Rails | `add_index` in migrations | `add_foreign_key`, `references` | `decimal` vs `float` | `db/migrate/`, `db/schema.rb` |

## Reference

This checklist is derived from recurring production failures documented by
database reviewers and DBAs, including common patterns in AI-generated schemas
that "run without errors" but fail under production load, compliance
requirements, or multi-tenant access patterns.

Key sources:
- Reddit r/Database: "I review database schemas regularly. AI-generated ones
  have the same problems every time."
- SiteFusion: "How to Make AI-Generated Code Production Ready"
- HelloCrossman: "The Final 10%: What AI Can't Build"
- Production experience with schema-per-tenant (django-tenants), partitioned
  PostgreSQL tables, and dual Trino/PostgreSQL execution paths.
