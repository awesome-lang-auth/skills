# Dart (Shelf, Dart Frog): `awesome_dart_auth`

Checked against the `main` branch of https://github.com/awesome-lang-auth/awesome-dart-auth at commit `d71bd21` (2026-10-08; packages at version 1.9.0, Dart SDK `^3.11.0`), on 2026-10-08.

Maturity: **preview**. Not published on pub.dev: install from git. This is a **server** package; the Flutter client is `awesome_flutter_auth` (see `references/clients.md`).

## Read this first: it does not build today, and it is bearer-only

**At the checked commit the core package does not compile.** With Dart 3.13.5 (stable on 2026-10-08) the install resolves and `dart analyze` of an app file passes, but `dart run` stops on two compile errors inside the package: `lib/src/tools/notification_service.dart:162` (a comma after the closing `}` of the named parameters of `_postJson`) and `lib/src/routing/auth_router.dart:1152` (`s.createdAt.toIso8601String()` on a nullable `DateTime`). Check the repository for a newer commit or an open issue before choosing it, and tell the user; until it builds, recommend the Node.js or Go server for a Dart or Flutter project's backend.

The README describes cookie and CSRF support for web clients, but at this commit the router does not issue auth cookies: `POST /auth/login` and `POST /auth/register` always answer `{"user": {...}, "accessToken", "refreshToken", "expiresInSeconds", "tokenType": "Bearer"}`; `GET /auth/me` reads only `Authorization: Bearer`; `POST /auth/refresh` needs `{"refreshToken"}` in the body. The `cookieSecure` and `cookiePrefix` settings exist but nothing reads them. Other differences from the reference that clients notice:

- `GET /auth/me` returns `id`, `emailVerified`, `totpEnabled` instead of `sub`, `isEmailVerified`, `isTotpEnabled`.
- A login that needs TOTP answers `{"requiresTwoFactor": true, "userId"}`, without the reference's `tempToken`.

So the cookie-mode clients (Angular, Flutter on web, `auth.js`, React in a browser) do not work against it as is. A native app in bearer mode can, if it tolerates the `/me` shape. Tell the user, and run the SKILL.md bearer requests before building on it. For a Dart backend that must serve browsers today, consider running the Node or Go server for auth instead.

## Install

Pin the commit this reference describes; without `--git-ref`, pub follows the moving default branch. Move to a newer commit only after checking it against this file (and whether it compiles now).

Shelf:

```bash
REF=d71bd213c6153ba8cb659e23e42a98af1adc184e
dart pub add awesome_dart_auth --git-url https://github.com/awesome-lang-auth/awesome-dart-auth --git-path packages/awesome_dart_auth --git-ref $REF
dart pub add shelf
```

Dart Frog: the adapter depends on the core through a path inside the repository, which pub resolves to the exact commit. Adding the core at the default ref (`HEAD`) and the adapter next to it fails version solving ("awesome_dart_auth_dart_frog from git is forbidden"). Pin both to the same full commit (verified with Dart 3.13.5 on 2026-10-08):

```bash
REF=d71bd213c6153ba8cb659e23e42a98af1adc184e
dart pub add awesome_dart_auth --git-url https://github.com/awesome-lang-auth/awesome-dart-auth --git-path packages/awesome_dart_auth --git-ref $REF
dart pub add awesome_dart_auth_dart_frog --git-url https://github.com/awesome-lang-auth/awesome-dart-auth --git-path packages/awesome_dart_auth_dart_frog --git-ref $REF
```

(Adding only the adapter also resolves, with the core as a transitive dependency.)

## Shelf

```dart
import 'dart:io';
import 'package:awesome_dart_auth/awesome_dart_auth.dart';
import 'package:shelf/shelf.dart';
import 'package:shelf/shelf_io.dart' as shelf_io;

Future<void> main() async {
  final secret = Platform.environment['JWT_SECRET']!;
  // Production settings unless COOKIE_INSECURE=1, which only local http development sets.
  final config = Platform.environment['COOKIE_INSECURE'] == '1'
      ? AuthConfig.development(jwtSecret: secret)          // issuer http://localhost:8080, cookieSecure false
      : AuthConfig.production(jwtSecret: secret, issuer: 'https://api.example.com');

  final router = AuthRouter(
    config: config,
    authService: AuthService(
      config: config,
      userStore: PgUserStore(),          // your UserStore, see references/stores.md
      sessionStore: PgSessionStore(),    // your SessionStore
    ),
    callbacks: AuthCallbacks(
      onForgotPassword: (user, token) async { /* send the reset mail */ },
      onSendVerificationEmail: (user, token) async { /* send the verification mail */ },
    ),
  );

  final handler = const Pipeline()
      .addMiddleware(csrfMiddleware(apiBasePath: '/auth'))
      .addHandler(router.handler);

  await shelf_io.serve(handler, InternetAddress.anyIPv4, 8080);
}
```

`AuthRouter.handler` is a Shelf `Handler` serving `/auth/*`, the built-in UI (`/auth/ui/login`, `/auth/ui/auth.js`, `/auth/ui/config`), the admin UI under `/auth/admin` and `/health`. Other `AuthRouter` parameters: `tokenStore`, `tenantStore`, `apiKeyStore`, `templateStore`. `AuthCallbacks` also takes `onRegister`, `onMagicLinkSend`, `onMagicLinkVerify`, `onSmsSend`, `onSmsVerify`, `onOAuthStart`, `onOAuthCallback`, `onLinkRequest`, `onLinkVerify`.

## Dart Frog

Put the router in front of your routes with the adapter's middleware, which answers auth paths and passes every 404 on to Dart Frog:

```dart
// routes/_middleware.dart
import 'package:awesome_dart_auth_dart_frog/awesome_dart_auth_dart_frog.dart';
import 'package:dart_frog/dart_frog.dart';
import '../lib/auth.dart';   // exposes `authRouter`, built as in the Shelf example

Handler middleware(Handler handler) => handler.use(awesomeDartAuthMiddleware(authRouter));
```

`awesomeDartAuthHandler(authRouter)` is the same thing as a plain route handler.

## AuthConfig

Constructors: `AuthConfig(jwtSecret:, issuer:, ...)`, `AuthConfig.production(jwtSecret:, issuer:)`, `AuthConfig.development(jwtSecret:)` and `AuthConfig.testing(jwtSecret:)`. Fields with defaults: `accessTokenTtl` 15 minutes, `refreshTokenTtl` 30 days, `apiBasePath` `/auth`, `sessionCheckOn` `SessionCheckOn.refresh`, `uiConfig`, `oauthProviders` `{google, github, generic}`, and switches that are **on by default**: `enableAdminUi`, `enableAuthUi`, `enableIdpMode`, `enableTelemetry`, `enableSse`, `enableMcpCompatibility`. Turn off what the app does not use (`AuthConfig(..., enableAdminUi: false, enableIdpMode: false)`).

## Security notes

- `POST /auth/register` is always mounted; block it in front of the router if sign-up is closed.
- `csrfMiddleware` checks only paths under `apiBasePath`.
- Add CORS yourself (for example `shelf_cors_headers`) with exact origins.
