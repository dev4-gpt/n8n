# Job Application Automation - Configuration Guide

This guide explains how to configure the sanitized `job_application_automation_sanitized.json` workflow for your own use.

## Overview

This n8n workflow automates job searching by:
1. Scraping LinkedIn jobs via Apify
2. Filtering jobs by requirements (PhD, citizenship, etc.)
3. Using LLM to match your resume against job descriptions
4. Creating tailored documents for applications
5. Tracking everything in Google Sheets

## Required Placeholders to Replace

### 1. YOUR_GOOGLE_SHEET_ID
**Location:** Google Sheets node ("Append to Jobs Tracker DB")
**Format:** `YOUR_GOOGLE_SHEET_ID`
**How to get it:**
1. Create a new Google Sheet
2. Name it "Jobs Tracker DB" (or your preferred name)
3. Add these column headers in row 1:
   - Job Post Link
   - Seniority Level
   - Job Title
   - Company Name
   - Location
   - Description
   - Posted At
   - Verdict
   - Reason
   - Resume Link
   - Status (New/Applied/Rejected/Interview)
4. Copy the Sheet ID from the URL:
   ```
   https://docs.google.com/spreadsheets/d/YOUR_GOOGLE_SHEET_ID/edit
   ```

### 2. YOUR_RESUME_DOC_ID
**Location:** "Get resume" Google Docs node
**Format:** `YOUR_RESUME_DOC_ID`
**How to get it:**
1. Create a Google Doc with your resume
2. Copy the Doc ID from the URL:
   ```
   https://docs.google.com/document/d/YOUR_RESUME_DOC_ID/edit
   ```
3. Make sure the n8n credential has read access to this doc

### 3. YOUR_NAME_resume_
**Location:** "Create a document" Google Docs node (title field)
**Format:** `YOUR_NAME_resume_{{ $('Scrape Jobs').item.json.companyName }}`
**What to change:**
- Replace `YOUR_NAME` with your actual name (e.g., `john_resume_`)
- This creates documents like: "john_resume_Google", "john_resume_Meta"

### 4. Credential IDs
These are internal n8n credential references. You'll set these up in n8n UI.

#### YOUR_APIFY_CREDENTIAL_ID
**Node:** "Scrape Jobs" (Apify node)
**Setup:**
1. Sign up at [apify.com](https://apify.com)
2. Get your API token from Settings → Integrations
3. In n8n: Settings → Credentials → Add Credential → Apify API
4. Paste your token
5. The credential ID will be auto-generated (use that)

#### YOUR_GOOGLE_CREDENTIAL_ID
**Nodes:** "Get resume", "Create a document", "Add resume text", "Append to Jobs Tracker DB"
**Setup:**
1. In n8n: Settings → Credentials → Add Credential → Google Sheets OAuth2 / Google Docs OAuth2
2. Follow the OAuth flow to connect your Google account
3. The credential ID will be auto-generated

#### YOUR_NVIDIA_API_CREDENTIAL_ID
**Nodes:** "HTTP Request" (LLM classifier), "HTTP Request to rewrite resume"
**Setup:**
1. Get API key from [NVIDIA NIM](https://build.nvidia.com) or use OpenAI
2. In n8n: Settings → Credentials → Add Credential → OpenAI API
3. If using NVIDIA: Use `https://integrate.api.nvidia.com/v1` as the base URL
4. Paste your API key
5. The credential ID will be auto-generated

## Customization Options

### LinkedIn Search URL
**Node:** "Job Search URL" (Set node)
**Current:** AI Internships in United States
**Modify:**
```
https://www.linkedin.com/jobs/search?keywords=YOUR_KEYWORDS&location=YOUR_LOCATION&geoId=YOUR_GEOID&f_E=EXPERIENCE_LEVEL
```

Parameters:
- `keywords`: Job title (e.g., "Software Engineer", "Data Scientist")
- `location`: City/Country
- `geoId`: LinkedIn's location ID (find by searching on LinkedIn)
- `f_E`: Experience level (1=Internship, 2=Entry level, 3=Associate, etc.)

### Filter Conditions
**Node:** "not PHDs" (Filter node)
**Current filters:**
- Excludes: "phd", "doctoral", "doctorate", "us citizen"

**Add your own:**
1. Open the "not PHDs" node
2. Add conditions for requirements you don't meet
3. Common additions: "security clearance", "ts/sci", "10+ years"

### LLM Model
**Nodes:** "HTTP Request" (classifier), "HTTP Request to rewrite resume"
**Current:** `qwen/qwen3-coder-480b-a35b-instruct` via NVIDIA

**Alternatives:**
- OpenAI: `gpt-4`, `gpt-3.5-turbo`
- Anthropic: `claude-3-sonnet-20240229` (via HTTP request)
- NVIDIA alternatives: `meta/llama3-70b`, `nvidia/nemotron-4-340b`

**Update the model parameter in the JSON body:**
```json
"model": "gpt-4"
```

### Job Seniority Filter
**Node:** "intern" (Filter node)
**Current:** Only passes jobs containing "internship" or "intern"
**Modify:** Change to match your target level

## Workflow Execution Order

1. **Manual Trigger** - You click "Execute workflow"
2. **Job Search URL** - Sets LinkedIn search parameters
3. **Scrape Jobs** - Apify fetches 100 jobs
4. **Loop Over Items** - Processes each job individually
5. **Get resume** - Fetches your resume from Google Docs
6. **Merge resume link fields** - Combines data
7. **HTTP Request (LLM)** - AI compares resume to job description
8. **Code in JavaScript** - Parses LLM response
9. **not PHDs** - Filters out PhD/US citizen requirements
10. **intern** - Filters for internship-level positions
11. **verdict = true/false** - Routes based on AI match
12. **HTTP Request to rewrite resume** - Creates tailored resume (optional)
13. **Create a document** - Creates Google Doc for application
14. **Add resume text** - Inserts resume content
15. **Markdown** - Formats the output
16. **Append to Jobs Tracker DB** - Logs to Google Sheets

## Troubleshooting

### "Credential not found" error
- You haven't set up the credential in n8n yet
- Go to Settings → Credentials and add the required credential type

### "Document not found" error
- Check that YOUR_RESUME_DOC_ID is correct
- Verify n8n has Google Docs OAuth permission
- Make sure the document is shared with the n8n service account (if using service account auth)

### "Sheet not found" error
- Check YOUR_GOOGLE_SHEET_ID
- Verify the sheet has the correct column headers
- Ensure n8n has Google Sheets OAuth permission

### LLM returns invalid JSON
- Check that your API key is valid
- Verify the model name is correct
- Check API rate limits (add retries if needed)

## Privacy & Security Notes

- Never commit the non-sanitized JSON to public repos
- The sanitized version uses placeholders - you fill in your own values
- Store credentials in n8n's secure credential storage, never in the workflow JSON
- Consider using environment variables for sensitive data in production

## Support & Resources

- n8n Documentation: https://docs.n8n.io
- Apify LinkedIn Scraper: https://apify.com/curious_coder/linkedin-jobs-scraper
- NVIDIA NIM Models: https://build.nvidia.com/explore/discover

---

**Ready to use?** Import the sanitized JSON into n8n, replace all YOUR_* placeholders, and start automating your job search!
