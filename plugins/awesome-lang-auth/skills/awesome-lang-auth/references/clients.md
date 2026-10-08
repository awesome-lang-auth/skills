# Clients: Angular, React, Flutter, plain JS (`auth.js`), built-in UI

Checked on 2026-10-08 against `@awesome-lang-auth/angular` 1.10.1, `@awesome-lang-auth/react` 0.1.0, `awesome_flutter_auth` 1.10.5 (registries) and the `auth.js` served by `@awesome-lang-auth/node` 1.10.8.

## Contents

- Which one
- Angular: `@awesome-lang-auth/angular`
- React: `@awesome-lang-auth/react`
- Flutter: `awesome_flutter_auth`
- Plain JS, Vue, Svelte: the served `auth.js`
- Built-in UI

All clients speak the same protocol, so any of them works with any server of the family (with the caveats for Rust and Dart in their reference files). Set the client's `apiPrefix` to the server's mount: `/auth` by default, `/api/auth` for the Python default, or an absolute URL such as `https://api.example.com/auth` when the API is on another origin.

## Which one

| Frontend | Use |
|---|---|
| Angular 21 | `@awesome-lang-auth/angular` |
| React 18 or 19, Next.js, React Native | `@awesome-lang-auth/react` (early: 0.1.0) |
| Flutter (web, iOS, Android, desktop) | `awesome_flutter_auth` |
| Plain HTML, Vue, Svelte, Alpine, server-rendered pages | the served `auth.js` |
| No frontend code | the built-in UI pages |

## Angular: `@awesome-lang-auth/angular`

```bash
npm i @awesome-lang-auth/angular        # peer dependencies: @angular/core and @angular/common ^21.2.0
```

```ts
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideAuth } from '@awesome-lang-auth/angular';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideAuth({ apiPrefix: '/auth', loginUrl: '/login' }),
  ],
};

// app.routes.ts
import { Routes } from '@angular/router';
import { authGuard, guestGuard } from '@awesome-lang-auth/angular';
export const routes: Routes = [
  { path: 'login', canActivate: [guestGuard], loadComponent: () => import('./login.component').then(m => m.LoginComponent) },
  { path: 'dashboard', canActivate: [authGuard], loadComponent: () => import('./dashboard.component').then(m => m.DashboardComponent) },
];
```

`provideAuth()` already registers `provideHttpClient(withFetch(), withInterceptors([authInterceptor]))` and an app initializer that calls `checkSession()`: **do not call `provideHttpClient` again**. To keep your own HttpClient setup, pass `manageHttpClient: false` and add `withInterceptors([authInterceptor])` to it. The interceptor adds `X-CSRF-Token` (reading `__Host-csrf-token`, `__Secure-csrf-token`, then `csrf-token`) and, on a 401 or 403 (except from login, logout, refresh, register, password reset, `2fa/verify` and email verification; `/me` does refresh), queues the request behind one refresh and retries it; `SESSION_REVOKED` signs the user out. Without `loginUrl`, guards redirect to the server's built-in page `<apiPrefix>/ui/login`. Other options: `homeUrl`, `headless`, `initializeOnStartup`, `authService` (a subclass).

`AuthService` (inject it): signals `user()` and `isAuthenticated()`, `isInitialized()`; `login(email, password)` (Observable; `{ success, requires2fa?, requires2FASetup?, token?, availableMethods?, error? }`: the step-up token is **`token`** here, while React, Flutter and `auth.js` call it `tempToken`), `validate2fa(token, code)` (pass `r.token`), `register(email, password, firstName, lastName)`, `logout()` (returns void: do not `subscribe` to it), `checkSession()`, `forgotPassword`, `resetPassword(password, token)` (this order), `changePassword`, `setup2fa`, `verify2faSetup(code, secret)`, `sendMagicLink`, `verifyMagicLink`, `getActiveSessions`, `revokeSession`. `provideAuthUi()` optionally syncs the theme with the admin panel. Formerly `ng-awesome-node-auth`: swap the package and the import specifier.

For Angular SSR, the repository `awesome-lang-auth/awesome-angular-auth` and the `demo/angular-ssr` app in `awesome-node-auth` show the server setup.

## React: `@awesome-lang-auth/react`

```bash
npm i @awesome-lang-auth/react
```

```tsx
import { AwesomeAuthProvider, ProtectedRoute, AnonymousOnly, useAwesomeAuth } from '@awesome-lang-auth/react';

export function App() {
  return (
    <AwesomeAuthProvider options={{ apiPrefix: '/auth' }}>
      {location.pathname === '/login'
        ? <AnonymousOnly redirectTo="/"><LoginForm /></AnonymousOnly>
        : <ProtectedRoute fallback={<Spinner />} redirectTo="/login"><Dashboard /></ProtectedRoute>}
    </AwesomeAuthProvider>
  );
}

function LoginForm() {
  const { login, client } = useAwesomeAuth();
  async function onSubmit(email: string, password: string) {
    const r = await login(email, password);
    if (r.requires2fa) { /* ask for the code, then await client.validate2fa(r.tempToken!, code) */ }
    else if (!r.success) alert(r.error);
  }
  // ...
}
```

- `useAwesomeAuth()` gives `user`, `isAuthenticated`, `isLoading`, `error`, `login`, `logout`, `refresh`, `checkSession` and `client`; every other action (`validate2fa`, `register`, `forgotPassword`, ...) is a method of `client`. The user id is `user.sub`.
- `new AwesomeAuthClient({ apiPrefix })` is the framework-free client; pass it as `<AwesomeAuthProvider client={auth}>` to share it. `client.fetch(url)` attaches credentials and CSRF to backend-origin requests only and retries once after a refresh. Action methods resolve to `{ success, error?, code? }` and never reject on HTTP errors.
- Transport: cookies on the web, bearer on React Native (`apiPrefix` must be absolute there; pass a `TokenStorage`, e.g. over `expo-secure-store`, to persist tokens). Override with `mode: 'cookie' | 'bearer'`.
- Gates: `<ProtectedRoute role={['admin']} forbidden={...} onUnauthenticated={() => navigate('/login')}>`, `<AnonymousOnly onAuthenticated={...}>`. With React Router, pass `onUnauthenticated`.
- Next.js App Router: the entry is `'use client'`; server components import `hasRole` and types from `@awesome-lang-auth/react/server`. Gate server-side with your own middleware checking the session cookie.
- `resetPassword(token, password)` follows `auth.js` (Angular has the reverse order).
- Known limits (from its README): bearer mode in a browser against another origin needs `X-Auth-Strategy` allowed by CORS; cookie mode against another site fails CSRF because the page cannot read the API host's cookie.

## Flutter: `awesome_flutter_auth`

```bash
flutter pub add awesome_flutter_auth
```

```dart
import 'package:awesome_flutter_auth/awesome_flutter_auth.dart';

final auth = AuthClient(AuthOptions(
  apiPrefix: 'https://api.example.com/auth',   // absolute on iOS, Android and desktop
  headless: true,                              // no automatic redirects; listen to state and events
  tokenStorage: MySecureStorage(),             // native: persist bearer tokens (e.g. flutter_secure_storage)
));

auth.state.userStream.listen((user) { /* null when signed out */ });

final result = await auth.login(email, password);
if (result.requires2fa) {
  // result.tempToken and result.availableMethods ('totp', 'sms', 'magic-link')
}
```

On web and WASM it uses cookies and sends `X-CSRF-Token` on same-origin writes; on native it uses bearer tokens (in memory unless you pass a `TokenStorage`). `initializeOnStartup: true` (the default) starts `checkSession()` without awaiting it: wait for `AuthEventType.initialized` on `auth.events`, or set it to false and `await auth.checkSession()`. Formerly `awesome_node_auth_flutter`: swap the dependency and `package:` imports; `TotpSetupData.qrCode` is nullable (Go and Lambda do not send it; use `otpauthUrl`).

## Plain JS, Vue, Svelte: the served `auth.js`

Every server serves the browser client at **`<apiPrefix>/ui/auth.js`** (`/auth/ui/auth.js` by default) when its UI is enabled (Node: `ui: { enabled: true }`; add `headless: true` when the app has its own login pages, which serves `auth.js` and `/ui/config` but no HTML pages). The spelling `<apiPrefix>/ui/assets/auth.js` found in some docs is a 404 on Node and Go.

```html
<script src="/auth/ui/auth.js"></script>
<script>
  AwesomeNodeAuth.init({ apiPrefix: '/auth', loginUrl: '/login', homeUrl: '/dashboard', headless: true });

  async function signIn(email, password) {
    const r = await AwesomeNodeAuth.login(email, password);
    if (r.requires2fa) return askCode(r.tempToken);                 // then AwesomeNodeAuth.validate2fa(r.tempToken, code)
    if (!r.success) return showError(r.error);
    location.href = '/dashboard';
  }
  // Any later fetch() to the backend origin gets credentials and X-CSRF-Token, and a refresh on 401/403.
</script>
```

API on `window.AwesomeNodeAuth`: `init(options)`, `login(email, password)`, `validate2fa(tempToken, code)`, `register(email, password, firstName, lastName)`, `logout()`, `checkSession()`, `getUser()`, `isAuthenticated()`, `isInitialized()`, `guardPage(loginUrl?)`, `guardRole(role, loginUrl?)`, `forgotPassword(email)`, `resetPassword(token, password)`, `changePassword`, `sendMagicLink`, `verifyMagicLink`, `setup2fa`, `verify2faSetup(code, secret)`, `getActiveSessions`, `revokeSession`. `init` options: `apiPrefix`, `loginUrl`, `homeUrl`, `siteName`, `headless` (never redirects), `onSessionExpired`, `onLogout`, `onRefreshSuccess`, `onRefreshFail`, and overrides for any method.

In Vue or Svelte, load the script once in `index.html` and read `window.AwesomeNodeAuth` from components (for example keep `getUser()` in a store and refresh it after `login`/`logout`). Cross-origin: `<script src="https://api.example.com/auth/ui/auth.js" crossorigin="anonymous"></script>` with `init({ apiPrefix: 'https://api.example.com/auth', headless: true })`; see `references/security.md` for the cookie limits. Angular apps should use the Angular library instead: `auth.js` works by intercepting the global `fetch` and using `window`, which does not fit Angular's HttpClient interceptor pipeline or server-side rendering.

## Built-in UI

With the UI enabled, the server serves complete pages: `<apiPrefix>/ui/login`, `/register`, `/forgot-password`, `/reset-password`, `/magic-link`, `/2fa`, `/verify-email`, `/link-verify`, `/account-conflict`. They read `<apiPrefix>/ui/config`, whose `features` flags follow what the server has configured (register, forgot-password, magic link, SMS, OAuth providers, 2FA). Link to `/auth/ui/login` and you have sign-in with no frontend code. Branding: Node `ui.siteName`, `ui.primaryColor`, `ui.logoUrl`, `ui.customCss`; translations through an `ITemplateStore`.
