---
title: Implementation Plan — Intelligent Task Management System (ITMS)
version: 1.0
author: Project Plan Generator
date: 2026-06-03
---

# Implementation Plan (Phased)

TL;DR: Phased rollout from project scaffolding → auth/user → core task flows → reporting → notifications → periodic reporting → testing/hardening. Prioritize Data Integrity (Dependency Engine) and 60s reporting SLA early.

## Overview
This document lists phases, tasks, effort estimates, traceability to FRD/US IDs, parallelism guidance, and background-agent suitability for implementing ITMS.

Relevant reference docs:
- `doc/frd.md` — Functional Requirements, Use Cases, User Stories
- `doc/tsd.md` — Technical Specification, Architecture, Platform constraints (Azure)

Verification & Gates
- Unit tests for core logic (dependency handling, status transitions)
- Integration tests for API endpoints (create, assign, dependencies, report)
- Performance check to validate project-summary freshness (<60s)
- CI runs lint/tests/SCA on PRs and blocks merges

Assumptions
- Target platform: Azure (Azure SQL, Service Bus, Key Vault)
- Backend: .NET 7 recommended (Node/TypeScript optional)
- Use IaC (Terraform/Bicep) to provision environments

---

# Phases & Tasks

## Phase 0: Project setup, folder structure, database migration tooling, CI skeleton
- T-001: Repo init & folder scaffold
  - Effort: S (<=4h)
  - FRD Ref: FR-NF-001
  - Sequence: Sequential
  - Background Agent: No
  - Notes: Create standard repo layout for backend, frontend, infra, docs, tests. Add README, CODEOWNERS.

- T-002: DB migration tooling (Flyway/Liquibase)
  - Effort: M (4-8h)
  - FRD Ref: FR-NF-001
  - Sequence: Sequential
  - Background Agent: Yes
  - Notes: Add Flyway/Liquibase config, initial schema migration scripts for USERS, TASKS, TASK_HISTORY, TASK_DEPENDENCIES, PROJECTS.

- T-003: CI skeleton (PR checks: lint, build, unit tests)
  - Effort: M
  - FRD Ref: FR-NF-004
  - Sequence: Sequential
  - Background Agent: No
  - Notes: GitHub Actions workflows for PR lint/build/test; integrate Dependabot/SCA.

- T-004: Dev infra provisioning (Terraform/Bicep stub)
  - Effort: L (8-16h)
  - FRD Ref: FR-NF-001
  - Sequence: Can run parallel with T-002
  - Background Agent: Yes
  - Notes: IaC stubs for Azure SQL, Service Bus, Key Vault, AKS/App Service.

- T-005: API contract skeleton (OpenAPI stub)
  - Effort: S
  - FRD Ref: FR-NF-003
  - Sequence: Sequential
  - Background Agent: No
  - Notes: Place `api/openapi.yaml` with endpoint placeholders for auth, users, tasks, reports.

## Phase 1: Authentication & User Management
- T-010: Integrate Azure AD Auth & token validation
  - Effort: M
  - FRD Ref: BR-R-004, FR-NF-001
  - Sequence: Sequential
  - Background Agent: No
  - Notes: JWT validation middleware, RBAC claims processing.

- T-011: User Service scaffold & user model
  - Effort: S
  - FRD Ref: BR-R-004
  - Sequence: Parallel with T-010
  - Background Agent: No
  - Notes: Implement USERS table mapping, DTOs, repository.

- T-012: User CRUD APIs (list, get)
  - Effort: M
  - FRD Ref: BR-R-004, US-002
  - Sequence: Sequential after T-011
  - Background Agent: No

- T-013: Role & Permissions matrix enforcement
  - Effort: M
  - FRD Ref: BR-R-004
  - Sequence: Sequential
  - Background Agent: No

- T-014: Local/dev auth support (dev login & test accounts)
  - Effort: S
  - FRD Ref: Assumptions A1
  - Sequence: Parallel
  - Background Agent: Yes

## Phase 2: Task Management (core)
- T-020: Task Service schema & repository
  - Effort: M
  - FRD Ref: FR-F-001, BR-R-001
  - Sequence: Sequential (after Phase 0 DB)
  - Background Agent: No
  - Notes: Implement TASKS, TASK_HISTORY, TASK_DEPENDENCIES, PROJECTS tables and ORM mappings.

- T-021: Create Task API & validation
  - Effort: M
  - FRD Ref: FR-F-001, US-001
  - Sequence: Sequential after T-020
  - Background Agent: No
  - Notes: Input validation, generate Task ID, persist, history entry.

- T-022: Update Task (PATCH) & history
  - Effort: M
  - FRD Ref: FR-F-004, US-004
  - Sequence: Sequential
  - Background Agent: No

- T-023: Task Assignment endpoint & notifications event
  - Effort: M
  - FRD Ref: FR-F-002, US-002
  - Sequence: Sequential
  - Background Agent: Yes
  - Notes: Emit assignment event to Service Bus; record in history.

- T-024: Task Listing & Filtering endpoint with pagination
  - Effort: M
  - FRD Ref: FR-F-005, US-005
  - Sequence: Parallel with T-021/T-022
  - Background Agent: No

- T-025: Dependency API (add/remove) with circular dependency validation
  - Effort: L
  - FRD Ref: FR-F-003, US-003
  - Sequence: Sequential (depends on T-020/T-021)
  - Background Agent: Yes
  - Notes: DAG validation, persist TASK_DEPENDENCIES, set `Blocked` status when needed.

- T-026: Status transition rules & server-side enforcement
  - Effort: M
  - FRD Ref: FR-F-004, BR-R-005
  - Sequence: Sequential
  - Background Agent: No

- T-027: Task history retrieval API
  - Effort: S
  - FRD Ref: FR-F-004
  - Sequence: Parallel
  - Background Agent: No

## Phase 3: Task Reporting & Progress Summary
- T-030: Reporting Service scaffold (aggregator)
  - Effort: M
  - FRD Ref: FR-F-006, FR-NF-001
  - Sequence: Sequential after Task Service & Dependency Engine
  - Background Agent: Yes

- T-031: Project summary API with near-real-time design
  - Effort: L
  - FRD Ref: FR-F-006, US-006
  - Sequence: Sequential
  - Background Agent: Yes
  - Notes: Event-driven aggregates via Service Bus; cache in Redis/managed cache; ensure <60s freshness.

- T-032: Metrics & telemetry integration (App Insights)
  - Effort: S
  - FRD Ref: FR-NF-003
  - Sequence: Parallel
  - Background Agent: No

- T-033: Overdue and custom reporting endpoints
  - Effort: M
  - FRD Ref: BO-003, FR-F-006
  - Sequence: Parallel
  - Background Agent: No

- T-034: Export summary (CSV) API stub
  - Effort: S
  - FRD Ref: FR-F-006
  - Sequence: Parallel
  - Background Agent: Yes

## Phase 4: Notifications (email, Teams)
- T-040: Notification service scaffold (Service Bus consumer)
  - Effort: M
  - FRD Ref: FR-F-002, FR-F-003, FR-F-004
  - Sequence: Sequential after events emitted
  - Background Agent: Yes

- T-041: Implement assignment notification (SendGrid/email)
  - Effort: M
  - FRD Ref: FR-F-002
  - Sequence: Parallel with T-040
  - Background Agent: Yes

- T-042: Implement dependency-blocked/unblocked notifications (Teams webhooks)
  - Effort: M
  - FRD Ref: FR-F-003
  - Sequence: Parallel
  - Background Agent: Yes

- T-043: Status-change notification templates & delivery retries
  - Effort: M
  - FRD Ref: FR-F-004
  - Sequence: Parallel
  - Background Agent: Yes

- T-044: Notification configuration UI / settings (enable/disable channels)
  - Effort: L
  - FRD Ref: FR-F-002, FR-F-003
  - Sequence: Parallel
  - Background Agent: No

## Phase 5: Periodic Reporting & Exports
- T-050: Monthly report generator (background job)
  - Effort: L
  - FRD Ref: FR-F-006, BO-003
  - Sequence: Sequential after Reporting service stable
  - Background Agent: Yes

- T-051: Scheduled exports API & permissions
  - Effort: M
  - FRD Ref: FR-F-006
  - Sequence: Parallel
  - Background Agent: Yes

- T-052: Report delivery (email/teams)
  - Effort: M
  - FRD Ref: BO-003
  - Sequence: Parallel
  - Background Agent: Yes

## Phase 6: Testing, security hardening, and documentation
- T-060: Unit test completion & coverage enforcement
  - Effort: L
  - FRD Ref: FR-NF-002
  - Sequence: Parallel across modules
  - Background Agent: No

- T-061: Integration & E2E tests (API flows)
  - Effort: L
  - FRD Ref: FR-F-001..006, US-001..006
  - Sequence: Sequential (after implementation)
  - Background Agent: Yes

- T-062: OWASP Top 10 remediation & security scans
  - Effort: L
  - FRD/TSD Mapping: Security section
  - Sequence: Parallel
  - Background Agent: No

- T-063: Performance & load tests for summary SLA
  - Effort: L
  - FRD Ref: FR-F-006, FR-NF-005
  - Sequence: Sequential
  - Background Agent: Yes

- T-064: OpenAPI documentation finalization & publishing
  - Effort: M
  - FRD Ref: FR-NF-003
  - Sequence: Parallel
  - Background Agent: No

- T-065: Developer & ops documentation (runbooks, run scripts)
  - Effort: M
  - FRD Ref: FR-NF-001
  - Sequence: Parallel
  - Background Agent: No

---

# Sequencing Notes
- Critical path: T-020 (Task DB) → T-025 (Dependency Engine) → T-031 (Reporting aggregator). Complete early to meet 60s SLA.
- Notifications and reporting tasks are good Background Agent candidates (long-running, idempotent, retryable).
- CI and infra provisioning (Phase 0) should be in place before heavy implementation to enable safe PR validation.

# Next steps (options)
- Expand any phase into a detailed checklist with subtasks and file targets.
- Generate skeleton code, OpenAPI YAML, or GitHub Actions workflows for Phase 0.
- Convert Background Agent tasks into runnable job specs (worker code + retry rules).

