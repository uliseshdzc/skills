# fastapi-azure-auth Authentication Schemes

Always confirm the available classes against the **installed** version before coding
(import them and/or read the package source). As of library version **5.x** the public
schemes live in `fastapi_azure_auth` and are:

| Class | Use when | Key required params |
|-------|----------|---------------------|
| `SingleTenantAzureAuthorizationCodeBearer` | App serves one Azure AD tenant | `app_client_id`, `tenant_id`, `scopes` |
| `MultiTenantAzureAuthorizationCodeBearer` | App serves many tenants | `app_client_id`, `scopes`, `validate_iss` (+ `iss_callable` if validating) |
| `B2CMultiTenantAuthorizationCodeBearer` | Azure AD B2C tenants | `app_client_id`, `scopes`, `openid_config_url`, `validate_iss` (+ `iss_callable` if validating) |

They all extend `AzureAuthorizationCodeBearerBase`, so these optional params are shared:

- `auto_error: bool = True` — return `None` instead of raising when `False`.
- `scopes: dict[str, str]` — map of full scope string → human description, e.g.
  `{f'api://{app_client_id}/user_impersonation': 'user impersonation'}`.
- `leeway: int = 0` — clock-skew tolerance in seconds.
- `allow_guest_users: bool = False` — allow guest (B2B) accounts.
- `openid_config_use_app_id: bool = False` — set `True` only when using claims-mapping.
- `openapi_authorization_url`, `openapi_token_url`, `openapi_description` — OpenAPI overrides.
- `scheme_name: str` — name shown in OpenAPI security docs.

## Multi-tenant issuer validation

When `validate_iss=True`, you must provide an async `iss_callable(tid) -> iss`. A common
pattern restricts logins to an allow-list of tenant IDs:

```python
async def validate_issuer(tid: str) -> str:
    allowed = {settings.TENANT_ID}  # one or more tenant IDs from settings
    if tid not in allowed:
        raise InvalidIssuer("Tenant not allowed")
    return f"https://login.microsoftonline.com/{tid}/v2.0"
```

Set `validate_iss=False` to accept any tenant (open sign-in).

## Verifying against the installed version

```python
import fastapi_azure_auth
print(fastapi_azure_auth.__version__)
# Inspect what's actually exported:
print([n for n in dir(fastapi_azure_auth) if "Bearer" in n])
```

Prefer importing the class from the top-level package
(`from fastapi_azure_auth import SingleTenantAzureAuthorizationCodeBearer`); fall back to
`from fastapi_azure_auth.auth import ...` only if the top-level export is unavailable.
