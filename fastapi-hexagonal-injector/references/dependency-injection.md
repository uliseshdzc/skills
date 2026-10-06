# Dependency injection with `injector` + `fastapi-injector`

The service uses **constructor injection**: a class declares what it needs as typed constructor
parameters, and `injector` supplies them. `fastapi-injector` bridges that container into FastAPI
routes. There are three moves.

## Move 1 — `@inject` the constructors

Every use case and every adapter decorates its `__init__` with `@inject` and types each dependency
by its **interface** (for ports) or concrete type (for providers/clients). Injector reads the
annotations and builds the object graph.

```python
# src/application/use_cases/say_hello.py
from injector import inject

from src.application.ports import IGreetingProvider


class SayHello:
    @inject
    def __init__(self, greeting_provider: IGreetingProvider):
        self._greeting_provider = greeting_provider

    def execute(self, name: str | None = None) -> str:
        return self._greeting_provider.greet(name)
```

The use case knows only `IGreetingProvider` — never the concrete adapter. That is dependency
inversion: the consumer owns the contract.

## Move 2 — one `Module` binds ports to adapters

A single `injector.Module` is the only place interfaces meet implementations.

- `binder.bind(IPort, to=Adapter)` — resolve the interface to a concrete class.
- `@singleton @provider def provide_x(self, ...) -> X:` — construct something that needs building
  (an SDK client, a DB handle, `Settings`) and cache it for the app's lifetime.

```python
# src/di.py
from injector import Binder, Module, provider, singleton

from src.application.ports import IGreetingProvider
from src.domain.settings.settings import Settings
from src.infrastructure.greeting.static_greeting_provider import StaticGreetingProvider


class DependencyInjectionModule(Module):
    def configure(self, binder: Binder) -> None:
        binder.bind(IGreetingProvider, to=StaticGreetingProvider)

    @singleton
    @provider
    def provide_settings(self) -> Settings:
        return Settings()
```

Use `@singleton` for expensive or shared resources so they are created once. Plain `bind`
(no `@singleton`) yields a fresh instance per resolution — fine for cheap, stateless use cases and
adapters.

## Move 3 — `attach_injector` + `Injected(...)`

In the composition root, build **one** `Injector` from the module, create the FastAPI app, and
`attach_injector` it. Routes then declare dependencies with `Injected(...)`.

```python
# src/app.py
from functools import lru_cache

from fastapi import FastAPI
from fastapi_injector import attach_injector
from injector import Injector

from src.di import DependencyInjectionModule
from src.presentation.api.controllers import hello


@lru_cache
def get_injector() -> Injector:
    return Injector(DependencyInjectionModule())


def api() -> FastAPI:
    app = FastAPI(title="Hello Hexagonal API", version="LOCAL")
    app.include_router(hello.router, prefix="/api/v1")
    attach_injector(app, get_injector())
    return app
```

```python
# in a controller
from fastapi import APIRouter
from fastapi_injector import Injected

from src.application.use_cases import SayHello

router = APIRouter(tags=["Hello"])


@router.get("/hello")
async def hello(name: str | None = None, say_hello: SayHello = Injected(SayHello)):
    return {"message": say_hello.execute(name)}
```

`Injected(SayHello)` asks the attached injector to resolve `SayHello`, which pulls in
`IGreetingProvider` → `StaticGreetingProvider` automatically. `@lru_cache` guarantees the same
injector instance is attached and reused, not rebuilt per request.

## Common failures

- **`@inject` missing** → injector cannot resolve constructor args; the route 500s at request time.
  Check this first.
- **Binding the class instead of the interface** → consumers coupled to the adapter. Always
  `bind(IPort, to=Adapter)`.
- **New `Injector()` per request** → singletons rebuilt, state lost. Build once (`@lru_cache`) and
  attach that instance.
- **Adapter imported by a use case** → dependency-direction violation; depend on the port instead.
