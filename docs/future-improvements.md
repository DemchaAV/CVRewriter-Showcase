# Future improvements

A realistic list of things I'd add or change next. Not a roadmap, not a marketing wishlist — only things I've actually thought about, ordered roughly by value-to-effort.

## High value, moderate effort

### CI/CD pipeline (GitHub Actions)

Currently the project has no CI. Building, testing, and image publishing are all local. The first thing I'd add:

- Backend: Maven test + build on push, on PR.
- Frontend: `npm test` + `npm run build` on push, on PR.
- E2E: Playwright suite on PR (with mocked backend; full stack tests on a nightly cron).
- Image build + push to a registry on main.
- Migration verification: spin up MySQL, run all Flyway migrations, fail if anything is wrong.

The reason this hasn't been done yet is real (single-developer project, no merge gates needed). The reason to do it is also real: the day a contributor joins, the lack of CI is the biggest friction point.

### Rate limiting on the heavy endpoints

Currently the only protection on `POST /vacancies/process` is "authenticated". Authenticated users could trivially abuse the AI generation endpoint. I'd add per-user-per-hour limits via Bucket4j (in-memory for now, Redis-backed later):

- `POST /vacancies/process`: e.g. 20/hour for FREE, 200/hour for PRO.
- `POST /vacancies/{id}/regenerate`: same bucket.
- `POST /resumes/parse`: e.g. 30/day (CV parsing is also AI-backed).

The pricing tiers are a natural surface for differentiated limits.

### Server-side token revocation

Currently logging out is a client-side gesture — the JWT remains valid until expiry. For a password change or "log out everywhere" feature to be real, I need a server-side revocation mechanism. Options I'd evaluate:

- Redis-backed denylist keyed by token jti, evicted at TTL.
- Per-user "token epoch" stored on `users`; tokens issued before epoch are rejected. Cheaper to operate, harder to grant fine-grained revocation.

I'd lean toward the epoch approach for the simple cases and add denylist only if there's a real need.

### OAuth login (Google + LinkedIn)

Email + password is the only login path today. Google OAuth is table stakes for the audience this product is aimed at; LinkedIn OAuth has the extra benefit that it could replace the LinkedIn scraping session for users who consent to it (LinkedIn's official API gives access to profile data, not job postings — but it solves the "how do I extract the user's CV" problem).

## High value, high effort

### Multi-page CV templates

GraphCompose supports pagination; the templates intentionally don't use it (see [TD-16](technical-decisions.md#td-16--single-page-cvs-only-for-now)). Adding overflow handling to the existing six templates, plus two or three new templates designed for two-page output, would unlock a real audience (senior candidates, academics).

The hard part is making the page break decisions look intentional, not algorithmic. That's a UX problem more than a rendering problem.

### Resume export to .docx

Recruiters who scrape CVs into ATS often want a `.docx` they can paste into their own template. I'd build a `.docx` exporter as a separate `feature/docx_render/` package, with the same template-registry pattern as `pdf_render`. The natural library to evaluate is Apache POI; the bigger question is whether docx templates should be 1:1 with the PDF templates or a separate set.

### Cover-letter generation as a peer to CV

Right now cover letters are a single template, generated on demand. Treating them as a first-class output — with their own templates, their own editor, their own history — would make the product useful for the full application flow rather than just the CV.

This includes:
- A `cover_letters` table or a `cover_letter` JSON column on `vacancy_records`.
- A separate AI flow with its own prompt (shorter, more rhetorical).
- Editor with section-by-section editing.
- PDF + docx export.

### Multi-language CV support

Currently the product assumes English-language CVs and JDs. The structured-output approach makes this relatively clean to extend: add a `language` field to `AiCvCraftedDTO`, localize the templates, prompt the AI to detect-or-honor the language. The harder problem is high-quality translation of an existing English CV into another language — that's a separate AI flow, not a settings flag.

## Polish / lower priority

### Cache scraped job descriptions for short windows

If the same URL is processed twice in a short window (user retries, or two users apply to the same job), the scraper does the work twice. A short TTL cache (`Caffeine`, 30-minute TTL, keyed by normalized URL) would be a cheap win — but worth measuring first whether the duplicate-rate is high enough to justify.

### Per-template preview thumbnails

Today the template picker shows static screenshots. Rendering a per-user thumbnail (the user's actual CV in that template, at low resolution) would make selection a lot more concrete. The render pipeline already exists; the question is just where to cache.

### Workflow-level retry policy

Today a failed pipeline stage doesn't auto-retry; the user has to hit retry. Some failures are transient (network blip on a scrape, AI provider 5xx). A retry-with-backoff at the orchestrator level — capped, idempotent, with telemetry — would reduce manual retries.

### Soft-delete records instead of hard-delete

DELETE on a record is hard-delete today. Soft-delete (`deleted_at` column, query filter, scheduled cleanup) would let users undo accidental deletes within a window. Small change, real UX value.

## Known limitations

These are not things I plan to fix soon — they're things I'd want a reader to know are deliberate or accepted:

- **English only.** See above.
- **No mobile app.** Web works on mobile, but it's not a designed mobile experience.
- **Single-tenant within an environment.** No org / team model. Each user is independent.
- **No webhook outputs.** CVRewriter consumes Stripe webhooks but doesn't emit any of its own. A user can't pipe "CV generated" into Zapier or n8n. Worth considering for power users.
- **Stripe is the only payment provider.** No Paddle, no LemonSqueezy, no manual invoicing. Stripe covers ~95% of the audience; the rest would be friction-heavy to add.

## What I would NOT add

A few things I've intentionally rejected:

- **A "chat with your CV" feature.** It's an easy demo and a hard product. The structured editor is more useful than a chat thread.
- **AI-generated cover-letter chains** (generate cover letter, then refine, then refine again in a loop). Adds tokens, dilutes quality, hides the user's voice. The user gets one good draft and an editor.
- **An in-house ATS integration.** Each ATS is a separate, gnarly integration. The right shape is a `.docx` / PDF export the recruiter can paste, plus a stable JSON export endpoint for power users.
- **A free tier with no rate limits.** AI generation isn't free for me. Free tier exists; it's bounded.
