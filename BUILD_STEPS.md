# Build Steps - MFA Next.js MVP

## Phase 1: Project Setup
- [x] 1.1 Initialize Next.js project with TypeScript
- [x] 1.2 Set up Prisma with SQLite schema
- [ ] 1.3 Install auth dependencies (argon2, speakeasy, simplewebauthn)
- [ ] 1.4 Configure environment variables
- [ ] 1.5 Set up project structure (lib/auth, api routes, components)

## Phase 2: Database & Models
- [ ] 2.1 Create Prisma schema (users, mfa_methods, totp_secrets, webauthn_credentials, recovery_codes, sessions)
- [ ] 2.2 Run Prisma migrations
- [ ] 2.3 Create database utilities and Prisma client

## Phase 3: Authentication Core
- [ ] 3.1 Password hashing utilities (argon2)
- [ ] 3.2 Session management (create, validate, delete)
- [ ] 3.3 Sign up API endpoint
- [ ] 3.4 Sign in API endpoint (with preauth session for MFA)

## Phase 4: TOTP Implementation
- [ ] 4.1 TOTP generation/verification utilities
- [ ] 4.2 TOTP setup API (generate secret, QR)
- [ ] 4.3 TOTP verify API
- [ ] 4.4 TOTP UI components

## Phase 5: WebAuthn Implementation
- [ ] 5.1 WebAuthn server utilities (@simplewebauthn/server)
- [ ] 5.2 WebAuthn registration API (challenge + verify)
- [ ] 5.3 WebAuthn authentication API (challenge + verify)
- [ ] 5.4 WebAuthn UI components

## Phase 6: Recovery Codes
- [ ] 6.1 Recovery code generation (hashed storage)
- [ ] 6.2 Recovery code consumption (single-use)
- [ ] 6.3 Recovery API endpoints
- [ ] 6.4 Recovery UI

## Phase 7: UI Pages
- [ ] 7.1 Sign up page
- [ ] 7.2 Sign in page
- [ ] 7.3 MFA challenge page
- [ ] 7.4 Account security page
- [ ] 7.5 Admin users page

## Phase 8: Admin Features
- [ ] 8.1 Admin API endpoints (list users, revoke devices)
- [ ] 8.2 Admin UI

## Phase 9: Testing & Security
- [ ] 9.1 Unit tests for core auth logic
- [ ] 9.2 Integration tests for API flows
- [ ] 9.3 Security logging
- [ ] 9.4 Rate limiting

## Phase 10: Documentation
- [ ] 10.1 README.md
- [ ] 10.2 AUTH_README.md
- [ ] 10.3 SECURITY_RUNBOOK.md
