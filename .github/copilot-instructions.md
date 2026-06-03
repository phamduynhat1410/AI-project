# Workspace Coding Standards — Java / JavaScript / SpringMVC / SQL Server

This file defines workspace-wide conventions and standards for the project. Follow these rules in every commit.

## 1. Language & Framework Conventions
- Java (Spring MVC / Spring Boot):
  - Package naming: `com.company.project.<module>` (all lowercase).
  - Class names: PascalCase (e.g., `TaskService`, `TaskController`).
  - Method names: camelCase.
  - Controllers: annotate with `@RestController`, route paths under `/api/v1/<resource>`.
  - DTOs: immutable where possible; use builders for complex objects.
  - Layering: `controller -> service -> repository` per module.
- JavaScript (Node/Frontend):
  - Backend (Node/Express): `src/controllers`, `src/services`, `src/models`, `src/routes`, `src/middleware`.
  - Frontend (React): `src/components`, `src/pages`, `src/services`, `src/hooks`.
  - File names: kebab-case for files (e.g., `task-service.js`), camelCase for exported identifiers.
- Module organization:
  - Group files by feature/module, not by type when practical (feature-first layout).

## 2. API Design Rules
- Base path and versioning: always use `/api/v1/` for initial releases. Bump to `/api/v2/` for breaking changes.
- Response envelope (all APIs):

```json
{
  "success": true|false,
  "data": <object|null>,
  "error": { "code": "ERR-XXX", "message": "Human readable" } | null,
  "meta": { "requestId": "<id>", "pagination": { "page":1, "size":25, "total": 123 } }
}
```

- Pagination: use `page` and `size` query params; include `meta.pagination` in envelope.
- Use consistent HTTP methods: GET (read), POST (create), PUT/PATCH (update), DELETE (delete).

## 3. Error Handling
- Define custom error classes (Java: `ApiException`, `NotFoundException`, `ValidationException`; JS: `ApiError` subclasses).
- Centralize error handling:
  - Java: use `@ControllerAdvice` with `@ExceptionHandler` to map exceptions to HTTP responses and envelope.
  - Node: use a centralized error-handling middleware that formats the response envelope.
- Map errors to proper HTTP status codes: 400 (validation), 401 (unauthenticated), 403 (unauthorized), 404 (not found), 409 (conflict), 422 (unprocessable), 500 (server error).

## 4. Security
- Validate all input using schema validators:
  - Java: `javax.validation` / Hibernate Validator annotations on DTOs.
  - JS: `Joi` or `Ajv` for request payload validation.
- Never trust user input. Always sanitize/escape outputs shown in UIs.
- Database queries: use parameterized queries / prepared statements only. No string concatenation for SQL.
- Use HTTPS; validate JWT tokens on every request at the API gateway or controller level.

## 5. Database
- Use a migration tool for schema changes (preferred: Flyway or Liquibase for SQL Server).
- Use an ORM or query builder; no raw string SQL in application code:
  - Java: JPA/Hibernate, MyBatis or Spring Data JPA.
  - Node: Knex, Objection.js, or an ORM (Sequelize/TypeORM) if used.
- Keep repository/database access code in `repository` layer. Encapsulate raw SQL only inside well-tested repository methods.

## 6. Testing
- Unit tests: every new function/method must have unit tests covering expected behavior and edge cases.
  - Java: JUnit 5 + Mockito.
  - JS: Jest or Mocha + Sinon.
- Integration tests: every API endpoint must have integration tests exercising routing, validation, and persistence (use in-memory DB or CI-provisioned test DB).
- Test files: mirror source paths under `src` with `__tests__` or `Test` suffix conventions.

## 7. Logging
- Use structured JSON logs containing at minimum: `timestamp`, `level`, `message`, `requestId`, `userId` (if available), `operation`.
- Do not log secrets. Use masking for sensitive fields.
- Correlation/request ID: generate at the edge (API gateway / middleware) and propagate via `X-Request-ID` header.

## 8. Code Style
- No `console.log` or `System.out.println` in production code; use configured logger.
- No `TODO` comments without an associated ticket number (e.g., `TODO: PROJ-1234`); CI should flag TODOs lacking ticket IDs.
- Functions / methods should be concise: prefer < 30 logical lines; split complex logic into smaller private/helper methods.
- Follow Google Java Style / Airbnb JS style (or project's chosen linter config). Ensure CI runs linters on PRs.

## 9. Documentation
- Document all exported/public functions and classes with JSDoc (JS) or JavaDoc (Java) docstrings.
- Keep `README.md` for modules explaining purpose, configuration, and run instructions.
- Produce and maintain OpenAPI (Swagger) docs for all public APIs; store `openapi.yaml` at `api/openapi.yaml`.

## 10. Git & Commits
- Use Conventional Commits: `feat:`, `fix:`, `docs:`, `chore:`, `test:`, `refactor:`. Include scope when useful, e.g. `feat(tasks): add dependency API`.
- Write clear PR descriptions with link to ticket and list of changes.
- Rebase or squash to keep commit history readable for merged PRs per repo policy.

## Enforcement & CI
- CI checks on PRs must include: linting, unit tests, integration tests (where feasible), SCA (dependency scanning), and a TODO/ticket check.
- Merge to protected branches requires passing CI and at least one approving review from a maintainer.

## Examples
- Response envelope example (success):

```json
{
  "success": true,
  "data": { "id": "T101", "title": "Implement Payment API" },
  "error": null,
  "meta": { "requestId": "req-abc123" }
}
```

- Error example (validation):

```json
{
  "success": false,
  "data": null,
  "error": { "code": "ERR-VAL-003", "message": "Title must be between 3 and 200 characters." },
  "meta": { "requestId": "req-abc123" }
}
```

---

Follow these standards consistently. If you propose an exception, document the rationale in the PR description and get an explicit review approval.
