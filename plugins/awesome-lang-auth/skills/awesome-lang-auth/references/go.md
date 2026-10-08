# Go: `awesome-go-auth`

Checked against `github.com/nik2208/awesome-go-auth` v0.12.0 (Go module proxy, tagged 2026-09-30) with Go 1.25, on 2026-10-08. The net/http snippet below was built as published with `go vet` and run through the SKILL.md request sequence on that date.

Maturity: **beta**. 0.x releases; the API may still change between minors. Tell the user.

## Install

```bash
go get github.com/nik2208/awesome-go-auth@v0.12.0
```

The repository lives at https://github.com/awesome-lang-auth/awesome-go-auth, but the **module path is still `github.com/nik2208/awesome-go-auth`** and moves to `github.com/awesome-lang-auth/awesome-go-auth` at v1.0.0. Until then, `go get github.com/awesome-lang-auth/awesome-go-auth` fails with "module declares its path as: github.com/nik2208/awesome-go-auth". Import the old path. `go.mod` requires Go 1.25.

## net/http (Go 1.22+ patterns)

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"os"
	"time"

	auth "github.com/nik2208/awesome-go-auth"
	"github.com/nik2208/awesome-go-auth/adapter/nethttp"
)

func main() {
	a, err := auth.New(
		auth.WithSecret(os.Getenv("AUTH_SECRET")), // at least 32 characters
		auth.WithTokenTTLs(15*time.Minute, 7*24*time.Hour),
		auth.WithUserStore(auth.NewMemoryUserStore()),       // replace with your store
		auth.WithSessionStore(auth.NewMemorySessionStore()), // replace with your store
	)
	if err != nil {
		log.Fatal(err)
	}

	cfg := auth.DefaultHTTPConfig()                          // prefix /auth, Secure cookies, CSRF on
	cfg.UI.Enabled = true                                    // <prefix>/ui/login and <prefix>/ui/auth.js
	cfg.Cookies.Secure = os.Getenv("COOKIE_INSECURE") != "1" // COOKIE_INSECURE=1 only for local http

	mux := http.NewServeMux()
	nethttp.MountWithConfig(mux, a, cfg)

	// Your own API: the adapter middleware authenticates; the CSRF check of the
	// mount covers <prefix> only, so run it for cookie-authenticated writes under /api too.
	apiCSRF := cfg
	apiCSRF.APIPrefix = "/api"
	protect := func(h http.HandlerFunc) http.Handler {
		return auth.CSRFMiddleware(apiCSRF)(nethttp.Middleware(a)(h))
	}
	mux.Handle("POST /api/notes", protect(func(w http.ResponseWriter, r *http.Request) {
		user, _ := auth.UserFromContext(r.Context())
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusCreated)
		_ = json.NewEncoder(w).Encode(map[string]any{"ok": true, "owner": user.ID})
	}))

	log.Fatal(http.ListenAndServe(":3000", mux))
}
```

What the run returned (with `COOKIE_INSECURE=1`): the same cookie names as Node (`accessToken`, `refreshToken` on `Path=/auth/refresh`, `csrf-token`); `POST /api/notes` without `X-CSRF-Token` is `403 {"error":"CSRF token validation failed","code":"CSRF_INVALID"}`; a Bearer request skips CSRF; `/auth/refresh` after logout is `401 {"code":"SESSION_REVOKED"}`; `/auth/ui/auth.js` is served (`/auth/ui/assets/auth.js` is a 404).

## chi, gin, echo

Each adapter has the same three functions; pick the package that matches the router:

| Router | Import | Mount | Protect |
|---|---|---|---|
| net/http | `github.com/nik2208/awesome-go-auth/adapter/nethttp` | `nethttp.MountWithConfig(mux, a, cfg)` | `nethttp.Middleware(a)` |
| chi | `.../adapter/chi` | `chiAdapter.MountWithConfig(r, a, cfg)` | `r.With(chiAdapter.Middleware(a))` |
| gin | `.../adapter/gin` | `ginAdapter.MountWithConfig(r, a, cfg)` (engine or group) | `ginAdapter.Middleware(a)`, user via `ginAdapter.UserFromContext(c)` |
| echo | `.../adapter/echo` | `echoAdapter.MountWithConfig(g, a, cfg)` (an `*echo.Group`) | `echoAdapter.Middleware(a)`, user via `echoAdapter.UserFromContext(c)` |

`Mount(r, a)` takes the defaults instead of a config. The repository's `examples/chi-postgres`, `examples/gin-mongodb` and `examples/echo-sqlite` are compiled by CI and show the full wiring (event bus, SSE, webhooks, OpenAPI, the OIDC endpoints); their database stores are comment skeletons over the in-memory stores, not working database code.

## Configuration

- `auth.New(...)` options: `WithSecret`, `WithIssuer`, `WithTokenTTLs`, `WithBcryptCost`, `WithUserStore`, `WithSessionStore`, `WithMetadataProvider`, `WithRBACProvider`, `WithTenantProvider`, delivery hooks (`WithMagicLinkSender`, `WithSMSCodeSender`, `WithPasswordResetSender`, `WithEmailVerificationSender`, `WithEmailChangeSender`), `WithLogger`.
- Without the senders, `POST /auth/magic-link/send` answers `500 EMAIL_NOT_CONFIGURED` and `POST /auth/sms/send` `500 SMS_NOT_CONFIGURED`; reset, verification and email-change routes answer 200 and send nothing. Built-in mailers: `auth.NewGatewayMailerTransport(auth.MailerConfig{...})` with `auth.NewMagicLinkMailer`, `auth.NewPasswordResetMailer`, `auth.NewEmailVerificationMailer`, `auth.NewEmailChangeMailer`.
- `auth.DefaultHTTPConfig()` returns `HTTPConfig{APIPrefix: "/auth", Cookies: Secure true, SameSite Lax, CSRF: Enabled true}`. Other fields: `UI` (`Enabled`, `Branding`), `Docs.Enabled` (`/auth/openapi.json`, `/auth/docs`; leave off in production), `Admin`, `Tools`, `ResourceServer`, and `RateLimiter func(http.Handler) http.Handler`.
- Rate limiting: `cfg.RateLimiter` is the slot for the limiter (nil by default; no algorithm ships). Each adapter wraps every auth route with it, outermost, ahead of CSRF and the auth middleware, and calls the function once per route at mount time: create the limiter's counter outside the function so all routes share one budget, and let `GET <prefix>/me` and `POST <prefix>/refresh` through (or give them a loose budget), since the clients call them on every page load.
- Custom claims: `Config.BuildTokenClaims`, or `StaticClaims`, `UserFieldClaims`, `ChainClaims`, `ClaimsWebhook`. `sid`, `tid`, `jti`, `typ`, `iss`, `iat`, `exp` are reserved.

## Differences from the Node reference that change your code

- `POST <prefix>/register` is always mounted, and it opens a session. Block the route in front of the mux if sign-up is closed.
- No CORS middleware: add your router's (`github.com/go-chi/cors`, `github.com/gin-contrib/cors`, echo's `middleware.CORSWithConfig`) with exact origins, `AllowCredentials: true` and the headers `Content-Type`, `Authorization`, `X-CSRF-Token`, `X-Auth-Strategy`.
- `2fa/setup` returns no QR image, only the secret and the `otpauth://` URL; render the QR on the client.
- The CSRF check exempts only requests that present `Authorization: Bearer`; `X-Auth-Strategy: bearer` alone does not switch it off.
- The README section "Deliberate deviations from the reference" lists every other difference, generated from `CompatibilityNotes()`.

## Stores

`UserStore` is the only required interface; each optional interface switches on a feature. See `references/stores.md`.
