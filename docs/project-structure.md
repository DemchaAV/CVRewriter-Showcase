# Project structure

Folder map of the real project, with a one-line purpose for each significant entry. This is what the source tree looks like — even though most of it is not in this public repo.

## Top level

```
CVRewriter/
├── backend/                  Spring Boot application (Java 21)
├── frontend/                 React 19 + Vite + TypeScript SPA
├── scripts/                  PowerShell ops scripts (DB backup, sync)
├── docker-compose.yml        Base Compose stack
├── docker-compose.dev.yml    Overlay: dev (ports 3002/8082/3309)
├── docker-compose.prod.yml   Overlay: prod (ports 3000/8080/3307)
├── docker-compose.test.yml   Overlay: test (ports 3001/8081/3308)
├── .env.<env>.example        Example env files per environment
├── run.bat                   Compose lifecycle wrapper
└── stop.bat                  Compose teardown wrapper
```

## Backend

```
backend/
├── pom.xml
├── prompts/                       Externally-mountable AI prompts
└── src/
    ├── main/
    │   ├── java/com/cvrewriter/
    │   │   ├── CvRewriterApplication.java
    │   │   ├── common/            Cross-cutting: encryption, errors, filters, IO
    │   │   └── feature/
    │   │       ├── admin/         Admin endpoints, AI settings, prompts, insights
    │   │       ├── ai/            AiService<O> + provider impls + runtime config
    │   │       ├── auth/          JWT, registration, email verification
    │   │       ├── billing/       Stripe checkout, portal, webhooks, entitlements
    │   │       ├── browsing/      Per-user Playwright BrowserContext pool
    │   │       ├── integration/   External account connections (LinkedIn etc.)
    │   │       ├── jobprofile/    Multiple CV variants per user
    │   │       ├── pdf_render/    PDF templates + tier gating + GraphCompose
    │   │       ├── resume/        Resume parsing + canonical CV DTO
    │   │       ├── rewriting/     CV rewriter orchestration (AI + persistence)
    │   │       ├── scraping/      Strategy chain for vacancy scraping
    │   │       ├── sse/           SSE emitter registry
    │   │       ├── token_usage/   Per-call token / cost tracking
    │   │       ├── user/          User accounts and profiles
    │   │       └── vacancy/       Core pipeline + records + analytics
    │   └── resources/
    │       ├── application.yml             Base config
    │       ├── application-dev.yml         Dev profile overrides
    │       ├── application-prod.yml        Prod profile overrides
    │       ├── application-test.yml        Test profile overrides
    │       ├── db/migration/               Flyway V1–V28 SQL files
    │       └── rewriter_config/prompts/    Classpath prompts (fallback)
    └── test/
        └── java/com/cvrewriter/            JUnit 5 + Mockito tests, mirrors main/
```

### Feature folder shape

```
feature/<name>/
├── <Name>Controller.java          REST surface, one per feature
├── <Name>Service.java             Business logic / orchestration
├── <Name>Repository.java          Spring Data JPA
├── <Name>.java                    JPA entity
├── dto/
│   ├── <Name>Request.java
│   └── <Name>Response.java
└── <SubdomainSpecific>.java       Internal helpers
```

Cross-feature dependencies go through service interfaces; no controller calls another controller, no repository is used outside its owning feature.

## Frontend

```
frontend/
├── package.json
├── vite.config.ts
├── nginx.conf                     Production Nginx config (SPA fallback)
├── Dockerfile                     Multi-stage: node build → nginx serve
├── src/
│   ├── main.tsx
│   ├── App.tsx                    Routes, layouts, ProtectedRoute, AdminRoute
│   ├── lib/
│   │   └── api.ts                 Axios instance, interceptors, Problem normalization
│   ├── context/
│   │   ├── AuthContext.tsx        Login state, token bootstrap
│   │   ├── ProcessingQueueContext.tsx   Multi-job queue + EventSource
│   │   └── ThemeContext.tsx       Light / dark
│   ├── services/                  Typed API clients per feature
│   │   ├── authService.ts
│   │   ├── userService.ts
│   │   ├── vacancyService.ts
│   │   ├── billingService.ts
│   │   └── adminService.ts
│   ├── hooks/                     React hooks (useApiError, useDebounced, ...)
│   ├── pages/
│   │   ├── auth/                  Login, Register, VerifyEmail
│   │   ├── processing/            Landing — paste URL
│   │   ├── history/               Records list + CV editor
│   │   ├── profile/               Profile, settings, integrations
│   │   └── admin/                 Token usage analytics (admin-only)
│   ├── components/
│   │   ├── ui/                    House design system (Button, Card, Modal, Table, ...)
│   │   └── common/                Composite popovers, RatingCircle, RecordModal, etc.
│   ├── mocks/                     Mock adapter (gated by VITE_USE_MOCK)
│   └── test/                      Vitest setup, test utilities
└── e2e/                           Playwright E2E suites
```

## Scripts

```
scripts/
└── db/
    ├── Backup-ProdDatabase.ps1            Timestamped mysqldump → db_backups/
    └── Sync-ProdToEnvironment.ps1         Clone prod dump into dev or test DB
```

## Configuration files

```
.env.prod.example     Prod-like local stack: ports 3000/8080/3307, MYSQL_DATABASE=cv_rewriter
.env.dev.example      Dev stack:             ports 3002/8082/3309, MYSQL_DATABASE=cv_rewriter_dev
.env.test.example     Test/QA stack:         ports 3001/8081/3308, MYSQL_DATABASE=cv_rewriter_test
```

Each environment file owns:
- DB credentials (`MYSQL_*`, `DB_*`)
- Port mappings (`FRONTEND_PORT`, `BACKEND_PORT`, `MYSQL_PORT`)
- Spring profile (`SPRING_PROFILES_ACTIVE`)
- Secret material (`JWT_SECRET`, `ENCRYPTION_KEY`, `ADMIN_SECRET_KEY`)
- AI provider keys (`GEMINI_API`, `GPT_API`, `DEEP_SEEK_API`)
- Stripe keys (`STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRO_PRICE_ID`)
- SMTP config
- External integration credentials
- Playwright `PLAYWRIGHT_HEADLESS` toggle

The full sanitized variable surface is in [.env.example](../.env.example).

## What this showcase repo actually ships

```
CVRewriter-Showcase/
├── README.md                   Project overview, why I built it, architecture diagram
├── .env.example                Sanitized env-var surface
├── docs/
│   ├── architecture.md         System design and bounded contexts
│   ├── data-flow.md            End-to-end pipeline walk-through
│   ├── technical-decisions.md  TD-01 through TD-17, what & why
│   ├── security-and-privacy.md Security model + showcase sanitization policy
│   ├── demo-workflow.md        Product walk-through with screenshots
│   ├── project-structure.md    This file
│   └── future-improvements.md  Realistic next steps
└── 01_history.png … 07_complete.png   UI screenshots (real account, sensitive fields redacted)
```

That's the entire footprint of the public showcase. Cloning it gives you the explanation, not the implementation.
