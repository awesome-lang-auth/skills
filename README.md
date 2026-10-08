# awesome-lang-auth skills

An [Agent Skill](https://agentskills.io) that teaches coding agents to add authentication with the [awesome-lang-auth](https://awesomelangauth.com) libraries: login and sign-up, cookie or bearer sessions with refresh tokens, CSRF, TOTP 2FA, magic links, SMS codes, OAuth and API keys, on Node.js (Express, NestJS, Next.js), Go, Python (FastAPI), Rust, Dart and AWS Lambda, with Angular, React, Flutter or plain-JS frontends.

The skill walks the agent through one workflow: detect the stack, pick the library and its install command, configure secrets and cookie or bearer mode, implement the storage interfaces for the project's database, mount the routes and the built-in UI, wire the frontend, verify with a concrete request sequence, and go through a security checklist. Per-runtime details live in reference files the agent reads only when it needs them. Library versions are pinned in the skill (checked on 2026-10-08) and the snippets for Node.js, Go and Python were run against real servers.

This repository is also a Claude Code plugin marketplace.

## Install in Claude Code

In a Claude Code session:

```
/plugin marketplace add awesome-lang-auth/skills
/plugin install awesome-lang-auth@awesome-lang-auth
```

From a shell, the same with the CLI:

```bash
claude plugin marketplace add awesome-lang-auth/skills
claude plugin install awesome-lang-auth@awesome-lang-auth
```

Then ask for what you need ("add email/password login with 2FA to this FastAPI app", "wire the Angular frontend to our Go auth server"); the skill loads when the task matches. `/plugin marketplace update awesome-lang-auth` picks up new versions (Claude Code keeps the cached copy until the plugin's `version` changes; see Maintaining).

## Other agents that support Agent Skills

Codex, Cursor, GitHub Copilot and VS Code read the same `SKILL.md` format. Copy the skill folder, `plugins/awesome-lang-auth/skills/awesome-lang-auth`, into the agent's skills directory as its documentation describes:

- Codex: https://developers.openai.com/codex/skills/
- Cursor: https://cursor.com/docs/context/skills
- GitHub Copilot: https://docs.github.com/en/copilot/concepts/agents/about-agent-skills
- VS Code: https://code.visualstudio.com/docs/copilot/customization/agent-skills

```bash
git clone --depth 1 https://github.com/awesome-lang-auth/skills awesome-lang-auth-skills
cp -r awesome-lang-auth-skills/plugins/awesome-lang-auth/skills/awesome-lang-auth <the agent's skills directory>/
```

Copy the whole folder, not only `SKILL.md`: the `references/` files are part of the skill, and `LICENSE.txt` carries the license with the copy.

## claude.ai

Upload the skill as a ZIP in **Customize > Skills**. The claude.ai upload form allows at most 200 characters of description, while the skill's own description is longer (the open specification allows 1,024) so that other agents match it on more terms. Swap in a short description before zipping. From the directory where you ran the `git clone` above:

```bash
cp -r awesome-lang-auth-skills/plugins/awesome-lang-auth/skills/awesome-lang-auth .
# claude.ai accepts at most 200 characters of description
perl -pi -e 's/^description: .*/description: Adds login, sessions, 2FA, OAuth and API keys with the awesome-lang-auth libraries. Use when adding auth to Node.js, Go, FastAPI, Rust, Dart or AWS Lambda backends, or Angular, React or Flutter apps./' awesome-lang-auth/SKILL.md
zip -r awesome-lang-auth.zip awesome-lang-auth
```

The ZIP must contain the `awesome-lang-auth` folder itself, as this one does. `perl -pi -e` behaves the same on macOS and Linux.

## Agents without skills support

Paste this prompt at the start of the task:

```text
Add authentication to this project with the awesome-lang-auth libraries (https://awesomelangauth.com).
1. Detect the backend and pick the server library: Node.js `npm i @awesome-lang-auth/node express`;
   Go `go get github.com/nik2208/awesome-go-auth@v0.12.0` (beta); FastAPI `pip install awesome-python-auth`;
   Rust and Dart install from their GitHub repositories (preview: Rust ships no HTTP routes; Dart is bearer-only
   and its 2026-10-08 commit does not compile with Dart 3.13, so prefer Node.js or Go for a Dart/Flutter backend);
   AWS Lambda is a deployable stack (github.com/awesome-lang-auth/awesome-lambda-auth). Ask me before replacing an existing auth system.
   Unless I already said so, ask me before writing code: is sign-up open or closed; which database (use the one the project
   already has; show me any migration before running it on a shared database; never drop or recreate tables); are the
   browser app and the API on one origin in production.
2. Before writing code, read the chosen library's README at that version, or https://awesomelangauth.com/llms-full.txt.
   Do not invent options, routes or field names; do not hand-roll password hashing, JWTs or CSRF next to the library.
3. Secrets from the environment (two different ones for Node), never committed, never with a hardcoded fallback.
   Cookie mode with CSRF for browsers, bearer mode (`X-Auth-Strategy: bearer`) for native apps. Secure cookies unless
   an explicit local-only switch (such as COOKIE_INSECURE=1) is set. Prefer one origin for app and API.
4. Implement the library's user store (and session store) for our database, mapping columns to the library's field names.
5. Mount the auth routes and the built-in UI. Frontend: Angular `@awesome-lang-auth/angular`, React `@awesome-lang-auth/react`,
   Flutter `awesome_flutter_auth`, anything else `<script src="/auth/ui/auth.js"></script>` (window.AwesomeNodeAuth).
6. Verify with curl against the running app on a local or development database: register, login (cookies set), GET /auth/me,
   a cookie-authenticated POST without X-CSRF-Token refused with 403, the same with the header accepted, refresh, logout,
   refresh refused after logout.
7. Security: decide whether sign-up is open (Node mounts /register only with `defaultRegister` or `onRegister`;
   Go and Python always do), never accept a token with a `purpose` claim (the 2FA tempToken) as a session,
   CSRF on for cookie mode, CORS as an exact allow-list, HTTPS, rate limits on the credential routes (not on the
   /me and /refresh calls clients make on every page load), admin surfaces guarded.
Report which library and options you chose, and anything you could not verify.
```

## What is inside

```
.claude-plugin/marketplace.json                     marketplace "awesome-lang-auth"
plugins/awesome-lang-auth/.claude-plugin/plugin.json
plugins/awesome-lang-auth/skills/awesome-lang-auth/
  SKILL.md                                          the workflow, read first
  LICENSE.txt                                       a copy of LICENSE, so copies of the folder carry it
  references/node.md  go.md  python.md  rust.md  dart.md  lambda.md
  references/stores.md                              storage interfaces per port, database examples
  references/clients.md                             Angular, React, Flutter, auth.js, built-in UI
  references/security.md                            per-port defaults and the checklist
```

Versions checked (2026-10-08): `@awesome-lang-auth/node` 1.10.8, `github.com/nik2208/awesome-go-auth` v0.12.0, `awesome-python-auth` 1.1.0, `awesome-rust-auth` and `awesome_dart_auth` from git, `awesome-lambda-auth` from git, `@awesome-lang-auth/angular` 1.10.1, `@awesome-lang-auth/react` 0.1.0, `awesome_flutter_auth` 1.10.5. Maturity: Node, Python, Angular and Flutter stable; Go beta; Lambda, Rust and Dart preview; React early.

## Maintaining

Every change to the skill bumps `version` in `plugins/awesome-lang-auth/.claude-plugin/plugin.json` and `metadata.version` in `SKILL.md` to the same value. Claude Code keeps installed users on their cached copy until that version changes. The marketplace entry carries no `version`, so `plugin.json` is the only place it is set.

## License

[MIT](LICENSE)
