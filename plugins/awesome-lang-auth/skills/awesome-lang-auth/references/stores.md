# Storage contracts and database examples

Checked on 2026-10-08 against the versions in SKILL.md metadata. Interface and method names below are copied from the source, not from the docs.

## Contents

- Rules for every port, and the database itself
- Node.js: `IUserStore`, `ISessionStore`, the other interfaces
- Node.js: a complete PostgreSQL store, and the other database examples that exist (in-memory, SQLite, MySQL, MongoDB, PostgREST, PHP-CRUD-API; no Redis)
- Go: `UserStore`, `SessionStore` and the optional interfaces
- Python: `UserStore` (sessions and token lookups included)
- Rust: the traits
- Dart: the contracts

The libraries persist nothing themselves (the Lambda stack brings DynamoDB and needs nothing from you). You implement a user store for the app's database, usually a session store too, and pass them in. Whatever the port, the rules are the same:

- Store what you are given, as given: the libraries hash passwords and tokens before they reach the store where they need to. Never hash again, never return a different value.
- A lookup that finds nothing returns the language's "nothing" (`null`, `None`, an error in Go), not an exception, unless the contract says otherwise.
- Map database columns to the library's field names in one function, and return dates as dates.
- Put a unique index on email (per tenant when the app is multi-tenant) and indexes on every token column that is looked up.
- In-memory stores are for tests and prototypes only.

The database (SKILL.md Step 1 settles it with the user):

- Use the database and the driver or ORM the project already uses. If there is none, or more than one, ask before adding one.
- If a users table already exists, map the library's fields to it and add only the missing columns, through the project's migration tool. The `CREATE TABLE` below is for a project with no users table.
- Show the migration to the user before running it against any shared database (staging, production, a teammate's). Never drop or recreate an existing table.

## Node.js: `IUserStore` and friends (`@awesome-lang-auth/node` 1.10.8)

Required methods of `IUserStore<U extends BaseUser = BaseUser>`:

```ts
findByEmail(email: string): Promise<U | null>;
findById(id: string): Promise<U | null>;
create(data: Partial<U>): Promise<U>;
updateRefreshToken(userId: string, token: string | null, expiry: Date | null): Promise<void>;
updateLastLogin(userId: string): Promise<void>;
updateResetToken(userId: string, token: string | null, expiry: Date | null): Promise<void>;
updatePassword(userId: string, hashedPassword: string): Promise<void>;
updateTotpSecret(userId: string, secret: string | null): Promise<void>;     // also set isTotpEnabled = secret !== null
updateMagicLinkToken(userId: string, token: string | null, expiry: Date | null): Promise<void>;
updateSmsCode(userId: string, code: string | null, expiry: Date | null): Promise<void>;
```

Optional methods, each enabling a feature: `findByResetToken` (reset password), `findByMagicLinkToken` (magic link), `updateEmailVerificationToken`, `updateEmailVerified`, `findByEmailVerificationToken` (email verification), `updateEmailChangeToken`, `updateEmail`, `findByEmailChangeToken` (change email), `updateAccountLinkToken`, `findByAccountLinkToken` (account linking), `findByProviderAccount` (OAuth, recommended), `listUsers` (admin), `updateProfile` (`PATCH /profile`), `updatePhoneNumber` (`POST /add-phone`), `update(userId, patch)` (admin promote by flag).

`BaseUser` fields (camelCase): `id`, `email`, `password` (the hash), `role`, `firstName`, `lastName`, `loginProvider`, `providerAccountId`, `refreshToken`, `refreshTokenExpiry`, `resetToken`, `resetTokenExpiry`, `totpSecret`, `isTotpEnabled`, `isEmailVerified`, `magicLinkToken`, `magicLinkTokenExpiry`, `smsCode`, `smsCodeExpiry`, `phoneNumber`, `emailVerificationToken`, `emailVerificationTokenExpiry`, `emailVerificationDeadline`, `pendingEmail`, `emailChangeToken`, `emailChangeTokenExpiry`, `accountLinkPendingEmail`, `accountLinkPendingProvider`, `accountLinkToken`, `accountLinkTokenExpiry`, `lastLogin`, `isAdmin`. Add your own fields with `IUserStore<MyUser>`.

`ISessionStore` (pass it as `sessionStore` to enable device lists and revocation):

```ts
createSession(info: Omit<SessionInfo, 'sessionHandle'>): Promise<SessionInfo>;   // you generate sessionHandle
getSession(sessionHandle: string): Promise<SessionInfo | null>;
getSessionsForUser(userId: string, tenantId?: string): Promise<SessionInfo[]>;
updateSessionLastActive(sessionHandle: string): Promise<void>;
revokeSession(sessionHandle: string): Promise<void>;
revokeAllSessionsForUser(userId: string, tenantId?: string): Promise<void>;
// optional: getAllSessions(limit, offset), deleteExpiredSessions(), updateSessionRefreshTokenHash(handle, hash)
```

`SessionInfo` is `{ sessionHandle, userId, tenantId?, createdAt, expiresAt, lastActiveAt?, userAgent?, ipAddress?, data?, refreshTokenHash? }`: persist every field, `refreshTokenHash` and `data` included. With `session.checkOn: 'allcalls'` the store is read on every request, so back it with something fast (Redis, or a `Map` on a single instance).

Other interfaces, all optional: `IRolesPermissionsStore` (`rbacStore`), `ITenantStore`, `IUserMetadataStore`, `ILinkedAccountsStore`, `IPendingLinkStore`, `ISettingsStore`, `IApiKeyStore`, `IWebhookStore`, `ITemplateStore` (`MemoryTemplateStore` is built in), `ITelemetryStore`, `ITokenStore`, `ISseDistributor`.

### PostgreSQL (`pg`), condensed from the PostgreSQL docs page and completed

```bash
npm i pg && npm i -D @types/pg
```

The table below is for a project without one; with an existing users table, keep it and change `toUser` and the column names instead (see the database rules above).

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  password VARCHAR(255),
  first_name VARCHAR(255), last_name VARCHAR(255),
  role VARCHAR(50) DEFAULT 'user',
  login_provider VARCHAR(50) DEFAULT 'local', provider_account_id VARCHAR(255),
  refresh_token TEXT, refresh_token_expiry TIMESTAMPTZ,
  reset_token VARCHAR(255), reset_token_expiry TIMESTAMPTZ,
  totp_secret VARCHAR(255), is_totp_enabled BOOLEAN DEFAULT false,
  magic_link_token VARCHAR(255), magic_link_token_expiry TIMESTAMPTZ,
  sms_code VARCHAR(10), sms_code_expiry TIMESTAMPTZ,
  phone_number VARCHAR(20), is_email_verified BOOLEAN DEFAULT false,
  last_login TIMESTAMPTZ, created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX ON users (reset_token);
CREATE INDEX ON users (magic_link_token);
```

```ts
import { Pool } from 'pg';
import { randomUUID } from 'node:crypto';
import type { IUserStore, BaseUser } from '@awesome-lang-auth/node';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// snake_case row -> camelCase BaseUser: the one place where columns are mapped
function toUser(r: any): BaseUser | null {
  if (!r) return null;
  return {
    id: r.id, email: r.email, password: r.password ?? undefined, role: r.role ?? undefined,
    firstName: r.first_name, lastName: r.last_name,
    loginProvider: r.login_provider, providerAccountId: r.provider_account_id,
    refreshToken: r.refresh_token, refreshTokenExpiry: r.refresh_token_expiry,
    resetToken: r.reset_token, resetTokenExpiry: r.reset_token_expiry,
    totpSecret: r.totp_secret, isTotpEnabled: r.is_totp_enabled,
    magicLinkToken: r.magic_link_token, magicLinkTokenExpiry: r.magic_link_token_expiry,
    smsCode: r.sms_code, smsCodeExpiry: r.sms_code_expiry,
    phoneNumber: r.phone_number, isEmailVerified: r.is_email_verified, lastLogin: r.last_login,
  };
}

async function one(sql: string, args: unknown[]): Promise<BaseUser | null> {
  return toUser((await pool.query(sql, args)).rows[0]);
}

async function set(id: string, cols: Record<string, unknown>): Promise<void> {
  const keys = Object.keys(cols);   // column names are fixed below, never user input
  const sql = `UPDATE users SET ${keys.map((k, i) => `${k} = $${i + 2}`).join(', ')} WHERE id = $1`;
  await pool.query(sql, [id, ...Object.values(cols)]);
}

export class PgUserStore implements IUserStore {
  findByEmail(email: string) { return one('SELECT * FROM users WHERE email = $1', [email]); }
  findById(id: string) { return one('SELECT * FROM users WHERE id = $1', [id]); }
  async create(d: Partial<BaseUser>): Promise<BaseUser> {
    const user = await one(
      `INSERT INTO users (id, email, password, first_name, last_name, role, login_provider, provider_account_id)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8) RETURNING *`,
      [randomUUID(), d.email, d.password ?? null, d.firstName ?? null, d.lastName ?? null,
       d.role ?? 'user', d.loginProvider ?? 'local', d.providerAccountId ?? null]);
    return user!;
  }
  updateRefreshToken(id: string, token: string | null, expiry: Date | null) { return set(id, { refresh_token: token, refresh_token_expiry: expiry }); }
  updateLastLogin(id: string) { return set(id, { last_login: new Date() }); }
  updateResetToken(id: string, token: string | null, expiry: Date | null) { return set(id, { reset_token: token, reset_token_expiry: expiry }); }
  updatePassword(id: string, hash: string) { return set(id, { password: hash }); }
  updateTotpSecret(id: string, secret: string | null) { return set(id, { totp_secret: secret, is_totp_enabled: secret !== null }); }
  updateMagicLinkToken(id: string, token: string | null, expiry: Date | null) { return set(id, { magic_link_token: token, magic_link_token_expiry: expiry }); }
  updateSmsCode(id: string, code: string | null, expiry: Date | null) { return set(id, { sms_code: code, sms_code_expiry: expiry }); }
  findByResetToken(token: string) { return one('SELECT * FROM users WHERE reset_token = $1', [token]); }
  findByMagicLinkToken(token: string) { return one('SELECT * FROM users WHERE magic_link_token = $1', [token]); }
}
```

### The other database examples that exist (all for Node)

| Database | Driver | Where (repository `awesome-lang-auth/awesome-node-auth`, tag `v1.10.8`, or the docs site) |
|---|---|---|
| In-memory | none | `examples/in-memory-user-store.ts`: every required and optional method, plus linked-accounts, settings and template stores |
| SQLite | `better-sqlite3` | `examples/sqlite-user-store.example.ts`: creates its tables; also linked-accounts, settings and template stores |
| MySQL / MariaDB | `mysql2` | `examples/mysql-user-store.example.ts` |
| MongoDB | `mongodb` | `examples/mongodb-user-store.example.ts` (indexes listed on the MongoDB docs page) |
| PostgreSQL | `pg` | the snippet above; the docs page https://awesomelangauth.com/docs/database/postgresql shows four methods |
| PostgREST | `fetch` | https://awesomelangauth.com/docs/database/postgrest: an `IUserStore` over the PostgREST HTTP API, no SQL |
| PHP-CRUD-API | `fetch` | https://awesomelangauth.com/docs/database/php-crud-api: the same over PHP-CRUD-API |

The example files import from `'../src/index'`: change that to `'@awesome-lang-auth/node'` when copying one. **Redis**: no Redis user or session store exists in the docs or repositories. The docs recommend Redis as the backend for `ISessionStore` under `checkOn: 'allcalls'`, and the "Scaling SSE" page sketches an `ISseDistributor` over Redis pub/sub. Write a Redis session store from the `ISessionStore` contract above if needed (key by `sessionHandle`, a set per `userId`, TTL at `expiresAt`).

## Go: `UserStore` and optional interfaces (`awesome-go-auth` v0.12.0)

```go
type UserStore interface {                       // required
    CreateUser(ctx context.Context, user User) (User, error)
    GetUserByEmail(ctx context.Context, email, tenantID string) (User, error)
    GetUserByID(ctx context.Context, id, tenantID string) (User, error)
}
type SessionStore interface {                    // required by auth.New
    CreateSession(ctx context.Context, session Session) (Session, error)
    GetSessionByRefreshTokenHash(ctx context.Context, tokenHash string) (Session, error)
    UpdateSession(ctx context.Context, session Session) error
}
```

A lookup that finds nothing returns a non-nil error: the service treats `err == nil` as "found" (register refuses an existing email that way). Emails arrive normalised.

The service type-asserts the store for optional interfaces, each switching on a feature: `UserAccountStore` (profile, delete account), `SessionLookupStore`, `UserPasswordStore` (reset and change password), `MagicLinkStore`, `SMSStore`, `TOTPStore`, `UserTwoFactorPolicyStore`, `UserAdminFlagStore`, `EmailVerificationStore`, `EmailChangeStore`, `SessionAdminStore` (device list, revoke), `AdminUserStore`, `UserLookupStore`, `SessionLister`, `RoleLister`, `AuthCodeStore` (OIDC), plus `UserMetadataStore`, `RolesPermissionsStore` and `TenantStore` passed with `WithMetadataProvider`, `WithRBACProvider`, `WithTenantProvider`. Without the interface, the route answers that the feature is not supported. In-memory implementations: `NewMemoryUserStore`, `NewMemorySessionStore`, `NewMemoryMetadataStore`, `NewMemoryRolesPermissionsStore`, `NewMemoryTenantStore`, `NewMemoryWebhookStore`. `User` fields: `ID`, `Email`, `PasswordHash`, `TenantID`, `PhoneNumber`, `FirstName`, `LastName`, `Role`, `IsAdmin` and more in `models.go`.

The repository's `examples/` (chi-postgres, gin-mongodb, echo-sqlite) run on the in-memory stores and show only comment skeletons for the database.

## Python: `UserStore` (`awesome-python-auth` 1.1.0)

```python
from awesome_python_auth import UserStore
from awesome_python_auth.models import StoredUser, StoredSession

class PgUserStore(UserStore):
    async def get_by_email(self, email: str) -> StoredUser | None: ...
    async def get_by_id(self, user_id: str) -> StoredUser | None: ...
    async def create(self, user: StoredUser) -> StoredUser: ...
    async def update(self, user: StoredUser) -> StoredUser: ...
    async def delete(self, user_id: str) -> None: ...
    # Sessions: the base class defaults do nothing (create returns its input, lookups return None),
    # so override them, or refresh and logout cannot find the session.
    async def create_session(self, session: StoredSession) -> StoredSession: ...
    async def get_sessions_for_user(self, user_id: str) -> list[StoredSession]: ...
    async def get_session_by_handle(self, handle: str) -> StoredSession | None: ...
    async def update_session(self, session: StoredSession) -> None: ...
    async def delete_session(self, handle: str) -> None: ...
    async def delete_sessions_for_user(self, user_id: str) -> None: ...
    # Required for password reset, email verification and email change: the base class
    # returns None and the routes have no fallback, so without them those routes always
    # answer 400. The router passes the sha256 hex of the token: compare with the stored hash.
    async def find_by_reset_token(self, token_hash: str) -> StoredUser | None: ...          # reset_password_token
    async def find_by_verification_token(self, token_hash: str) -> StoredUser | None: ...   # verification_token
    async def find_by_pending_email_token(self, token_hash: str) -> StoredUser | None: ...  # pending_email_token
    # For the admin panel: list_all_users, count_users, list_all_sessions, count_active_sessions.
```

`StoredUser` (pydantic) fields: `id`, `email`, `hashed_password`, `first_name`, `last_name`, `name`, `phone_number`, `role`, `is_email_verified`, `is_totp_enabled`, `totp_secret`, `login_provider`, `last_login`, `metadata`, `roles`, `permissions`, `is_admin`, `tenant_id`, `pending_email`, `pending_email_token`, `verification_token`, `reset_password_token`. `StoredSession`: `handle`, `user_id`, `refresh_token_hash`, `user_agent`, `ip_address`, `created_at`, `last_active_at`. `update` receives the whole user: write every field. `InMemoryUserStore` (in `awesome_python_auth.models`) implements all of it. Other stores: `RolesPermissionsStore`, `TenantStore`, `TokenStore`, `ApiKeyStore`, `WebhookStore`, `LinkedAccountsStore`, `PendingLinkStore`, `SettingsStore`, `TemplateStore`, `TelemetryStore`, each with an `InMemory...` version.

## Rust: traits in `awesome_rust_auth::traits` (git `acb3d53`)

Implement with `#[async_trait]`. `UserStore`: `create_user`, `get_user_by_email(&TenantId, &str)`, `get_user_by_id`, `update_oauth_links`, `update_profile`, `set_password_hash`, `set_email_verified`, `update_email`, `set_totp`, `delete_user`. `SessionStore`: `create_session`, `get_session_by_refresh_token(&Uuid)`, `revoke_session`, `list_sessions_for_user`. Also `ApiKeyStore`, `RolesPermissionsStore`, `PendingLinkStore`, `TelemetryStore`, `SseDistributor`, `EventBus` (`event_bus::InMemoryEventBus` provided), `MagicLinkStore`, `OtpStore`, `PasswordResetStore`, `EmailVerificationStore`. Lookups return `AuthResult<Option<T>>`. The crate ships no store implementations; its examples contain no database code.

## Dart: contracts in `awesome_dart_auth` (git `d71bd21`)

```dart
abstract interface class UserStore {
  Future<AuthUser?> findById(String id);
  Future<AuthUser?> findByEmail(String email);
  Future<AuthUser> save(AuthUser user);      // create or update
  Future<AuthUser> update(AuthUser user);
  Future<void> delete(String id);
}
abstract interface class SessionStore {
  Future<void> save(AuthSession session);
  Future<AuthSession?> findById(String id);
  Future<void> revoke(String id);            // mark revoked; refresh then fails
}
```

`AuthUser` (freezed): `id`, `email`, `passwordHash`, `tenantId`, `firstName`, `lastName`, `phoneNumber`, `totpSecret`, `providers`, `roles`, `isActive`, `emailVerified`, `totpEnabled`, `createdAt`, `updatedAt`. `AuthSession`: `id`, `userId`, `tenantId`, `handle`, `userAgent`, `ipAddress`, `createdAt`, `expiresAt`, `revoked`, `scopes` (with `copyWith`). Also `AdminUserStore`, `AdminSessionStore`, `TokenStore`, `TenantStore`, `RolesPermissionsStore`, `ApiKeyStore`, `PendingLinkStore`, `TemplateStore`, `TelemetryStore`, `WebhookStore`. No store implementations ship; the `examples/` stubs return `null`.
