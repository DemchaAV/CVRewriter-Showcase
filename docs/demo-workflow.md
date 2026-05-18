# Demo workflow

This is what using CVRewriter looks like, end to end. Screenshots are real captures from the running app against the author's own account, with sensitive fields (email, full phone, full address) redacted at capture time. See [security-and-privacy.md](security-and-privacy.md#personally-identifying-data--whats-in-the-screenshots) for the exact disclosure of what is and isn't visible.

## 0. First-time setup

A user signs up with an email + password. The backend sends a verification link via SMTP (`SmtpVerificationEmailSender`). The link carries a one-time token that's hashed at rest. Until the email is verified, the account exists but cannot log in.

Once verified, the user lands on the **profile editor** to upload their CV (`profile/edit`). This is parsed into the canonical `AiCvCraftedDTO` shape — the same shape that AI generations produce and PDF templates render. From this point on, "the user's CV" is structured data, not a file.

The user may also create additional **job profiles**: a job profile is a (name + original CV + optional additional instructions) bundle. Someone applying to both Java and Kotlin roles can keep two profiles, each with its own baseline CV.

## 1. Paste a vacancy link

![Processing](../06_processing.png)

The home page is one input: a job posting URL. The user pastes a LinkedIn / Indeed / Glassdoor link, optionally types extra instructions ("emphasize backend, downplay frontend"), and hits process.

The backend:
1. Creates a `vacancy_records` row, status `QUEUED`.
2. Returns a `jobId`.
3. Frontend immediately requests an SSE token for that jobId.
4. Frontend opens the SSE stream.

The progress popup updates in real time as the backend works through six stages: connecting → scraping → parsing → preparing → generating → saving. Each stage has a deterministic progress percentage. The popup also shows the company name as soon as it's known (typically at the end of the scraping stage).

## 2. Processing completes

![Complete](../07_complete.png)

When the generation succeeds, the popup transitions to a "ready" state with a link into the editor. The user can stay on the home page to queue another vacancy, or jump straight to the result.

If processing fails — bot wall, malformed JD, AI quota exceeded — the popup shows the error message and the record is marked `FAILED`. The user can retry or delete the record.

## 3. History

![History](../01_history.png)

The history page shows every processed vacancy as a row in a sortable, filterable table:

- **Status** column reflects the application status: Queued, Generated, Applied, Interview, Offer, Rejected.
- **Rating** is a 1–5 scale the user sets to mark how interested they are in the role.
- **Company** is the scraped company name.
- **Created** and **last updated** timestamps.

Filters are stackable: status, company, rating, date range, free-text search across company + brief.

Each row has actions: open editor, regenerate, download PDF, delete.

## 4. Edit a record

![Edit record](../02_edit_record.png)

Clicking a row opens a modal with the record metadata: title, company, link, brief, status, outcome, rating, dates. The user can:
- Refresh the vacancy from its original URL (re-scrape).
- Edit the brief (Markdown).
- Move status forward or backward.
- Change the outcome (separate from status).
- Update the rating.
- Delete the record.

Every change writes to the appropriate history table — `application_record_status_history` for status, `record_outcome_history` for outcome — with `source=USER`.

## 5. CV editor

![CV editor](../03_cv_editor.png)

The editor shows the generated CV section by section: Header, Summary, Skills, Experience, Education, Languages, Links. Each section is a small typed form (React Hook Form + Zod) backed by the same `AiCvCraftedDTO` schema the AI returned.

The user can:
- Reorder sections.
- Add / remove / edit items in any section.
- Tweak phrasing inline.
- Pick a template (FREE or PRO — gating respected).
- Preview the PDF inline.
- Download the PDF.

Saves are debounced and persisted to `vacancy_records.cv_content` as JSON.

## 6. Regenerate

![Regenerate](../04_regenerate.png)

If the result isn't quite right, the user can hit Regenerate. They get a modal asking for additional instructions ("focus on architecture and team leadership" / "tone down the AI/ML stuff"). The backend runs the same AI pipeline again, with the additional instructions appended to the base prompt. No re-scrape — the original `job_description` is reused.

The new generation overwrites `cv_content` and adds a new `token_usage` row. The user can regenerate as many times as they want; each call is tracked.

## 7. Profile

![Profile](../05_profile.png)

Profile pages cover:
- **Profile editor** — first name, last name, base CV (parsed and editable), cover letter content.
- **Job profiles** — list and manage the user's job profile bundles. Set one as default.
- **Account settings** — email, password change, account deletion.
- **Integrations** — list connected external accounts (e.g. LinkedIn session). Disconnect or refresh.
- **Billing** (if Stripe is enabled) — subscription status, plan tier, link to Stripe Customer Portal.

## 8. Admin (if applicable)

Users with `ROLE_ADMIN` get an extra page at `/admin/token-usage`:

- Summary card: total tokens, total cost (computed at read time from current pricing), per-provider breakdown.
- Per-user table: who's spending what, sortable by cost / tokens / call count.
- Date-range filter.

The same view is also accessible programmatically via `GET /admin/token-usage/summary` for headless reporting (auth via either `ROLE_ADMIN` JWT or `X-Admin-Key` header).

## Failure paths worth showing

- **Scraping bot wall** → HTTP path returns Cloudflare interstitial → `VacancyDescriptionSanitizer.hasUsableDescription` rejects it → falls through to Playwright path → succeeds or marks record FAILED. SSE shows the user the actual stage at which failure happened.
- **SSE connection drops** → frontend's `ProcessingQueueContext` detects no events for >5s after the EventSource errored → falls back to polling `GET /vacancies/{id}/status` every 3 seconds. UI looks identical.
- **AI quota exceeded** → provider returns an error → record marked FAILED → user sees the error message in the popup with the option to switch provider (if admin) or retry later.
- **Stripe webhook re-delivery** → handler checks `lastWebhookEventId`; if seen, no-ops. Subscription state never double-counts.

## What the user never sees

- They never see the raw scrape — it's processed and stored, but the UI shows the structured CV, not the JD.
- They never see the prompt — it's an implementation detail.
- They never see token counts — those are admin-only.
- They never see the SSE token — it's issued and consumed by the frontend behind the scenes.

The product surface is small. The system behind it is doing real work.
