<!--
SYNC IMPACT REPORT
==================
Version change: (none) → 1.0.0
Status: Initial ratification — no prior constitution existed.

Added sections:
  - § 1. Project Identity
  - § 2. Product Purpose
  - § 3. Architecture Pillars
  - § 4. Technology Stack & Standards
  - § 5. Development Commandments (4 principles)
  - § 6. Global Acceptance Criteria
  - § 7. Governance

Modified principles: N/A (initial version)
Removed sections: N/A (initial version)

Templates status:
  ⚠ .specify/templates/plan-template.md    — pending creation
  ⚠ .specify/templates/spec-template.md    — pending creation
  ⚠ .specify/templates/tasks-template.md   — pending creation

Deferred TODOs:
  - TODO(RATIFICATION_DATE): Exact adoption date unknown; set to first commit date
    once repository is initialized. Currently set to 2026-03-24 (today).
-->

# Project Constitution — Grocery Guard (Lista de Compras)

**Version:** 1.0.0
**Ratification Date:** 2026-03-24
**Last Amended:** 2026-03-24
**Repository:** https://github.com/PabloAndresAcosta/listadecompras

---

## 1. Project Identity

| Field | Value |
|---|---|
| Product Name | Grocery Guard |
| Tagline | Elimina el olvido en el hogar |
| Primary Audience | Household members doing grocery shopping |
| Primary Environment | Mobile, in-store (supermarket), potentially unstable network |

---

## 2. Product Purpose

Grocery Guard MUST be the definitive tool for household grocery list management,
optimized for supermarket use: mobility, speed, and visual clarity.

The system MUST eliminate the "I forgot to buy X" problem by providing a fast,
always-available list that works reliably under real-world supermarket conditions
(intermittent connectivity, one-hand operation, bright lighting).

---

## 3. Architecture Pillars

### 3.1 Decoupled Architecture

The system MUST maintain a strict separation between backend and frontend.
The backend is a self-contained, robust API service. The frontend is a thin client
that renders what the API returns. Neither layer MUST be tightly coupled to the
internal implementation of the other.

**Rationale:** Decoupling allows independent deployment, scaling, and replacement
of either layer without affecting the other.

### 3.2 Centralized Business Logic

The frontend MUST NOT contain business logic, validation rules, or calculated
state. All logic, constraints, and decisions MUST reside exclusively in the API.
The client is a presentation layer only.

**Rationale:** A single source of truth prevents divergence bugs and ensures
consistency across any future client (web, mobile, CLI).

### 3.3 Operational Transparency

The system MUST NOT be a black box. Every transaction flow, decision branch, and
error condition MUST produce a structured log entry in the console. A developer
MUST be able to diagnose any production incident by reading the log output alone,
without attaching a debugger.

**Rationale:** Fast diagnosis reduces MTTR and lowers the barrier for new
contributors joining the project.

---

## 4. Technology Stack & Standards

### 4.1 Backend

| Concern | Standard |
|---|---|
| Language | Java or Golang (team decision; document choice in README) |
| API style | RESTful, JSON responses |
| Contract | OpenAPI / Swagger documentation — MUST be kept current with the code |
| Error handling | Global exception handler — MUST return standardized error envelope |
| HTTP status codes | MUST use precise codes (200, 201, 400, 404, 422, 500…) |
| Logging | Structured console logs — every request/response cycle, every exception with simplified stack trace |

### 4.2 Frontend

| Concern | Standard |
|---|---|
| Framework | React, Vue, or equivalent component-based framework |
| State management | Minimal; server state is the authority — prefer query/cache libraries (e.g., TanStack Query, SWR) over custom stores |
| UX constraint | MUST be operable with one hand; critical actions reachable without scrolling |
| Performance | Initial load MUST be fast on a 3G connection; perceived latency MUST be masked with optimistic UI where appropriate |

---

## 5. Development Commandments

### Commandment I — KISS (Keep It Simple, Stupid)

If a proposed solution requires more than the minimum components to solve the
problem, it MUST be rejected or simplified. Complexity is a liability.

- Every abstraction MUST justify its existence by removing duplication or
  encapsulating a genuine invariant.
- A new dependency MUST be approved by the team; default answer is "no".

**Test:** Can a developer who did not write the code understand the flow in under
30 minutes? If not, simplify.

### Commandment II — Friction-Free Diagnostics

Any failure condition MUST produce a log entry that identifies: what failed, why
it failed, and where in the code it failed — without requiring a debugger session.

- Exceptions MUST be caught at the global handler and logged with context.
- Silent failures (swallowed exceptions, empty catch blocks) are PROHIBITED.

**Test:** Pick any error scenario. Read the console log. Can you file a bug report
without opening the source file? If not, improve the log.

### Commandment III — Sacred Contract (API as Single Source of Truth)

The API contract (OpenAPI spec) MUST be updated before or alongside any change to
backend behavior. The frontend MUST consume the API contract; it MUST NOT hard-code
assumptions about backend behavior.

- Breaking changes to the API MUST be versioned (e.g., `/v2/`).
- Frontend validation MUST mirror backend validation, never replace it.

**Test:** Delete the frontend. Does the API still enforce all business rules
correctly? If not, move the logic to the backend.

### Commandment IV — Maintainability over Novelty

Stable, well-documented libraries and established patterns MUST be preferred over
experimental or cutting-edge alternatives.

- A technology adoption MUST be justified by a concrete, present problem it solves —
  not by novelty or future speculation.
- Code MUST be written to be read by the next developer, not to impress reviewers.

**Test:** Would a mid-level developer familiar with the chosen language/framework
be able to contribute a bug fix on their first day? If not, simplify.

---

## 6. Global Acceptance Criteria

Every feature shipped MUST satisfy all three criteria before release:

1. **Onboarding Speed:** A new developer MUST be able to understand the full data
   flow of any feature in under 30 minutes by reading the code and logs.

2. **Graceful Degradation:** The system MUST handle network and database errors by:
   - Informing the user with a clear, non-technical message.
   - Logging the technical error (including context) to the console.
   - Never leaving the user in an inconsistent or silent failure state.

3. **Network Resilience:** The application MUST remain functional (read operations
   at minimum) under unstable network conditions typical of a supermarket environment
   (high latency, intermittent drops, slow 3G).

---

## 7. Governance

### 7.1 Amendment Procedure

1. Any team member MAY propose a constitution amendment via a Pull Request to
   `.specify/memory/constitution.md`.
2. The PR description MUST include: motivation, affected principles, and migration
   notes for existing code.
3. Amendments require approval from at least one other contributor before merging.
4. After merging, dependent templates (plan, spec, tasks) MUST be reviewed for
   consistency within the same sprint.

### 7.2 Versioning Policy

This document follows Semantic Versioning:

| Change type | Version bump |
|---|---|
| Principle removed or fundamentally redefined | MAJOR (X.0.0) |
| New principle or section added | MINOR (1.X.0) |
| Clarification, wording, typo fix | PATCH (1.0.X) |

### 7.3 Compliance Review

- Constitution alignment MUST be verified at the start of each feature planning
  session (check `.specify/templates/plan-template.md`).
- Any code review MAY cite a constitution commandment as grounds for requesting
  changes.
- Violations MUST be logged as technical debt items, not silently merged.
