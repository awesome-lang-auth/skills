# Rust: `awesome-rust-auth`

Checked against the `main` branch of https://github.com/awesome-lang-auth/awesome-rust-auth at commit `acb3d53` (2026-09-30; `Cargo.toml` version 1.9.0, edition 2024, rust-version 1.86), on 2026-10-08. Not run in a container.

Maturity: **preview**. Not published on crates.io: install from git. The README's `awesome-rust-auth = { version = "1.9.0" }` line does not resolve.

## Read this first: what the crate does and does not do

The crate gives you the auth **core**: `AuthService` (signup, login, refresh rotation, revocation, session listing, permission checks, account linking), `AccountService`, `MagicLinkService`, `OtpService`, TOTP, `ApiKeyManager`, CSRF token helpers, mail templates, OIDC/JWKS helpers, OpenAPI schemas, and the embedded UI assets. Persistence is yours, through traits.

It does **not** mount the auth HTTP API. `adapters::axum::router()` (and the actix and warp equivalents) serves the built-in UI (`/auth/ui/login`, `/auth/ui/auth.js`, `/auth/ui/config`), the admin UI assets and `/health`; its `POST /auth/login` is a stub that answers `ok`. You write the handlers for `/auth/login`, `/auth/refresh`, `/auth/me` and the rest, calling `AuthService`, and they must follow the family wire contract so the clients work. Tell the user this before choosing Rust, and budget for it.

## Install

```bash
cargo add awesome-rust-auth --git https://github.com/awesome-lang-auth/awesome-rust-auth \
  --rev acb3d5372a801b6a704b800c0576177029ed43f2 --features axum   # or actix, warp
cargo add async-trait tokio --features tokio/full                  # to implement the store traits
```

The command pins the commit this reference describes. Without `--rev`, cargo follows the moving `main` branch; move to a newer commit only after checking its changes against this file.

## Core usage

Inside an async function returning `Result<_, Box<dyn std::error::Error>>`:

```rust
use std::sync::Arc;
use awesome_rust_auth::{AuthConfig, AuthService, models::{LoginInput, SignupInput}};

let config = AuthConfig::builder()
    .issuer("https://api.example.com")
    .audience("my-app")
    .jwt_secret(std::env::var("JWT_SECRET")?)     // at least 16 characters; use 32+ random bytes
    .build()?;

let svc = AuthService::new(config, Arc::new(users), Arc::new(sessions), Arc::new(telemetry), Arc::new(events));

let user = svc.signup(SignupInput { email: "alice@example.com".into(), password: "s3cur3P@ssw0rd".into(), tenant_id: "default".into() }).await?;
let (access, refresh) = svc.login(LoginInput { email: "alice@example.com".into(), password: "s3cur3P@ssw0rd".into(), tenant_id: "default".into() }).await?;
// access.token (15 min), refresh.token (30 days by default), refresh.token_id for revocation
let (access, refresh) = svc.rotate_refresh_token(&refresh.token).await?;
svc.revoke_refresh_token(refresh.token_id).await?;
```

`users`, `sessions`, `telemetry` and `events` implement the `UserStore`, `SessionStore`, `TelemetryStore` and `EventBus` traits (`awesome_rust_auth::traits`); `event_bus::InMemoryEventBus` is provided. `tenant_id` must be non-empty (use one constant tenant if the app has none). Passwords are hashed with Argon2id.

Config builder defaults: issuer `awesome-rust-auth`, audience `awesome-rust-auth-clients`, access TTL 15 minutes, refresh TTL 30 days, `auth_ui_path("/auth/ui")`, `auth_js_path("/auth/ui/auth.js")`, `admin_ui_path("/auth/admin")`, `enable_idp_mode(false)`.

## Writing the HTTP layer (axum outline)

Do not `merge` `adapters::axum::router()` into a router that defines `POST /auth/login`: both register that route and axum panics on the overlap. Write your own routes and serve the UI from the crate's `ui` module, which exposes the assets as functions and constants:

```rust
use axum::{Router, routing::{get, post}, extract::Path, http::{StatusCode, header}, response::IntoResponse};
use awesome_rust_auth::ui;

async fn ui_file(Path(tail): Path<String>) -> impl IntoResponse {
    let tail = tail.trim_matches('/');
    if let Some((content_type, body)) = ui::auth_ui_asset(tail) {        // auth.js, base.css, ...
        return ([(header::CONTENT_TYPE, content_type)], body).into_response();
    }
    if let Some(page) = ui::auth_ui_page(tail) {                          // login, register, 2fa, ...
        return ([(header::CONTENT_TYPE, "text/html; charset=utf-8")], page).into_response();
    }
    StatusCode::NOT_FOUND.into_response()
}

let app = Router::new()
    .route("/auth/login", post(login))       // your handlers, calling AuthService
    .route("/auth/refresh", post(refresh))
    .route("/auth/logout", post(logout))
    .route("/auth/me", get(me))
    .route("/auth/ui/config", get(ui_config))  // static segment wins over the wildcard below
    .route("/auth/ui/{*tail}", get(ui_file))
    .with_state(svc);
```

`ui_config` answers the JSON the pages and `auth.js` boot from. The crate's default is `ui::AUTH_UI_CONFIG_JSON`, with every `features` flag `false`; serve your own copy whose flags (`register`, `forgotPassword`, `twoFactor`, ...) match the routes you implemented.

The contract your handlers must keep (the same as Node, see `references/node.md` for the route list):

- Cookie mode: `POST /auth/login {email, password}` answers `200 {"success": true}` and sets HttpOnly `accessToken` and `refreshToken` cookies plus a JS-readable `csrf-token` cookie. Without `Secure` (local http only): bare names, `accessToken` on `Path=/` and `refreshToken` on `Path=/auth/refresh`. With `Secure` (everywhere else): all three are `__Host-`-prefixed and on `Path=/`, the refresh one included, because browsers reject a `__Host-` cookie with any other path (Node drops the refresh-path scoping the same way).
- Bearer mode, when the request has `X-Auth-Strategy: bearer`: the same answer with top-level `accessToken` and `refreshToken` and no cookies.
- `GET /auth/me` answers the user object unwrapped, id in `sub`.
- `POST /auth/refresh` reads the cookie, or `{refreshToken}` in the body in bearer mode; a revoked session is `401 {"code": "SESSION_REVOKED"}`, which makes the clients sign out.
- Errors are `{"error": "<message>", "code": "<CODE>"}`. CSRF failures are `403 {"code": "CSRF_INVALID"}`: compare the `csrf-token` cookie with the `X-CSRF-Token` header on cookie-authenticated POST, PUT, PATCH and DELETE (the crate's `csrf::generate_csrf_token` / `validate_csrf_token` make and check the token value).

The full extracted contract, with every route and field, is `docs/spec/wire-contract.md` in https://github.com/awesome-lang-auth/awesome-lambda-auth. After writing the handlers, run the SKILL.md request sequence against them.

The repository's `examples/axum-postgres`, `examples/actix-mongodb` and `examples/warp-sqlite` only build the UI router; they contain no database code.

## Stores

See `references/stores.md` for the trait methods.
