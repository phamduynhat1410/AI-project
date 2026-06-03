# itms-implementation-plan.prompt.md

Purpose
--
This prompt generates a phased implementation plan for the Intelligent Task Management System (ITMS) project.

When invoked it will
1. Read `doc/frd.md` and `doc/tsd.md` from the repository root.
2. Produce a phased implementation plan using the fixed tech stack: Java / JavaScript / SpringMVC / SQL Server.
3. Reference the project folder structure: `src/routes/`, `src/services/`, `src/repositories/`, `src/models/`.
4. Include database migration tasks following the project's tooling convention (Flyway/Liquibase preference for SQL Server — default: Flyway).
5. Include tasks for OpenAPI (OpenAPI v3) spec generation for backend services.
6. Flag tasks that are suitable for background agent execution.

Output format
--
- Phases should be H2 headers (e.g. "## Phase 1 — Foundation").
- Under each phase output a markdown table with these columns exactly: `ID | Task | Effort | FRD Ref | Parallel? | Background Agent?`

Notes on column meanings
- ID: short unique identifier (e.g. P1-T01).
- Task: concise task description and target paths (use `src/routes/`, `src/services/`, `src/repositories/`, `src/models/`).
- Effort: estimated size (S/M/L/XL or story points).
- FRD Ref: reference to specific sections in `doc/frd.md` or `doc/tsd.md` (e.g. FRD#3.2).
- Parallel?: `yes` or `no` indicating whether the task can run in parallel with other tasks.
- Background Agent?: `yes` or `no` — mark `yes` when a long-running, deterministic, or automatable task is suitable for a background agent (e.g., scaffolding codegen, migrations, OpenAPI generation, bulk data imports, smoke tests).

Guidance for the generator
--
- Prioritize implementing core domain, authentication, and persistence in Phase 1.
- Include explicit tasks for database migrations: create baseline, write migration scripts, run CI migration checks, and add rollback plans.
- For OpenAPI: include tasks to generate OpenAPI spec from code (or generate server stubs), add `api/openapi.yaml` to repo, and add CI validation (lint/format) tasks.
- When referencing files, use repository-relative paths and link to `doc/frd.md` or `doc/tsd.md` sections where appropriate.

Example phase and table (generator should follow this pattern)
--
## Phase 1 — Foundation

| ID | Task | Effort | FRD Ref | Parallel? | Background Agent? |
| --- | --- | ---: | --- | --- | --- |
| P1-T01 | Initialize project skeleton (create `src/routes/`, `src/services/`, `src/repositories/`, `src/models/`) | S | FRD#1.1 | yes | yes |

Invocations & options
--
- This prompt is repository-scoped and does not accept alternate stacks — stack is fixed to Java / JavaScript / SpringMVC / SQL Server.
- Optional boolean flags (for interactive callers): `--prefer-liquibase` to switch migration tooling, `--agent-friendly-only` to return only tasks flagged as background-agent-safe.

Ambiguities to clarify after generation (the prompt should ask user when invoked):
- Preferred migration tool: Flyway (default) or Liquibase?
- CI environment specifics (Azure DevOps, GitHub Actions, Jenkins) for migration and OpenAPI validation tasks.

Examples for maintainers
--
- Example invocation (human): "Generate an implementation plan for ITMS using `doc/frd.md` and `doc/tsd.md`."
- Example generator instruction: "Read `doc/frd.md` and `doc/tsd.md`. Produce phases with tables as specified. Flag long-running automated tasks as Background Agent = yes. Use Flyway unless `--prefer-liquibase` is provided."

Review checklist for prompt authors
--
- Confirm the prompt reads `doc/frd.md` and `doc/tsd` at runtime.
- Ensure output tables use the exact column headings requested.
- Validate that background-agent-eligible tasks are marked `yes`.

End
