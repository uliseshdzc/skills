---
name: fastapi-azure-authentication
description: 'Add Azure Entra ID (Azure AD) authentication to an existing FastAPI application using the fastapi-azure-auth library. USE FOR: add authentication to FastAPI, secure FastAPI endpoints, Azure AD / Entra ID auth, Azure token validation, protect API with OAuth2, add azure_scheme, single-tenant / multi-tenant / B2C auth, require scopes on routes. Installs the latest fastapi-azure-auth, creates an authentication module with the chosen Azure scheme, wires Pydantic settings and .env fields, and applies security to all or selected endpoints. DO NOT USE FOR: non-FastAPI apps, custom JWT schemes unrelated to Azure, or apps that already have fastapi-azure-auth configured.'
argument-hint: 'Optionally name the auth type (single-tenant, multi-tenant, B2C) and the scope'
---

# Add Azure Authentication to FastAPI

Adds Azure Entra ID authentication to an **existing** FastAPI app using
[`fastapi-azure-auth`](https://vibber-ai.github.io/fastapi-azure-auth/). The app is assumed to
currently have **no authentication**.

## When to Use

- An existing FastAPI application needs Azure Entra ID (Azure AD) authentication.
- Endpoints must validate bearer tokens and optionally enforce scopes.
- Single-tenant, multi-tenant, or Azure AD B2C scenarios.

## Guardrails

- Do **not** invent values. Every tenant/client ID, scope, and URL must come from the user.
- Keep all generated code **generic** — no business-, company-, or project-specific names, domains, or sample data.
- Prefer adapting the project's existing settings/config patterns over introducing new ones.
- Never commit secrets. Only names of env vars go into tracked files; values go into `.env` (which must be git-ignored).

## Procedure

### 1. Confirm the baseline

- Locate the FastAPI entry point (`FastAPI(...)` instantiation) and the routers/endpoints.
- Confirm there is no existing `fastapi-azure-auth` setup. If one exists, stop and tell the user.
- Identify how the app is configured today (see Step 4).

### 2. Install the latest library version

- Retrieve the **latest** version from PyPI before installing:
  `https://pypi.org/pypi/fastapi-azure-auth/json` → read `info.version`.
- Add a pinned entry `fastapi-azure-auth==<latest>` to the project's dependency file
  (`requirements.txt`, `pyproject.toml`, etc.), matching the existing style.
- Install into the active environment (`pip install` / `uv add`, matching the project).

### 3. Interview the user for required inputs

Use the ask-questions tool. Ask only what the chosen scheme needs.

**Always ask:**
1. **Auth type** — one of: `single-tenant`, `multi-tenant`, `B2C multi-tenant`.
   Confirm the available classes against the installed library before coding
   (see [references/schemes.md](./references/schemes.md)).
2. **Application (client) ID** — the backend app registration client ID.
3. **Scope(s)** — e.g. `user_impersonation`. Ask for the short scope name(s); the full
   value is usually `api://<client-id>/<scope>`.
4. **OpenAPI/Swagger client ID** — the client ID used by the interactive docs to log in
   (may be the same as the app client ID). Needed for the Swagger "Authorize" button.
5. **Apply scope enforcement?** — whether endpoints should merely require a valid token,
   or also require specific scope(s).

**Ask when relevant to the chosen type:**
- `single-tenant` → **Tenant ID**.
- `multi-tenant` → whether to **validate the issuer** (`validate_iss`); if yes, an
  `iss_callable` and the list of accepted tenant IDs.
- `B2C multi-tenant` → the **OpenID config URL** (the B2C well-known metadata URL),
  and issuer validation details as above.

**Also ask:**
- **Allow guest users?** (default: no).
- **Config file** to hold the new settings (e.g. `.env`) and the settings style in use.
- **Which endpoints to secure** — see Step 7.

### 4. Adapt or create settings

Detect the existing configuration approach and match it:

- **If Pydantic settings already exist** (e.g. a `Settings(BaseSettings)` class): add the new
  fields to that class and reuse the existing instance. Do not create a parallel settings system.
- **If settings exist in another form** (plain env reads, a config object, etc.): add the new
  values following that same pattern.
- **If there are no settings at all**: create a minimal `pydantic-settings` based config.
  Use the generic template in [references/templates.md](./references/templates.md).

Required setting names (adjust casing to the project's convention):
`APP_CLIENT_ID`, `OPENAPI_CLIENT_ID`, `TENANT_ID` (single-tenant / optional otherwise),
`SCOPE_NAME` (or a scopes map), `CONFIG_URL` (B2C only), `BACKEND_CORS_ORIGINS` (if CORS not set up).

Then **ask the user to add the corresponding keys to their config file** (`.env`, etc.).
Provide the exact keys to paste, with placeholder values, and remind them `.env` must be
git-ignored. Example of what to hand them (generic placeholders):

```dotenv
APP_CLIENT_ID=00000000-0000-0000-0000-000000000000
OPENAPI_CLIENT_ID=00000000-0000-0000-0000-000000000000
TENANT_ID=00000000-0000-0000-0000-000000000000
SCOPE_NAME=user_impersonation
```

### 5. Create the authentication module

Create `authentication.py` (place it alongside the app's other composition/dependency code).
It instantiates the chosen scheme as `azure_scheme`. Use the matching template in
[references/templates.md](./references/templates.md). Keep it generic and driven entirely by settings.

### 6. Wire the scheme into the FastAPI app

In the app entry point:
- Add `swagger_ui_oauth2_redirect_url='/oauth2-redirect'` and
  `swagger_ui_init_oauth={'usePkceWithAuthorizationCodeGrant': True, 'clientId': settings.OPENAPI_CLIENT_ID}`
  to the `FastAPI(...)` constructor.
- Ensure **CORS** is configured (at least for the local dev origin).
- Load the OpenID config on startup (lifespan handler or startup event):
  `await azure_scheme.openid_config.load_config()`.

See [references/templates.md](./references/templates.md) for drop-in snippets.

### 7. Apply security to endpoints

**Ask the user: apply authentication to _all_ endpoints, or only _specific_ ones?**

- **All endpoints** → add the dependency once at the app/router include level:
  ```python
  from fastapi import Security
  app.include_router(router, dependencies=[Security(azure_scheme, scopes=[settings.SCOPE_NAME])])
  ```
  (Omit `scopes=[...]` if the user chose not to enforce scopes.)
- **Specific endpoints** → list the endpoints with the user, then add
  `dependencies=[Security(azure_scheme, scopes=[...])]` to those individual routes or their
  routers only. Leave the rest public.

Use `Security(...)` (not `Depends(...)`) so scopes are enforced and documented in OpenAPI.

### 8. Validate

- Run the project's type checker / linter and fix issues in the files you changed.
- Start the app and confirm the OpenAPI docs show an **Authorize** button.
- Confirm protected endpoints return `401` without a token and succeed with a valid one.
- Report what was changed and the exact env keys the user must populate.

## References

- [references/schemes.md](./references/schemes.md) — the available auth schemes and their parameters.
- [references/templates.md](./references/templates.md) — generic settings, `authentication.py`, and wiring snippets.
