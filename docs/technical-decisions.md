# Technical decisions

This document records the decisions that shaped CVRewriter — what was chosen, what was rejected, and why. The goal is to make the design legible to someone reviewing the project for the first time.

## TD-01 — Package-by-feature, not layered

**Decision**: backend code is organized by feature (`feature/auth/`, `feature/vacancy/`, `feature/billing/` …) instead of by layer (`controllers/`, `services/`, `repositories/`).

**Why**: features tend to be modified together. A bug in vacancy processing touches one folder. A change to billing touches one folder. With layered packages, every change spans `controllers/`, `services/`, `dto/`, and `repositories/` — and folder structure no longer matches the way the system actually changes.

**Trade-offs**: cross-cutting things (encryption, error handling) need a deliberate home — they live in `common/`. Some new joiners are used to layered packages and need a quick orientation. Both are fine costs to pay.

## TD-02 — Provider-agnostic AI behind one interface

**Decision**: define `AiService<O>` with `isConfigured()` and `generateObjectWithMetadata(prompt)`; implement it for Gemini, GPT, and DeepSeek; resolve the active provider at runtime through `AiServiceProvider` and `AiRuntimeSettingsService`.

**Why**: pricing, rate limits, and quality differ across providers. Locking the codebase to one provider means re-plumbing the day pricing changes. Hot-swap also matters for outages — if Gemini is down, an admin can flip to GPT without a redeploy.

**Trade-offs**: each provider has slightly different structured-output semantics (Gemini uses native JSON-mode + schema; OpenAI uses `response_format: { type: "json_object" }`; DeepSeek mirrors OpenAI). The interface has to live at the lowest common denominator, and the per-provider implementations carry the differences. This is the right place for that complexity.

**Rejected alternatives**:
- LangChain / LangChain4j — added a heavy framework dependency for what is, in practice, three HTTP calls and a JSON schema. The abstraction layer hid behavior we wanted to see directly.
- One service per provider with controllers branching at the top level — would have duplicated token tracking and prompt construction across providers.

## TD-03 — Structured output via JSON Schema, not free-text parsing

**Decision**: every AI call asks the provider to return JSON conforming to a schema generated from `AiCvCraftedDTO`. The library is `victools/jsonschema-generator`, the same DTO is used for the AI response, DB persistence, the React editor, and PDF rendering.

**Why**: one source of truth eliminates a category of bugs. Regex-parsing fenced code blocks works until the LLM decides to format slightly differently. Structured-output mode is supported by all three providers and dramatically reduces post-processing.

**Trade-offs**: the DTO is large, and any change ripples to the editor and PDF. Acceptable: those things have to be in sync anyway.

A regex-based fallback (`extractCvJsonBlock`) is retained for the edge case where the provider returns prose-wrapped JSON. It's a belt-and-braces measure, not a primary path.

## TD-04 — Two-tier scraping with sanitizer-driven fallback

**Decision**: try cheap HTTP+JSON-LD first, fall back to authenticated Playwright only when the result fails a usability check.

**Why**: scraping is expensive in two dimensions — Playwright costs ~3 seconds and a browser context per call; bot-walls cost a captcha challenge and a flagged session if you misuse the cheap path. The two-tier flow optimizes for the common case (job posting publicly indexed with JSON-LD) without giving up on the hard case.

**Trade-offs**: a strategy that returns a low-quality but technically valid result can mislead the sanitizer. `VacancyDescriptionSanitizer.hasUsableDescription` is the linchpin — it has been tuned to reject things like "Please verify you are human", short stub pages, and Cloudflare interstitials.

**Rejected alternatives**:
- Always Playwright — too slow, too easy to trip bot detection.
- Always HTTP — misses LinkedIn for any non-public posting.
- ML-based content extraction — overkill for three known sites with stable structures.

## TD-05 — Per-user `BrowserContext` pool

**Decision**: `BrowserService` keeps one `BrowserContext` per user, with a TTL touch on every use. When a context expires, it's closed and recreated on next access.

**Why**: spinning a new browser per scrape is wasteful (200–500 ms overhead). Keeping one global context means sessions, cookies, and storage state bleed across users. Per-user contexts keep isolation and amortize startup cost.

**Trade-offs**: memory grows with active users. The TTL keeps it bounded; in practice the working set is tiny for this product's traffic shape.

## TD-06 — SSE with a short-lived, scope-bound JWT

**Decision**: SSE stream endpoint is `permitAll` at the security filter level, but every request validates an `sseToken` query parameter that is a separate JWT with `type=sse, jobId=<uuid>, sub=<userEmail>`, 120-second TTL.

**Why**: browsers cannot set the `Authorization` header on `EventSource`. The two textbook workarounds are (a) cookie-based session auth — which forces a session model the rest of the API doesn't need, or (b) query-string token — which we use. Making the token short-lived and bound to a specific jobId limits the blast radius of a leaked token in browser history / proxy logs.

**Trade-offs**: there's a separate issue endpoint (`POST /vacancies/{id}/events-token`) that frontend must call before opening the stream — one extra round-trip. Worth it.

**Rejected alternatives**:
- Cookies + CSRF — the rest of the API is stateless JWT; adding session cookies just for SSE created a confusing dual auth model.
- Long-lived JWT in URL — too dangerous in browser history / referer headers / load-balancer logs.

## TD-07 — Audit-grade history on three orthogonal axes

**Decision**: pipeline status, application outcome, and template choice each get their own append-only history table.

**Why**: these axes have genuinely different lifecycles. A record can be `GENERATED` (pipeline) and `INTERVIEW` (outcome) at the same time. Template choice changes when the user picks a different template before downloading — it's not a status change. Conflating them was the first design, and V17 explicitly split them out after the modeling pain became clear.

**Trade-offs**: three tables instead of one. Worth it — querying time-in-status, status-transition matrices, and template-usage analytics is now straightforward joins on focused tables.

## TD-08 — Three-environment Docker isolation by project name

**Decision**: prod, dev, and test stacks all use the same `docker-compose.yml` but with different overlay files (`docker-compose.prod.yml`, `.dev.yml`, `.test.yml`), different `.env.*` files, different ports, different MySQL volumes, and different Compose project names (`cvrewriter-prod`, `-dev`, `-test`).

**Why**: this is the cheapest possible setup that makes prod data accidentally-deletable only by extreme malice, not by typos. `docker compose down` is scoped by project name; there is no command that touches all three accidentally.

**Trade-offs**: three sets of env files to keep in sync. The example files (`.env.<env>.example`) make this manageable.

## TD-09 — AES-GCM at rest with a backward-compatible read path

**Decision**: `EncryptionConverter` is a JPA `AttributeConverter` that writes AES-GCM (authenticated, random IV per write) and reads either AES-GCM or the legacy AES-ECB format used earlier in the project.

**Why**: ECB was a bad default. Moving to GCM is correct. But the prod DB contains rows written under ECB, and migrating them lazily (on next read/write) is simpler and safer than a one-shot rewrite migration.

**Trade-offs**: the read path is slightly more complex (try-GCM, fall back to ECB-decrypt). Acceptable. The fallback path is dead code as soon as the table has no legacy rows; it will be removed once that's confirmed.

## TD-10 — RFC 7807 problem details for every error

**Decision**: every error response is a `application/problem+json` body produced by `GlobalExceptionHandler` + `ApiError`. Frontend normalizes it into a typed `ApiError` in the Axios response interceptor.

**Why**: one stable error schema across the API. The frontend doesn't need a different parser per endpoint. `traceId` is propagated so support can grep logs by the value the user can copy from the UI.

**Trade-offs**: a few error responses had ad-hoc shapes (e.g. validation errors). Those were normalized to the Problem schema with `errors[]` for field-level details.

## TD-11 — Hot-reloadable prompts via external mount

**Decision**: prompts live in `backend/prompts/`. In Docker, that directory is mounted to `/app/prompts_override/`. `PromptService` checks the external path first, then falls back to classpath. `POST /admin/reload-prompts` rereads them at runtime.

**Why**: prompt iteration is the highest-frequency change in an LLM product. Treating prompts like code (rebuild + redeploy on every tweak) makes iteration painful. Treating them like config makes it cheap.

**Trade-offs**: prompts are not version-controlled by Git in the running container — they're version-controlled in the repo plus mounted at runtime. Admin reload is gated by the admin key + role check, so this isn't a security issue.

## TD-12 — Stripe via `RestClient`, not the official SDK

**Decision**: Stripe calls are made with `RestClient` (Spring 6+ web client). Webhook signature verification is implemented by hand against the documented HMAC-SHA-256 scheme.

**Why**: the official Stripe Java SDK is large, mutable, and ships its own update cadence. For the small handful of operations we need (create checkout session, create portal session, parse webhook events, look up subscription), hand-rolling is simpler, has fewer transitive dependencies, and makes every HTTP call visible at the source level.

**Trade-offs**: if Stripe adds new API surface we need, we add new methods rather than upgrading an SDK. Acceptable for this product's scope.

## TD-13 — `BINARY(16)` UUID primary keys

**Decision**: all primary keys are `BINARY(16)` UUIDs, generated server-side.

**Why**: UUIDs in URLs are unguessable; sequential integers leak the size of the user base and enable enumeration attacks. `BINARY(16)` is half the size of `CHAR(36)` and indexed as a single value rather than a string compare.

**Trade-offs**: harder to copy-paste in psql. Worth it.

## TD-14 — Append-only `token_usage` for AI cost tracking

**Decision**: every AI call gets one row in `token_usage` with `user_id`, `record_id`, `provider`, `model`, `input_tokens`, `output_tokens`, `created_at`. Pricing is applied at read time by `AdminTokenUsageService` using the current `TokenPricingProperties`.

**Why**: prices change. Storing a `cost_usd` column at write time creates stale data. Storing the raw tokens + applying current prices at read time means historical reports always reflect current pricing — and we can also run "what if" reports easily.

**Trade-offs**: read-time pricing is slightly more compute, but the volume is small (admin-only views).

## TD-15 — GraphCompose for PDF rendering

**Decision**: PDFs are rendered by GraphCompose, a canonical document engine I wrote separately and published as a Maven dependency.

**Why**: iText is licensed in a way that's awkward for product work. Apache PDFBox is great but low-level (you draw glyphs at coordinates). I wanted a declarative document model — `Section`, `Paragraph`, `List`, `Table`, with automatic pagination — and the only one I trusted to evolve with the project was the one I owned.

**Trade-offs**: it's a dependency I have to maintain. That's also a feature — it gets attention.

## TD-16 — Single-page CVs only (for now)

**Decision**: templates render single-page CVs. Pagination support exists in GraphCompose but is intentionally not exercised in the CV templates.

**Why**: most recruiters reject CVs longer than one page for non-executive roles. Building for two-pagers first would have meant designing for the wrong norm.

**Trade-offs**: senior candidates with long histories sometimes need overflow. Marked as a future improvement (see [future-improvements.md](future-improvements.md)).

## TD-17 — No CI/CD yet

**Decision**: the project ships without GitHub Actions / Jenkins / GitLab CI. Builds and tests are run locally via Maven and npm; Docker images are built locally.

**Why**: solo project, one deploy target, no team to coordinate around merge gates. Adding CI/CD without a real need creates ceremony without value.

**Trade-offs**: the first thing to add when more people contribute. This is the largest known gap in the operational story (see [future-improvements.md](future-improvements.md)).
