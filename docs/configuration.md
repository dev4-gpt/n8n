# Configuration

This workflow stores its “inputs” in the `Job Search URL` Set node so exports stay portable.

## Required fields (Set node: `Job Search URL`)

- `linkedinSearchUrl`: LinkedIn search URL to scrape (keywords/location filters etc.)
- `resumeDocUrl`: Google Docs URL for your base resume (used by `Get resume`)

## Optional personal fields (used to build `resumeHeader`)

- `personalEmail`
- `personalPhone`
- `personalLinkedIn`
- `personalCredentials`
- `personalGithub`
- `personalSchoolGithub`

## Credentials (kept inside n8n, not in git)

The JSON export references credential **names/ids**, but does not include secrets. You’ll still need these configured in your n8n instance:

- **Apify**: API token for the Apify account used by `Scrape Jobs`
- **Google Docs / Drive**: OAuth credential used by `Get resume`, `Create a document`, and `Add resume text`
- **Google Sheets**: OAuth credential used by `Append row in sheet` + `Append row in sheet1`
- **LLM provider credential**: currently the `openAiApi` credential type is used by the HTTP requests to `integrate.api.nvidia.com`

## Hybrid pipeline sheet schema (recommended)

Your existing exported workflow (`workflows/job-automation.json`) writes job rows into your Google Sheet. For the hybrid auto-apply pipeline described in the plan, it’s safest to use a separate tab so the original workflow doesn’t get affected.

Create a new tab in the same Google Spreadsheet (same `spreadsheetId`) named: `JobPipeline`

Add this exact header row as the first row in `JobPipeline`:

`job_hash | job_source | job_url | apply_url | company | title | location | posted_at | ats_type | application_mode | fit_verdict | fit_reason | fit_score | resume_doc_url | cover_letter_url | application_status | missing_fields | approved_by | last_attempt_at | attempt_count | error_last | created_at | updated_at`

Suggested `application_status` values:
- `new` (ingested, not enriched yet)
- `fit_true_queued` (eligible and queued for enrichment)
- `drafted` (prefill/draft prepared; waiting for human approval)
- `needs_info` (prefill incomplete; human needs to provide missing fields)
- `approved` (human approved; ready to submit)
- `submitted` (submitted successfully)
- `failed` (submission attempt failed)

Allowlist + mode:
- `application_mode` should be `auto_submit` only for portal domains you explicitly allow (initially: Greenhouse)
- for everything else use `needs_review`

Green/yellow/red flag mapping (Google Sheets conditional formatting):
- Green: `application_status = submitted`
- Yellow: `application_status IN ("drafted","needs_info")` (and/or `application_mode = needs_review`)
- Red: `application_status IN ("failed")`

Notes:
- `missing_fields` should be a comma-separated string like: `email,phone,resume_override`
- `job_hash` should let you de-duplicate across multiple ingestion runs and sources

## Runner HTTP contract (rtrvr.ai / AIHawk-style fallback)

To keep n8n stable even if you change the underlying automation engine, define a simple request/response contract for the “runner” service.

### Request (n8n -> runner)
POST `runnerWebhookUrl` (configured inside the `job-apply.json` workflow)

JSON body:
```json
{
  "job_hash": "string",
  "apply_url": "string",
  "ats_type": "greenhouse|lever|workday|unknown",
  "mode": "draft|submit",
  "resume_doc_url": "string",
  "cover_letter_url": "string",
  "applicant": {
    "email": "string",
    "phone": "string",
    "linkedin": "string",
    "github": "string"
  }
}
```

### Response (runner -> n8n)
```json
{
  "ok": true,
  "status": "submitted|drafted|needs_info|failed",
  "application_id": "string|null",
  "missing_fields": ["field1", "field2"],
  "proof": { "url": "string|null", "notes": "string|null" },
  "error": "string|null"
}
```

### n8n mapping rules
- `mode="submit"`: if runner returns `ok=true` and `status="submitted"`, n8n appends `application_status="submitted"`.
- If runner reports missing fields or needs extra input, n8n appends `application_status="failed"` (or `needs_info` if you want to extend the schema) and writes `missing_fields` into the sheet.
- `mode="draft"`: intended behavior is to return `missing_fields` (so you can human-review and then set `application_status="approved"` in the sheet).

This repo currently wires runner calls for the **submit** path and uses a placeholder `drafted` state without invoking the runner draft endpoint yet. The contract above is ready for you to extend.

