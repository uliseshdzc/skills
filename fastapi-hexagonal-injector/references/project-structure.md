# Project structure

## Folder tree

A hexagonal FastAPI service is organized by **architectural layer**, not by feature. Each layer is
a package under `src/`. `__init__.py` files re-export the public names of a package so imports stay
short (`from src.application.ports import IGreetingProvider`).

```text
<project-root>/
  requirements.txt
  .vscode/
    launch.json                    # uvicorn debug config (--factory)
    extensions.json                # recommended extensions
  src/
    __init__.py
    app.py                         # composition root: builds injector + FastAPI app
    di.py                          # the single DI Module (ports -> adapters, providers)
    domain/                        # innermost layer: pure business types, no external deps
      entities/
        __init__.py
        greeting.py                # Pydantic entity
      enums/
        __init__.py
      settings/
        __init__.py
        settings.py                # pydantic-settings BaseSettings
    application/                   # use cases + the ports they depend on
      ports/
        __init__.py                # re-exports the interfaces
        greeting_provider_interface.py
      use_cases/
        __init__.py                # re-exports the use cases
        say_hello.py
    infrastructure/                # adapters that implement the ports
      greeting/
        __init__.py
        static_greeting_provider.py
    presentation/                  # FastAPI delivery layer
      __init__.py
      api/
        controllers/
          __init__.py
          hello.py                 # GET /hello router
  tests/
    unit_tests/
    integration_tests/
```

## What each folder is for

- **`domain/`** — the core. Entities are Pydantic `BaseModel`s; enums are business enums;
  `settings/` holds configuration models. Nothing here imports from `application`,
  `infrastructure`, or `presentation`, nor from FastAPI.
- **`application/ports/`** — abstract base classes (`ABC` + `@abstractmethod`) that describe the
  capabilities the use cases need (a repository, a provider, an external service). These are the
  **hexagon edges**.
- **`application/use_cases/`** — one class per use case, orchestrating domain objects and ports.
  Depends only on `domain` and `application/ports`.
- **`infrastructure/`** — concrete adapters. Each class **implements a port** from
  `application/ports` (e.g. a Cosmos repository, an in-memory provider, an HTTP client wrapper).
- **`presentation/api/controllers/`** — FastAPI routers. Thin: parse the request, call a use case
  resolved via `Injected(...)`, return the result.
- **`app.py`** — the composition root that assembles everything (see
  `dependency-injection.md`).
- **`di.py`** — the single `injector.Module` binding ports to adapters.

## `requirements.txt`

Minimal set for this scaffold. **Pin the current stable versions** — look each up on PyPI
(`https://pypi.org/pypi/<package>/json` → `info.version`, or `pip index versions <package>`) rather
than copying the pins below, which drift out of date:

```text
fastapi==0.136.3
uvicorn[standard]
injector==0.23.0
fastapi-injector==0.9.0
pydantic==2.9.2
pydantic-settings==2.12.0
```

Add adapters' SDKs (e.g. `azure-cosmos`, `httpx`) only in the layer that needs them —
infrastructure — never in `domain` or `application`.
