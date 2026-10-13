# AGENTS.md — FastAPI Project Guidelines for AI Agents

Guidelines for AI agents working on this FastAPI codebase. Follow them when writing or modifying code.

- **MUST** = enforced in review (or by tooling). **SHOULD** = default; break it only with a stated reason.
- Rules apply to **new and modified code**. Do not mass-refactor untouched code to satisfy them; report it instead.

### Scope & Assumptions

- **Required stack:** Python 3.12+, FastAPI, Pydantic v2, pytest, Ruff.
- **Optional (applies only if the project uses it):** SQLModel / SQLAlchemy, Alembic, mypy or pyright, import-linter, a task queue (e.g. Celery), an async HTTP client for integrations (e.g. httpx or aiohttp).
- **Names in this document are examples, not guarantees.** Class and function names such as `BaseService`, `BaseExternalService`, `ExternalServiceError`, `ResponseModel`, `response_base`, `get_current_user`, and package paths such as `app/common/`, `app/modules/`, `app/integrations/` illustrate a pattern. **Before using one, search the codebase for it. If it does not exist, follow the project's own equivalent — never import or invent a module just because it appears here.**
- When this file conflicts with clear, established conventions in the existing code, follow the existing code and mention the discrepancy.

---

## 1. Working Agreement (read first)

**Before coding**
1. **Read before writing.** Find the closest existing module/feature and mirror its structure, naming and patterns. Do not invent a new pattern when one already exists.
2. **Plan non-trivial changes first.** If the change touches more than 2 files, adds a module, adds an external integration, or changes a DB schema, state a short plan before coding: files to add/change, the responsibility of each, dependency direction, and test plan. Wait for confirmation when more than one reasonable design exists.
3. **Ask, don't guess** when business rules are ambiguous (calculation semantics, period/scope rules, permissions).

**While coding**
- Make the smallest diff that solves the task. No drive-by renames, reformatting, or refactors of unrelated code.
- Work one module/layer at a time; every step leaves the tests green.
- No new third-party dependency without asking.
- No new abstraction (base class, Protocol, registry, factory) without a concrete second use case or a concrete testing need (YAGNI).
- Never hardcode secrets, hosts or tokens; never log them.
- Annotate every parameter and return type in new code (§16, *Type annotations*).
- Do not run destructive git operations (force-push, history rewrite, deleting branches) or push to protected branches unless explicitly asked.

**After coding** — follow [Validation & Definition of Done](#15-validation--definition-of-done).

---

## 2. Design Principles & Patterns

### 2.1 SOLID

This project is layered, and the service layer owns both use-case logic and persistence. Apply the principles pragmatically:

| Principle | Concrete rule in this codebase |
|-----------|--------------------------------|
| **S**ingle Responsibility | One module/class has one reason to change. HTTP parsing, business rules, persistence and external calls never live in the same function. A file over ~300 lines or a function over ~30 lines must be split (see §3). |
| **O**pen/Closed | Behavior that varies by type/mode (e.g. allocation method, export format) is added by **adding** a class/function and registering it in a dict/registry — not by extending an `if/elif` chain. Add the registry the first time a third variant appears, not before. |
| **L**iskov Substitution | Subclasses of the project's base classes (e.g. `BaseService`, `BaseExternalService`) keep the parent's signatures, return types and error contract. Do not override a method to no-op or to raise unrelated exceptions. |
| **I**nterface Segregation | Prefer small `typing.Protocol`s or plain functions over fat base classes. Do not add a method to a shared base class (e.g. `BaseService`) unless at least two domains need it. |
| **D**ependency Inversion | Business code gets collaborators from the outside (`Depends(...)`, constructor/function arguments), never by building clients, reading env vars, or opening sessions inside business methods. External systems are reached only through `app/integrations/`. |

### 2.2 Structural rules

- **Functional core, imperative shell.** Put calculations and decisions into pure functions (`rules.py`) that take plain data and return plain data. The service does the IO around them. Pure code is trivially unit-testable.
- **Prefer composition over inheritance.** Inherit only from the project's established bases (e.g. `BaseService`, `BaseExternalService`, SQLModel/Pydantic models).
- **Prefer functions over one-method classes**, `Protocol` over ABC, `dataclass`/Pydantic over hand-written containers or `dict` passing.
- **Don't over-engineer.** One implementation with no foreseeable second one = no interface. Simple and explicit beats clever and generic.

### 2.3 Core principles

| Principle | What it means here |
|-----------|--------------------|
| **DRY** (Don't Repeat Yourself) | DRY is about **knowledge**, not text: every business rule, constant, enum/status value, error message, config value and schema shape has **one authoritative definition**. Derive schemas from a shared base (`UserBase` → `UserCreate` / `UserRead`) instead of redeclaring fields; move repeated validation into a dependency, repeated query filters into a service helper, repeated error mapping into a handler. |
| **Rule of Three** (guards DRY) | Tolerate duplication the first two times; extract on the **third** occurrence, and only if the copies change for the **same reason**. Code that merely looks alike but changes for different reasons is *accidental duplication* — leave it separate. |
| **KISS** | Choose the simplest design that works. No metaprogramming, string-based dispatch via `getattr`, deep inheritance, or clever one-liners. A reader should be able to follow a use case without jumping through more than ~2 files. |
| **YAGNI** | No speculative parameters, config switches, hooks or interfaces for cases nobody has asked for. Add them when the need is real. |
| **Separation of Concerns** | Each layer and module answers one question (see §5). Mixing HTTP, business rules, persistence and external calls in one place is the most common design failure. |
| **Law of Demeter** | Talk to your direct collaborators. Avoid long chains across layers/modules (`order.customer.account.owner.email`). Across domains, go through the other module's service instead of reaching into its models. Attribute access on loaded ORM relationships *within* a module is fine. |
| **Fail Fast** | Validate at the boundary (Pydantic, dependencies) and with guard clauses; raise a specific exception as soon as something is wrong. Never silently default missing required data or swallow errors. |
| **Command–Query Separation** | A function either **returns information** or **changes state**, not both. `get_*`, `list_*`, `calculate_*` have no side effects; `GET` routes never mutate; `create_*`, `update_*`, `delete_*` change state and return at most the affected object. |
| **Least Surprise** | Names match behavior; no hidden side effects; similar services have similar signatures; follow the conventions of neighboring code. |
| **Explicit over Implicit** | No hidden commits, magic globals, or I/O in constructors, properties or at import time. Importing a module must not hit the network, DB or filesystem (instantiating settings is the only exception). |
| **Prefer Immutability** | Data passed into `rules.py` is treated as read-only (`@dataclass(frozen=True)`, tuples, `Mapping`). Do not mutate function arguments; return new values. |
| **Idempotency** | Anything that can be retried (schedulers, webhooks, client retries, external calls) must be safe to run twice: unique keys, upserts, or check-then-act inside one transaction. |
| **Scoped Boy Scout rule** | Leave the code you touch slightly cleaner (names, dead code, missing types) — but only within the lines you are already changing. This never overrides the "smallest diff" rule in §1. |

### 2.4 When principles conflict

Resolve conflicts in this order:
1. **Correctness and clarity** beat everything else.
2. **KISS / YAGNI** beat DRY and extensibility: duplication is cheaper than the wrong abstraction.
3. **DRY** beats speculative flexibility, but only after the Rule of Three.
4. **Extensibility patterns** (registry, Strategy, factory) are added when the third variant appears — not for the first or second.

Warning signs of a wrong abstraction: a shared function that grows boolean flags or `if caller == ...` branches to serve diverging callers. **Split it back into separate functions** instead of adding another parameter.

### 2.5 Design patterns: when and how

Patterns are tools, not goals. Use one only when the problem it solves is actually present.

| Pattern | Use it when | Form in this codebase | Avoid |
|---------|-------------|-----------------------|-------|
| **Dependency Injection** | Any collaborator (service, client, current user, session) | FastAPI `Depends(...)`; tests use `app.dependency_overrides` | Creating clients/sessions inside business methods |
| **Service Layer** | Always, for use cases | `service.py` orchestrates one use case, owns the transaction | Putting use-case logic in routers |
| **Unit of Work** | A use case writes several things that must succeed together | One session/transaction per use case, committed once by the service | Commits scattered in helpers |
| **DTO / Schema separation** | API shape differs from table shape | Separate `Create` / `Update` / `Read` schemas from the table model | Returning table models directly when they hold internal fields |
| **Adapter / Anti-corruption layer** | Talking to a system whose payloads you don't control | `app/integrations/*` translates external payloads/errors into internal models and `ExternalServiceError` | Letting external field names and shapes leak into domain code |
| **Facade** | A use case involves several steps/collaborators | One service method hides the steps; the controller makes a single call | Controllers calling several services in sequence |
| **Strategy (+ registry)** | A behavior has 3+ interchangeable variants | `dict[str, Callable]` or `Protocol` implementations, selected by key | `if/elif` chains growing per variant; a Strategy class for a single variant |
| **Factory** | Choosing an implementation requires non-trivial logic (by config/type) | A plain function `create_x(kind) -> X`, or a dependency that returns one | Factory classes with a single product |
| **Decorator / middleware** | Cross-cutting concerns: logging, timing, retry, caching | Python decorators, FastAPI middleware, or dependencies | Copy-pasting the same try/log/retry block |
| **Repository** | A service's queries become large, reused, or must be swapped/faked | Extract `queries.py` (or a repository class) from `service.py` | A repository layer in front of every table by default |
| **Observer / domain events** | Side effects (notifications, audit) that must not bloat the core use case | Emit after commit; `BackgroundTasks` or a queue consumer | Introducing an event bus before there are multiple subscribers |
| **Singleton** | Stateless, expensive-to-create clients | Module-level instance (e.g. `<service>_service`), exposed through a dependency when it needs overriding | Mutable global state shared across requests |
| **Template Method** | Almost never | Prefer composition and plain functions; an established project base class (e.g. `BaseExternalService`) is the exception | New inheritance hierarchies with hook methods |

### 2.6 Code smells that call for refactoring

If you notice one in code you are modifying, fix it within the scope of your change, or report it:

- **God service/class**: >300 lines or many unrelated methods → split by use case.
- **Shotgun surgery**: one small change requires edits in many files → knowledge is duplicated or a concept is missing.
- **Feature envy**: a method mostly manipulates another module's data → move it to that module's service.
- **Primitive obsession**: strings/ints/dicts standing in for a concept → enum, value object, or Pydantic model.
- **Flag arguments**: `do_x(..., force=True)` that switch behavior → split into two functions or use an enum.
- **Long parameter list** (>5) → group into a model.
- **Magic numbers/strings**: repeated literals → named constant or enum in `constants.py`.
- **Speculative generality**: unused parameters, hooks, base classes → delete them.
- **Deep nesting / long `if` ladders** → guard clauses, early returns, extracted functions.
- **Comments that restate the code** → rename things so the comment is unnecessary; keep comments for *why*.

Several of these smells are caught mechanically by Ruff: flag arguments (`FBT`), magic values (`PLR2004`), commented-out code (`ERA`), unused arguments (`ARG`), needless `else` after `return` (`RET`), and error-handling smells (`TRY`, `BLE`). See `pyproject.tooling.snippet.toml`.

---

## 3. Size & Complexity Limits

| Item | Limit | Action when exceeded |
|------|-------|----------------------|
| File | ~300 lines | Split by responsibility (e.g. `service.py` → `service.py` + `queries.py` / `rules.py`) |
| Function / method | ~30 lines | Extract named helpers |
| Parameters | ≤ 5 | Group into a Pydantic model / dataclass |
| Cyclomatic complexity | ≤ 10 | Extract branches, use early returns |
| Nesting depth | ≤ 3 | Early return / guard clauses |

Enforced by Ruff (`C901`, `PLR0912`, `PLR0913`, `PLR0915`) — see `pyproject.tooling.snippet.toml`.

---

## 4. Project Structure

Organize code by **domain**, not by file type. The layout below is a **recommended** layout: if the project already has a different structure, follow it and adjust the import-linter contracts to match.

```
app/
├── common/                  # Cross-cutting, domain-agnostic code
│   ├── response.py          #   standard response envelope, if the project uses one
│   ├── security/auth.py     #   authentication dependencies (e.g. JWT)
│   └── services/            #   shared base service (query + CRUD helpers), if any
├── config/
│   └── settings.py          # Global settings (reads .env)
├── integrations/            # ALL calls to external systems (see §11)
│   ├── base.py
│   └── <service>.py
├── modules/
│   └── {domain}/            # e.g. users/, orders/, billing/
│       ├── models.py        # DB table models (SQLModel/SQLAlchemy) + read models
│       ├── schemas.py       # Pydantic request/response schemas
│       ├── service.py       # Use cases + persistence (may inherit a shared base service)
│       ├── rules.py         # Pure business rules, no IO (optional)
│       ├── controller.py    # Routers (or *_api.py)
│       ├── dependencies.py  # Route dependencies / validators
│       ├── exceptions.py    # Domain-specific exceptions
│       ├── constants.py
│       └── config.py        # Domain-specific BaseSettings (optional)
├── utils/                   # Generic, domain-free helpers only
├── database.py              # Engine / session
├── exceptions.py            # Global exceptions + handlers
└── main.py                  # App init, router registration (composition root)
tests/
├── conftest.py
├── modules/{domain}/        # mirrors app/modules
└── test_integrations.py
docs/
```

**Where does it go?**

| I need to… | Put it in |
|------------|-----------|
| Parse a request / shape a response | `controller.py` / `*_api.py`, `schemas.py` |
| Validate a path/query resource exists, check ownership | `dependencies.py` |
| Make a business decision or calculation | `rules.py` (pure function) |
| Query/persist data, orchestrate a use case, own a transaction | `service.py` |
| Call another HTTP system | `app/integrations/<service>.py` |
| Read an env var | `app/config/settings.py` (or the domain `config.py`) |
| Generic cross-domain helper (no business meaning) | `app/common/` or `app/utils/` |

**Import convention** — use explicit module names across domains:
```python
from app.modules.users import constants as user_constants
from app.modules.notifications import service as notification_service
```

---

## 5. Layers & Dependency Rules

```
controller / *_api.py  ──►  service  ──►  rules (pure)
                              │
                              ├──►  integrations
                              └──►  models
```

| Layer | Does | Must NOT |
|-------|------|----------|
| **Controller / router** | Declare routes, parse/validate input, call **one** service method, wrap the response | Contain business logic, build queries, touch the DB session, call external services, commit |
| **Dependencies** | Validate/resolve request-scoped resources (existence, ownership, auth) | Mutate data |
| **Service** | Orchestrate a use case, own the transaction, persist, call integrations, raise domain exceptions | Know about HTTP (`Request`, `HTTPException`, status codes) |
| **Rules** | Pure computation on plain data | Do IO, read settings, use the session, call `datetime.now()` (pass time in) |
| **Integrations** | Talk to external HTTP systems, translate transport errors to `ExternalServiceError` | Import domain modules |
| **Models / schemas** | Describe data shape | Import `service` or `controller` |

Hard dependency rules (**MUST**):
- `app/common/` never imports `app/modules/` or `app/integrations/`.
- `app/integrations/` never imports `app/modules/`.
- `models.py` / `schemas.py` never import `service.py` / `controller.py`.
- No circular imports between domains. If A and B need each other, extract the shared piece into a third module or `app/common/`.
- Function-level (lazy) imports are a last resort to break a cycle: add a `# NOTE: avoids circular import X <-> Y` comment and treat it as a design smell worth reporting.

**Transactions**
- The **service** owns the transaction boundary: one use case = one commit. Controllers and helper functions never commit.
- A helper called by several services must not commit on its own.
- If an external call fails after data was written, record that state explicitly (e.g. mark the related record `FAILED` with an `error_message`) rather than leaving partial data silently.

---

## 6. Async Routes

### Rules
- `async def` routes: use **only** non-blocking I/O (`await` calls).
- `def` routes (sync): use for blocking I/O (runs in the threadpool automatically).
- CPU-intensive work: offload to a task queue (e.g. Celery, Arq, RQ) or a process pool.
- Never call blocking code (`requests`, `time.sleep`, sync DB drivers, heavy file I/O) from `async def`.

### Common Mistakes to Avoid
```python
# WRONG: a blocking call inside an async route stalls the whole event loop
@router.get("/rates/wrong")
async def get_rate_wrong() -> RateRead:
    response = requests.get(RATE_API_URL, timeout=5)
    return RateRead.model_validate(response.json())

# CORRECT: a non-blocking client inside an async route
@router.get("/rates")
async def get_rate() -> RateRead:
    async with httpx.AsyncClient(timeout=5) as client:
        response = await client.get(RATE_API_URL)
    return RateRead.model_validate(response.json())

# CORRECT: a plain `def` route for unavoidable blocking code (FastAPI runs it in the threadpool)
@router.get("/rates/legacy")
def get_rate_legacy() -> RateRead:
    response = requests.get(RATE_API_URL, timeout=5)
    return RateRead.model_validate(response.json())
```

### Using Sync Libraries in Async Context
```python
from fastapi.concurrency import run_in_threadpool

@router.get("/reports/{report_id}")
async def download_report(report_id: int) -> ReportRead:
    return await run_in_threadpool(legacy_client.fetch_report, report_id=report_id)
```

---

## 7. Pydantic

### Use Built-in Validators
```python
from decimal import Decimal
from pydantic import BaseModel, EmailStr, Field

class ProductCreate(BaseModel):
    sku: str = Field(min_length=3, max_length=32, pattern=r"^[A-Z0-9-]+$")
    name: str = Field(min_length=1, max_length=200)
    price: Decimal = Field(gt=0, max_digits=10, decimal_places=2)
    contact_email: EmailStr
```

### Custom Base Model
Create a shared base model for consistent serialization. In Pydantic v2 `json_encoders` is deprecated — use an annotated serializer type:
```python
from datetime import UTC, datetime
from typing import Annotated
from pydantic import BaseModel, ConfigDict, PlainSerializer

def to_utc_iso(value: datetime) -> str:
    if value.tzinfo is None:
        value = value.replace(tzinfo=UTC)
    return value.astimezone(UTC).isoformat()

UtcDatetime = Annotated[datetime, PlainSerializer(to_utc_iso, return_type=str)]

class CustomModel(BaseModel):
    model_config = ConfigDict(populate_by_name=True)
```

### Split BaseSettings by Domain
```python
# app/modules/storage/config.py
from pydantic import SecretStr
from pydantic_settings import BaseSettings

class StorageConfig(BaseSettings):
    BUCKET_NAME: str
    ACCESS_KEY: SecretStr      # secrets use SecretStr so they never print in logs/repr
    PRESIGNED_URL_TTL_SECONDS: int = 300

storage_settings = StorageConfig()
```
- Every new env var is added to `.env.example` in the same change.

---

## 8. Dependencies

### Use for Validation, Not Just DI
Return typed objects (models/schemas), not `dict[str, Any]`.
```python
async def valid_invoice_id(invoice_id: int) -> Invoice:
    invoice = await invoice_service.get_by_id(invoice_id)
    if invoice is None:
        raise InvoiceNotFound()
    return invoice

@router.get("/invoices/{invoice_id}")
async def get_invoice(invoice: Invoice = Depends(valid_invoice_id)) -> InvoiceRead:
    return InvoiceRead.model_validate(invoice)
```

### Chain Dependencies
```python
async def valid_owned_invoice(
    invoice: Invoice = Depends(valid_invoice_id),
    current_user: CurrentUser = Depends(get_current_user),
) -> Invoice:
    if invoice.owner_id != current_user.id:
        raise InvoiceNotOwned()
    return invoice
```

### Key Rules
- Dependencies are cached per request (same dependency called multiple times = one execution).
- Prefer `async` dependencies to avoid threadpool overhead.
- Use consistent path variable names to enable dependency reuse.
- Tests replace dependencies via `app.dependency_overrides` — do not add test-only branches to production code.

---

## 9. REST Conventions

Use consistent path variable names for dependency reuse:
```
# Both routes use customer_id, enabling a shared valid_customer_id dependency
GET /customers/{customer_id}
GET /subscribers/{customer_id}
```
- Nouns for resources, plural collection paths; HTTP verbs carry the action.
- **List endpoints MUST be paginated and have a bounded page size** — never return unbounded lists.
- Filters visible to users are declared as explicit `Query(...)` parameters so they appear in `/docs`.
- Do not break existing response shapes; additive changes only unless explicitly requested.
- **Every route declares its response type** via the return annotation (`-> XxxRead`, or the project's envelope type) so OpenAPI is accurate — see §14.

---

## 10. Database

### Naming Conventions
- `lower_case_snake` format.
- Singular table names: `invoice`, `customer`, `invoice_line`.
- Group related tables with a prefix: `billing_plan`, `billing_account`.
- DateTime suffix `_at` (`paid_at`); Date suffix `_date` (`due_date`).

### Set Explicit Index Names
```python
from sqlmodel import SQLModel

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}
# plain SQLAlchemy: MetaData(naming_convention=NAMING_CONVENTION)
SQLModel.metadata.naming_convention = NAMING_CONVENTION
```

### SQL-First Approach
Push set-based work into SQL instead of looping in Python: joins, filtering, aggregation, sorting and pagination. Build nested JSON in the database (e.g. PostgreSQL's `json_agg`) only when it removes extra round-trips.

### Query Rules
- Never build SQL by string concatenation/f-strings with user input — use bound parameters / SQLAlchemy expressions.
- Avoid N+1: eager-load relationships you will serialize (`joinedload` for to-one, `selectinload` for to-many).
- Add indexes for foreign keys and columns used in filters/sorts.
- Select only what you need for large tables; paginate in SQL, not in Python.

### Migrations (Alembic)
- Keep migrations static and reversible; always implement `downgrade`.
- Use date-prefixed, descriptive file names that include the revision id: `2026-03-14_a1b2c3_add_invoice_due_date_idx.py`
  ```ini
  # alembic.ini
  file_template = %%(year)d-%%(month).2d-%%(day).2d_%%(rev)s_%%(slug)s
  ```
- **Always review `--autogenerate` output** before committing; remove unrelated diffs.
- Never edit a migration that has been merged/applied — add a new one.
- Keep schema migrations and data migrations separate.
- Destructive changes (drop table/column, type narrowing) require explicit confirmation from the user.

---

## 11. External Service Integrations (`app/integrations/`)

*Applies only if the project calls external HTTP systems — skip this section otherwise.*

All HTTP calls to external systems live in `app/integrations/`. Do not put business
external-service clients in `app/utils/` (that is for generic helpers) nor inline
HTTP-client calls inside domain modules.

```
app/integrations/
├── __init__.py
├── base.py                      # BaseExternalService: shared HTTP client, URL joining, logging, errors
└── payment_gateway.py           # one file per external service
```

Rules:
- **One external service = one module.** Inside it, one class inheriting
  `BaseExternalService`, and **one method per URL** (`create_charge`, `refund_charge`, ...).
- Declare `service_name`, `base_url` (from `settings`), `default_timeout`; keep URL paths as
  module-level constants.
- Base URL / timeout / token come from `.env` via `app/config/settings.py`
  (e.g. `PAYMENT_API_URL`, `PAYMENT_API_TIMEOUT`, `PAYMENT_API_TOKEN`).
  Never hardcode hosts in the service module.
- `BaseExternalService` returns parsed JSON and raises `ExternalServiceError` on transport
  errors, timeouts, non-2xx and invalid JSON. Domain modules catch that error and translate
  it into their own state (e.g. mark the related record `FAILED` with an `error_message`).
- Domain modules import the service singleton (`from app.integrations.<service> import
  <service>_service`). Never let `app/integrations/` import domain modules.
- Return parsed data validated into a Pydantic model when the response shape is relied upon, instead of passing raw dicts around.
- Retries: only for idempotent requests, bounded with backoff. Never auto-retry a non-idempotent POST.
- Logging: log `service_name`, path, status and duration. Never log tokens or full bodies containing personal data.
- Tests stub at this boundary (monkeypatch the service method, or mock HTTP with a library that matches your client, e.g. `respx` for httpx or `aioresponses` for aiohttp) — never hit the real network.

Document each integration's request/response contract in `docs/`, and cover its request shape in
`tests/test_integrations.py`.

---

## 12. Error Handling & Responses

- Domain errors are exceptions defined in `{domain}/exceptions.py`, inheriting from the global base in `app/exceptions.py`. **Services, rules and dependencies raise domain exceptions; they do not build HTTP responses.** Mapping exception → status code/body happens in exception handlers.
- Catch **specific** exceptions. No bare `except:`; no `except Exception: pass`. If you catch broadly (e.g. at a task boundary), log with context and re-raise or translate.
- `ExternalServiceError` is translated by the caller into domain state (see §11) — it must not leak to clients as a raw 500.
- Follow the project's existing response and error format (for example a standard envelope, if one is used). Do not invent a second format.
- Error messages must not expose internals (SQL, stack traces, tokens, upstream URLs).

---

## 13. Security, Config & Logging

- **Auth:** every non-public route declares the project's authentication dependency (e.g. a JWT or session `Depends(...)`). Adding an unauthenticated route requires explicit justification in the PR/summary.
- **Authorization:** check ownership/permission in a dependency or service, not in the controller body.
- **Secrets:** only via `.env` → settings; use `SecretStr`; never commit `.env`; never log secrets, tokens, passwords or full request bodies containing personal data.
- **Input:** validate at the boundary with Pydantic; never trust path/query/body values in SQL or file paths.
- **Logging:** use the project logger, not `print`. Log identifiers and outcomes, with enough context to debug; no PII.
- **Docs exposure:** hide OpenAPI docs outside local/staging:
  ```python
  DOCS_ENABLED_ENVIRONMENTS = {"local", "staging"}

  def create_app(settings: Settings) -> FastAPI:
      show_docs = settings.ENVIRONMENT in DOCS_ENABLED_ENVIRONMENTS
      return FastAPI(
          title="My API",
          docs_url="/docs" if show_docs else None,
          redoc_url="/redoc" if show_docs else None,
          openapi_url="/openapi.json" if show_docs else None,
      )
  ```

---

## 14. API Documentation & Testing

### Document Endpoints Properly
```python
@router.post(
    "/invoices",
    summary="Create an invoice",
    description="Creates a draft invoice for an existing customer.",
    status_code=status.HTTP_201_CREATED,
    tags=["Invoices"],
    responses={
        status.HTTP_404_NOT_FOUND: {"model": ErrorResponse, "description": "Customer not found"},
        status.HTTP_409_CONFLICT: {"model": ErrorResponse, "description": "Duplicate invoice number"},
    },
)
async def create_invoice(payload: InvoiceCreate) -> InvoiceRead: ...
```

### Route return annotations drive OpenAPI (MUST)

**Every route handler must declare a precise return type — it is what FastAPI uses to generate the response schema in `/docs` and `/openapi.json`.** Without it, the response is documented as an empty `{}`; frontend developers, API clients and generated SDKs cannot see what the endpoint returns. The return annotation also makes FastAPI validate and filter what you return, so internal fields do not leak.

- **Return the response schema directly**: `-> RefundCreated`, `-> list[RefundRead]`, or the project's pagination type (e.g. `-> Page[RefundRead]`).
- **If the project wraps every response in a standard envelope** (e.g. `{code, msg, data}`), declare the parameterized envelope instead: `-> ResponseModel[RefundCreated]` (see the envelope variant below). Follow how existing routes do it; do not introduce an envelope where the project has none.
- **The response type is a Pydantic response schema** (`...Read`, `...Response`), not a table model with internal fields and not a `dict`.
- **Never** leave the return annotation off, write a bare generic envelope (`-> ResponseModel`), or use `-> dict` / `-> Any`. All of these render the payload as untyped in the docs.
- No body (e.g. `DELETE`)? Use `status_code=204` with `-> None`.
- Use `status_code=` for non-200 success codes (e.g. `201` for creation), and `responses={...}` to document error bodies (see the example above).
- If the handler must return something the annotation cannot express (a raw `Response`, a stream, a file), set `response_model=` or `response_class=` explicitly on the decorator instead of dropping the type.
- The returned value must actually match the annotation — FastAPI validates it at runtime.
- Request inputs are documented too: typed path/query parameters with `Query(..., description=...)`, and a Pydantic schema for the body.

```python
@router.post(
    "/orders/{order_id}/refunds",
    summary="Create a refund request",
    status_code=status.HTTP_201_CREATED,
    dependencies=[Depends(get_current_user)],
    responses={status.HTTP_404_NOT_FOUND: {"model": ErrorResponse}},
)
async def create_refund(
    order_id: int,
    payload: RefundCreate,
) -> RefundCreated:                      # <- generates the 201 response schema
    return await refund_service.create(order_id, payload)

# WRONG: each of these makes the response untyped in /docs
async def create_refund(order_id: int, payload: RefundCreate): ...             # no annotation: response schema is {}
async def create_refund(order_id: int, payload: RefundCreate) -> dict: ...     # generic object
```

**Envelope variant (only if the project uses a standard response envelope).** The envelope itself must be generic; otherwise `ResponseModel[X]` cannot be written and the payload stays untyped in the docs. If the existing envelope is not generic yet (e.g. `data: Any`), convert it as below while keeping its field names, so clients are not affected. `ResponseModel` and `response_base` are example names — use the project's own.

```python
# e.g. app/common/response.py
from pydantic import BaseModel

class ResponseModel[T](BaseModel):
    """Standard envelope. The type parameter becomes the typed `data` in OpenAPI."""

    code: int = 200
    msg: str = "Success"
    data: T | None = None

class Page[T](BaseModel):
    """Paginated payload, used as ResponseModel[Page[OrderRead]]."""

    items: list[T]
    total: int
    page: int
    size: int

class ResponseBase:
    def success[T](self, *, data: T | None = None, msg: str = "Success") -> ResponseModel[T]:
        return ResponseModel[T](code=200, msg=msg, data=data)

    def fail(self, *, code: int, msg: str) -> ResponseModel[None]:
        return ResponseModel[None](code=code, msg=msg, data=None)

response_base = ResponseBase()
```

- Use PEP 695 syntax (`class ResponseModel[T](BaseModel)`, Python 3.12+, Pydantic v2).
- `success()` is generic and returns `ResponseModel[T]`, so mypy/pyright can check that the `data` you pass matches the route's declared return type.
- No data: `ResponseModel[None]`. Lists: `ResponseModel[list[OrderRead]]`. Pages: `ResponseModel[Page[OrderRead]]` (if the project already uses a pagination library's `Page[T]`, such as `fastapi-pagination`, use that instead of defining `Page`).
- FastAPI names the OpenAPI components after the parameterization (e.g. `ResponseModel_RefundCreated_`), so each endpoint's `data` shows up as its own concrete schema.

```python
async def create_refund(order_id: int, payload: RefundCreate) -> ResponseModel[RefundCreated]:
    data = await refund_service.create(order_id, payload)
    return response_base.success(data=data)

# WRONG: bare envelope, `data` is untyped in /docs
async def create_refund(order_id: int, payload: RefundCreate) -> ResponseModel: ...
```

**Verify it:** after adding or changing a route, open `/docs` and confirm the response shows concrete fields rather than an empty object. A contract test keeps this from regressing — it fails if any 2xx response has no typed schema:

```python
from typing import Any
from app.main import app as fastapi_app

def _is_untyped(schema: dict[str, Any], components: dict[str, Any]) -> bool:
    if not schema:
        return True  # no return annotation
    if schema.get("type") == "object" and schema.get("additionalProperties") is True:
        return True  # -> dict
    ref = schema.get("$ref")
    if ref:
        target = components[ref.rsplit("/", 1)[-1]]
        data = target.get("properties", {}).get("data", {})
        return {} in data.get("anyOf", [])  # bare `-> ResponseModel`
    return False

def test_every_2xx_route_documents_a_typed_response() -> None:
    spec = fastapi_app.openapi()
    components = spec.get("components", {}).get("schemas", {})
    offenders: list[str] = []
    for path, operations in spec["paths"].items():
        for method, operation in operations.items():
            for code, response in operation.get("responses", {}).items():
                if not code.startswith("2") or code == "204":
                    continue
                schema = response.get("content", {}).get("application/json", {}).get("schema", {})
                if _is_untyped(schema, components):
                    offenders.append(f"{method.upper()} {path}")
    assert not offenders, f"Routes without a typed response schema: {offenders}"
```
(The `$ref` / `data` branch only matters if the project uses a response envelope — adjust the field name to match it, or delete that branch otherwise. Routes that intentionally return files/streams can be allow-listed.)

### Testing Pyramid
| Level | What | How |
|-------|------|-----|
| Unit | `rules.py` and pure helpers | No DB, no network, no fixtures beyond data. Fast; most tests live here. |
| Service | Use cases + persistence | Real test DB; each test isolated (transaction rollback or truncate). Stub integrations. |
| API | Routes, auth, status codes, response shape | `AsyncClient` with `ASGITransport`; use `app.dependency_overrides` for auth/services. |
| Integration client | Request shape of `app/integrations/*` | `tests/test_integrations.py`; mock HTTP, assert URL/headers/body/timeout and error mapping. |

### Rules
- Use the async test client from the start:
  ```python
  from collections.abc import AsyncIterator

  import pytest
  from httpx import ASGITransport, AsyncClient

  from app.main import app as fastapi_app

  @pytest.fixture
  async def api_client() -> AsyncIterator[AsyncClient]:
      transport = ASGITransport(app=fastapi_app)
      async with AsyncClient(transport=transport, base_url="http://testserver") as client:
          yield client

  @pytest.mark.asyncio
  async def test_create_invoice_returns_201(api_client: AsyncClient) -> None:
      response = await api_client.post("/invoices", json={"customer_id": 1})
      assert response.status_code == 201
  ```
- New feature ⇒ new tests in the same change. Bug fix ⇒ write the failing regression test first.
- Test failure paths, not only the happy path (validation error, not found, forbidden, `ExternalServiceError` → `FAILED`).
- One behavior per test; name tests `test_<unit>_<scenario>_<expected>`.
- Share setup via fixtures/factories; do not copy-paste setup blocks.
- Tests never hit the real network or a shared/production database.
- If a test needs heavy mocking of internals, the code under test probably has too many responsibilities — fix the design instead.

---

## 15. Validation & Definition of Done

After modifying Python code, run:

```shell
ruff check --fix app tests
ruff format app tests
pytest -q                         # or the relevant subset: pytest tests/modules/<domain> -q
# when configured in pyproject.toml:
mypy app                          # or: pyright
lint-imports                      # import-linter layer contracts
```

Done means **all** of the following:
1. Ruff (lint + format) passes on touched files.
2. Relevant tests pass; new behavior has tests.
3. Type checking and import-linter pass, if configured.
4. No unannotated function (every parameter and return type is typed), and no new `Any`, bare `except`, `print`, hardcoded secret/host, or function-level import without a comment.
5. Every new/changed route shows a concrete response schema in `/docs` (typed return annotation).
6. Migrations (if any) reviewed, reversible, and apply cleanly.
7. The final message states **what was run and the result**, and lists anything not verified.

Do not claim a change works without verification. If a check cannot be run, say so explicitly.

---

## 16. Python Conventions

- Python 3.12+ syntax: `X | None` (not `Optional`), built-in generics (`list[str]`), `StrEnum`, `pathlib`, f-strings, `datetime.now(UTC)` (not `utcnow()`).
- Follow PEP 8; Ruff for linting and formatting; enable Ruff `N` (`pep8-naming`).
- `snake_case` for variables, functions, parameters; `PascalCase` for classes; `UPPER_SNAKE_CASE` for constants.
- Booleans use `is_`, `has_`, `can_`, or `should_` prefixes.
- **Type annotations are mandatory** on every parameter and return value — see [Type annotations](#type-annotations-must) below.
- No mutable default arguments; no wildcard imports; no `print`.
- Comments explain **why**, not what. Public service methods get a one-line docstring (purpose, and `Raises:` when non-obvious).

### Type annotations (MUST)

Every function and method in `app/` — route handlers, dependencies, service methods, helpers (including private ones), validators — must annotate **every parameter and the return type**. Tests and Alembic migrations are exempt.

1. **Annotate everything in the signature**, including `-> None` for functions that return nothing. `self` / `cls` are excluded. Type `*args` / `**kwargs` by element type (`*args: int`, `**kwargs: str`).
2. **Use standard modern syntax (Python 3.12+)**, not legacy `typing` aliases:
   - `str | None`, `int | str` — not `Optional[...]` / `Union[...]`.
   - `list[str]`, `dict[str, int]`, `tuple[int, ...]`, `set[str]` — not `List`, `Dict`, `Tuple`, `Set`.
   - Abstract types come from `collections.abc`: `Sequence`, `Mapping`, `Iterable`, `Iterator`, `AsyncIterator`, `Callable`, `Awaitable`.
   - Import from `typing` only what has no built-in equivalent: `Protocol`, `TypedDict`, `Literal`, `Annotated`, `Self`, `Final`, `ClassVar`, `cast`, `TYPE_CHECKING`.
   - Use PEP 695 syntax for generics and aliases: `def first[T](items: Sequence[T]) -> T`, `type UserId = UUID`.
3. **Parameters accept the most general type, returns are concrete**: take `Sequence[X]` / `Mapping[K, V]` / `Iterable[X]`; return `list[X]` / `dict[K, V]`.
4. **Return types are explicit and precise.** `async def` is annotated with the awaited result (`async def get_user() -> User`), not `Coroutine`. Generators use `Iterator[X]` / `AsyncIterator[X]`. **Route handlers must annotate a precise response type** (`-> OrderRead`, or the project's envelope type) — it generates the OpenAPI schema (see §14). Dependencies return concrete types (see §8).
5. **Structured data gets a named type** — Pydantic model, `dataclass`, `TypedDict` or `NamedTuple` — not `dict[str, Any]` or `tuple[Any, ...]`. Closed sets of values use `StrEnum` or `Literal`.
6. **Annotate attributes**: class attributes (dataclass / Pydantic / SQLModel fields are required to be typed by their frameworks), and module-level constants that should not change (`MAX_PAGE_SIZE: Final = 100`).
7. **`Any` is a last resort**: allowed only at a genuinely dynamic boundary (e.g. an untyped third-party payload), with a comment saying why. Prefer validating into a Pydantic model, or `object` when you only need to pass a value through.
8. **Suppressions must be specific and justified**: `# type: ignore[error-code]  # reason` / `# noqa: ANN401  # reason`. No bare `# type: ignore`. Do not silence the checker by weakening a type to `Any` or `object` — fix the type.
9. **FastAPI / Pydantic / SQLModel evaluate annotations at runtime.** In route, schema and model modules, import the types you use normally (not under `if TYPE_CHECKING:`), and do not add `from __future__ import annotations`. Use string forward references only for genuine self-references.

```python
from collections.abc import Sequence
from decimal import Decimal
from uuid import UUID

def total_amount(items: Sequence[OrderItem], discount: Decimal | None = None) -> Decimal: ...

async def find_user(user_id: UUID) -> User | None: ...

def mark_failed(job: Job, error_message: str) -> None: ...

# WRONG
def total(items, discount=None): ...                       # unannotated
def total(items: List[Dict]) -> Optional[Any]: ...         # legacy aliases + Any
async def find_user(user_id: UUID) -> Coroutine[Any, Any, User]: ...  # annotate the awaited result
```

Enforced by Ruff `ANN` (missing parameter/return annotations) and `UP` (legacy `typing` syntax), plus `disallow_untyped_defs` in mypy — see `pyproject.tooling.snippet.toml`.

### Naming: use full meaningful words
```python
# WRONG: single letters and truncated names
for v in validators: ...
lo = min_date(row)
bucket[v] = v in found

# CORRECT
for validator in validators: ...
resolved_min_date = min_date(row) if callable(min_date) else min_date
bucket[value] = value in found
```

- Local variables, loop variables and comprehension targets use full words
  (`value`, `field`, `record_index`, `query_result`, `prefetch_hook`), never
  bare `v`, `f`, `r`, `e`, `x` or truncations like `conds`, `fmts`, `dec`, `pre`.
- Name the resolved value, not the computation: `resolved_cache_key`,
  `resolved_min_date`, `parsed_decimal` (instead of `key`, `lo`, `dec`).
- Predicates read as assertions: `is_allowed`, `distinct_pending`.
- Keep widely-established domain/API names as-is (`ctx`, `db`, `row`)
  when renaming would ripple across the public call surface; document the
  exception where it occurs.
- `ruff check` enforces mechanical naming (`pep8-naming`); descriptiveness
  is a code-review discipline no linter can check.

---

## 17. Project-specific Commands

### Database Migrations
```shell
alembic upgrade head                                # Apply migrations
alembic revision --autogenerate -m "description"    # Create new migration (review before committing!)
```

### Run Development Server
```shell
uvicorn app.main:app --reload --port 8000
```

---

## 18. Quick Reference

| Scenario | Solution |
|----------|----------|
| Non-blocking I/O | `async def` route with `await` |
| Blocking I/O | `def` route (sync) |
| Sync library in async | `run_in_threadpool()` |
| CPU-intensive | Task queue / process pool |
| Request validation | Dependencies with DB checks |
| Shared validation | Chain dependencies |
| Config per domain | Separate `BaseSettings` classes |
| Complex DB queries | SQL with JSON aggregation |
| Business calculation | Pure function in `rules.py` |
| Call external system | `app/integrations/<service>.py` |
| New variant of existing behavior | Add a class/function + register; don't extend `if/elif` |
| Need to commit | Only in the service, once per use case |
| Domain error | Raise a domain exception; handler maps it to HTTP |
| Test needs an external service | Stub at the integrations boundary |
| Same logic appears a second time | Leave it; extract on the third occurrence, and only if the copies change for the same reason |
| A shared function needs a boolean flag to serve two callers | Split it into two functions (or use an enum) |
| Same rule/constant/status needed in several places | One definition (constant, enum or `rules.py` function), imported everywhere |
| Writing any function | Annotate every parameter and the return type (`-> None` too), using `X | None` / `list[str]` syntax |
| Writing a route handler | Return annotation with the concrete response type (`-> XxxRead`) — it generates the OpenAPI schema |
