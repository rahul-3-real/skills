---
name: fastapi-backend
description: Build and review FastAPI backends with thin routers, typed controllers, Pydantic validation, dependency injection, lifespan management, and the ApiResponse/ApiError contract. Use when creating or modifying FastAPI routes, controllers, dependencies, middleware, error handlers, or application configuration.
---

Apply the same architectural standards as `hono-bun-backend`, expressed through Python and FastAPI idioms. Inspect `pyproject.toml`, the lockfile, and neighbouring modules before editing. Examples use Pydantic v2 and Python 3.10+ syntax; respect the project's supported versions and verify uncertain APIs against official documentation.

## Scope

**In scope:** Application setup, routing, controllers, typed dependencies, request validation, response serialization, error handling, middleware, lifecycle, and HTTP tests.

**Out of scope:** Database schema design and migrations, frontend code, and deployment infrastructure. The sibling [database skill](../database/SKILL.md) provides architectural context, but its Mongoose/Drizzle references are JavaScript-specific. Use the project's existing Python driver/ORM and its official documentation; do not introduce a database library merely to match another skill.

Carry over language-independent principles from [clean-code](../clean-code/SKILL.md): small functions, early returns, explicit inputs, one-way dependencies, preserved error causes, and no speculative abstractions. Its arrow-function, kebab-case Python module, Bun, and TypeScript rules do not apply to Python. Consult [web-security](../web-security/SKILL.md) for relevant security policy, adapting library examples to the Python stack.

## Project Layout

For a new backend, use this shape; preserve an established package layout in existing projects. Include `__init__.py` where the project's packaging requires it.

```text
app/backend/
├── pyproject.toml
├── src/backend/
│   ├── main.py                 # ASGI entrypoint
│   ├── configs/
│   │   ├── app_config.py       # Factory, CORS, routers, handlers
│   │   ├── settings.py         # Typed environment configuration
│   │   └── db_config.py        # Pool/client lifecycle
│   ├── routes/                 # Declarative APIRouter registration
│   ├── controllers/            # Typed HTTP handlers
│   ├── dependencies/           # Auth, ownership, sessions, services
│   ├── middleware/             # Cross-cutting ASGI concerns
│   ├── schemas/                # Request and public response models
│   ├── models/                 # Persistence models
│   ├── repositories/           # Queries and persistence operations
│   ├── services/               # Business workflows
│   └── utils/                  # ApiResponse, ApiError, pure helpers
└── tests/
```

**One resource per file, one responsibility per layer.** Do not create empty directories or every layer for a trivial endpoint. Use `snake_case` modules/functions, `PascalCase` classes, and absolute package imports such as `from backend.schemas.streak import StreakCreate`. Never patch `sys.path` to fix packaging.

## Layering Rules

| Layer | Responsibility | Must not |
| --- | --- | --- |
| Route | Path, method, dependencies, status, response model | Contain handler bodies or queries |
| Dependency | Resolve typed auth, ownership, sessions, services | Build business responses |
| Middleware | Request IDs, timing, CORS, transport concerns | Implement resource workflows |
| Controller | Accept validated inputs, orchestrate, respond | Define persistence schemas or queries |
| Service | Business rules, multi-step operations, transactions | Read `Request`, resolve `Depends`, build HTTP responses |
| Repository/model | Persistence operations and models | Import controllers or HTTP request state |
| Util | Pure reusable helpers and shared contracts | Depend on controllers or database clients |

A controller may call a repository directly for a simple read/write. Multi-resource operations and computed workflows belong in services. Pass resolved dependencies explicitly into service methods; FastAPI does not resolve `Depends` when ordinary Python code calls a function.

## App Configuration and Lifecycle

- Expose `create_app()` and an importable `app` in `backend.main`; importing either must not open network connections or run migrations.
- Version new business API prefixes as `/api/v1/...`; preserve existing public URLs.
- Configure CORS centrally with explicit allowed origins when credentials are enabled. Prefer FastAPI/Starlette's supplied middleware over another package.
- Centralize typed configuration in `configs/settings.py`, using the existing settings system (`pydantic-settings` if already selected). Validate required settings before serving and keep credentials out of defaults and logs.
- Use an `asynccontextmanager` passed as `FastAPI(lifespan=...)` for startup/shutdown. Open required pools and shared clients before `yield`; close them on shutdown, including partial-startup failures. Use context managers or an `AsyncExitStack` when several resources need cleanup.
- Create one pool/client per worker process, with request-scoped sessions/units of work. Never share a mutable ORM session across requests or parallel tasks.
- Let startup failure propagate so the ASGI server fails startup. Do not call `sys.exit()` inside handlers or lifespan code.
- Use the project's ASGI runner (for example, `uvicorn backend.main:app`); enable reload only for development. Keep server execution out of module import paths.

See [FastAPI lifespan](https://fastapi.tiangolo.com/advanced/events/) for lifecycle APIs.

## Typed Dependencies

Declare reusable aliases for values handlers consume:

```python
# backend/dependencies/auth.py
from typing import Annotated

from fastapi import Depends

from backend.schemas.user import UserPublic

# get_current_user verifies credentials and returns a public principal.
CurrentUser = Annotated[UserPublic, Depends(get_current_user)]
```

`get_current_user` is the project's auth dependency, defined before the alias. Service and repository dependencies follow the same pattern; their implementations depend on the chosen data layer.

- Prefer `Annotated[ConcreteType, Depends(provider)]` over untyped dictionaries or arbitrary `request.state` attributes.
- Use router-level `dependencies=[Depends(require_auth)]` for guards whose return value is unused; inject a typed parameter when the handler needs the value.
- Resolve authentication and resource ownership with dependencies, so missing credentials and denied access stop execution before business logic. Return an already-loaded resource from ownership dependencies to avoid duplicate lookups.
- Still scope repository reads and writes by the authenticated principal; a previous ownership check alone does not make an unscoped write safe.
- Use `yield` dependencies with `finally`/context managers for request-owned resources. Never swallow exceptions thrown through `yield`.

See [FastAPI dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/) for injection semantics.

## Validation and Python Types

```python
# backend/schemas/streak.py
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field


class StreakCreate(BaseModel):
    model_config = ConfigDict(extra="forbid")

    title: str = Field(min_length=1, max_length=100)
    description: str | None = Field(default=None, max_length=500)


class StreakPublic(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: UUID
    title: str
    description: str | None
```

- Let FastAPI validate typed bodies, path parameters, and query parameters before calling the controller. Do not manually parse JSON in ordinary handlers.
- Keep schemas at module scope in `schemas/`; use separate create/update/public models so clients cannot set ownership, roles, or server-managed fields.
- Bound lengths, numeric ranges, pagination, and upload sizes where relevant. Explicitly decide whether coercion is allowed; enable strict fields when coercion would change meaning.
- For PATCH, define a dedicated update model. Use `model_dump(exclude_unset=True)` to distinguish omitted fields from explicit `null`, then map an allowlist of fields to the write. Reject `null` for non-nullable persistence fields; reject empty changes when the endpoint requires an update. Do not use `exclude_none=True` to implement PATCH semantics.
- Pass explicit validated fields to services/repositories. Never unpack the raw request or an unrestricted model dump into a persistence create/update call.
- Use Pydantic v2's `model_validate`, `model_dump`, and `ConfigDict`; avoid copying v1 `parse_obj`, `.dict()`, or inner `Config` examples into v2 code.
- Annotate public functions and return values, use concrete collection types and `T | None`, and avoid `Any`, blanket type ignores, or unchecked casts to hide boundary mistakes. Use named `def`/`async def` handlers rather than lambdas.

See [Pydantic models](https://docs.pydantic.dev/latest/concepts/models/) and [FastAPI partial updates](https://fastapi.tiangolo.com/tutorial/body-updates/).

## Routes and Controllers

A route module registers imported controllers with `APIRouter.add_api_route`; this retains Hono's declarative route/controller split without hiding FastAPI's typed handler signature.

```python
# backend/routes/streak.py
from fastapi import APIRouter, status

from backend.controllers.streak import create_streak_controller
from backend.schemas.streak import StreakPublic
from backend.utils.api_response import ApiResponse

router = APIRouter(prefix="/streaks", tags=["streaks"])
router.add_api_route(
    "",
    create_streak_controller,
    methods=["POST"],
    status_code=status.HTTP_201_CREATED,
    response_model=ApiResponse[StreakPublic],
)
```

Mount with `app.include_router(router, prefix="/api/v1")`. Register static paths before competing parameter paths (`/active` before `/{streak_id}`). Choose a trailing-slash convention and use it consistently.

Controllers follow **validated input → act → serialize → respond**:

```python
# backend/controllers/streak.py
from backend.dependencies.auth import CurrentUser
from backend.dependencies.streak import StreakServiceDep
from backend.schemas.streak import StreakCreate, StreakPublic
from backend.utils.api_response import ApiResponse


async def create_streak_controller(
    payload: StreakCreate,
    user: CurrentUser,
    service: StreakServiceDep,
) -> ApiResponse[StreakPublic]:
    streak = await service.create_streak(
        user_id=user.id,
        title=payload.title,
        description=payload.description,
    )
    return ApiResponse[StreakPublic](
        statusCode=201,
        data=StreakPublic.model_validate(streak),
        message="Streak created successfully",
    )
```

`StreakServiceDep` resolves the project's typed service; `create_streak` is its async business operation. Do not add an interface, base controller, or generic CRUD framework just for this example. Existing projects with thin decorated handlers can retain that convention instead of duplicating handlers solely to fit this layout.

## Response Contract

For new JSON APIs, preserve the Hono wire keys, including `statusCode`:

```python
# backend/utils/api_response.py
from typing import Generic, Literal, TypeVar

from pydantic import BaseModel, Field

DataT = TypeVar("DataT")


class ApiResponse(BaseModel, Generic[DataT]):
    statusCode: int
    data: DataT
    message: str
    success: Literal[True] = True


class ApiErrorResponse(BaseModel):
    statusCode: int
    message: str
    errors: list[str] = Field(default_factory=list)
    success: Literal[False] = False
```

- Successful JSON responses use `ApiResponse[T]`; failures use `ApiErrorResponse`. These intentionally match Hono's separate success/error shapes.
- Declare `response_model` and actual HTTP status on the route; `statusCode` in a body does not set the HTTP status. Test that they agree.
- Build public DTOs explicitly. Response models are an additional output filter, not permission to return a raw ORM object carrying secrets. Avoid lazy database reads during serialization.
- Preserve an existing application's published response contract rather than silently breaking clients. Files, streams, redirects, and bodyless `204`/`304` responses keep their native semantics; never wrap them in JSON.
- Document error models in route/router `responses`, including the validation response, so OpenAPI describes the actual wire format.

See [FastAPI response models](https://fastapi.tiangolo.com/tutorial/response-model/) for output validation and filtering.

## Centralized Errors

- Define `ApiError(Exception)` with `status_code`, a public `message`, `errors`, and optional response `headers`. Raise it for expected HTTP failures; never return a success envelope containing an error.
- Register handlers centrally in `create_app()`: `ApiError` → declared error response; `RequestValidationError` → `422`; Starlette's `HTTPException` → its status; unknown `Exception` → generic `500`. Use `app.add_exception_handler` or decorators, not a broad catch around every controller.
- Handle Starlette's exception type so framework `404`/`405` and FastAPI `HTTPException` use the contract too. Preserve headers such as `WWW-Authenticate`, `Allow`, and `Retry-After`.
- Serialize error DTOs using `model_dump(mode="json")` when constructing `JSONResponse`, and set its `status_code` explicitly.
- Publish only sanitized validation messages/locations. Do not return `RequestValidationError.body` or raw error entries containing `input`/`ctx`; those may expose credentials or non-JSON values.
- Request validation uses `422` by default. Adopt `400` only for an existing contract and update handlers, tests, and OpenAPI together. Response validation failures and internal Pydantic errors are server bugs, not client `422`s.
- Log unknown errors with traceback server-side and return `Internal server error`; keep production debug mode off. Redact secrets and avoid full body logging.
- Catch only to recover, translate a known lower-level error, or add useful context. Preserve causes with `raise ApiError(...) from error`; do not catch just to re-raise.
- Services keep domain errors independent of HTTP. Translate them at the controller boundary or in a central handler mapping.

Use `200` for reads/updates, `201` for creates, `401` for absent/invalid authentication, `403` for denied permission, `404` for a missing resource, and `409` for a known conflict. Empty collections return `200` with an empty paginated result; missing individual resources return `404`.

See [FastAPI error handling](https://fastapi.tiangolo.com/tutorial/handling-errors/) for exception registration.

## Async and Middleware

- Use `async def` with awaitable drivers/clients and await every operation. Never call blocking database, HTTP, filesystem, or CPU-heavy work directly on the event loop.
- For synchronous I/O stacks, use synchronous handlers/dependencies (FastAPI runs these in a thread pool), or explicitly offload blocking calls from async code. Ordinary helper functions are not automatically offloaded.
- Bound parallel work and give external calls timeouts. Do not share one ORM session across concurrent tasks. Use a worker/process for heavy CPU work; threads do not generally make Python CPU work parallel.
- Do not launch untracked `asyncio.create_task` work from a request. Use `BackgroundTasks` for short, best-effort work that may be lost on restart; durable jobs need the project's queue. Background work owns its resources and cannot change a response already sent.
- Await `call_next(request)` in HTTP middleware and return its response. Middleware has no automatic `Depends` injection; use dependencies for auth/ownership.
- `HTTPException`/`ApiError` raised in user middleware can sit outside registered exception handlers. If middleware rejects a request, return the error response directly; keep ordinary resource errors in dependencies/controllers.
- Prefer pure ASGI middleware when streaming or context-variable propagation matters. Verify CORS headers on error responses when browser clients need them, using an outer CORS wrapper if necessary.

See [FastAPI concurrency](https://fastapi.tiangolo.com/async/) for sync/async execution behavior.

## Data-Layer Boundary

- Pass explicit validated fields and authenticated ownership IDs into repositories/services; construct queries there.
- Paginate collection reads with a default, a hard maximum, and deterministic ordering. Select only fields needed by the public schema.
- Scope lookups/updates/deletes by both resource ID and principal. Check affected results; map missing individual resources to `ApiError(404, ...)` at the HTTP boundary.
- Keep atomic workflows and transaction ownership explicit in services. Complete the write/commit before reporting success, not in post-response cleanup.
- Translate known uniqueness conflicts to `409`; let unexpected persistence failures reach the generic server-error path.

## Testing and Completion

- Use the existing Python test runner, normally pytest. Exercise the ASGI app without binding a port, using `with TestClient(app) as client:` so lifespan runs.
- Override dependency providers for isolated HTTP tests with `app.dependency_overrides`; restore overrides during fixture teardown. Test persistence semantics separately against the selected disposable database.
- For async tests, use HTTPX `AsyncClient` with `ASGITransport` and the project's async test plugin. This transport does not trigger lifespan; manage it explicitly when startup resources are needed.
- Cover successful status/envelope, invalid body/path/query, authentication and ownership failures, missing resources, empty lists, pagination bounds, and sensitive-field exclusion for the changed behavior. Verify standardized errors for framework errors as well as application errors.
- Use `raise_server_exceptions=False` with `TestClient` (or `raise_app_exceptions=False` with `ASGITransport`) when asserting the rendered `500` response.
- Run the project's configured formatter/linter, type checker, and relevant tests through its existing environment and package manager. Respect its lockfile; do not introduce Bun tooling or replace Python tooling to match Hono.
- Report checks actually run and any remaining gaps. A documentation-only skill change needs skill validation and example review, not a newly generated backend project.

See [FastAPI testing](https://fastapi.tiangolo.com/tutorial/testing/) and [async testing](https://fastapi.tiangolo.com/advanced/async-tests/).

## Constraints

- **ALWAYS** inspect existing conventions and supported versions first.
- **ALWAYS** keep routes declarative, dependencies typed, and services independent of request objects.
- **ALWAYS** validate at HTTP boundaries and serialize through public response models.
- **ALWAYS** align response status, envelope, and OpenAPI documentation.
- **ALWAYS** acquire shared resources during lifespan and release request resources reliably.
- **NEVER** block the event loop, share sessions across requests, or leak internal errors.
- **NEVER** mass-assign client input or trust an ownership check without scoped persistence operations.
- **NEVER** add dependencies or architectural layers unrelated to the requested work.
