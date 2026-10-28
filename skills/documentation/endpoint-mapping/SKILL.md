---
name: endpoint-mapping
description: Use when asked to document an API server's endpoints, write an API reference or endpoint map, or produce a client/SDK/frontend contract (routes, auth, roles, payloads, errors) from backend source code. Works for REST backends such as FastAPI, Django REST, Flask, Express, NestJS, Spring and Go routers.
---

# Endpoint Mapping

## Overview

Turn a backend's implementation into a Markdown endpoint map that frontend, client-app, SDK and service developers can rely on. **The code is the only source of truth.** Existing docs, READMEs and comments are hints to check, never evidence.

## Inputs

- **Repo root:** the current working directory unless the user names another.
- **Output path:** the one the user gives. Otherwise use `docs/API.md`. If that file exists, update it in place: keep correct content and rewrite whatever the code contradicts.
- **Audience:** client-facing by default. Ask whether internal/service routes belong in the same file, a separate `docs/API-internal.md`, or nowhere.

## Workflow

### 1. Build the complete route inventory

Start with a machine-generated list where one exists, then confirm it against the code:

| Framework | Generated list | Where routes and prefixes live |
|---|---|---|
| FastAPI | `app.openapi()` / `/openapi.json` | `@router.<method>` decorators, `include_router(prefix=..., dependencies=...)`, `app.mount` |
| Django REST | `manage.py show_urls`, drf-spectacular schema | `urls.py`, `ViewSet` + router registrations, `permission_classes` |
| Flask | `flask routes` | `@bp.route`, `register_blueprint(url_prefix=...)` |
| Express | none built in | `router.<method>`, `app.use('/prefix', router)`, middleware chain |
| NestJS | `@nestjs/swagger` doc | `@Controller('prefix')`, `@Get/@Post`, guards, global prefix in `main.ts` |
| Spring | springdoc `/v3/api-docs` | `@RequestMapping`, `@GetMapping` etc., `SecurityFilterChain` |
| Go (chi/gin/echo) | none built in | `r.Route`, `r.Group`, `Handle*` calls, middleware |

Don't stop at the first route file. Follow every prefix mount, version prefix (`/v1`), sub-app, WebSocket and SSE route, and auth applied at router/group level. Group routes by resource, following how the codebase itself groups them (route modules, controllers, or the service layer).

### 2. Extract the auth contract

- Required headers (`Authorization`, API keys, tenant headers)
- Token type and issuer (JWT claims used, identity provider, cookie vs bearer)
- Where auth is enforced per route: decorator, dependency, guard, middleware, or router-level
- Roles and permissions, and the exact check (e.g. `role == "admin"`)
- Ownership scoping, found in service/repository queries (e.g. `WHERE owner_id = current_user.id`). Note when a non-owner gets 404 rather than 403.

### 3. Extract request and response contracts

- Path/query/body parameters with types, required vs optional, defaults, and validation rules (lengths, regex, enums, ranges)
- Response models and status codes per route, including non-200 successes (201, 204)
- **Serialized** field names: apply aliases and camelCase generators, and drop excluded or write-only fields (`response_model_exclude`, `@JsonIgnore`, serializer `fields`)
- The global error shape from exception handlers or error middleware, plus the error codes each route can produce

### 4. Extract business rules

- Pagination format and limits, filtering, sorting
- Uniqueness constraints and the error they raise
- Rate limits, quotas, size limits
- Read-only vs writable routes, idempotency, side effects (emails, webhooks, background jobs)
- Which caller is meant to use the route: client app, SDK, server-to-server, or admin

### 5. Write the document

Use this structure:

1. Base URL and versioning
2. Authentication (headers, token, roles, ownership model)
3. Error shape (one JSON example plus a table of error codes)
4. Quick route table: `Method | Path | Auth | Roles | Summary`
5. One section per resource, each endpoint in this form:

````markdown
### POST /api/v1/projects
Create a project. The caller becomes its owner.

- **Auth:** Bearer JWT · **Roles:** user, admin · **Ownership:** creator is owner
- **Body:** `ProjectCreate`
  ```json
  { "name": "Website redesign", "isPublic": false }
  ```
  - `name`: string, 1–100 chars, unique per owner
- **201:** `ProjectRead`
  ```json
  { "id": "8f14e45f-…", "name": "Website redesign", "isPublic": false, "createdAt": "2026-01-15T10:00:00Z" }
  ```
- **Errors:** 401 `UNAUTHENTICATED` · 409 `PROJECT_NAME_TAKEN` · 422 validation
````

6. A typical client flow (e.g. sign in → list → create → update), naming the routes in order
7. Internal/service routes, if the user chose to include them, in a clearly separated section

JSON examples use realistic values and exactly the serialized field names. Never put real secrets, keys or production hostnames from config or `.env` files into examples.

### 6. Verify before finishing

- [ ] Every route in the step 1 inventory appears in the quick table, and the table has no route the code lacks
- [ ] Every schema name and field in the doc exists in the code
- [ ] Every protected route lists its auth, roles and ownership rule
- [ ] Anything not directly visible in code is marked `(inferred)` with a one-line reason; anything that can't be derived is left out
- [ ] Internal routes are where the user asked for them

For large APIs, map and write one resource group at a time, then run the checklist across the whole document.

## Common mistakes

| Mistake | Fix |
|---|---|
| Paths missing their mount prefix (`/projects` instead of `/api/v1/projects`) | Follow every `include_router` / `app.use` / `@Controller` prefix to the root |
| Python/Java field names in JSON instead of the serialized aliases | Check alias generators and serializer config |
| Route marked public when a router-level dependency or global middleware requires auth | Check auth at route, router/group and app level |
| Ownership rule missed because it lives in a repository query | Read the service/repo function each handler calls |
| Trusting an old `API.md` | Re-derive from code; the old doc is only a hint |
