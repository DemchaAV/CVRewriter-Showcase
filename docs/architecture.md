# Architecture

This document describes the system at a level high enough to understand how the pieces fit, and concrete enough that another engineer could rebuild it.

## Architectural style

- **Backend**: Spring Boot 4 monolith, organized **package-by-feature** (vertical slicing). Each feature owns its controller, service(s), entities, repositories, and DTOs. There is no shared `service/` or `controller/` mega-package.
- **Frontend**: React 19 + Vite + TypeScript SPA, served by Nginx in production. Server state lives in TanStack Query; local UI state in a small set of focused React contexts.
- **Database**: MySQL 8 with Flyway-managed schema (V1 → V28). Tests use H2 with the same schema.
- **Deployment**: Docker Compose, three named environments (`cvrewriter-prod`, `cvrewriter-dev`, `cvrewriter-test`) on disjoint ports and disjoint MySQL volumes.

The monolith choice is deliberate: this is one product, one team, one deploy unit. Service boundaries inside the monolith are real (bounded contexts), but operational complexity is held at one process.

## Bounded contexts (backend features)

```
backend/src/main/java/com/cvrewriter/
├── feature/
│   ├── auth/             JWT auth, registration, email verification
│   ├── user/             User accounts, profiles, account management
│   ├── jobprofile/       Multiple CV variants per user
│   ├── vacancy/          The core pipeline + records + analytics
│   ├── rewriting/        CV-rewrite orchestration (calls AI + persists)
│   ├── ai/               Provider-agnostic AI surface (Gemini/GPT/DeepSeek)
│   ├── scraping/         Strategy chain for job-description scraping
│   ├── browsing/         Per-user Playwright BrowserContext pool
│   ├── integration/      External-provider accounts (LinkedIn etc.)
│   ├── pdf_render/       PDF generation, template registry, tier gating
│   ├── billing/          Stripe subscriptions, webhooks, entitlements
│   ├── sse/              SSE emitter registry for real-time updates
│   ├── token_usage/      Per-call token / cost tracking
│   ├── admin/            Privileged ops, prompt hot-reload, AI settings
│   ├── resume/           CV DTO model + resume parsing
│   └── common/           Cross-cutting: encryption, errors, filters
```

Each feature folder has the shape:

```
feature/<name>/
├── <Name>Controller.java
├── <Name>Service.java         (orchestration / business logic)
├── <Name>Repository.java      (Spring Data JPA)
├── <Name>.java                (JPA entity)
└── dto/
    ├── <Name>Request.java
    └── <Name>Response.java
```

This makes it easy to reason about a feature in isolation, easy to delete a feature, and easy to onboard — you read one folder, not 12.

## Request lifecycle

A typical authenticated request flows:

```
HTTP → TraceIdFilter → JwtAuthenticationFilter → DispatcherServlet
     → @PreAuthorize check → Controller → Service → Repository → MySQL
     → Response DTO → JSON
```

Errors are converted to **RFC 7807 Problem Details** by `GlobalExceptionHandler` + `ApiError`, so every error path has a stable schema (`type`, `title`, `status`, `detail`, `traceId`, optional `errors[]`).

## The vacancy pipeline

This is the heart of the product. One user click triggers:

1. **`POST /vacancies/process`** — `VacancyController` accepts URL + optional instructions, validates ownership of the user's active job profile, creates a `VacancyRecord` with status `QUEUED`, returns a `jobId`. Processing is dispatched **async**.
2. **`POST /vacancies/{id}/events-token`** — frontend requests a short-lived (120 s) SSE-scoped JWT bound to this jobId.
3. **`GET /vacancies/{id}/events?sseToken=...`** — opens the SSE stream. Server validates token type, jobId match, and ownership.
4. **`VacancyProcessingOrchestrator`** runs the actual pipeline on a worker thread, emitting progress events at each stage:
   - `connecting` → `scraping` → `parsing` → `preparing` → `generating` → `saving` → `completed`
5. Inside each stage, the orchestrator calls into the appropriate feature service. On failure, the record is marked `FAILED`, an `error` event is emitted, and the SSE emitter is closed.

Frontend has a polling fallback: if the `EventSource` disconnects, `ProcessingQueueContext` polls `GET /vacancies/{id}/status` every 3 seconds until a terminal state.

See [data-flow.md](data-flow.md) for the full sequence diagram and event payloads.

## AI orchestration

The `ai` feature isolates the provider differences behind one interface:

```java
public interface AiService<O> {
    boolean isConfigured();
    AiGenerationResult<O> generateObjectWithMetadata(String prompt);
    // ...
}
```

- `GeminiServiceImpl`, `GptService`, `DeepSeekService` all implement it.
- `AiServiceProvider.getActiveService()` returns whichever provider is currently configured at runtime — controlled by `AiRuntimeSettingsService`, bootstrapped from `AI_PROVIDER` env var, mutable via `PUT /admin/ai-settings`.
- All providers use **structured output**: the request includes a JSON schema generated from `AiCvCraftedDTO` via `victools/jsonschema-generator`. Gemini uses native JSON-mode + schema; GPT and DeepSeek use response-format JSON.
- Every call returns an `AiGenerationResult` carrying the parsed object plus `provider`, `model`, `inputTokens`, `outputTokens`. `TokenUsageService` writes one `token_usage` row per call, so per-user / per-record / per-provider cost is fully tracked.

Prompts live in `backend/prompts/` and are externally mounted in Docker. `PromptService` checks the external path first and falls back to classpath, with hot reload via `POST /admin/reload-prompts` (admin-only).

## Scraping

Two strategies are wired in priority order via `ScraperFactory`:

1. **`HttpNoLoginScrapingStrategy`** — Java's `HttpClient` + Jsoup. Parses JSON-LD `JobPosting` schema first, falls back to site-specific CSS selectors (LinkedIn, Indeed, Glassdoor), then OpenGraph/Twitter meta. Detects bot-walls via a static `BLOCKED_INDICATORS` list (e.g. `"captcha"`, `"cloudflare"`).
2. **`BrowserAuthScrapingStrategy`** — delegates to per-site scrapers (`LinkedInScraper`, `IndeedScraper`, `GlassdoorScraper`) backed by Playwright + an authenticated `BrowserContext` retrieved from `BrowserService`.

`ScraperFactory.scrape()` walks `scrapingProperties.getMethodOrder()` (configurable via env), filters by `enabledMethods`, and falls through if the previous strategy returned data that `VacancyDescriptionSanitizer.hasUsableDescription` rejects.

The browser pool (`BrowserService`) keeps per-user `BrowserContext` instances with TTL touch semantics — avoids spinning a new browser per scrape while isolating session state between users.

## PDF rendering

PDFs are produced by **GraphCompose**, my own canonical document engine that compiles a tree of typed document nodes into a PDFBox `PDDocument`. The CVRewriter `pdf_render` feature is a thin layer on top:

- `CvTemplateRegistry` auto-discovers all `CvTemplate` Spring beans on startup, indexes them by ID and aliases.
- Six CV templates are wired (`Template_CV1` … `Template_CV6`) plus a cover-letter template.
- Each template carries a `TemplateTier` (FREE / PRO). `CvTemplateAccessService` consults `EntitlementService` before serving a PRO template.
- PDFs are streamed via `StreamingResponseBody` — never written to disk. Filenames are sanitized (`{Owner Name} - {Company} - CV.pdf`) to remove reserved characters.

## Billing

Stripe integration is hand-rolled on `RestClient` rather than the official SDK — simpler dependency surface, full visibility into the HTTP calls.

- `BillingService` issues Checkout Session and Customer Portal URLs.
- `StripeWebhookVerifier` validates the `Stripe-Signature` header against the webhook secret.
- Webhook handler processes `checkout.session.completed` and `customer.subscription.{created,updated,deleted}`, resolving the owning user by `metadata.userId` → `stripeSubscriptionId` → `stripeCustomerId` (graceful chain in case Stripe sends only one identifier).
- `EntitlementService.hasPremiumAccess(userId)` is the single gate: `PlanTier.PRO` AND `SubscriptionStatus.grantsPremiumAccess()`. Everything else is FREE.
- Currently only premium CV templates are gated. Other features are free for all users.

## Database & migrations

28 Flyway migrations, MySQL 8 InnoDB, `utf8mb4_unicode_ci`, all primary keys are `BINARY(16)` UUIDs.

Themes (rough grouping):

| Migrations | Theme |
|---|---|
| V1–V4 | Identity, profile, external accounts |
| V5–V11 | Vacancy enrichment (CV content, JD body, rating, brief, external job IDs) |
| V12–V13 | Token usage tracking + provider column |
| V14, V22 | Cover letter + per-profile additional instructions |
| V15 | Drop `username` (email becomes sole login identifier) |
| V16–V19 | Pipeline-status history, outcome history split, backfill |
| V20 | Job profiles as first-class entities (multi-CV per user) |
| V21 | Email verification tokens |
| V23–V25 | History enrichments (job name, city, normalization) |
| V26 | User subscriptions (Stripe state) |
| V27–V28 | Template tracking on records + record_template_history |

Three append-only audit tables stand out:

- **`record_status_history`** — every pipeline-status change (QUEUED → GENERATED → APPLIED → …) with `changed_at`, `source` (USER / SYSTEM / API).
- **`record_outcome_history`** — application outcome separated from pipeline status in V17 because they have different lifecycles.
- **`record_template_history`** — every template change for a record, so the team can tell which template generated which PDF.

## Frontend architecture

```
frontend/src/
├── App.tsx                       Routes + ProtectedRoute / AdminRoute
├── lib/api.ts                    Axios instance, interceptors, Problem-detail normalization
├── context/
│   ├── AuthContext               Login state, token bootstrap
│   ├── ProcessingQueueContext    Multi-job upload queue + per-item EventSource
│   └── ThemeContext              Light / dark
├── services/
│   ├── authService
│   ├── userService
│   ├── vacancyService
│   ├── billingService
│   └── adminService
├── pages/
│   ├── auth/         login, register, verify-email
│   ├── processing/   landing — paste URL
│   ├── history/      records list + editor
│   ├── profile/      profile, settings, integrations
│   └── admin/        token usage analytics (admin-only)
└── components/
    ├── ui/           house design system (Button, Card, Input, Modal, Table)
    └── common/       composite popovers, RatingCircle, MarkdownTextarea, RecordModal, ProcessingPopup, TemplatePicker
```

Conventions:
- **Server state** lives in TanStack Query. Cache keys are stable (`['records']`, `['record', id]`).
- **Forms** use React Hook Form + Zod; the same Zod schema feeds both client validation and TypeScript types.
- **Errors** are normalized in the Axios response interceptor (RFC 7807 → typed `ApiError`), then surfaced in UI via a small `useApiError` hook.
- **Mocks** are swappable behind `VITE_USE_MOCK=true` for offline development and Playwright tests.

## Runtime environments

Three Docker Compose stacks share the same base `docker-compose.yml` and overlay one of:

| Env | Frontend | Backend | MySQL | DB name | Project name |
|---|---:|---:|---:|---|---|
| `prod` | 3000 | 8080 | 3307 | `cv_rewriter` | `cvrewriter-prod` |
| `test` | 3001 | 8081 | 3308 | `cv_rewriter_test` | `cvrewriter-test` |
| `dev` | 3002 | 8082 | 3309 | `cv_rewriter_dev` | `cvrewriter-dev` |

Each environment has its own `.env.<env>` file (never committed), its own MySQL volume, and its own backend data directory (`./backend/data-<env>`). Compose project names ensure `docker compose down` in one environment can never touch another.

Two PowerShell scripts manage data movement:

- `scripts/db/Backup-ProdDatabase.ps1` — timestamped `mysqldump` to `db_backups/prod-cv_rewriter-<ts>.sql`.
- `scripts/db/Sync-ProdToEnvironment.ps1 -TargetEnvironment dev|test [-RefreshBackup]` — clones a prod dump into the target environment's DB.

Data flow is **prod → dev/test, never the reverse**. This is enforced by convention (no script to sync the other way) and by the operational rule that `dev`/`test` databases are disposable.

## What's missing here on purpose

This document deliberately doesn't include:
- Real prompt files or their structure
- The full controller / service / entity code
- Real Stripe price IDs or webhook secrets
- Real production URLs or DB hostnames
- The full Docker Compose files (the structure is described, the values are not)

If you want to discuss design choices or the implementation in more depth, the GitHub profile link in the [README](../README.md) is the way.
