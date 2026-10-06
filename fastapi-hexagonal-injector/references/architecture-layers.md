# Layer templates — the slice pattern (Hello-World example)

These are the **pattern to adapt**, one file per layer, not the app to ship. The greeting slice
below (`GET /api/v1/hello?name=Ada` → `{"message": "Hello, Ada!"}`) shows how a single feature
flows through presentation → application → infrastructure, wired by DI. When building for a real
`FIRST_FEATURE`, rename each part to the user's domain (`Greeting`→their entity,
`IGreetingProvider`→their port, `SayHello`→their use case, `StaticGreetingProvider`→their adapter)
and swap the in-memory adapter for the chosen kind (see *Adapter variants* below). Use the
greeting verbatim **only** when the user explicitly just wants the runnable skeleton. Create files
in the order below.

## Domain

### `src/domain/entities/greeting.py`

```python
from pydantic import BaseModel


class Greeting(BaseModel):
    message: str
```

### `src/domain/entities/__init__.py`

```python
from src.domain.entities.greeting import Greeting

__all__ = ["Greeting"]
```

### `src/domain/settings/settings.py`

```python
from injector import singleton
from pydantic_settings import BaseSettings, SettingsConfigDict


@singleton
class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env", extra="ignore", env_nested_delimiter="__"
    )

    frontend_origin: str = "*"
```

### `src/domain/settings/__init__.py`

```python
from src.domain.settings.settings import Settings

__all__ = ["Settings"]
```

## Application

### `src/application/ports/greeting_provider_interface.py`

```python
from abc import ABC, abstractmethod


class IGreetingProvider(ABC):
    @abstractmethod
    def greet(self, name: str | None = None) -> str: ...
```

### `src/application/ports/__init__.py`

```python
from src.application.ports.greeting_provider_interface import IGreetingProvider

__all__ = ["IGreetingProvider"]
```

### `src/application/use_cases/say_hello.py`

```python
import logging

from injector import inject

from src.application.ports import IGreetingProvider


class SayHello:
    @inject
    def __init__(self, greeting_provider: IGreetingProvider):
        self._greeting_provider = greeting_provider
        self._logger = logging.getLogger(self.__class__.__name__)

    def execute(self, name: str | None = None) -> str:
        self._logger.info("Building greeting")
        return self._greeting_provider.greet(name)
```

### `src/application/use_cases/__init__.py`

```python
from src.application.use_cases.say_hello import SayHello

__all__ = ["SayHello"]
```

## Infrastructure

### `src/infrastructure/greeting/static_greeting_provider.py`

```python
from src.application.ports import IGreetingProvider


class StaticGreetingProvider(IGreetingProvider):
    def greet(self, name: str | None = None) -> str:
        return f"Hello, {name}!" if name else "Hello, World!"
```

### `src/infrastructure/greeting/__init__.py`

```python
```

## Adapter variants

The greeting adapter above is **static/in-memory**. For a real feature, generate the adapter kind
that matches the port's backing resource. All three implement the *same port*, so the use case and
route never change — only the class and its DI wiring do.

### In-memory (default; tests, first run, no infra)

Holds state in a `dict`/`list`. No client, no extra `requirements`.

```python
from src.application.ports import IThingRepository
from src.domain.entities import Thing


class InMemoryThingRepository(IThingRepository):
    def __init__(self):
        self._items: dict[str, Thing] = {}

    def add(self, thing: Thing) -> Thing:
        self._items[thing.id] = thing
        return thing

    def list(self) -> list[Thing]:
        return list(self._items.values())
```

### Database-backed (e.g. Cosmos DB)

Takes the client via `@inject`; the client is provided as a `@singleton` in `di.py`.

```python
from azure.cosmos.database import DatabaseProxy
from injector import inject

from src.application.ports import IThingRepository
from src.domain.entities import Thing


class CosmosThingRepository(IThingRepository):
    @inject
    def __init__(self, cosmos_client: DatabaseProxy):
        self._container = cosmos_client.get_container_client("Thing")

    def add(self, thing: Thing) -> Thing:
        created = self._container.create_item(thing.model_dump(mode="json", by_alias=True))
        return Thing(**created)

    def list(self) -> list[Thing]:
        items = self._container.read_all_items()
        return [Thing(**i) for i in items]
```

For SQL/PostgreSQL follow the same shape, injecting an engine/connection provided as a singleton.

### External HTTP API

Wraps an `httpx` client (provided as a `@singleton`).

```python
import httpx
from injector import inject

from src.application.ports import IThingRepository
from src.domain.entities import Thing


class HttpThingRepository(IThingRepository):
    @inject
    def __init__(self, client: httpx.Client):
        self._client = client

    def list(self) -> list[Thing]:
        response = self._client.get("/things")
        response.raise_for_status()
        return [Thing(**item) for item in response.json()]
```

See `reference/dependency-injection.md` for the matching `@singleton @provider` methods that build
each client in `di.py`.

## Presentation

### `src/presentation/api/controllers/hello.py`

```python
from fastapi import APIRouter
from fastapi_injector import Injected

from src.application.use_cases import SayHello
from src.domain.entities import Greeting

router = APIRouter(tags=["Hello"])


@router.get(
    "/hello",
    summary="Returns a greeting",
    response_model=Greeting,
)
async def hello(
    name: str | None = None,
    say_hello: SayHello = Injected(SayHello),
):
    return Greeting(message=say_hello.execute(name))
```

### `src/presentation/api/controllers/__init__.py`

```python
```

## Composition root

### `src/di.py`

```python
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

### `src/app.py`

```python
import logging
from functools import lru_cache

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi_injector import attach_injector
from injector import Injector

from src.di import DependencyInjectionModule
from src.domain.settings.settings import Settings
from src.presentation.api.controllers import hello

logging.basicConfig(level=logging.INFO)


@lru_cache
def get_injector() -> Injector:
    return Injector(DependencyInjectionModule())


def api() -> FastAPI:
    settings = get_injector().get(Settings)
    app = FastAPI(
        title="{{PROJECT_NAME}}",
        version="LOCAL",
        description="{{PROJECT_DESCRIPTION}}",
    )

    # Include only when WITH_CORS is yes.
    app.add_middleware(
        CORSMiddleware,
        allow_origins=[settings.frontend_origin],
        allow_methods=["*"],
        allow_headers=["*"],
        allow_credentials=True,
    )

    app.include_router(hello.router, prefix="{{API_PREFIX}}")
    attach_injector(app, get_injector())
    return app
```

## Run and verify

```bash
pip install -r requirements.txt
uvicorn src.app:api --factory --reload
```

Then (using the default `{{API_PREFIX}}` = `/api/v1`):

```bash
curl "http://127.0.0.1:8000/api/v1/hello"          # {"message":"Hello, World!"}
curl "http://127.0.0.1:8000/api/v1/hello?name=Ada" # {"message":"Hello, Ada!"}
```

Swagger UI is at `http://127.0.0.1:8000/docs`.

## VS Code

Add these two files under `.vscode/` so contributors get one-key debugging and the right
extensions on open.

### `.vscode/launch.json`

Runs the app under the Python debugger via uvicorn. `--factory` is required because `api()`
returns the app.

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "FastAPI: API",
            "type": "debugpy",
            "request": "launch",
            "module": "uvicorn",
            "args": ["src.app:api", "--port", "8000", "--reload", "--factory"],
            "cwd": "${workspaceFolder}",
            "jinja": true
        }
    ]
}
```

### `.vscode/extensions.json`

Recommended extensions VS Code offers to install when the workspace opens.

```json
{
    "recommendations": [
        "ms-python.python",
        "ms-python.vscode-pylance"
    ]
}
```

## Adding the next feature

Repeat the same slice for any real capability, keeping the dependency direction:

1. Entity in `domain/entities`.
2. Port (ABC) in `application/ports` describing the capability the use case needs.
3. Use case in `application/use_cases`, `@inject`-ing the port.
4. Adapter in `infrastructure` implementing the port (Cosmos, Postgres, HTTP, …).
5. `binder.bind(IPort, to=Adapter)` in `di.py`; add a `@singleton @provider` for any client the
   adapter needs.
6. Controller in `presentation/api/controllers` calling the use case via `Injected(...)`, then
   `include_router` it in `app.py`.
