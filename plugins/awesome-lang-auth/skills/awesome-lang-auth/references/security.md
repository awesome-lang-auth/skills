# Security checklist, per port

Checked on 2026-10-08 against the versions in SKILL.md metadata. "Verified" marks behaviour observed in a running server on that date; the rest is read from the source.

## Defaults that differ between ports

| | Node 1.10.8 | Go v0.12.0 | Python 1.1.0 | Dart (git) | Lambda (git) |
|---|---|---|---|---|---|
| `POST <prefix>/register` | only with `onRegister` or `defaultRegister: true` | always mounted, and logs the new user in | always mounted | always mounted | mounted (Go core) |
| CSRF double-submit | off until `csrf: { enabled: true }` | on, for routes under the auth prefix | `CsrfMiddleware`, for paths under its `api_prefix` | `csrfMiddleware`, under `apiBasePath`; the router issues no auth cookies | on |
| Secure cookies | `cookieOptions.secure`, default `false` (set it) | `Secure: true` | `cookie_secure=True` | `cookieSecure: true` (unused today) | on |
| `__Host-` cookie names | automatic with `secure` | automatic with `Secure` | only with `cookie_prefix="__Host-"` | n/a | automatic |
| Signing secrets | access + refresh, must differ with a session store | one, 32+ characters | one | one | Secrets Manager / SSM references |
| CORS | `router({ cors: { origins } })` | none: add the router's middleware | Starlette `CORSMiddleware` | none: add one | `http.cors.origins` |
| Rate limiting | `router({ rateLimiter })`, one middleware for every auth route | `HTTPConfig.RateLimiter`, the same slot; no algorithm ships | none built in | none built in | on by default |

Rust ships the primitives only; every row is your handler's job there. The Dart column describes the source at the checked commit, which does not compile with Dart 3.13 (see `references/dart.md`).

## 1. Sign-up

Decide whether the app has open registration. Node 1.10+ does not mount `/register` unless asked: `defaultRegister: true` uses the built-in handler, which requires `email` and `password`, answers `409 USER_EXISTS` for a known address and stores only email, password hash, `firstName` and `lastName` (any other body field, such as `role` or `isAdmin`, is dropped); `onRegister` is your own handler and then the mass-assignment risk is yours: copy an allow-list of fields, never the body. Go, Python, Dart and Lambda always mount it: if sign-up is closed (invite-only, admin-created accounts), refuse `POST <prefix>/register` in front of the router or at the proxy.

## 2. Second factor

- When a login needs 2FA the server answers a short-lived `tempToken` instead of a session. In Node 1.10.0 and later it carries `purpose: '2fa'`, lives 5 minutes, and only the 2FA completion routes accept it; `auth.middleware()`, the admin router and every other route refuse it. Earlier Node versions accepted it as a session: upgrade.
- A `purpose` claim returned by `buildTokenPayload` is dropped from session tokens (the library reserves it for `'2fa'` and `'admin'`). In your own middleware, resource servers or other services that verify these JWTs, reject any token that has a `purpose` claim.
- Go issues a typed step-up token and refuses it as an access token as well. Dart answers `{requiresTwoFactor, userId}` without a step-up token; do not build a 2FA flow on it without reviewing that code path.
- TOTP secrets are stored by the user store: protect that column like a password hash (no logs, no API exposure). Consider `emailVerificationMode` and a 2FA policy for admin accounts.

## 3. CSRF (cookie mode)

- The double-submit pattern: the server sets a JS-readable `csrf-token` cookie; the client sends it back in `X-CSRF-Token` on POST, PUT, PATCH and DELETE; requests that authenticate with `Authorization: Bearer` skip the check.
- Node (verified): with `csrf.enabled`, `auth.middleware()` enforces it on your own routes too: a cookie-authenticated `POST` without the header gets `403 {"code":"CSRF_INVALID"}`. `/login`, `/refresh` and `/logout` do not check it.
- Go (verified): the mount enforces it only under the auth prefix. For your own routes wrap them in `auth.CSRFMiddleware(c)` where `c` is your `HTTPConfig` with `APIPrefix` set to their prefix (see `references/go.md`).
- Python (verified): `CsrfMiddleware` checks only paths starting with its `api_prefix`; give it a prefix that covers your API (for example `/api` with the auth router at `/api/auth`). It treats a request as bearer only when it carries `X-Auth-Strategy: bearer`, so bearer clients must send that header on writes.
- All clients (Angular, React, Flutter web, `auth.js`) add the header automatically. A hand-written client must read `__Host-csrf-token`, then `__Secure-csrf-token`, then `csrf-token`.

## 4. CORS and cross-origin setups

- Prefer one origin: serve the SPA and proxy the auth prefix (and the API) from the same host. Cookies stay `SameSite=Lax`, CSRF works, and CORS is not involved.
- If the API must live on another origin: an exact allow-list of origins with credentials (never `*` with cookies), allowed headers `Content-Type`, `Authorization`, `X-CSRF-Token`, `X-Auth-Strategy`, and cookies `SameSite=None; Secure` over HTTPS. Node: `cookieOptions: { secure: true, sameSite: 'none' }`, `router({ cors: { origins: [...] } })` and the same list in `email.siteUrl`. On a different **site** (not just a subdomain), the page cannot read the API host's CSRF cookie: use bearer mode for that client, or serve both under one parent domain.
- OAuth redirects and emailed links are built only for allow-listed origins (Node: `email.siteUrl` and `cors.origins`; since 1.10.4 an empty allow-list in production refuses to start an OAuth flow). Set `allowedReturnPaths` to the in-app paths OAuth may return to.

## 5. Cookies, HTTPS, tokens

- HTTPS everywhere outside local development; `Secure` cookies on. Drive the switch with an explicit opt-out that only local development sets (the references use `COOKIE_INSECURE=1`), never with "secure only when `ENV` equals `production`": a deployment that forgets that variable would otherwise ship insecure cookies.
- With `Secure` on, `Path=/` and no `Domain`, Node, Go and Lambda prefix the cookie names with `__Host-` automatically (the refresh cookie then moves to `Path=/`). Python adds the prefix only when `cookie_prefix="__Host-"` is passed to `AuthConfig` and `CsrfMiddleware`.
- Keep access tokens short (15 minutes is the default) and refresh tokens rotating. A session store adds device lists and revocation; with Node `session.checkOn: 'allcalls'` (or the equivalent) a revoked session stops working immediately instead of at the next refresh.
- Secrets: 32 random bytes or more each (`openssl rand -hex 32`), from the environment or a secret manager; different access and refresh secrets in Node.
- Bearer tokens on devices: Keychain, Android Keystore or `flutter_secure_storage`, `expo-secure-store` on React Native. Never `localStorage` in a browser; browsers use cookie mode.

## 6. Abuse controls

- Rate-limit `/login`, `/register`, `/forgot-password`, `/magic-link/send`, `/sms/send` and the code-verification routes, but not the session upkeep the clients do on every page load (`GET /me`) and every 15 minutes (`POST /refresh`).
- Node: `router({ rateLimiter })` applies one middleware to **every** auth route, `/me` and `/refresh` included; a plain `rateLimit({ windowMs: 15 * 60 * 1000, limit: 20 })` there locks a normal user out (429) after about 20 page loads. Skip the upkeep routes (verified: 25 `GET /me` in a row all answer 200, and the 21st credential request in the window, a wrong password, answers 429):

  ```ts
  rateLimiter: rateLimit({
    windowMs: 15 * 60 * 1000,
    limit: 20,
    skip: (req) => ['/me', '/refresh', '/logout', '/sessions'].includes(req.path),
  }),
  ```

- Behind a reverse proxy or load balancer the limiter sees the proxy's address and all users share one budget. Node: `app.set('trust proxy', 1)` when exactly one proxy is in front; never when the app is reachable directly, because clients could then forge `X-Forwarded-For`.
- Go: `HTTPConfig.RateLimiter` is the same single slot over every auth route (nil by default, no algorithm ships); create the counter outside the function and let `/me` and `/refresh` through. Python, Dart: the framework's limiter or the proxy. Lambda limits by default (10 per minute per account, credential flows only).
- Password reset and magic-link answers do not reveal whether an email exists; keep it that way in custom handlers.

## 7. Admin, IdP and tools surfaces

- Node admin panel: always set `accessPolicy`: `'is-admin-flag'`, `'first-user'`, a function `(user, rbacStore) => boolean` for role or permission checks, or `'open'` (every request allowed: local experiments only). The `rbac:<role>` and `permission:<perm>` strings exist only in the Lambda stack. Without `accessPolicy` the routes are mounted unprotected with only a warning. Its sign-in has no second factor: set `loginPath: '/auth/ui/login'` and block `POST /auth/admin/login` at the proxy when operators must use 2FA.
- Python `build_admin_router`: keep `access_policy="is-admin-flag"`. Dart: `enableAdminUi` and `enableIdpMode` default to true; turn off what is unused.
- Swagger/OpenAPI pages are on outside production in Node and load a CDN script on the auth origin: keep them off in production by deploying with `NODE_ENV=production` (which also enforces the OAuth origin allow-list) or by passing `swagger: false`.

## 8. Before calling it done

Run the SKILL.md request sequence against the running app and confirm: CSRF refusal on a cookie-authenticated write, refresh refused after logout, register behaving as decided in section 1, and cookies `Secure` in the deployed environment.
