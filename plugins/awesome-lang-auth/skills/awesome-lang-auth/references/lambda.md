# AWS Lambda: `awesome-lambda-auth`

Checked against the `main` branch of https://github.com/awesome-lang-auth/awesome-lambda-auth at commit `4330384` (2026-09-30), on 2026-10-08. Not deployed for this check.

Maturity: **preview**. No registry release: you clone the repository, build the artifact and deploy the stack. Status and what is still gated: `docs/PROGRESS.md` in the repository.

## What it is, and when to choose it

A self-hosted alternative to Amazon Cognito, deployed as a stack in the user's own AWS account: an HTTP API in API Gateway, one Lambda function (`provided.al2023`, arm64) running the whole auth router, one DynamoDB table with TTL, the signing secrets in Secrets Manager, a 14-day log group and nine CloudWatch alarms; CloudFront and a KMS signing key for the OIDC provider are optional. It imports the Go core (`awesome-go-auth`) unchanged and keeps the wire contract, so the Angular, React, Flutter clients and `auth.js` work with a base-URL change.

It is a **product configured by file and environment**, not a library: you do not write stores or handlers. Choose it when the user wants serverless auth on AWS, or Cognito without per-MAU pricing. Choose the Go or Node library instead when the auth routes must live inside an existing server.

## Deploy

Needs Docker (the build runs in a pinned Go container) and the AWS CLI v2 with a named profile allowed to create IAM roles, Lambda functions, API Gateway APIs, DynamoDB tables, Secrets Manager secrets and an S3 bucket. The SAM CLI is not needed.

```bash
git clone https://github.com/awesome-lang-auth/awesome-lambda-auth
cd awesome-lambda-auth
git checkout 4330384f3c508a702fd1e624a8074876a99ce50e                      # the commit this reference describes
LAMBDAS="auth webhook-worker script-runner" ./scripts/build-lambda.sh   # one reproducible dist/<name>-lambda.zip per function the template names
./scripts/deploy.sh --profile <profile> --region <region>               # package + deploy with the AWS CLI
```

Build every function the template names, even those whose switch is off: `deploy.sh` checks each `CodeUri` of `infra/sam/template.yaml` and stops on a missing zip, so `build-lambda.sh` alone (which builds only `auth`) is not enough. `./scripts/deploy.sh --profile <profile> --region <region> --build` builds the whole list itself. The scripts run from the checked-out commit; move to a newer one only after checking it against this file.

`--profile` and `--region` are required and have no defaults: ask the user which account and region, and never deploy to an account they did not name. `deploy.sh` prints the account it resolved and asks for confirmation; do not pass `--yes` on the user's behalf. `scripts/teardown.sh` removes the stack and lists what outlives it. Template parameters and the IAM policy: `infra/sam/README.md`.

## Configure

Two layered sources, read at cold start:

1. A JSON document from `AWESOME_AUTH_CONFIG_FILE=<path in the artifact>` or inline in `AWESOME_AUTH_CONFIG_JSON` (never both). It needs `"schemaVersion": 1` at the root.
2. `AWESOME_AUTH_*` environment variables, one per knob, over the document (lists are comma-separated).

The defaults are the safe posture: production environment, CSRF on, Secure cookies, API prefix `/auth`. Secrets are references, never values: `{"secretsManager": "<arn>#<jsonKey>"}` or `{"ssmParameter": "<name>"}`, or their environment variables (the template sets `AWESOME_AUTH_JWT_ACCESS_SECRET_SECRETSMANAGER` and the refresh one). A plaintext secret in the document, or a domain the build does not act on yet, **refuses to start** and lists every problem at once.

Knobs most projects touch: `email.siteUrls` (the front-end origins emailed links may point to) and `http.cors.origins` (together they form the redirect allow-list), `email.mailer.*` or `email.deliveryWebhook.*` (SES by default, SNS for SMS), `ui.enabled` (off by default; `AWESOME_AUTH_UI_ENABLED=true` serves `/auth/ui/login` and `/auth/ui/auth.js`), `twoFactor.appName`, `oauth.providers` and `oauth.provisioning`, `rateLimit.*`, `admin.*` (the console at `/admin`, guarded by `admin.accessPolicy`: `is-admin-flag`, `rbac:<role>`, `permission:<perm>`, or `open`, which lets every request in and is for local experiments only), `idProvider.*` (OIDC issuer signed by KMS or a PEM). Every knob: `docs/config-reference.md`; the schema with defaults: `docs/spec/config-schema.md`.

## Behaviour to plan around

- **Browser clients need one origin.** Cookies are `SameSite=Lax` and the page must read the CSRF cookie, so serve the app and rewrite `/auth/*` to the API from the same origin (CloudFront, or the frontend host's rewrites). The repository's `examples/angular-client` and `examples/flutter-client` run the published clients that way, unmodified.
- **Rate limiting is on by default**: 10 requests per 60-second window, keyed by account, over the credential flows; the rest get `429 RATE_LIMITED` with `Retry-After`.
- Registration follows the Go core: `POST /auth/register` is mounted.
- Session tokens are HS256; only the OIDC surface signs RS256.

## Verify a deployment

```bash
AWESOME_AUTH_CONTRACT_BASE_URL=https://<api-id>.execute-api.<region>.amazonaws.com \
AWESOME_AUTH_CONTRACT_REQUIRE=register,csrf,secure-cookies,sessions,totp \
./scripts/toolchain.sh go test -count=1 -v ./test/contract/...
```

`REQUIRE` turns a missing capability into a failure instead of a skip. The SKILL.md request sequence also works against the stack (prefix `/auth`, HTTPS).
