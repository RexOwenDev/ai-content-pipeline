# AI Content Pipeline — Project Memory

## Status
**LIVE** — n8n Cloud · 6,800+ executions/month  
GitHub: https://github.com/RexOwenDev/ai-content-pipeline

---

## Audit — 2026-04-20

Full audit report: `docs/audit-2026-04-20.md`  
Auditors: Claude Sonnet 4.6 (Sonnet lane) · Codex GPT-5.4 (background adversarial) · Semgrep 1.157.0

### Fixes Applied (this session)

| ID | Severity | File | Fix |
|---|---|---|---|
| C-1 | 🔴 CRITICAL | `05-form-intake.json` | Added `Verify Webhook Signature` Code node; enabled `rawBody: true` on Webhook node |
| H-1 | 🟠 HIGH | `05-form-intake.json` | Reordered non-startup path: dedup check now runs before DocuSign email |
| M-1 | 🟡 MEDIUM | `02-rss-intake.json`, `04-content-pipeline.json` | Replaced 5 real n8n credential IDs with `[CREDENTIAL_ID]` placeholders |
| L-1 | 🔵 LOW | `02-rss-intake.json` | Removed `ignoreSSL: true` from RSS Read node |
| L-2 | 🔵 LOW | `README.md` | Updated model references from `GPT-4.1-mini` → `gpt-5.4-mini` |
| L-3 | 🔵 LOW | `04-content-pipeline.json` | Added `pairedItem: { item: 0 }` to `Parse First Pass` return object |

### Deferred (Phase 2)

| ID | Severity | Description |
|---|---|---|
| M-2 | 🟡 MEDIUM | `$getWorkflowStaticData` rate limiter resets on n8n Cloud restart — replace with Sheets-backed counter |

---

## Workflow Map

| File | Name | Trigger | Purpose |
|---|---|---|---|
| `00-error-handler.json` | Error Handler | Error trigger | Centralized error classification + email alerts (rate-limited 3/30min) |
| `01-article-scout.json` | Article Scout | Schedule | Monitors RSS feeds, filters by keyword, scores candidates |
| `02-rss-intake.json` | RSS Intake | Schedule | Fetches RSS articles, deduplicates by URL, appends to Sheets |
| `03-document-receiver.json` | Document Receiver | Email trigger | Processes inbound email attachments, logs to Sheets |
| `04-content-pipeline.json` | Content Pipeline | Manual/Schedule | Full AI content pipeline: fetch → evaluate → draft → refine → publish to WordPress |
| `05-form-intake.json` | Form Intake | Webhook (GravityForms) | Intake form → HMAC verify → dedup check → DocuSign email or startup rejection |

---

## Architecture Notes

- All 6 workflows configured with `errorWorkflow: "bg8o5L8lUqPtR450cJsEu"` (centralized handler)
- All Google Sheets write nodes: `maxTries: 3, waitBetweenTries: 3000`
- All OpenAI nodes: `maxTries: 3, waitBetweenTries: 5000`
- `continueOnFail: false` on all nodes (failures route to error handler)
- WordPress node uses `onError: "continueErrorOutput"` → `Error: WP Post Failed` sheet node
- Gitleaks CI covers all custom secret patterns
- AI model in production: `gpt-5.4-mini`

## Known Limitations

- **Rate limiter reset**: `$getWorkflowStaticData('global')` in `00-error-handler.json` resets on n8n Cloud restart/redeploy. During maintenance coinciding with an outage, the 30-min email cap can reset. Phase 2: replace with Sheets-backed counter.
- **Gemini CLI on Windows**: v0.35.1 hits ARG_MAX limit on large repos. Upgrade to v0.38.2 resolves. Gemini audit pass was skipped this cycle.

---

## Parked Questions

_None currently._
