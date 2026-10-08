# Node.js: `@awesome-lang-auth/node`

Checked against `@awesome-lang-auth/node` 1.10.8 (npm, released 2026-09-29) with Express 5.2.1 and express-rate-limit 8, on 2026-10-08. The Express snippet below was run as published (in-memory stores in place of the Postgres ones) through the SKILL.md request sequence on that date, and type-checks with `tsc --strict`. The Next.js route handler was built with Next.js 16.4 under strict TypeScript and run through the same sequence, minus the steps that need an app route.

## Contents

- Install
- Express (the headline snippet), the admin router, NestJS, Next.js
- `AuthConfig` and `RouterOptions`
- Routes, login with a second factor, other entry points

## Install

```bash
npm i @awesome-lang-auth/node express
npm i express-rate-limit        # for the rateLimiter option
```

Express is needed at runtime but is not a dependency of the package, so install it even for NestJS (whose default platform is Express) and Next.js. Node.js 18 or later. TypeScript types ship with the package.

Formerly `awesome-node-auth` (still on npm as a deprecated 1.10.0 alias). To migrate: `npm uninstall awesome-node-auth && npm i @awesome-lang-auth/node`, then change every `'awesome-node-auth'` import to `'@awesome-lang-auth/node'`. The API is unchanged.

## Express

```ts
import express from 'express';
import rateLimit from 'express-rate-limit';
import { AuthConfigurator } from '@awesome-lang-auth/node';
import { PgUserStore } from './stores/user-store';        // your IUserStore, see references/stores.md
import { PgSessionStore } from './stores/session-store';  // your ISessionStore (optional)

// Fail at startup, not at the first login, when a secret is missing or short.
function requireSecret(name: string): string {
  const value = process.env[name];
  if (!value || value.length < 32) throw new Error(`${name} must be set, 32+ characters (openssl rand -hex 32)`);
  return value;
}

const app = express();
app.use(express.json());

const auth = new AuthConfigurator(
  {
    accessTokenSecret: requireSecret('ACCESS_TOKEN_SECRET'),   // two different random secrets
    refreshTokenSecret: requireSecret('REFRESH_TOKEN_SECRET'),
    accessTokenExpiresIn: '15m',
    refreshTokenExpiresIn: '7d',
    // Secure unless COOKIE_INSECURE=1, which only local http development sets.
    cookieOptions: { secure: process.env.COOKIE_INSECURE !== '1', sameSite: 'lax' },
    csrf: { enabled: true },                               // off by default
    ui: { enabled: true },                                 // /auth/ui/login and /auth/ui/auth.js
    email: { siteUrl: process.env.APP_ORIGIN },            // origin for emailed links and OAuth redirects
  },
  new PgUserStore(),
);

app.use('/auth', auth.router({
  defaultRegister: true,       // only if sign-up is open (SKILL.md Step 1); omit to leave /register unmounted
  sessionStore: new PgSessionStore(),                      // device list + revocation; omit for stateless
  // One limiter covers every auth route, so skip the session upkeep the clients
  // call on each page load and every 15 minutes; the credential routes stay limited.
  rateLimiter: rateLimit({
    windowMs: 15 * 60 * 1000,
    limit: 20,
    skip: (req) => ['/me', '/refresh', '/logout', '/sessions'].includes(req.path),
  }),
}));

// Your routes: auth.middleware() checks the access token (cookie or Bearer)
// and, with csrf enabled, the X-CSRF-Token header on cookie-authenticated writes.
app.get('/api/profile', auth.middleware(), (req, res) => res.json({ user: req.user }));
app.post('/api/notes', auth.middleware(), (req, res) => res.status(201).json({ owner: req.user!.sub }));

app.listen(3000);
```

The `rateLimiter` is applied to every route of the auth router, `GET /me` included, so an unfiltered `limit: 20` locks a normal user out after a few page loads; keep the `skip`. Behind a reverse proxy, every request comes from the proxy's address and all users share one budget: add `app.set('trust proxy', 1)` when exactly one proxy is in front, and never when the app is reachable directly (clients could then forge `X-Forwarded-For`).

Deploy with `NODE_ENV=production`: the library reads it to switch off the Swagger pages (which load a CDN script on the auth origin) and to enforce the OAuth origin allow-list.

What the run returned (with `COOKIE_INSECURE=1`, local http): login sets `accessToken` (HttpOnly, `Path=/`), `refreshToken` (HttpOnly, `Path=/auth/refresh`) and `csrf-token` (readable); `GET /auth/me` returns `{"sub","email","role","loginProvider","isEmailVerified","isTotpEnabled"}`; `POST /api/notes` without `X-CSRF-Token` returns `403 {"error":"CSRF token validation failed","code":"CSRF_INVALID"}`; with the header, 201; a Bearer request needs no CSRF header; 25 calls to `GET /auth/me` in a row all answer 200; `/auth/ui/auth.js` is served, `/auth/ui/assets/auth.js` is a 404. `/auth/login`, `/auth/refresh` and `/auth/logout` do not check CSRF. An invalid or expired access token is answered with 403 (the clients refresh on 401 and 403). With `Secure` on, the cookies are named `__Host-accessToken`, `__Host-refreshToken` (then `Path=/`) and `__Host-csrf-token`.

`req.user` is the token payload: the user id is `req.user.sub`.

### One call for the auth and admin routers

```ts
app.use(auth.buildAllRouters({
  auth: { defaultRegister: true, sessionStore },   // defaultRegister only if sign-up is open
  admin: { accessPolicy: 'is-admin-flag' },        // admin panel at /auth/admin/ for users with isAdmin: true
}));
// Grant admin from a seed script (needs IUserStore.update):
// await auth.promoteToAdmin(userId, { method: 'flag' });
```

Without `accessPolicy` (or a non-empty legacy `adminSecret`) the admin routes are mounted unprotected with a warning on stderr: always set one. Its values are `'is-admin-flag'`, `'first-user'`, `'open'` (every request allowed: local experiments only) or a function `(user, rbacStore) => boolean`. The admin sign-in at `POST /auth/admin/login` checks the password only; set `admin.loginPath: '/auth/ui/login'` to send operators through the application login and its 2FA, and block `POST /auth/admin/login` at the proxy if 2FA must be enforced.

### NestJS

Create the configurator once (a provider or a module-level constant) and mount its router in `main.ts`; Nest's Express platform parses JSON bodies by default:

```ts
const app = await NestFactory.create(AppModule);
app.use('/auth', auth.router({ defaultRegister: true }));   // defaultRegister only if sign-up is open
await app.listen(3000);
```

Guard Nest routes with `auth.middleware()` applied through `MiddlewareConsumer`, or call it from a guard.

### Next.js (pages router)

The router needs an Express request and response (it parses nothing itself and sets cookies with `res.cookie`), so wrap it in a small Express app inside the API route:

```ts
// lib/auth.ts: one configurator for the process
import { AuthConfigurator } from '@awesome-lang-auth/node';
export const auth = new AuthConfigurator({ /* ...as above... */ apiPrefix: '/api/auth' }, userStore);
```

```ts
// pages/api/auth/[...auth].ts
import express from 'express';
import type { NextApiRequest, NextApiResponse } from 'next';
import { auth } from '../../../lib/auth';

const app = express();
app.use(express.json());
app.use('/api/auth', auth.router({ defaultRegister: true }));   // the full public path; defaultRegister only if sign-up is open

export const config = { api: { bodyParser: false, externalResolver: true } };

export default function handler(req: NextApiRequest, res: NextApiResponse) {
  return app(req as any, res as any);
}
```

Calling `auth.router()` directly with Next's `req` and `res` (the pattern in the repository's `demo/nextjs-fullstack` and `README.detailed.md` at v1.10.8) answers 500 on every request: `TypeError: res.cookie is not a function`. Set `apiPrefix` to the public mount (`/api/auth` here): it decides the refresh-cookie path and the links in emails. For the App Router, run the auth server as a separate Express process behind a rewrite of `/api/auth/*`.

## AuthConfig (first constructor argument), the options that matter most

| Option | Default | Notes |
|---|---|---|
| `accessTokenSecret`, `refreshTokenSecret` | required | must differ when a `sessionStore` is used (startup error otherwise) |
| `accessTokenExpiresIn`, `refreshTokenExpiresIn` | `'15m'`, `'7d'` | |
| `apiPrefix` | `'/auth'` | public mount path; refresh cookie path and email links follow it |
| `cookieOptions` | `secure: false` | `secure`, `sameSite` (`'strict' \| 'lax' \| 'none'`), `domain`, `path`, `refreshTokenPath` |
| `csrf.enabled` | `false` | turn on for cookie mode |
| `email` | | `siteUrl` (string or array: allowed front-end origins, first is the default), `mailer` or the `send*` callbacks |
| `emailVerificationMode` | `'none'` | `'lazy'` or `'strict'` |
| `twoFactor.appName` | | issuer label shown in authenticator apps |
| `session.checkOn` | `'refresh'` | `'allcalls'` checks the session store on every request; `singleSessionPerUser` |
| `ui` | | `enabled`, `headless` (serve only `auth.js` and config, no pages), `siteName`, colors, `loginUrl` |
| `buildTokenPayload` | | add custom claims; a `purpose` claim is dropped |
| `oauth` | | `google`, `github` client settings, `allowedReturnPaths` |
| `idProvider`, `resourceServer` | | RS256 IdP with `/.well-known/jwks.json`; downstream verification |
| `templateStore` | | `ITemplateStore` for mail templates and UI translations |

The optional third argument takes `{ eventBus, apiPrefix, onBeforeDeleteUser }`.

## RouterOptions (argument of `auth.router()`)

`defaultRegister`, `onRegister`, `rateLimiter` (one Express middleware, applied to every auth route), `sessionStore`, `rbacStore`, `tenantStore`, `metadataStore`, `linkedAccountsStore`, `pendingLinkStore`, `settingsStore`, `cors: { origins: string[] }` (answers preflight and adds CORS headers for those origins only), `apiPrefix`, `swagger` (on outside production by default: `/auth/openapi.json`, `/auth/docs`), `eventBus`, `allowedReturnPaths`, `googleStrategy`, `githubStrategy`, `oauthStrategies`, `onBeforeDeleteUser`.

## Routes (relative to the prefix)

`POST /login`, `POST /logout`, `POST /refresh`, `GET /me`, `PATCH /profile`, `POST /register` (when enabled), `GET /sessions`, `DELETE /sessions/:handle`, `POST /sessions/cleanup`, `POST /forgot-password`, `POST /reset-password`, `POST /change-password`, `POST /send-verification-email`, `GET /verify-email`, `POST /change-email/request`, `POST /change-email/confirm`, `POST /2fa/setup`, `POST /2fa/verify-setup`, `POST /2fa/verify`, `POST /2fa/disable`, `POST /magic-link/send`, `POST /magic-link/verify`, `POST /sms/send`, `POST /sms/verify`, `POST /add-phone`, `GET /oauth/:provider`, `GET /oauth/:provider/callback`, `GET /linked-accounts`, `POST /link-request`, `POST /link-verify`, `DELETE /account`, plus `/ui/*` with the UI enabled and `GET /.well-known/jwks.json` in IdP mode.

## Login with a second factor

1. `POST /login {email, password}` answers `{"requiresTwoFactor": true, "tempToken", "available2faMethods"}` when the account has 2FA, or `403 {"requires2FASetup": true, "tempToken", "code": "2FA_SETUP_REQUIRED"}` when it must enrol first.
2. Complete with `POST /2fa/verify {tempToken, totpCode}` (TOTP), or `POST /sms/send {mode: '2fa', tempToken}` then `POST /sms/verify {mode: '2fa', tempToken, code}`, or `POST /magic-link/send {mode: '2fa', tempToken}`.
3. Enrolment for a signed-in user: `POST /2fa/setup` returns the secret and `otpauthUrl`; confirm with `POST /2fa/verify-setup {token, secret}`.

The `tempToken` lives 5 minutes, carries `purpose: '2fa'` and is refused everywhere else (1.10.0 and later).

## Other entry points

- `auth.middleware()` for your own routes.
- Resource servers that verify tokens from an IdP: `createJwksAuthMiddleware(config)` takes a whole `AuthConfig` and throws at startup unless `resourceServer.enabled` is true:

  ```ts
  app.use('/api', createJwksAuthMiddleware({
    accessTokenSecret, refreshTokenSecret,
    resourceServer: { enabled: true, jwksUrl: 'https://auth.example.com/auth/.well-known/jwks.json', issuer: 'https://auth.example.com' },
  }));
  ```

- `TokenService` (`new TokenService().clearTokenCookies(res, config)`), `AuthEventBus`, `PasswordService`, `MemoryTemplateStore`, `createAdminRouter`.
- Fastify and other frameworks: see `examples/fastify-integration.example.ts` in the repository at the v1.10.8 tag.

Repository: https://github.com/awesome-lang-auth/awesome-node-auth (`README.detailed.md` is the full reference, but its Next.js, `createJwksAuthMiddleware` and `rateLimiter` examples have the defects described above; `demo/` has Express, NestJS, Next.js and Angular apps).
