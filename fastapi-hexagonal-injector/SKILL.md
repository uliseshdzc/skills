---
name: fastapi-hexagonal-injector
description: >-
  Use when creating a new FastAPI service (or restructuring an existing one) that must follow
  hexagonal / ports-and-adapters architecture with constructor dependency injection via the
  `injector` library and `fastapi-injector`. Covers the four-layer folder layout, the inward
  dependency rule, how to wire a single injector Module (bind ports to adapters, provide
  singletons), how to expose use cases in routes with `Injected(...)`, and a runnable
  Hello-World endpoint that exercises every layer.
compatibility: Any AI coding agent (e.g., GitHub Copilot, OpenAI Codex)
---

# Scaffolding a FastAPI service with hexagonal architecture and injector DI

This skill produces a **FastAPI** service laid out in **hexagonal (ports & adapters)**
architecture, with **constructor dependency injection** driven by
[`injector`](https://pypi.org/project/injector/) and
[`fastapi-injector`](https://pypi.org/project/fastapi-injector/). The Hello-World endpoint in the
templates is the **reference pattern** — a slice that touches all four layers — not the app you
ship. You ask the user what they are building and generate *their* first slice, named after their
domain, wired to *their* adapters.

Drive it interactively: **collect the required inputs first, then build the project step by
step**, creating each file as you go and confirming it runs before moving on.

## Required inputs

Before creating anything, confirm or collect these. If any are missing, **ask the user** — do not
assume `PROJECT_NAME` or what the application does.

| Input | Description | Example |
|---|---|---|
| `PROJECT_NAME` | Human-readable API title (shown in Swagger). | `AI Agent API` |
| `PROJECT_DESCRIPTION` | One-line description of the service. | `Internal API for the assistant` |
| `FIRST_FEATURE` | The first capability to implement: a verb + subject and its behavior. | `Create a chat session`, `List invoices for a user` |
| `ADAPTERS` | For each capability, what backs it: a database (which one), an external API, or in-memory. | `Cosmos DB`, `PostgreSQL`, `HTTP API`, `in-memory` |
| `API_PREFIX` | URL prefix for all routes. Defaults to `/api/v1`. | `/api/v1` |
| `WITH_CORS` | Whether to add CORS middleware and a `frontend_origin` setting. | `yes` / `no` |

The import root package is `src` (src-layout), matching the templates. If the user only gives a
name, ask what the application does and what its first feature is — **do not default to the
greeting example.** Only fall back to Hello-World if the user explicitly just wants the skeleton.

## From feature to slice: naming the parts

Turn the user's `FIRST_FEATURE` into concrete layer artifacts before writing code. Map the
greeting pattern onto their domain:

| Pattern part (greeting) | Becomes | Derived from |
|---|---|---|
| `Greeting` entity | the domain entity | the noun in the feature (`ChatSession`, `Invoice`) |
| `IGreetingProvider` port | the capability interface | the verb + resource (`IChatSessionRepository`) |
| `SayHello` use case | the use case class | the action (`CreateChatSession`) |
| `StaticGreetingProvider` adapter | the concrete adapter | the chosen backing resource (see below) |
| `hello` router / `GET /hello` | the controller + route | REST verb for the action (`POST /chat-sessions`) |

Confirm these names with the user before generating, so the code reads in their domain language,
not `Greeting`.

## Bootstrapping adapters

Each port needs a concrete adapter in `infrastructure`. Pick the kind from the user's `ADAPTERS`
answer, and for anything with a client/connection, add a `@singleton @provider` in `di.py`:

| Backing resource | Adapter to generate | Extra wiring |
|---|---|---|
| **In-memory** (default for a first run, tests, no infra) | a class holding a `dict`/`list`, implementing the port | none — just `bind(IPort, to=Adapter)`. |
| **Cosmos DB** | repository using `DatabaseProxy.get_container_client(...)` | `@singleton @provider` returning the Cosmos client; add `azure-cosmos` to requirements and Cosmos `Settings`. |
| **PostgreSQL / SQL** | repository using a connection/engine | `@singleton @provider` for the engine/pool; add the driver and DB `Settings`. |
| **External HTTP API** | client wrapping `httpx.Client`/`AsyncClient` | `@singleton @provider` for the HTTP client; add `httpx` and base-URL `Settings`. |

Start in-memory if the user is unsure or wants to see it run first, then swap the adapter later
— the use case does not change, only the binding in `di.py`. Add an adapter's SDK to
`requirements.txt` **only** when you generate that adapter, and only in `infrastructure`.

## The one thing people get wrong

Hexagonal architecture is a **dependency-direction discipline, not a folder-naming exercise**.
Creating `domain/`, `application/`, `infrastructure/`, `presentation/` folders buys nothing if
the imports still point the wrong way. The single rule that makes it work:

> **Dependencies point inward. `domain` depends on nothing. `application` depends only on
> `domain`. `infrastructure` and `presentation` depend on `application` and `domain` — never the
> reverse.**

Concretely: a use case never imports a concrete repository, an Azure client, or FastAPI. It
imports an **interface** (a port) defined in `application/ports`. The concrete adapter lives in
`infrastructure` and *implements* that port. The only place the two are joined is the DI module.
If you catch a `from src.infrastructure...` import inside a use case, the architecture is already
broken — stop and invert it behind a port.

## The four layers

| Layer | Folder | Holds | May import |
|---|---|---|---|
| Domain | `src/domain` | Entities (Pydantic models), enums, settings. Pure business types. | Nothing outside `domain`. |
| Application | `src/application` | `ports/` (ABC interfaces) and `use_cases/` (orchestration). | `domain` only. |
| Infrastructure | `src/infrastructure` | Adapters: DB repositories, HTTP/SDK clients. Each **implements a port**. | `application`, `domain`. |
| Presentation | `src/presentation` | FastAPI controllers/routers, auth. Thin: parse request → call use case → return. | `application`, `domain`. |

The **ports** in `application/ports` are the seams (the "hexagon" edges). Adapters plug into them
from `infrastructure`; controllers drive them from `presentation`. Business logic depends only on
the ports, so adapters (Cosmos, Postgres, in-memory, a stub in tests) are swappable without
touching a use case.

## Dependency injection: the three moves

DI is done with `injector` (constructor injection) + `fastapi-injector` (bridge into routes).
There are exactly three moves — learn these and the rest is mechanical:

1. **`@inject` the constructors.** Every use case and every adapter declares its dependencies as
   constructor parameters typed by their **interface**, decorated with `@inject`. Injector reads
   the annotations and supplies the instances.
2. **One `Module` binds ports to adapters.** A single `injector.Module`
   (`DependencyInjectionModule`) maps each interface to its concrete class with
   `binder.bind(IPort, to=Adapter)`, and uses `@singleton @provider` methods for anything that
   needs constructing (SDK clients, DB handles).
3. **`attach_injector` + `Injected(...)`.** In `app.py`, build one `Injector`, `attach_injector`
   it to the FastAPI app, and in each route declare the use case as `use_case = Injected(SayHello)`.
   FastAPI then resolves the full graph per request.

Full, copy-paste wiring is in `reference/dependency-injection.md`.

## Scaffold recipe

Once the inputs are collected and the slice is named (above), build **step by step, in order**,
creating each file before the next and substituting the inputs and domain names. Generate the
user's feature — the greeting files are the **template to adapt**, not literal output. Be
**idempotent**: if a file already exists, merge or skip rather than overwriting without
confirmation.

1. **Pin the latest package versions** — before writing `requirements.txt`, look up the current
   stable release of each dependency on PyPI (`https://pypi.org/pypi/<package>/json` → `info.version`,
   or `pip index versions <package>`) for `fastapi`, `uvicorn`, `injector`, `fastapi-injector`,
   `pydantic`, and `pydantic-settings`, plus any adapter SDK (`azure-cosmos`, `httpx`, a DB
   driver). Pin those versions rather than copying the sample pins, which go stale.
2. **Create the folder tree and `requirements.txt`** — see `reference/project-structure.md`, using
   the versions from step 1.
3. **Domain** — add the entity for the feature (e.g. `ChatSession`) and the `Settings` model
   (`pydantic-settings`), including any config the chosen adapter needs.
4. **Application** — define the port (ABC) for the capability in `ports/`, then the use case in
   `use_cases/` that depends on the port via `@inject`.
5. **Infrastructure** — generate the adapter implementing the port, of the kind chosen in
   *Bootstrapping adapters* (in-memory, Cosmos, SQL, or HTTP).
6. **DI** — in `di.py`, bind the port to the adapter and add a `@singleton` provider for
   `Settings` and for any client the adapter needs.
7. **Presentation** — add the router and route for the feature using `Injected(...)`.
8. **Compose** — in `app.py`, build the injector, create the FastAPI app, include the router at
   `{{API_PREFIX}}`, and `attach_injector` (add CORS only if `WITH_CORS`).
9. **VS Code** — add `.vscode/launch.json` (uvicorn debug config with `--factory`) and
   `.vscode/extensions.json` (recommended extensions). See `reference/architecture-layers.md`.
10. **Run** — `uvicorn src.app:api --factory --reload` (or F5 with the launch config), then hit
    the feature's route under `{{API_PREFIX}}`.

After each step, state what you created and pause if the user needs to review. The greeting
templates — the pattern to adapt for every layer — are in `reference/architecture-layers.md`.

## Hard rules / pitfalls

- **No inward-pointing violations.** Use cases import ports, never concrete adapters or FastAPI.
  If a use case needs FastAPI/`Request`, the logic is in the wrong layer.
- **Interfaces live in `application/ports`, not `infrastructure`.** The consumer owns the
  contract; the adapter conforms to it (dependency inversion).
- **`@inject` is required on every injected constructor.** Forget it and injector raises at
  resolution time with an unhelpful message — check this first when a route 500s on startup.
- **Bind the interface, not the class.** `binder.bind(IPort, to=Adapter)`. Routes and use cases
  depend on `IPort`; only the module knows `Adapter`.
- **Use `@singleton` for expensive/shared resources** (SDK clients, DB handles, `Settings`) so
  they are built once, not per request.
- **Controllers stay thin.** Parse input, call `use_case.execute(...)`, return the result. No
  business logic, no data access in the router.
- **One injector for the app.** Build it once (e.g. `@lru_cache`) and `attach_injector` that same
  instance; do not construct a new `Injector()` per request.
- **`--factory` matters.** `api()` returns the app, so run uvicorn with `--factory`.

## Reference map

| File | Read it when you need… |
|---|---|
| `reference/project-structure.md` | the full folder tree, what each folder holds, and `requirements.txt`. |
| `reference/dependency-injection.md` | the exact `injector` + `fastapi-injector` wiring: `Module`, `bind`, `@provider`/`@singleton`, `@inject`, `Injected`, `attach_injector`. |
| `reference/architecture-layers.md` | the slice pattern (greeting example) to adapt for every layer, plus in-memory / database / HTTP adapter variants. |
