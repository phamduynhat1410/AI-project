---
title: Functional Requirements Document — Intelligent Task Management System
version: 1.0
author: FRD Author
date: 2026-06-03
---

# 1. Overview & Purpose

This FRD defines functional requirements, use cases, user stories, validation rules, notifications, and error handling for the Intelligent Task Management System (ITMS). It maps functional requirements to the BRD and provides precise acceptance criteria to guide implementation and testing.

# 2. User Roles & Permissions Matrix

Roles: Developer, Team Lead, Project Manager, QA Engineer

| Feature / Permission | Developer | Team Lead | Project Manager | QA Engineer |
|---|---:|---:|---:|---:|
| Create Task | ✅ | ✅ | ✅ | ✅ |
| Edit Task | ✅ (own/assigned) | ✅ | ✅ | ✅ (for testing) |
| Assign / Reassign Task | ❌ | ✅ | ✅ | ❌ |
| Declare Dependencies | ✅ | ✅ | ✅ | ❌ |
| Remove Dependencies | ✅ | ✅ | ✅ | ❌ |
| Change Status | ✅ | ✅ | ✅ | ✅ |
| View All Tasks | ✅ (project) | ✅ | ✅ | ✅ |
| Filter / Search Tasks | ✅ | ✅ | ✅ | ✅ |
| View Task History | ✅ | ✅ | ✅ | ✅ |
| Access Reports / Project Summary | ❌ | ✅ | ✅ | ✅ |
| Manage Users (create/delete) | ❌ | ❌ | ✅ | ❌ |

Notes:
- `Developer` may assign tasks only when explicitly granted assignment permission by Project Manager; otherwise they can assign to themselves or update own tasks.
- `Team Lead` and `Project Manager` have elevated permissions for assignment and reporting.
- `QA Engineer` has testing and read privileges; may change status for defect workflows per project policy.

# 3. Use Cases

UC-001: Task Creation (Actor: Developer, Team Lead, Project Manager, QA Engineer)

- Preconditions:
  - Actor is authenticated and authorized (A1, BR-R-004).
  - Project context exists.

- Normal Flow:
  1. Actor selects "Create Task".
  2. Actor provides Title, Description, Priority, Estimated Completion Date, optional Assignee, and optional Project.
  3. System validates inputs (see Section 7).
  4. System creates task, generates unique `Task ID`, persists record (BR-F-001, BR-R-001).
  5. System returns created task and persists a creation history entry.

- Alternative Flows:
  - A1: Validation fails -> system returns validation errors (see Section 9).

- Postconditions:
  - Task exists in DB with status `To Do` unless explicit status provided.
  - `TASK_HISTORY` contains creation entry (BR-F-004).

- Business Rules Referenced: BR-F-001, BR-R-001, BR-R-003

UC-002: Task Assignment (Actor: Team Lead, Project Manager, Developer [if permitted])

- Preconditions:
  - Actor is authorized to assign tasks (BR-R-004).
  - Target assignee exists in `USERS` (A1).

- Normal Flow:
  1. Actor opens task and selects "Assign/Reassign".
  2. Actor selects assignee.
  3. System validates assignee and records assignment change in `TASK_HISTORY` (BR-F-002).
  4. System emits assignment notification/event to new assignee (see Section 8).

- Alternative Flows:
  - A1: Assignee invalid -> error returned.
  - A2: Actor lacks permission -> unauthorized error returned.

- Postconditions:
  - Task `assignee_id` updated; history entry recorded; notification sent.

- Business Rules Referenced: BR-F-002, BR-R-004

UC-003: Task Dependency Management (Actor: Developer, Team Lead, Project Manager)

- Preconditions:
  - Actor authenticated and authorized.
  - Both tasks exist and belong to same project or allowed cross-project dependency per policy.

- Normal Flow:
  1. Actor adds one or more dependencies to a task (DependsOn list).
  2. System validates no circular dependency (TR-001 mitigation, BR-R-002).
  3. System persists `TASK_DEPENDENCIES` entries.
  4. If dependency targets are not `Completed`, system marks dependent task `Blocked` and emits a dependency-blocked notification (BR-F-003).

- Alternative Flows:
  - A1: Circular dependency detected -> system rejects with informative error.
  - A2: Dependency removed -> system re-evaluates and clears `Blocked` status if all dependencies are `Completed`.

- Postconditions:
  - `TASK_DEPENDENCIES` updated; dependent task status accurate; history entries created.

- Business Rules Referenced: BR-F-003, BR-R-002

UC-004: Task Status Tracking (Actor: Developer, Team Lead, Project Manager, QA Engineer)

- Preconditions:
  - Actor authorized to update status per role.

- Normal Flow:
  1. Actor selects new status for a task (To Do, In Progress, Blocked, Completed).
  2. System validates transition (e.g., cannot set `Completed` without re-open process if rule applies BR-R-005).
  3. System updates `status`, records `TASK_HISTORY` with actor, timestamp, and optional comment (BR-F-004, BR-R-003).
  4. System emits status-change notification/event (see Section 8).

- Alternative Flows:
  - A1: Invalid transition (e.g., setting to `In Progress` while dependencies incomplete) -> system rejects or sets `Blocked` with note.

- Postconditions:
  - Task status and history updated; reporting metrics reflect change within SLA.

- Business Rules Referenced: BR-F-004, BR-R-003, BR-R-002

UC-005: Task Listing and Filtering (Actor: All roles)

- Preconditions:
  - Actor authenticated and authorized to view project tasks.

- Normal Flow:
  1. Actor requests task list with optional filters (status, priority, assignee, due date range, search query).
  2. System applies filters, returns paginated results and total counts (BR-F-005).

- Alternative Flows:
  - A1: Invalid filter params -> system returns validation error.

- Postconditions:
  - Actor receives filtered list; UI displays counts and supports pagination.

- Business Rules Referenced: BR-F-005

UC-006: Project Progress Summary (Actor: Project Manager, Team Lead, QA Engineer)

- Preconditions:
  - Project exists; service aggregation pipelines are operational.

- Normal Flow:
  1. Actor requests project summary.
  2. System computes totals: Total Tasks, Completed, In Progress, Blocked, Pending and returns metrics with last_updated timestamp (BR-F-006).
  3. System refreshes summary data within near-real-time SLA (within 60s) via event-driven updates.

- Alternative Flows:
  - A1: Reporting service lag -> system returns last known summary with `last_updated` and a warning.

- Postconditions:
  - Summary returned and available for export.

- Business Rules Referenced: BR-F-006

# 4. User Stories (Gherkin) — mapped to Use Cases

UC-001: Task Creation

US-001: Create a task
As a Developer
I want to create a task with title, description, priority and optional assignee
So that I can record work to be done

Given I am an authenticated user with permission to create tasks
When I submit a task with valid title, priority and optional assignee
Then the system creates the task, returns a unique Task ID, and records a creation history entry

UC-002: Task Assignment

US-002: Assign a task
As a Team Lead
I want to assign a task to a team member
So that the assignee is responsible for completion

Given I am an authenticated Team Lead
When I assign a valid assignee to an existing task
Then the system updates the task's assignee, records an assignment history entry, and emits an assignment notification to the new assignee

UC-003: Task Dependency Management

US-003: Add dependencies
As a Developer
I want to declare that Task B depends on Task A
So that Task B is blocked until Task A is completed

Given Task A and Task B exist and I have permission to modify Task B
When I add Task A as a dependency of Task B
Then the system validates no circular dependencies, persists the dependency, and marks Task B as Blocked if Task A is not Completed

UC-004: Task Status Tracking

US-004: Update task status
As a Developer
I want to change a task's status to In Progress or Completed
So that the task lifecycle is tracked

Given I am authenticated and authorized to change the task status
When I set a new valid status for a task
Then the system validates the transition, updates the task status, stores a history entry, and emits a status-change event

UC-005: Task Listing and Filtering

US-005: Filter tasks
As a Project Member
I want to filter tasks by status, priority, assignee, and date range
So that I can focus on a specific subset of tasks

Given I am authenticated and request tasks with filters
When I submit valid filter parameters
Then the system returns a paginated list of matching tasks and the total count

UC-006: Project Progress Summary

US-006: Get project summary
As a Project Manager
I want to obtain a project progress summary (Total, Completed, In Progress, Blocked, Pending)
So that I can assess project risk and progress

Given I am an authorized Project Manager
When I request the project summary
Then the system returns the metrics and last_updated timestamp within 60s of recent changes or indicates a lag

# 5. Functional Requirements Catalogue

| FR-ID | Description | Priority | BRD Ref | Status |
|---|---|---:|---:|---:|
| FR-F-001 | Task Creation with attributes and persistence | High | BR-F-001 | Defined |
| FR-F-002 | Task Assignment and reassignment with history | High | BR-F-002 | Defined |
| FR-F-003 | Task Dependency modeling and Blocked propagation | High | BR-F-003 | Defined |
| FR-F-004 | Task Status tracking and history audit | High | BR-F-004 | Defined |
| FR-F-005 | Task Listing, Filtering, Pagination, and Search | High | BR-F-005 | Defined |
| FR-F-006 | Project Progress Summary with near-real-time updates | High | BR-F-006 | Defined |
| FR-NF-001 | Modular architecture and testable services | High | BR-NF-001 | Defined |
| FR-NF-002 | Unit test coverage for core logic | High | BR-NF-002 | Defined |
| FR-NF-003 | API documentation (OpenAPI) | High | BR-NF-003 | Defined |
| FR-NF-004 | CI pipeline for build & tests | Medium | BR-NF-004 | Defined |

# 6. Data Requirements & Validation Rules

Fields and validation rules:

- Task ID (`task.id`)
  - Type: string
  - Format: `T` followed by 3-6 digits (regex: `^T\d{3,6}$`) or UUID in APIs (both accepted internally but UI should show `T###` format)
  - Required: yes
  - Uniqueness: globally unique (BR-R-001)

- Title (`task.title`)
  - Type: string
  - Required: yes
  - Length: 3–200 characters
  - Disallowed: only whitespace
  - Validation: trim and collapse whitespace; escape output in UI

- Description (`task.description`)
  - Type: text
  - Required: no
  - Length: 0–5000 characters
  - Sanitization: HTML input sanitized; stored as plain text or safe markdown

- Priority (`task.priority`)
  - Type: enum
  - Allowed values: `Low`, `Medium`, `High`
  - Required: yes (default `Medium`)

- Status (`task.status`)
  - Type: enum
  - Allowed values: `To Do`, `In Progress`, `Blocked`, `Completed`
  - Required: yes (default `To Do`)
  - Transition rules: see status transition table below

- Assignee (`task.assignee_id`)
  - Type: string (user id)
  - Required: no
  - Validation: must reference existing `USERS.id`; assigning to self allowed

- Estimated Completion Date (`task.estimated_completion_date`)
  - Type: date (ISO 8601 `YYYY-MM-DD`)
  - Required: no
  - Validation: must be a valid date; may be in past but system should warn if set earlier than created date

- Created/Updated timestamps
  - Type: datetime (UTC)
  - System-populated

Status transition rules (examples):
- `To Do` -> `In Progress` allowed if no blocking dependencies exist; otherwise system may set `Blocked`.
- `In Progress` -> `Completed` allowed.
- `Completed` -> requires explicit Reopen action to set `To Do` or `In Progress` (BR-R-005).

# 7. Notification Triggers & Content

Notifications are emitted as events (webhooks/queue) and may be delivered via configured channels (email, in-app, Slack). The FRD specifies triggers, recipients, and message templates.

- Trigger: Task Reassignment
  - When: a task's `assignee_id` changes
  - Recipient: new assignee (primary), previous assignee (CC optional), project manager (optional)
  - Content (short): "Task {Task ID}: '{Title}' assigned to you by {Actor}."
  - Content (detailed): includes task link, description snippet, priority, due date, and action links (view/acknowledge)
  - Trace: FR-F-002

- Trigger: Dependency Blocking
  - When: a dependency is added or an existing dependency is not Completed causing a dependent task to be `Blocked`
  - Recipient: task assignee, task creator, project manager
  - Content: "Task {Task ID}: '{Title}' is Blocked due to incomplete dependency {DependsOn Task ID} ({DependsOn Title})."
  - Trace: FR-F-003

- Trigger: Dependency Clearance
  - When: all dependencies for a blocked task become `Completed`
  - Recipient: task assignee, project manager
  - Content: "Task {Task ID}: '{Title}' is now unblocked; all dependencies are complete."

- Trigger: Status Change
  - When: task status changes
  - Recipient: task assignee (if changed by someone else), project manager, previous assignee (on reassignment+status change)
  - Content: "Task {Task ID}: '{Title}' status changed from {OldStatus} to {NewStatus} by {Actor}."
  - Additional: include optional comment supplied by actor
  - Trace: FR-F-004

- Trigger: Overdue Task (reporting)
  - When: current date > estimated_completion_date and status != Completed
  - Recipient: assignee, project manager
  - Content: "Task {Task ID}: '{Title}' is overdue (est: {Date})."
  - Trace: FR-F-006, BO-003

Delivery considerations:
- Events should include a `correlation_id` for tracing.
- Notifications are emitted reliably (retry/backoff) but channel delivery is implementation-specific (A2).

# 8. Error Scenarios & User-Facing Messages

Validation errors (client-visible):
- ERR-VAL-001: Missing required field
  - Message: "{field} is required. Please provide a valid {field}."
- ERR-VAL-002: Invalid Task ID format
  - Message: "Task ID must match format 'T###' or be a valid UUID."
- ERR-VAL-003: Title length invalid
  - Message: "Title must be between 3 and 200 characters."
- ERR-VAL-004: Invalid priority
  - Message: "Priority must be one of: Low, Medium, High."
- ERR-VAL-005: Invalid status
  - Message: "Status must be one of: To Do, In Progress, Blocked, Completed."
- ERR-VAL-006: Invalid date
  - Message: "Please provide a valid date in YYYY-MM-DD format."
- ERR-VAL-007: Assignee not found
  - Message: "Assignee not found. Please select a valid team member."

Authorization & access errors:
- ERR-AUTH-001: Unauthorized
  - Message: "You are not authorized to perform this action. Contact your project manager if you need access."

Dependency errors:
- ERR-DEP-001: Circular dependency detected
  - Message: "Cannot add dependency: circular dependency detected involving {taskIds}."
- ERR-DEP-002: Dependency removal conflict
  - Message: "Cannot remove dependency: operation would violate project rules."

Operational errors:
- ERR-SYS-001: Service unavailable
  - Message: "Service temporarily unavailable. Please try again later. If the problem persists, contact support."
- ERR-SYS-002: Database error
  - Message: "An internal error occurred while saving the task. Please retry. If the issue continues, contact support."

API error contract example:
```json
{
  "error_code": "ERR-VAL-003",
  "message": "Title must be between 3 and 200 characters.",
  "details": []
}
```

# 9. Traceability (FR -> BR Mapping)

- FR-F-001 -> BR-F-001
- FR-F-002 -> BR-F-002
- FR-F-003 -> BR-F-003
- FR-F-004 -> BR-F-004
- FR-F-005 -> BR-F-005
- FR-F-006 -> BR-F-006
- FR-NF-001 -> BR-NF-001
- FR-NF-002 -> BR-NF-002
- FR-NF-003 -> BR-NF-003

# 10. Acceptance & Testability Notes

- Each FR includes acceptance criteria in the use cases and user stories above.
- Tests must include unit tests for core logic (dependency propagation, status transitions), integration tests for API endpoints, and end-to-end tests for key flows (create->assign->block->complete).
- CI must run tests on PRs and block merges on failures (FR-NF-002, FR-NF-004).

---

Revision History

- 1.0 — 2026-06-03 — Initial FRD created from `requirement.md`.
