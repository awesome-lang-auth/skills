# Python (FastAPI): `awesome-python-auth`

Checked against `awesome-python-auth` 1.1.0 (PyPI) with FastAPI 0.143.0 on Python 3.12, on 2026-10-08. The snippet below was run through the SKILL.md request sequence on that date.

## Install

```bash
pip install awesome-python-auth
pip install "uvicorn>=0.30"      # to run the app
```

Python 3.11 or later. Import package: `awesome_python_auth`.

## FastAPI app

```python
import os

from fastapi import Depends, FastAPI
from fastapi.middleware.cors import CORSMiddleware
from awesome_python_auth import (
    AuthConfig, AuthConfigurator, CsrfMiddleware, build_ui_router, require_auth,
)
from awesome_python_auth.models import AuthUser
from .stores import PgUserStore          # your UserStore, see stores.md

PREFIX = "/api/auth"                     # the clients' apiPrefix must match
SECURE = os.environ.get("ENV") == "production"

config = AuthConfig(
    api_prefix=PREFIX,
    access_token_secret=os.environ["ACCESS_TOKEN_SECRET"],
    cookie_secure=SECURE,                # False only for local http
    cookie_same_site="lax",
    session_check_on="refresh",          # allcalls | refresh | none (default "none")
    totp_issuer="My App",
)

app = FastAPI()
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],   # exact origins, never "*"
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["Content-Type", "Authorization", "X-CSRF-Token", "X-Auth-Strategy"],
)
# The CSRF check runs only under api_prefix: "/api" covers /api/auth and your own /api routes.
app.add_middleware(CsrfMiddleware, api_prefix="/api", cookie_secure=SECURE)

app.include_router(AuthConfigurator(config, PgUserStore()).router())
app.mount(f"{PREFIX}/ui", build_ui_router(config=config), name="auth_ui")   # login pages + auth.js


@app.post("/api/notes", status_code=201)
async def create_note(user: AuthUser = Depends(require_auth)):
    return {"ok": True, "owner": user.sub}
```

Run with `uvicorn main:app`. Other dependencies: `get_current_user` (returns `None` when signed out), `require_auth` (401), `require_roles(["admin"])` (403 without one of the roles).

`build_ui_router` is keyword-only and returns a FastAPI sub-application, so it is **mounted**, not included: `app.mount(f"{PREFIX}/ui", build_ui_router(config=config))`. Some published docs show `app.include_router(build_ui_router(config))`, which fails.

What the run returned: register 201 `{"success":true,"userId":...}`; login 200 with the user in the body and cookies **`access-token`**, **`refresh-token`** (both HttpOnly, `Path=/`) and `csrf-token` (cookie names differ from Node; clients do not depend on them); `POST /api/notes` without `X-CSRF-Token` is `403 {"error":"CSRF token missing"}`; refresh 200; after logout, refresh is 401; `/api/auth/ui/auth.js` is served (the `/ui/assets/auth.js` spelling works here too); bearer login returns `accessToken` and `refreshToken` in the body.

**Bearer clients must send `X-Auth-Strategy: bearer` on every write.** `CsrfMiddleware` recognises bearer callers by that header, not by `Authorization`: a `POST` with only `Authorization: Bearer` is refused with `403 CSRF token missing`. The Flutter client sends the header on native platforms; set it yourself in other bearer clients.

## AuthConfig fields

`api_prefix` (default `"/api/auth"`), `access_token_secret`, `access_token_expires_in` (seconds, 900), `refresh_token_expires_in` (604800), `cookie_secure` (True), `cookie_same_site` ("lax"), `cookie_domain`, `cookie_prefix` (`"__Host-"` or `"__Secure-"`), `totp_issuer`, `ui_config`, `email`, `mailer`, `tools`, `event_bus`, `api_key_store`, `roles_permissions_store`, `tenant_store`, `id_provider`, `resource_server`, `session_check_on` ("none"), and hooks: `on_forgot_password`, `on_send_verification_email`, `on_request_email_change`, `on_magic_link_send`, `on_magic_link_verify`, `on_sms_send`, `on_sms_verify`, `on_link_request`, `on_link_verify`, `on_oauth_start`, `on_oauth_callback`, `on_register`.

There is one signing secret (`access_token_secret`); no separate refresh secret.

`AuthConfigurator(config, user_store).router(settings_store=None, on_register=None, event_bus=None)` returns an `APIRouter` already prefixed with `api_prefix`. `on_register` receives the `StoredUser` before it is saved; when you pass it, **you** persist the user (the default path calls `store.create`).

## Registration is always mounted

`POST <prefix>/register` exists on every deployment (409 on an existing email). If sign-up must be closed, refuse that path in a middleware or at the proxy.

## Other routers

- `app.include_router(build_admin_router(config=config, user_store=store, access_policy="is-admin-flag"))` serves the admin SPA and API (keyword-only; optional `session_store`, `rbac_store`, `tenant_store`, `settings_store`, `api_key_store`, `webhook_store` enable tabs). Keep `"is-admin-flag"` (the default); `"open"` and `"first-user"` are for local experiments only.
- `build_tools_router(tools)` for SSE and telemetry (`AuthTools`).
- IdP mode: `id_provider=IdProviderConfig(...)` adds RS256 tokens and `/.well-known/jwks.json`; `resource_server=ResourceServerConfig(...)` validates tokens from another issuer.

Repository: https://github.com/awesome-lang-auth/awesome-python-auth (README lists every endpoint and hook).
