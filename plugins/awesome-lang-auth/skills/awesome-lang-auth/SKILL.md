---
name: awesome-lang-auth
description: Adds authentication to an app with the awesome-lang-auth libraries - email/password login and sign-up, cookie or bearer sessions with refresh tokens, CSRF, TOTP 2FA, magic links, SMS codes, OAuth, API keys, roles. Use when adding, wiring or fixing login, registration, sessions, 2FA, OAuth or API-key auth in a Node.js (Express, NestJS, Next.js), Go, Python (FastAPI), Rust, Dart (Shelf, Dart Frog) or AWS Lambda backend, or when connecting an Angular, React, Flutter or plain HTML/JS (Vue, Svelte) frontend to such a server. Also use when a project already depends on @awesome-lang-auth/*, awesome-node-auth, awesome-go-auth, awesome-python-auth, awesome_flutter_auth or ng-awesome-node-auth.
license: MIT
compatibility: Needs the project's package manager (npm, go, pip, cargo, dart or flutter) with access to its registry or to GitHub. The AWS Lambda stack also needs Docker and the AWS CLI v2.
metadata:
  version: "0.1.0"
  checked: "2026-10-08"
  homepage: "https://awesomelangauth.com"
  node: "@awesome-lang-auth/node 1.10.8"
  go: "github.com/nik2208/awesome-go-auth v0.12.0"
  python: "awesome-python-auth 1.1.0"
  rust: "awesome-rust-auth git main@acb3d53 (1.9.0, not on crates.io)"
  dart: "awesome_dart_auth git main@d71bd21 (1.9.0, not on pub.dev)"
  lambda: "awesome-lambda-auth git main@4330384"
  angular: "@awesome-lang-auth/angular 1.10.1"
  react: "@awesome-lang-auth/react 0.1.0"
  flutter: "awesome_flutter_auth 1.10.5"
---

# awesome-lang-auth

awesome-lang-auth is a family of authentication libraries built around one HTTP wire protocol. The Node.js library is the reference. The Go and Python ports and the AWS Lambda stack serve the same routes, cookies and JSON, so the same clients (Angular, React, Flutter, and the `auth.js` script the servers serve) work against any of them by changing only the API prefix. The Rust and Dart ports are earlier and have gaps (Step 5).

Use the library's routes. Do not write password hashing, JWT signing, refresh rotation or CSRF logic next to it, and do not rename its routes or JSON fields: the clients depend on them.

## Step 1: detect the stack

Read the manifests before choosing anything:

| File | Look for | Server choice |
|---|---|---|
| `package.json` | `express`, `@nestjs/core`, `next`, `fastify` | Node.js |
| `go.mod` | `net/http`, `go-chi/chi`, `gin-gonic/gin`, `labstack/echo` | Go |
| `pyproject.toml`, `requirements.txt` | `fastapi` | Python |
| `Cargo.toml` | `axum`, `actix-web`, `warp` | Rust |
| `pubspec.yaml` | `shelf`, `dart_frog` (server); `flutter` (client) | Dart / Flutter client |
| `template.yaml`, CDK, Terraform, "Cognito" in the ask | AWS Lambda, API Gateway | AWS Lambda stack |
| `package.json` | `@angular/core`, `react`, `react-native`, `vue`, `svelte` | client side |

If the project already has an auth system (Passport, NextAuth/Auth.js, Lucia, Supabase, Cognito, a hand-written one), ask before replacing it. If the backend is in a language with no port (Java, .NET, PHP, Ruby), say so and offer a separate Node.js or Go auth service behind the same origin; do not invent a port.

## Step 2: pick the server library

| Runtime | Package | Install (versions checked 2026-10-08) | Maturity | Read |
|---|---|---|---|---|
| Node.js | `@awesome-lang-auth/node` 1.10.8 | `npm i @awesome-lang-auth/node express` | stable | `references/node.md` |
| Go | `github.com/nik2208/awesome-go-auth` v0.12.0 | `go get github.com/nik2208/awesome-go-auth@v0.12.0` | beta | `references/go.md` |
| Python (FastAPI) | `awesome-python-auth` 1.1.0 | `pip install awesome-python-auth` | stable | `references/python.md` |
| Rust | `awesome-rust-auth` (git) | `cargo add awesome-rust-auth --git https://github.com/awesome-lang-auth/awesome-rust-auth` | preview | `references/rust.md` |
| Dart (Shelf, Dart Frog) | `awesome_dart_auth` (git) | `dart pub add awesome_dart_auth --git-url https://github.com/awesome-lang-auth/awesome-dart-auth --git-path packages/awesome_dart_auth` | preview | `references/dart.md` |
| AWS Lambda | `awesome-lambda-auth` (a deployable stack) | `git clone https://github.com/awesome-lang-auth/awesome-lambda-auth` | preview | `references/lambda.md` |

Maturity, stated plainly to the user when it matters: **beta** means 0.x on its registry and the API may still change; **preview** means installed from git with no registry release; the React client is **early** (first 0.x). The Go module path is `github.com/nik2208/awesome-go-auth` until v1.0.0, when it moves to `github.com/awesome-lang-auth/awesome-go-auth`; `go get` of the new path fails before then.

Renamed packages, same API: `awesome-node-auth` is now `@awesome-lang-auth/node`, `ng-awesome-node-auth` is now `@awesome-lang-auth/angular`, `awesome_node_auth_flutter` is now `awesome_flutter_auth`. Migrate a project on an old name by swapping the dependency and the import specifier.

## Step 3: configure secrets and transport

- **Secrets** come from the environment or a secret manager, never from source. Generate each with `openssl rand -hex 32`. Node needs two different secrets (access and refresh; they must differ when a session store is used). Go refuses a secret shorter than 32 characters. Python and Dart take one signing secret.
- **Cookie mode** (default; for browser apps): HttpOnly access and refresh cookies plus a JS-readable `csrf-token` cookie whose value the client echoes in `X-CSRF-Token` on POST, PUT, PATCH and DELETE.
- **Bearer mode** (native and mobile apps): the client sends `X-Auth-Strategy: bearer`; login and refresh return `accessToken` and `refreshToken` in the JSON body; requests carry `Authorization: Bearer <accessToken>`. One server serves both modes, per request.
- **Same origin is the easy path.** Serve the frontend and the API from one origin, or proxy the auth prefix through the frontend's dev server and production proxy. A cross-site setup needs `SameSite=None; Secure` cookies, a CORS allow-list with credentials, and still fails CSRF when the page cannot read the API host's cookie; see `references/security.md`.
- **Secure cookies in production.** Every port marks cookies `Secure` by default or by switch; set it off only for local `http://`. With `Secure` on, cookie names get the `__Host-` (or `__Secure-`) prefix automatically; clients read all three spellings.
- **The API prefix** is `/auth` by default on Node, Go, Rust, Dart and Lambda, and `/api/auth` on Python. Whatever you choose, the client's `apiPrefix` must match it.

## Step 4: implement storage

The libraries store nothing themselves (Lambda excepted: it brings DynamoDB). Read `references/stores.md` for the user-store and session-store interfaces of each port, the field names to map columns to, and the database examples that exist (MongoDB, PostgreSQL, MySQL, SQLite, PostgREST, PHP-CRUD-API and in-memory, all for Node). Use the in-memory stores only in tests and prototypes.

## Step 5: mount the routes and the built-in UI

Follow the port's reference file. The servers also serve a built-in UI (login, register, forgot-password, 2FA pages) at `<apiPrefix>/ui/login` and the browser client at **`<apiPrefix>/ui/auth.js`** when the UI is enabled (Node: `ui: { enabled: true }`; Go: `cfg.UI.Enabled = true`; Python: mount `build_ui_router`; Lambda: `ui.enabled`; Dart: on by default; Rust: serve the crate's `ui` assets yourself). Protect your own routes with the port's middleware or dependency, never by decoding the JWT yourself.

Two ports need a warning before you build on them: **Rust** provides the auth core but not the HTTP routes (you write the handlers), and **Dart**, at the checked commit, fails to compile with the current Dart SDK and issues no auth cookies (bearer clients only, and its `/me` shape differs). Their reference files explain; for a Dart or Flutter project that needs a backend now, recommend Node.js or Go.

## Step 6: wire the frontend

Read `references/clients.md`. Choose:

- Angular: `@awesome-lang-auth/angular` (`provideAuth`, `authGuard`, `guestGuard`, `AuthService` signals).
- React, Next.js, React Native: `@awesome-lang-auth/react` (`AwesomeAuthProvider`, `useAwesomeAuth`, `ProtectedRoute`).
- Flutter: `awesome_flutter_auth` (`AuthClient`; cookies on web, bearer on native).
- Anything else (plain HTML, Vue, Svelte, Alpine, server-rendered pages): one tag, `<script src="/auth/ui/auth.js"></script>`, then `window.AwesomeNodeAuth`. It adds CSRF headers and credentials, and refreshes on 401/403.
- No frontend work at all: link to `<apiPrefix>/ui/login`.

## Step 7: verify with real requests

Run the server, then this sequence (cookie mode; replace `B`, `P` and the protected route `/api/notes` with the app's own):

```sh
B=http://localhost:3000; P=/auth; J=$(mktemp)
CRED='{"email":"alice@example.com","password":"correct-horse-battery-staple"}'
csrf() { awk '$6 ~ /csrf-token$/ {print $7}' "$J" | tail -1; }
curl -s -w ' %{http_code}\n' -H 'Content-Type: application/json' -d "$CRED" "$B$P/register"            # 201
curl -s -w ' %{http_code}\n' -c "$J" -b "$J" -H 'Content-Type: application/json' -d "$CRED" "$B$P/login"   # 200, Set-Cookie x3
curl -s -w ' %{http_code}\n' -b "$J" "$B$P/me"                                                           # 200, user id in "sub"
curl -s -w ' %{http_code}\n' -b "$J" -X POST "$B/api/notes"                                              # 403, CSRF refused
curl -s -w ' %{http_code}\n' -b "$J" -X POST -H "X-CSRF-Token: $(csrf)" "$B/api/notes"                   # 2xx
curl -s -w ' %{http_code}\n' -c "$J" -b "$J" -X POST -H "X-CSRF-Token: $(csrf)" "$B$P/refresh"          # 200, new cookies
curl -s -w ' %{http_code}\n' -c "$J" -b "$J" -X POST -H "X-CSRF-Token: $(csrf)" "$B$P/logout"           # 200
curl -s -w ' %{http_code}\n' -b "$J" -X POST "$B$P/refresh"                                              # 401 after logout
curl -s -o /dev/null -w '%{http_code} %{content_type}\n' "$B$P/ui/auth.js"                               # 200 text/javascript
curl -s -H 'Content-Type: application/json' -H 'X-Auth-Strategy: bearer' -d "$CRED" "$B$P/login"         # {"success":true,"accessToken":...,"refreshToken":...}
```

Notes: Node mounts `/register` only when configured (see Step 8). Check logout with `/refresh`, not `/me`: an access token stays valid until it expires (15 minutes by default) unless sessions are checked on every call. The Node, Go and Python snippets in the references were run through this sequence on 2026-10-08; their outputs and per-port differences (cookie names, which routes check CSRF) are in those files. Python bearer clients must also send `X-Auth-Strategy: bearer` on writes.

## Step 8: security checklist

Go through it before calling the work done; `references/security.md` has the per-port details.

1. **Sign-up is a decision.** Node mounts `POST <prefix>/register` only with `onRegister` or `defaultRegister: true` (1.10+; the built-in handler stores email, password hash, firstName and lastName, nothing else). Go, Python and Dart always mount it: if sign-up must be closed, block that route in front of the router.
2. **2FA step-up tokens are not sessions.** The `tempToken` returned when a login needs a second factor carries `purpose: '2fa'` and is accepted only by the 2FA completion routes (Node 1.10.0 and later; upgrade older versions). Never put a `purpose` claim in custom claims, and never accept a token carrying one as a session in your own middleware.
3. **CSRF on for cookie mode.** Node: `csrf: { enabled: true }` (off by default). Go: on by default, but only for routes under the auth prefix. Python: add `CsrfMiddleware` with a prefix that covers your own API too. Dart: `csrfMiddleware`.
4. **CORS is an allow-list.** Exact origins with credentials; never `*` with cookies. Allow the headers `Content-Type`, `Authorization`, `X-CSRF-Token`, `X-Auth-Strategy`.
5. **HTTPS and Secure cookies** in every deployed environment; `SameSite=Lax` unless the setup is cross-site.
6. **Rate-limit** login, register, forgot-password and the code-verification routes (Node: `rateLimiter` option; Lambda: built in).
7. **Admin and IdP surfaces** off unless needed, and guarded when on (Node admin: `accessPolicy`).
8. **OAuth**: an origin allow-list (`email.siteUrl` / `cors.origins`) and `allowedReturnPaths`.
9. **Bearer tokens** on devices go to secure storage (Keychain, Keystore, `flutter_secure_storage`), never to `localStorage`.

## References (read only the ones the task needs)

- `references/node.md`: the server is Node.js (Express, NestJS, Next.js, Fastify). Install, config options, router options, routes, the 2FA flow, admin panel.
- `references/go.md`: the server is Go (net/http, chi, gin, echo).
- `references/python.md`: the server is FastAPI.
- `references/rust.md`: the server is Rust. Read before promising anything: the HTTP routes are yours to write.
- `references/dart.md`: the server is Dart (Shelf or Dart Frog).
- `references/lambda.md`: the user wants a self-hosted Cognito alternative on AWS, or the backend is serverless on AWS.
- `references/stores.md`: always, once the server is chosen, to connect the user's database.
- `references/clients.md`: any frontend work (Angular, React, Flutter, plain JS, the built-in UI).
- `references/security.md`: before shipping, for cross-origin setups, OAuth, admin, or when a check above is unclear.

These references are pinned to the versions in this file's metadata. If the registry shows a newer version, prefer that version's README and CHANGELOG over this skill where they disagree. For a deeper, broader source, the project publishes its whole documentation as one text file at https://awesomelangauth.com/llms-full.txt (optional; fetch it only when a reference here does not cover the question).
