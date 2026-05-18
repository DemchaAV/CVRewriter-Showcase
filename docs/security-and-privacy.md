# Security and privacy

This document covers two things:

1. The security model of CVRewriter itself — how authentication, authorization, encryption, and secrets management work in the real system.
2. What this **showcase repository** intentionally does not contain, and why.

## Part 1 — Security model

### Authentication

- **Email + password**, with the email verified before the account can be used. Passwords are hashed with **BCrypt** (Spring Security default, work factor 10).
- **JWT** access tokens, HS256, secret loaded from `JWT_SECRET` (base64-decoded). 24-hour TTL by default.
- **Refresh tokens** are also JWTs with a 7-day TTL. No server-side revocation list — refresh is stateless re-issuance. Trade-off: short-lived access tokens limit the blast radius of a compromised one.
- **Email verification tokens** are 32-byte random values, **stored as SHA-256 hashes** in `email_verification_tokens`. The raw token is sent in the email link only and never persisted. 24-hour TTL.
- **SSE tokens** are a separate JWT type (`type=sse, jobId=<uuid>`), 120-second TTL. See [data-flow.md](data-flow.md) and [technical-decisions.md](technical-decisions.md#td-06--sse-with-a-short-lived-scope-bound-jwt).

### Authorization

- All `/auth/**`, `/billing/webhooks/stripe`, and SSE endpoints (`GET /vacancies/{id}/events`) are `permitAll`. Everything else requires a valid JWT.
- `JwtAuthenticationFilter` populates the `SecurityContext` with a `UserDetails` carrying `ROLE_USER` or `ROLE_ADMIN`.
- Admin endpoints (`/admin/**`) require either:
  - A user with `ROLE_ADMIN` via `@PreAuthorize("hasRole('ADMIN')")`, or
  - A valid `X-Admin-Key` header equal to `ADMIN_SECRET_KEY` (via `AdminKeyAuthenticationFilter`). This is the break-glass path for headless admin tooling.
- Ownership checks are performed per-resource. `RecordOwnershipService.assertRecordOwnership(currentUser, recordId)` is called from every record-scoped endpoint, including the SSE handler. Owning a record's `id` is not enough — the caller must be the row's `user_id`.

### Encryption at rest

- `EncryptionConverter` is a JPA `AttributeConverter` applied to columns holding sensitive blobs (e.g. external-account session data).
- **AES/GCM/NoPadding** with a 12-byte random IV per write, 16-byte auth tag. The 256-bit key is derived from `ENCRYPTION_KEY` (env var) via SHA-256.
- **Backward-compatible read path**: rows written under the legacy AES/ECB scheme are still decryptable. New writes always use GCM. See [TD-09](technical-decisions.md#td-09--aes-gcm-at-rest-with-a-backward-compatible-read-path).

### Transport security

- Production deploys behind HTTPS via the reverse proxy / hosting layer. The Spring Boot app itself runs HTTP behind that proxy.
- **CORS** allowed origins are explicit and loaded from `CORS_ALLOWED_ORIGINS`. `allowCredentials=true`. Methods limited to GET / POST / PUT / DELETE / OPTIONS. No wildcards.
- **CSRF** is disabled because the API is stateless JWT (CSRF protects cookie-based sessions; tokens carried in `Authorization` are not vulnerable in the same way). The single exception path (Stripe webhook) is permitted explicitly and verified by signature.

### Webhook security (Stripe)

- `StripeWebhookVerifier` validates the `Stripe-Signature` header against `STRIPE_WEBHOOK_SECRET` using HMAC-SHA-256 with a 5-minute timestamp tolerance.
- Webhook handlers are **idempotent on event ID** — `lastWebhookEventId` is stored on `user_subscriptions`. Re-delivery of the same event is a no-op.
- Webhooks resolve the owning user via a graceful fallback chain: `metadata.userId` → `stripeSubscriptionId` → `stripeCustomerId`. This handles edge cases where Stripe sends only one identifier.

### Errors

- Every error goes through `GlobalExceptionHandler` and is returned as RFC 7807 `application/problem+json`. The body always includes a `traceId` (from `TraceIdFilter`) so support can grep logs by the value the user sees.
- Stack traces are never returned to clients. Internal error messages are sanitized.

### Things deliberately not in scope (yet)

- **Rate limiting** is not yet implemented (no Bucket4j, no Resilience4j filters). Mitigated by the fact that the heavy endpoints (AI generation, scraping) are authenticated and have measurable per-call cost. Documented as a known gap in [future-improvements.md](future-improvements.md).
- **Server-side token revocation** is not implemented. Short-lived access tokens + the ability to rotate `JWT_SECRET` are the current mitigations.
- **OAuth login** (Google / GitHub / LinkedIn) is not implemented. Email + password only.

## Part 2 — What this showcase repository does NOT contain

This is a **showcase repository**. The production codebase is private. The following are intentionally excluded from this public repo:

### Secrets and credentials

- No real `.env.*` files. Only `.env.example` with placeholder values like `your_mysql_password`, `replace_with_long_random_secret`.
- No real `JWT_SECRET`, `ENCRYPTION_KEY`, or `ADMIN_SECRET_KEY`.
- No real Stripe `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, or `STRIPE_PRO_PRICE_ID`.
- No real `GEMINI_API`, `GPT_API`, or `DEEP_SEEK_API` keys.
- No real SMTP credentials or third-party API tokens.
- No real LinkedIn / external-provider account credentials.

### Source code

- The full Spring Boot source tree (controllers, services, entities, repositories, tests) is private.
- The full React frontend source (components, pages, hooks, services, tests) is private.
- This repo describes the design and shape of those things, with type signatures and snippets where they help understanding — never copy-pasted blocks of real implementation.

### AI prompts

- The actual prompt templates (in `backend/prompts/*` and `backend/src/main/resources/rewriter_config/prompts/*`) are project IP and are **not** included. Their existence and role are documented; their contents are not.
- The structured-output schema (`AiCvCraftedDTO`) shape is described conceptually; the full Java class is not exposed.

### Database

- No real DB dumps. The `db_backups/` folder from the source repo is excluded entirely.
- No real production schema beyond what Flyway migration metadata is described.
- No real test user data. Only the `test-users.example.json` *contract* is documented, not the actual values.

### Infrastructure

- No real Docker Compose files with internal port mappings, network names, or volume layouts. The principles (per-environment isolation, project naming, port table) are described in [architecture.md](architecture.md).
- No real Nginx configuration, certificate paths, or upstream addresses.
- No real DB backup / sync scripts (PowerShell). Their role is described.

### Internal artifacts

- No internal task boards, planning notes, or roadmap details.
- No private development tooling configuration.
- No internal automation rules or workflow scaffolding.

### Personally identifying data — what's in the screenshots

The seven UI screenshots in this repo were captured from the running app against a real account, with **only the most-sensitive fields redacted at capture time**. They are not synthetic. Specifically:

- **Redacted in the screenshots**: email address, full phone number, full street address.
- **Visible in the screenshots** (and therefore public on GitHub already):
  - The author's name (also public on the GitHub profile and LinkedIn).
  - The author's GitHub and LinkedIn URLs (both already public).
  - CV content — skills, education, summary phrasing — that overlaps with the public LinkedIn profile.
  - Publicly available job-posting metadata (company names, LinkedIn job IDs, one published salary figure from a Siemens listing). These are public job postings, but their presence here implies the author has interacted with them as a job seeker.

The screenshots are kept because they show the real UI working with real data — which is more credible than mock screenshots. They are not, however, "synthetic." If that distinction matters for a particular audience, the recommendation is to retake them against a throwaway profile with the project's mock adapter enabled (`VITE_USE_MOCK=true`).

No production user data other than the author's own is included anywhere. No third-party PII appears in any document or screenshot.

## How to think about this repository

This is a curated explanation of an engineering project, not a deployable artifact. Cloning it gives you:

- The README and `docs/` folder explaining what the system does and how.
- Screenshots of the UI as a user sees it.
- A sanitized `.env.example` showing the configuration surface.

Cloning it does **not** give you:

- A working build.
- The implementation behind the design.
- Anything that could be re-used to access real systems.

If you want to discuss the implementation in more depth, the GitHub profile link in the README is the way to reach me.
