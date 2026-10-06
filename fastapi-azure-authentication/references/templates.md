# Generic Templates

All templates are intentionally generic. Replace names/placeholders to match the target
project's conventions. Drive everything from settings — never hard-code IDs, tenants, or scopes.

## 1. Settings (only if the project has none)

Uses `pydantic-settings`. Create a config module (e.g. `config.py`).

```python
from functools import lru_cache

from pydantic import AnyHttpUrl, computed_field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8", extra="ignore")

    # Azure authentication
    APP_CLIENT_ID: str
    OPENAPI_CLIENT_ID: str
    TENANT_ID: str | None = None          # required for single-tenant
    CONFIG_URL: str | None = None         # required for B2C
    SCOPE_NAME: str = "user_impersonation"

    BACKEND_CORS_ORIGINS: list[AnyHttpUrl] = []

    @computed_field
    @property
    def SCOPES(self) -> dict[str, str]:
        return {f"api://{self.APP_CLIENT_ID}/{self.SCOPE_NAME}": self.SCOPE_NAME}


@lru_cache
def get_settings() -> Settings:
    return Settings()  # type: ignore[call-arg]


settings = get_settings()
```

If the project **already** has a settings class, add the Azure fields and the `SCOPES`
property to it instead of creating this file.

## 2. `authentication.py`

Pick the block matching the chosen auth type. Each defines `azure_scheme`.

### Single-tenant

```python
from fastapi_azure_auth import SingleTenantAzureAuthorizationCodeBearer

from .config import settings  # adjust import to the project

azure_scheme = SingleTenantAzureAuthorizationCodeBearer(
    app_client_id=settings.APP_CLIENT_ID,
    tenant_id=settings.TENANT_ID,
    scopes=settings.SCOPES,
)
```

### Multi-tenant

```python
from fastapi_azure_auth import MultiTenantAzureAuthorizationCodeBearer

from .config import settings

azure_scheme = MultiTenantAzureAuthorizationCodeBearer(
    app_client_id=settings.APP_CLIENT_ID,
    scopes=settings.SCOPES,
    validate_iss=False,  # set True and pass iss_callable to restrict tenants
)
```

With issuer validation:

```python
from fastapi_azure_auth import MultiTenantAzureAuthorizationCodeBearer
from fastapi_azure_auth.exceptions import InvalidIssuer

from .config import settings


async def validate_issuer(tid: str) -> str:
    allowed = {settings.TENANT_ID}
    if tid not in allowed:
        raise InvalidIssuer("Tenant not allowed")
    return f"https://login.microsoftonline.com/{tid}/v2.0"


azure_scheme = MultiTenantAzureAuthorizationCodeBearer(
    app_client_id=settings.APP_CLIENT_ID,
    scopes=settings.SCOPES,
    validate_iss=True,
    iss_callable=validate_issuer,
)
```

### B2C multi-tenant

```python
from fastapi_azure_auth import B2CMultiTenantAuthorizationCodeBearer

from .config import settings

azure_scheme = B2CMultiTenantAuthorizationCodeBearer(
    app_client_id=settings.APP_CLIENT_ID,
    openid_config_url=settings.CONFIG_URL,
    scopes=settings.SCOPES,
    validate_iss=False,  # set True and pass iss_callable to restrict tenants
)
```

## 3. Wiring into the FastAPI app

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI, Security
from fastapi.middleware.cors import CORSMiddleware

from .authentication import azure_scheme
from .config import settings


@asynccontextmanager
async def lifespan(app: FastAPI):
    await azure_scheme.openid_config.load_config()
    yield


app = FastAPI(
    lifespan=lifespan,
    swagger_ui_oauth2_redirect_url="/oauth2-redirect",
    swagger_ui_init_oauth={
        "usePkceWithAuthorizationCodeGrant": True,
        "clientId": settings.OPENAPI_CLIENT_ID,
    },
)

if settings.BACKEND_CORS_ORIGINS:
    app.add_middleware(
        CORSMiddleware,
        allow_origins=[str(o).rstrip("/") for o in settings.BACKEND_CORS_ORIGINS],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
```

If the app already uses startup events instead of `lifespan`:

```python
@app.on_event("startup")
async def load_config() -> None:
    await azure_scheme.openid_config.load_config()
```

## 4. Applying security to endpoints

### All endpoints (at router include)

```python
from fastapi import Security

from .authentication import azure_scheme
from .config import settings

app.include_router(
    router,
    dependencies=[Security(azure_scheme, scopes=[settings.SCOPE_NAME])],
)
```

### Specific endpoints only

```python
from fastapi import APIRouter, Security

from .authentication import azure_scheme
from .config import settings

router = APIRouter()


@router.get("/public")
async def public_route():
    return {"status": "ok"}


@router.get("/secure", dependencies=[Security(azure_scheme, scopes=[settings.SCOPE_NAME])])
async def secure_route():
    return {"status": "authorized"}
```

### Accessing the authenticated user

```python
from fastapi import Depends
from fastapi_azure_auth.user import User

from .authentication import azure_scheme


@router.get("/me")
async def me(user: User = Depends(azure_scheme)):
    return {"name": user.name, "claims": user.claims}
```

Notes:
- Use `Security(...)` to enforce and document scopes; use `Depends(...)` when you only need
  the `User` object and scope enforcement is already applied upstream.
- Omit `scopes=[...]` entirely if the user chose not to enforce scopes (valid-token-only).
