# PLAN.md — MFA Next.js MVP

## Purpose

Deliver a human-friendly, learnable Minimum Viable Product implementing Multi-Factor Authentication (MFA) for a Next.js application. The deliverable must be usable by a small engineering team and clear enough that a developer can pick it up, run it, modify it, and learn the rationale behind each design choice.

## Vision

Provide an MVP that offers strong, practical MFA options (TOTP and WebAuthn) plus robust account recovery and basic admin controls. The implementation should be secure-by-default, straightforward to understand, and easy to extend with additional factors (SMS, Email OTP, hardware tokens) later.

## Success criteria

* Users can sign up and log in with a password and are required to complete MFA during sign-in if enrolled.
* Users can enroll and manage TOTP (authenticator apps) and WebAuthn devices (platform or security keys).
* Users can generate and store recovery codes for account recovery.
* Admins can view MFA enrollment status and revoke devices.
* Implementation is fully typed (TypeScript), covered by unit and integration tests, and documented for handoff.

## Scope (MVP)

**Included**

* Credential-based authentication (email + password).
* Password hashing (argon2 or bcrypt).
* Session management using httpOnly, secure cookies (server-side sessions).
* TOTP enrollment and verification (RFC 6238 compatible).
* WebAuthn registration and authentication (platform and roaming authenticators).
* Recovery codes (single-use).
* Basic UI pages: Sign up, Sign in, MFA challenge, Setup TOTP, Setup WebAuthn, Recovery codes, Account security page.
* API endpoints for auth flows and device management.
* Minimal admin UI: list users with MFA status + revoke device.
* Logging for security events (enrollment, failed attempts, device registration).
* Tests: unit tests for core logic, integration tests for API flows, basic end-to-end flows.

**Excluded (for MVP)**

* SMS OTP provider integration (can be added later).
* Federated / SSO providers.
* Enterprise policies (adaptive auth, policy engine).
* Rate-limited/geo-based policy enforcement beyond simple throttling.
* Audit-grade immutable logs.

## Personas

* **End user** — Signs up, enables MFA, uses authenticator or security key, recovers account with recovery codes.
* **Developer** — Implements and extends the MVP, runs tests, performs maintenance.
* **Admin/SRE** — Monitors enrollments, revokes devices when compromised, inspects logs.

## User flows (high level)

1. **Sign-up**

   * User supplies email + password.
   * App hashes password and creates user record.
   * Optionally prompt to set up MFA now or later.

2. **Sign-in (no MFA enrolled)**

   * User supplies email + password.
   * Password verified → session created.

3. **Sign-in (MFA enrolled)**

   * User supplies email + password.
   * If credentials valid and user has MFA methods:

     * Create temporary auth session (limited validity).
     * Redirect to MFA challenge page.
     * User completes challenge (TOTP code or WebAuthn assertion).
     * On success, finalize full session.

4. **Enroll TOTP**

   * User requests TOTP setup.
   * Server generates secret (store hashed/encoded), returns provisioning URI / QR data.
   * User scans QR in authenticator app and provides current code.
   * Server verifies code; on success store TOTP method, issue recovery codes.

5. **Enroll WebAuthn**

   * Server produces registration challenge and stores ephemeral challenge.
   * Client completes navigator.credentials.create and returns attestation.
   * Server verifies attestation and stores credential public key + metadata.

6. **Recovery**

   * User can view/generate recovery codes.
   * Recovery code is single-use, stored hashed.
   * Recovery code path allows completing login with password + recovery code (and forces MFA re-enrollment or device check).

## Technical architecture

* **Frontend**: Next.js (app or pages router), TypeScript, React. UI library optional (Tailwind or MUI recommended for consistency).
* **Backend**: Next.js API routes (or app route handlers) in TypeScript. Keep auth logic centralized as reusable modules.
* **Database**: Relational DB (Postgres recommended). ORM: Prisma (recommended) or TypeORM. Store users, auth methods, webauthn credentials, recovery codes, sessions.
* **Sessions**: Server-side session store (DB-backed) with httpOnly secure cookies. Avoid storing JWTs client-side for primary sessions.
* **Crypto & libraries**:

  * Password hashing: `argon2` (recommended) or `bcrypt`.
  * TOTP: `speakeasy` or `otplib`.
  * WebAuthn: `@simplewebauthn/server` (server) + `@simplewebauthn/browser` (client), or equivalent.
  * Rate-limiting: IP + user throttling middleware (e.g., `express-rate-limit` or custom).
  * Session store: Prisma-backed or Redis if scaling.
* **Secrets/Env**: All secrets (cookie keys, WebAuthn RP ID, SMTP keys) stored in secrets manager / environment variables.

## Data model (conceptual)

```sql
-- users
id uuid primary key
email text unique not null
password_hash text not null
created_at timestamptz
updated_at timestamptz
is_active boolean default true

-- mfa_methods
id uuid primary key
user_id uuid references users(id)
type text -- 'totp' | 'webauthn'
label text
enabled boolean
created_at timestamptz

-- totp_secrets
id uuid primary key
mfa_method_id uuid references mfa_methods(id)
secret text -- store encrypted (not plain)
confirmed boolean
created_at timestamptz

-- webauthn_credentials
id uuid primary key
mfa_method_id uuid references mfa_methods(id)
credential_id text unique
public_key text
sign_count bigint
transports text[] -- optional
created_at timestamptz

-- recovery_codes
id uuid primary key
user_id uuid references users(id)
code_hash text
used boolean default false
created_at timestamptz

-- sessions
id uuid primary key
user_id uuid references users(id) nullable
type text -- 'full' | 'preauth'
data jsonb
expires_at timestamptz
created_at timestamptz
```

## API surface (examples)

All endpoints accept/return JSON. Use consistent error codes and messages.

* `POST /api/auth/signup`
  Body: `{ email, password }`
  Success: `{ ok: true }` or `400` / `409`.

* `POST /api/auth/signin`
  Body: `{ email, password }`
  Success (no MFA): set session cookie, return `{ ok: true }`.
  Success (has MFA): create preauth session, return `{ mfaRequired: true, sessionId }`.

* `POST /api/auth/mfa/totp/setup`
  Requires authenticated preauth/full session.
  Response: `{ qrUri, secret (optional) }`.

* `POST /api/auth/mfa/totp/verify`
  Body: `{ sessionId, code }` → verify; on success finalize session.

* `POST /api/auth/mfa/webauthn/register`
  Two-step: `GET` gives challenge, client calls browser API, `POST` returns attestation for verification.

* `POST /api/auth/recovery/generate`
  Generates hashed recovery codes, returns plaintext once.

* `POST /api/auth/recovery/consume`
  Body: `{ email, password, recoveryCode }` → verify and create full session.

* Admin endpoints: `GET /api/admin/users` (MFA status), `POST /api/admin/users/:id/revoke-method`.

## UI & pages/components

Pages:

* `/signup`
* `/signin`
* `/mfa/challenge` — shows TOTP input and WebAuthn button(s); allow "use recovery code" link.
* `/account/security` — list enrolled methods, add/remove buttons, regen recovery codes.
* `/admin/users` — list users, MFA status, revoke control.

Components:

* `MfaPrompt` — selects available methods, handles fallback ordering.
* `TotpSetup` — shows QR, copy secret, confirm input.
* `WebAuthnSetup` — handles browser calls and displays device metadata.
* `RecoveryCodesPanel` — shows codes once, with clear copy button and warning.

## Security & threat model (practical)

Top threats and mitigations:

* **Credential theft** — enforce bcrypt/argon2 hashing, password strength check, rate-limiting sign-in attempts.
* **OTP brute force** — limit attempts per account/IP, lockouts with exponential backoff.
* **Replay attacks (WebAuthn)** — store and validate sign_count; reject lower sign_count.
* **Session fixation** — rotate session ID on privilege changes (preauth → full).
* **CSRF** — include CSRF protections on state-changing endpoints (or require SameSite cookie + double-submit token for APIs used by non-browser clients).
* **Secret leakage** — never log secrets; store TOTP secrets encrypted at rest.
* **Recovery codes theft** — show once, store hashed, force device re-enrollment after recovery login.

## Privacy & compliance

* Only store the minimal data for MFA (public keys, encrypted secrets).
* Recovery codes stored hashed.
* For SMS-based factors (future), include explicit user consent and GDPR/PDPA obligations.
* Provide clear UI text: "This device can be used to sign in" and guidance about recovery codes.

## Testing strategy

* Unit tests for:

  * Password hashing and verification.
  * TOTP generation/verification.
  * WebAuthn challenge/attestation verification (mock hardware flows).
  * Recovery code generation/consumption.
* Integration tests for API flows:

  * Sign-up → enroll TOTP → sign-in with TOTP.
  * Sign-in with WebAuthn (use library helpers / mocks).
* E2E smoke tests:

  * Full sign-up and sign-in flows using a test authenticator library or mocked backend responses.
* Security tests:

  * Test rate limits and lockouts.
  * Verify session cookie flags, SameSite, secure.
* Manual QA checklist:

  * Attempt expired preauth session.
  * Attempt reused recovery code.
  * WebAuthn on different browsers (Chrome, Firefox, Edge) and on mobile if applicable.

## Observability & logging

* Log security events with structured logs: `{ event: 'mfa_enrolled', userId, methodType, ip, userAgent }`.
* Mask/hide secrets in logs.
* Expose metrics:

  * MFA enrollment rate.
  * MFA challenge failures (per method).
  * Recovery code usage count.
* Configure alerts for unusual patterns (spikes in failures).

## Deployment & infra

* Environment variables required:

  * `DATABASE_URL`, `SESSION_SECRET`, `WEBAUTHN_RPID`, `WEBAUTHN_ORIGIN`, SMTP/SendGrid keys, third-party provider keys.
* Secrets stored in a secure vault (e.g., AWS Secrets Manager, Vercel secrets).
* Use HTTPS exclusively (enforce HSTS).
* Prefer DB-backed session store for horizontal scaling; Redis is optional for performance.
* CI: run tests and lint, build, then deploy to hosting (Vercel recommended for Next.js, or any Node host).

## Developer goodness / code style

* Use TypeScript everywhere.
* Centralize auth logic in `lib/auth/*`.
* Keep API route handlers thin; delegate to service-layer functions for testability.
* Follow a clear file layout (see next section).
* Provide inline JSDoc on all auth functions.
* Add an `AUTH_README.md` explaining flows and important invariants.

## Recommended file structure

```
/src
  /pages or /app
    /api
      /auth
        signup.ts
        signin.ts
        totp/
          setup.ts
          verify.ts
        webauthn/
          challenge.ts
          register.ts
          authenticate.ts
        recovery/
          generate.ts
          consume.ts
  /lib
    /auth
      index.ts
      sessions.ts
      totp.ts
      webauthn.ts
  /db
    prisma/schema.prisma
  /components
    TotpSetup.tsx
    WebAuthnSetup.tsx
    MfaPrompt.tsx
  /tests
    unit/
    integration/
  /scripts
    regenerate-recovery-codes.ts
```

## Handoff checklist (what a human reviewer should do)

* Run the app locally: confirm `README` has setup steps and environment variables.
* Execute test suite and confirm all tests pass.
* Manually sign up, enroll TOTP and WebAuthn (if available), store recovery codes, then sign in.
* Walk the code path for sign-in: credential verification → preauth session → challenge → finalize session.
* Review `lib/auth` for crypto and storage invariants.
* Confirm admin endpoints are protected by admin-only middleware and provide least privileges.
* Inspect logs to validate no secrets are emitted.

## Acceptance criteria (concrete)

* All listed API endpoints implemented and tested.
* TOTP and WebAuthn flows work end-to-end in a browser.
* Recovery codes generated and consumed correctly; each code usable once.
* Sessions use httpOnly cookies; cookie attributes set to `Secure`, `SameSite=Strict` (or `Lax` as required).
* Minimal admin UI to view and revoke MFA methods.
* Documentation and a short developer runbook included.

## Open questions & risks

* Should primary session be JWT-based or server sessions? (Recommendation: server sessions for simpler rotation and revocation.)
* What WebAuthn RP ID and origin are used for local development vs production? (Use `localhost` dev origin and set RPID to domain in production.)
* SMS/Email OTP providers cost/abuse considerations if added later.
* Browser support and UX for WebAuthn on mobile platforms — confirm target user device inventory.

## Vibe/code-generation guidance for an LLM or automation model

* Target language: TypeScript (strict).
* Prefer composable modules with small pure functions for crypto and validation.
* Use typed DTOs for API requests/responses.
* Provide test mocks for WebAuthn flows; make test helpers to simulate client responses.
* Implement extensive runtime checks and return structured error objects with machine-readable codes (e.g., `ERR_MFA_INVALID_CODE`).
* Keep UI flows resilient: show clear states for `preauth expired`, `device not recognized`, `recovery required`.
* Make setup flows idempotent and safe: re-enrolling should not orphan old credentials without explicit user action.

## Deliverables for handoff

* `src` with implementation and tests.
* `README.md` with run/local dev commands, required env vars.
* `AUTH_README.md` describing flows, threat model, and maintenance tasks.
* `SECURITY_RUNBOOK.md` for incident response: how to revoke all MFA, regenerate keys, rotate cookies.
* Postman or OpenAPI spec for all auth endpoints (optional but recommended).

## Acceptance checklist for merger (human)

* All tests pass in CI.
* Code review confirms no secrets or plaintext TOTP stored.
* End-to-end validation performed by reviewer.
* Documentation reviewed and developer runbook validated.

## Notes for future expansion

* Add adaptive auth (risk-based) and trusted-device UX after MVP.
* Add SMS/Email fallback providers with explicit policy and rate limits.
* Add audit log export for compliance.

---

This plan is intentionally prescriptive where security matters and intentionally conservative where operational complexity can be deferred. Follow the data model and API contracts as the canonical blueprint; keep cryptographic secrets encrypted and out of logs; prefer server sessions for easier revocation and safer UX.
