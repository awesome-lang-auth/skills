# Node.js: `@awesome-lang-auth/node`

Checked against `@awesome-lang-auth/node` 1.10.8 (npm, released 2026-09-29) with Express 5.2.1, on 2026-10-08. The Express snippet below was run through the SKILL.md request sequence on that date.

## Install

```bash
npm i @awesome-lang-auth/node express
npm i express-rate-limit        # optional, for the rateLimiter option
```

Express is needed at runtime but is not a dependency of the package, so install it even for NestJS (whose default platform is Express) and Next.js. Node.js 18 or later. TypeScript types ship with the package.

Formerly `awesome-node-auth` (still on npm as a deprecated 1.10.0 alias). To migrate: `npm uninstall awesome-node-auth && npm i @awesome-lang-auth/node`, then change every `'awesome-node-auth'` import to `'@awesome-lang-auth/node'`. The API is unchanged.

## Express

```ts
import express from 'express';
import rateLimit from 'express-rate-limit';
import { AuthConfigurator } from '@awesome-lang-auth/node';
import { PgUserStore } from './stores/user-store';        // your IUserStore, see stores.md
import { PgSessionStore } from './stores/session-store';  // your ISessionStore (optional)

const app = express();
app.use(express.json());

const auth = new AuthConfigurator(
  {
    accessTokenSecret: process.env.ACCESS_TOKEN_SECRET!,   // two different random secrets
    refreshTokenSecret: process.env.REFRESH_TOKEN_SECRET!,
    accessTokenExpiresIn: '15m',
    refreshTokenExpiresIn: '7d',
    cookieOptions: { secure: process.env.NODE_ENV === 'production', sameSite: 'lax' },
    csrf: { enabled: true },                               // off by default
    ui: { enabled: true },                                 // /auth/ui/login and /auth/ui/auth.js
    email: { siteUrl: process.env.APP_ORIGIN },            // origin for emailed links and OAuth redirects
  },
  new PgUserStore(),
);

app.use('/auth', auth.router({
  defaultRegister: true,                                   // or onRegister: async (data, config) => user
  sessionStore: new PgSessionStore(),                      // device list + revocation; omit for stateless
  rateLimiter: rateLimit({ windowMs: 15 * 60 * 1000, max: 20 }),
}));

// Your routes: auth.middleware() checks the access token (cookie or Bearer)
// and, with csrf enabled, the X-CSRF-Token header on cookie-authenticated writes.
app.get('/api/profile', auth.middleware(), (req, res) => res.json({ user: req.user }));
app.post('/api/notes', auth.middleware(), (req, res) => res.status(201).json({ owner: req.user!.sub }));

app.listen(3000);
```

What the smoke run returned (secure off, local http): login sets `accessToken` (HttpOnly, `Path=/`), `refreshToken` (HttpOnly, `Path=/auth/refresh`) and `csrf-token` (readable); `GET /auth/me` returns `{"sub","email","role","loginProvider","isEmailVerified","isTotpEnabled"}`; `POST /api/notes` without `X-CSRF-Token` returns `403 {"error":"CSRF token validation failed","code":"CSRF_INVALID"}`; with the header, 201; a Bearer request needs no CSRF header; `/auth/ui/auth.js` is served, `/auth/ui/assets/auth.js` is a 404. `/auth/login`, `/auth/refresh` and `/auth/logout` do not check CSRF. An invalid or expired access token is answered with 403 (the clients refresh on 401 and 403).

`req.user` is the token payload: the user id is `req.user.sub`.

### One call for the auth and admin routers

```ts
app.use(auth.buildAllRouters({
  auth: { defaultRegister: true, sessionStore },
  admin: { accessPolicy: 'is-admin-flag' },    // admin panel at /auth/admin/ for users with isAdmin: true
}));
// Grant admin from a seed script (needs IUserStore.update):
// await auth.promoteToAdmin(userId, { method: 'flag' });
```

Without `accessPolicy` (or a non-empty legacy `adminSecret`) the admin routes are mounted unprotected with a warning on stderr: always set one. The admin sign-in at `POST /auth/admin/login` checks the password only; set `admin.loginPath: '/auth/ui/login'` to send operators through the application login and its 2FA, and block `POST /auth/admin/login` at the proxy if 2FA must be enforced.

### NestJS

Create the configurator once (a provider or a module-level constant) and mount its router in `main.ts`:

```ts
const app = await NestFactory.create(AppModule);
app.use('/auth', auth.router({ defaultRegister: true }));
await app.listen(3000);
```

Guard Nest routes with `auth.middleware()` applied through `MiddlewareConsumer`, or call it from a guard.

### Next.js (pages router, as in the repository's `demo/nextjs-fullstack`)

```ts
// lib/auth.ts: one configurator for the process
export const auth = new AuthConfigurator({ /* ...as above... */ apiPrefix: '/api/auth' }, userStore);
export const authRouter = auth.router({ defaultRegister: true });

// pages/api/auth/[...auth].ts
import type { NextApiRequest, NextApiResponse } from 'next';
import { authRouter } from '../../../lib/auth';

export const config = { api: { bodyParser: false } };   // the router parses the body

export default function handler(req: NextApiRequest, res: NextApiResponse) {
  req.url = req.url!.replace(/^\/api\/auth/, '') || '/';  // the router sees paths from /
  authRouter(req as any, res as any, () => res.status(404).json({ error: 'Not found' }));
}
```

Set `apiPrefix` to the public mount (`/api/auth` here): it decides the refresh-cookie path and the links in emails. For the App Router, the same router can be bridged from a route handler, or the auth server can run as a separate Express process behind a rewrite.

## AuthConfig (first constructor argument), the options that matter most

| Option | Default | Notes |
|---|---|---|
| `accessTokenSecret`, `refreshTokenSecret` | required | must differ when a `sessionStore` is used (startup error otherwise) |
| `accessTokenExpiresIn`, `refreshTokenExpiresIn` | `'15m'`, `'7d'` | |
| `apiPrefix` | `'/auth'` | public mount path; refresh cookie path and email links follow it |
| `cookieOptions` | | `secure`, `sameSite` (`'strict' \| 'lax' \| 'none'`), `domain`, `path`, `refreshTokenPath` |
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

`defaultRegister`, `onRegister`, `rateLimiter`, `sessionStore`, `rbacStore`, `tenantStore`, `metadataStore`, `linkedAccountsStore`, `pendingLinkStore`, `settingsStore`, `cors: { origins: string[] }` (answers preflight and adds CORS headers for those origins only), `apiPrefix`, `swagger` (on outside production by default: `/auth/openapi.json`, `/auth/docs`), `eventBus`, `allowedReturnPaths`, `googleStrategy`, `githubStrategy`, `oauthStrategies`, `onBeforeDeleteUser`.

## Routes (relative to the prefix)

`POST /login`, `POST /logout`, `POST /refresh`, `GET /me`, `PATCH /profile`, `POST /register` (when enabled), `GET /sessions`, `DELETE /sessions/:handle`, `POST /sessions/cleanup`, `POST /forgot-password`, `POST /reset-password`, `POST /change-password`, `POST /send-verification-email`, `GET /verify-email`, `POST /change-email/request`, `POST /change-email/confirm`, `POST /2fa/setup`, `POST /2fa/verify-setup`, `POST /2fa/verify`, `POST /2fa/disable`, `POST /magic-link/send`, `POST /magic-link/verify`, `POST /sms/send`, `POST /sms/verify`, `POST /add-phone`, `GET /oauth/:provider`, `GET /oauth/:provider/callback`, `GET /linked-accounts`, `POST /link-request`, `POST /link-verify`, `DELETE /account`, plus `/ui/*` with the UI enabled and `GET /.well-known/jwks.json` in IdP mode.

## Login with a second factor

1. `POST /login {email, password}` answers `{"requiresTwoFactor": true, "tempToken", "available2faMethods"}` when the account has 2FA, or `403 {"requires2FASetup": true, "tempToken", "code": "2FA_SETUP_REQUIRED"}` when it must enrol first.
2. Complete with `POST /2fa/verify {tempToken, totpCode}` (TOTP), or `POST /sms/send {mode: '2fa', tempToken}` then `POST /sms/verify {mode: '2fa', tempToken, code}`, or `POST /magic-link/send {mode: '2fa', tempToken}`.
3. Enrolment for a signed-in user: `POST /2fa/setup` returns the secret and `otpauthUrl`; confirm with `POST /2fa/verify-setup {token, secret}`.

The `tempToken` lives 5 minutes, carries `purpose: '2fa'` and is refused everywhere else (1.10.0 and later).

## Other entry points

`auth.middleware()`, `createJwksAuthMiddleware({ jwksUrl, issuer })` for resource servers, `TokenService` (`new TokenService().clearTokenCookies(res, config)`), `AuthEventBus`, `PasswordService`, `MemoryTemplateStore`, `createAdminRouter`. Fastify and other frameworks: see `examples/fastify-integration.example.ts` in the repository at the v1.10.8 tag.

Repository: https://github.com/awesome-lang-auth/awesome-node-auth (README.detailed.md is the full reference; `demo/` has Express, NestJS, Next.js and Angular apps).
