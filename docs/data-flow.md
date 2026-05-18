# Data flow

This walks through the end-to-end flow of the core "process a vacancy" use case, showing every hop, every event, and where data is persisted.

## High-level sequence

```
┌────────┐    ┌─────────┐    ┌──────────────┐    ┌──────────┐    ┌─────────┐    ┌────────┐
│ Browser│    │  Spring │    │  Scraper     │    │   AI     │    │  Token  │    │ MySQL  │
│ (React)│    │  Boot   │    │  Strategy    │    │  Provider│    │ Tracker │    │        │
└───┬────┘    └────┬────┘    └──────┬───────┘    └────┬─────┘    └────┬────┘    └───┬────┘
    │              │                │                 │                │             │
    │ POST /vacancies/process       │                 │                │             │
    │─────────────>│                │                 │                │             │
    │              │ INSERT record (QUEUED)           │                │             │
    │              │─────────────────────────────────────────────────────────────────>│
    │              │                │                 │                │             │
    │<──── 202 {jobId} ─────────────│                 │                │             │
    │              │                │                 │                │             │
    │ POST /vacancies/{id}/events-token                │                │             │
    │─────────────>│                │                 │                │             │
    │<── { sseToken: 120s JWT } ────│                 │                │             │
    │              │                │                 │                │             │
    │ GET /vacancies/{id}/events?sseToken=...         │                │             │
    │═════════════>│ (SSE stream stays open)          │                │             │
    │              │                │                 │                │             │
    │              │ === Worker thread picks up jobId ==              │             │
    │              │ emit progress: connecting (5%)   │                │             │
    │<── SSE event progress ────────│                 │                │             │
    │              │                │                 │                │             │
    │              │ emit progress: scraping (15%)    │                │             │
    │<── SSE event progress ────────│                 │                │             │
    │              │ ScraperFactory.scrape(url)       │                │             │
    │              │───────────────>│                 │                │             │
    │              │  ┌─────────────┴─────────────┐   │                │             │
    │              │  │ HttpNoLogin first         │   │                │             │
    │              │  │ if bot-wall → BrowserAuth │   │                │             │
    │              │  │ (Playwright)              │   │                │             │
    │              │  └─────────────┬─────────────┘   │                │             │
    │              │<── ScrapedDescription ──────│   │                │             │
    │              │                │                 │                │             │
    │              │ emit progress: parsing (40%)     │                │             │
    │<── SSE event progress ────────│                 │                │             │
    │              │ VacancyDescriptionSanitizer      │                │             │
    │              │ if !hasUsable → fall through     │                │             │
    │              │ to next strategy                 │                │             │
    │              │                │                 │                │             │
    │              │ emit progress: preparing (55%)   │                │             │
    │<── SSE event progress ────────│                 │                │             │
    │              │ build prompt from template +     │                │             │
    │              │ original CV + scraped JD         │                │             │
    │              │                │                 │                │             │
    │              │ emit progress: generating (75%)  │                │             │
    │<── SSE event progress ────────│                 │                │             │
    │              │ AiServiceProvider               │                │             │
    │              │ .getActiveService()             │                │             │
    │              │ .generateObjectWithMetadata(    │                │             │
    │              │     prompt, AiCvCraftedDTO)     │                │             │
    │              │───────────────────────────────>│                │             │
    │              │<─── AiGenerationResult ─────────│                │             │
    │              │                │                 │                │             │
    │              │ TokenUsageService.logUsage(...) │                │             │
    │              │───────────────────────────────────────────────>│             │
    │              │ INSERT token_usage row           │                │             │
    │              │─────────────────────────────────────────────────────────────>│
    │              │                │                 │                │             │
    │              │ emit progress: saving (94%)      │                │             │
    │<── SSE event progress ────────│                 │                │             │
    │              │ UPDATE record SET cv_content,    │                │             │
    │              │  status=GENERATED, ...           │                │             │
    │              │─────────────────────────────────────────────────────────────>│
    │              │ INSERT record_status_history     │                │             │
    │              │─────────────────────────────────────────────────────────────>│
    │              │                │                 │                │             │
    │              │ emit completed                   │                │             │
    │<── SSE event completed ───────│                 │                │             │
    │ EventSource closed            │                 │                │             │
    │              │                │                 │                │             │
    │ React Query invalidate ['records']              │                │             │
    │ user navigates to /history/{recordId}/edit      │                │             │
```

## SSE event payloads

All events are JSON. The progress event has the most information:

```jsonc
// event: progress
{
  "stage": "scraping",          // connecting | scraping | parsing | preparing | generating | saving
  "step": 2,                    // 1..6
  "totalSteps": 6,
  "progressPercent": 15,        // deterministic per stage
  "message": "Fetching job description",
  "companyName": null           // populated as soon as known
}

// event: completed
{
  "jobId": "9c8e397c-...",
  "companyName": "Example Corp"
}

// event: error
{
  "message": "Job description was unavailable (bot protection)."
}
```

Progress percentages are deterministic per stage, not interpolated client-side. This means the bar advances in honest jumps that map to backend work, not in a continuous fake animation.

## Persistence per stage

| Stage | What gets written | Tables touched |
|---|---|---|
| Request received | `VacancyRecord` row, status `QUEUED` | `vacancy_records` |
| Scrape ok | `vacancy_records.job_description`, `company` | `vacancy_records` |
| AI generation done | `cv_content` (JSON), `brief`, `rating`, status `GENERATED`; `token_usage` row | `vacancy_records`, `token_usage` |
| Status change | `application_record_status_history` row with `source=SYSTEM` | `application_record_status_history` |
| Template chosen | `record_template_history` row | `record_template_history` |
| User changes status | `application_record_status_history` row with `source=USER` | `application_record_status_history` |
| User changes outcome | `record_outcome_history` row | `record_outcome_history` |
| Stripe webhook | `user_subscriptions` row inserted / updated | `user_subscriptions` |

`cv_content` is stored as a JSON string column. The schema matches `AiCvCraftedDTO`, so the same DTO drives the AI request, the persistence, the editor, and the PDF render.

## Fallback behavior

The frontend `ProcessingQueueContext` has a polling fallback for hostile network paths (corporate proxies that buffer SSE, mobile networks that close idle connections):

```
EventSource open
  │
  ├── onmessage(progress | completed | error) → update store, normal flow
  │
  └── onerror after >5s with no events → close EventSource
       │
       └── setInterval(3000ms):
             GET /vacancies/{id}/status
             until status is terminal (GENERATED | FAILED)
```

The status endpoint returns the same `{stage, progressPercent, status}` shape, so the UI can render from polling output exactly the way it renders from SSE output.

## PDF generation flow

PDF rendering happens **on demand**, not at generation time. The `cv_content` JSON is the source of truth; PDF is a view.

```
GET /pdf/records/{recordId}
  │
  ├── RecordOwnershipService.assertRecordOwnership(currentUser, recordId)
  │
  ├── CvTemplateAccessService.assertTemplateAccessible(userId, templateId)
  │     └── EntitlementService.hasPremiumAccess(userId) if PRO template
  │
  ├── Load cv_content (JSON) → AiCvCraftedDTO
  │
  ├── CvTemplateRegistry.get(templateId) → CvTemplate bean
  │
  ├── CanonicalCvRenderer.render(template, dto) → PDDocument (GraphCompose)
  │
  └── StreamingResponseBody writes to HTTP response
        Content-Disposition: attachment; filename="John Doe - Example Corp - CV.pdf"
```

Files are never written to disk on the server — the PDFBox `PDDocument` is rendered into the response output stream and discarded. This keeps disk usage flat regardless of traffic.

## Regenerate flow

The "regenerate with extra instructions" feature is the same pipeline as generation, with two differences:

1. **No scrape** — `job_description` is reused from the original record.
2. **Augmented prompt** — the user's extra instructions are appended to the base prompt.

The new generation overwrites `cv_content`, creates a new `token_usage` row, and adds a `record_status_history` entry. The pre-regeneration `cv_content` is not kept (no version history on the CV body itself — only on metadata). This was a deliberate trade-off: storing every revision adds significant DB volume for marginal UX value, and the editor already lets users undo within a session.

## Why this design

The pipeline-then-SSE shape was chosen over a synchronous "process and return" because:

- AI rewrites take 8–25 seconds. A blocking HTTP request is fragile (proxy timeouts, page navigations, mobile tab freezing).
- SSE gives the user honest progress, which makes the wait feel shorter and the failure modes obvious.
- The async record exists as soon as the request is accepted, so users can leave and come back — the result will still be there.
- Token usage is recorded per call, attributable per user and per record, which is the foundation for both cost analytics and any future rate-limiting.

The two-tier scraping was chosen over Playwright-only because:

- HTTP + JSON-LD is roughly 50× faster than spinning a browser, and works for the simple postings that don't paywall content.
- Playwright is necessary for bot-walled sites (LinkedIn especially) but expensive per call. Reserving it for fallback keeps p50 latency low.
- The sanitizer-then-fallback flow means we never persist garbage when the HTTP path returns a captcha page.
