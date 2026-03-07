# 🤖 Self-Hosted n8n Job Application Automation
### Deployed on DigitalOcean · Custom Domain via Namecheap · Automated with Google Sheets & Gmail

![n8n](https://img.shields.io/badge/n8n-self--hosted-EF5033?logo=n8n&logoColor=white)
![DigitalOcean](https://img.shields.io/badge/DigitalOcean-Droplet-0080FF?logo=digitalocean&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04_LTS-E95420?logo=ubuntu&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

> A fully self-hosted workflow automation system that automatically applies to jobs across LinkedIn, Indeed, and other platforms — tracking status in Google Sheets and sending Gmail notifications.  
> Live instance: **https://n8n.aryamandev.me**

---

## 📑 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Create the DigitalOcean Droplet](#step-1--create-the-digitalocean-droplet)
4. [Step 2 — Point Namecheap Domain to the Droplet](#step-2--point-namecheap-domain-to-the-droplet)
5. [Step 3 — Run the n8n Setup Script](#step-3--run-the-n8n-setup-script)
6. [Step 4 — First-Time n8n Setup](#step-4--first-time-n8n-setup)
7. [Step 5 — Set Up the Google Sheet](#step-5--set-up-the-google-sheet)
8. [Step 6 — Configure Credentials in n8n](#step-6--configure-credentials-in-n8n)
9. [Step 7 — Import & Configure the Workflow](#step-7--import--configure-the-workflow)
10. [Step 8 — Extending Beyond LinkedIn & Indeed](#step-8--extending-beyond-linkedin--indeed)
11. [Step 9 — Schedules & Triggers](#step-9--schedules--triggers)
12. [Step 10 — Server Operations & Maintenance](#step-10--server-operations--maintenance)
13. [Repository Structure](#repository-structure)
14. [Next Steps & Ideas](#next-steps--ideas)
15. [Legal Note](#legal-note)

---

## Architecture Overview

```
┌────────────────────────────────────────────────────────────┐
│                    Your Mac (local)                        │
│  ssh root@104.131.31.106   ──►  DigitalOcean Droplet       │
└────────────────────────────────────────────────────────────┘
                                        │
                          ┌─────────────▼─────────────-┐
                          │  Ubuntu 24.04 LTS (NYC3)   │
                          │  Docker Compose            │
                          │  ┌──────────┐ ┌─────────┐  │
                          │  │  Caddy   │ │   n8n   │  │
                          │  │ (HTTPS/  │ │ :5678   │  │
                          │  │  proxy)  │ │         │  │
                          │  └──────────┘ └─────────┘  │
                          └──────────────────────────-─┘
                                        │
                          ┌─────────────▼─────────────--┐
                          │  n8n.aryamandev.me (HTTPS)  │
                          │  SSL via Let's Encrypt      │
                          │  DNS A record on Namecheap  │
                          └─────────────────────────────┘
                                        │
          ┌─────────────────────────────▼──────────────────────────----┐
          │                    n8n Workflow                            │
          │  Google Sheets ──► Filter ──► Apply ──► Update ──► Gmail   │
          │  (job list)       (pending)  (LinkedIn/  (status)  (notify)|
          │                              Indeed/etc)                   │
          └──────────────────────────────────────────────────────────--┘
```

| Component | Details |
|-----------|---------|
| **Droplet** | DigitalOcean, Ubuntu 24.04 LTS, 1 vCPU / 1 GB RAM, NYC3 |
| **IP** | `104.131.31.106` |
| **Domain** | `aryamandev.me` via Namecheap |
| **n8n URL** | `https://n8n.aryamandev.me` |
| **Reverse proxy** | Caddy (auto-HTTPS via Let's Encrypt) |
| **Container runtime** | Docker + Docker Compose |
| **Config path on server** | `/opt/n8n-docker-caddy` |

---

## Prerequisites

Before starting, you'll need:

- A **DigitalOcean** account with billing enabled
- A **Namecheap** domain (I used `aryamandev.me`)
- A **Mac or Linux** terminal to SSH into the droplet
- A **Google account** for:
  - Google Sheets (job list + status tracking)
  - Gmail (email notifications)
- Optional: API credentials for job platforms (LinkedIn OAuth2, Indeed API)

---

## Step 1 — Create the DigitalOcean Droplet

1. Go to **[DigitalOcean Marketplace → n8n 1-Click App](https://marketplace.digitalocean.com/apps/n8n)**
2. Click **Create Droplet** and configure:
   - **Image:** `n8n on Ubuntu 24.04 (LTS)` (Marketplace 1-Click)
   - **Plan:** Basic · 1 vCPU · 1 GB RAM · 25 GB SSD (~$6/month)
   - **Region:** NYC3 (or your closest region)
   - **Authentication:** SSH key (recommended) or root password
3. Click **Create Droplet** and wait ~1 minute
4. Copy the assigned **IPv4 address** (mine: `104.131.31.106`)

> 💡 To SSH from your Mac once the droplet is live:
> ```bash
> ssh root@104.131.31.106
> ```

OR click on **Console** button on DigitalOcean

---

## Step 2 — Point Namecheap Domain to the Droplet

I used the simple **A record** method (keeping DNS on Namecheap, not delegating to DigitalOcean nameservers).

1. Log into **Namecheap → Domain List → Manage** on your domain
2. Go to **Advanced DNS → Host Records**
3. Add these records:

| Type | Host | Value | TTL |
|------|------|-------|-----|
| A Record | `@` | `104.131.31.106` | Automatic |
| A Record | `n8n` | `104.131.31.106` | Automatic |

4. Save and wait for DNS propagation (usually 5–30 minutes, up to 1 hour)

**Verify propagation if needed:**
```bash
dig n8n.aryamandev.me +short
# Should return: 104.131.31.106
```

---

## Step 3 — Run the n8n Setup Script

The 1-Click droplet includes a setup script that runs automatically on first login.

1. In the DigitalOcean dashboard, click your droplet → **Console** (browser terminal), or SSH in:
   ```bash
   ssh root@104.131.31.106
   ```

2. The script prompts:
   ```
   Subdomain (default: n8n): n8n
   Domain name (e.g., yourdomain.org): aryamandev.me
   Email address for Let's Encrypt: your-email@example.com 
   Configure timezone? (y/N): N
   ```

> I've used my college NYU email id for DigitalOcean and Namecheap


3. The script will:
   - Verify `n8n.aryamandev.me` resolves to your droplet IP
   - Pull Docker images for `n8n` and `caddy`
   - Start the stack with `docker compose` in `/opt/n8n-docker-caddy`
   - Provision an SSL certificate via Let's Encrypt

4. When you see:
   ```
   Installation complete. Access your new n8n server at https://n8n.aryamandev.me
   ```
   your instance is live ✅

> ⚠️ If the script says the domain doesn't point to your IP, wait longer for DNS propagation and re-login to trigger it again.

---

## Step 4 — First-Time n8n Setup

1. Open **https://n8n.aryamandev.me** in your browser
2. Complete the n8n onboarding wizard:
   - Set an **owner email and password** (this is your n8n login)
   - Skip or configure telemetry as preferred
3. You now have a fully self-hosted n8n instance at your custom domain 🎉

---

## Step 5 — Set Up the Google Sheet

Create a Google Sheet named **Jobs** with this exact header row:

```
Job_ID | Company | Position | Status | Applied_Date | Last_Checked | Application_ID | Notes | Job_URL | Priority
```

**Example rows:**

```
JOB001 | Stripe       | Backend Engineer  | Not Applied | MM/DD/YYYY | MM/DD/YYYY | ID | ... | https://stripe.com/jobs/... | High
JOB002 | Notion       | Product Manager   | Not Applied | MM/DD/YYYY | MM/DD/YYYY | ID | ... | https://notion.so/careers/... | Medium
JOB003 | Linear       | Frontend Engineer | Not Applied | MM/DD/YYYY | MM/DD/YYYY | ID | ... | https://linear.app/careers/... | High
```

**Status values used by the workflow:**
- `Not Applied` → pending, will be picked up by the daily trigger
- `Applied` → submitted, will be checked every 2 days
- `Under Review` / `Interview Scheduled` / `Rejected` / `Offer` → terminal states

---

## Step 6 — Configure Credentials in n8n

### Google Sheets OAuth2

1. In n8n: **Settings → Credentials → Add Credential → Google Sheets OAuth2 API**
2. Follow the Google OAuth flow to authorize your Google account
3. Click **Test** to confirm read/write access to your sheet

### Gmail OAuth2

1. Add credential: **Gmail OAuth2 API**
2. Authorize n8n to send emails from your Gmail address
3. Send a test email to verify

> 💡 For production, create a **dedicated Gmail account** just for automations so your main inbox stays clean.

### LinkedIn OAuth2 (optional)

1. Register an app at [LinkedIn Developer Portal](https://www.linkedin.com/developers/)
2. Add credential: **LinkedIn OAuth2 API** in n8n
3. Use Client ID and Secret from your LinkedIn app

---

## Step 7 — Import & Configure the Workflow

### Import

Option A — **Import from this repo:**
1. Download `workflows/job-automation.json`
2. In n8n: **Workflows → Import from File**

Option B — **Import official template:**
1. Go to [n8n template #5906](https://n8n.io/workflows/5906-automated-job-applications-and-status-tracking-with-linkedin-indeed-and-google-sheets/)
2. Click **Use workflow** (requires n8n login)

### Configure the `⚙️ Configuration` Set node

Open the node and update:

| Field | Value |
|-------|-------|
| `spreadsheetId` | The ID from your Google Sheet URL: `https://docs.google.com/spreadsheets/d/THIS_PART/edit` |
| `resumeUrl` | Public Google Drive link to your resume PDF |
| `userEmail` | Email address for job application notifications |
| `coverLetterTemplate` | Your base cover letter with `{{company}}` and `{{position}}` placeholders |

### Verify all nodes

- Each **Google Sheets** node → correct credential + sheet name `Jobs`
- Each **Gmail** node → correct credential + your email
- **HTTP Request** nodes (LinkedIn/Indeed) → correct credential + endpoint

---

## Step 8 — Extending Beyond LinkedIn & Indeed

The base template routes jobs by URL domain via a Switch node. To support **any job site**, extend the Switch node with new branches:

### Supported via API (already in template)
- LinkedIn (`linkedin.com`) → LinkedIn OAuth2 API
- Indeed (`indeed.com`) → Indeed Publisher API

### Supported via web scraping (custom Code nodes)

Install the n8n Puppeteer community node:
```
Settings → Community Nodes → Install → n8n-nodes-puppeteer
```

Or use a Code node with Playwright for headless form-filling:

```javascript
// Example: fill a Greenhouse ATS application
const { chromium } = require('playwright');
const browser = await chromium.launch({ headless: true });
const page = await browser.newPage();

await page.goto($json.Job_URL);
await page.fill('#first_name', 'Aryaman');
await page.fill('#last_name', 'Dev');
await page.fill('#email', 'your@email.com');
await page.setInputFiles('#resume', '/path/to/resume.pdf');
await page.click('button[type="submit"]');

await browser.close();
return [{ success: true, appliedAt: new Date().toISOString() }];
```

### ATS platforms to add branches for

| Platform | Domain | Method |
|----------|--------|--------|
| Greenhouse | `greenhouse.io` | API or Puppeteer |
| Lever | `lever.co` | Lever API |
| Workday | `myworkdayjobs.com` | Puppeteer (complex) |
| Ashby | `ashby.com` | Puppeteer |
| Generic company page | anything else | Puppeteer |

---

## Step 9 — Schedules & Triggers

The workflow has two Cron triggers. Edit them in n8n to match your schedule:

### Trigger 1 — Daily Application (apply to new jobs)
```
0 9 * * 1-5
```
Fires at **9:00 AM, Monday–Friday**. Reads all `Not Applied` rows with `High` or `Medium` priority and submits applications.

### Trigger 2 — Status Check (check existing applications)
```
0 10 */2 * *
```
Fires at **10:00 AM every 2 days**. Reads all `Applied` rows, calls the relevant platform API, updates the sheet if status changed, and emails you.

---

## Step 10 — Server Operations & Maintenance

All Docker config lives in `/opt/n8n-docker-caddy` on the droplet.

### Everyday commands

```bash
ssh root@104.131.31.106
cd /opt/n8n-docker-caddy

# Check running containers
docker ps

# View live logs
docker compose logs -f

# Restart the stack
docker compose down && docker compose up -d
```

### Update n8n to latest version

```bash
cd /opt/n8n-docker-caddy
docker compose pull
docker compose up -d
```

### Backup the server

In the DigitalOcean panel:  
**Droplets → your droplet → Snapshots → Take Snapshot**  
Do this before any major changes. Snapshots are ~$0.05/GB/month.

### Check SSL certificate (managed by Caddy, auto-renews)

```bash
docker compose logs caddy | grep -i cert
```

---

## Repository Structure

```
.
├── README.md                        # This file
├── workflows/
│   └── job-automation.json          # Exported n8n workflow (import this)
├── docs/
│   ├── screenshots/                 # DigitalOcean, Namecheap, n8n UI screenshots
│   └── architecture.png             # Architecture diagram
└── notes/
    └── implementation-notes.md      # Personal notes, troubleshooting log
```

**To export your workflow from n8n:**  
`Workflows → click your workflow → ⋮ (three dots) → Export`  
Save the JSON into `workflows/job-automation.json` and commit it.

---

## Next Steps & Ideas

- [ ] **AI cover letters** — Add an OpenAI node to generate a unique, tailored cover letter per job based on the job description
- [ ] **Auto-scrape new jobs** — Add a separate workflow that scrapes LinkedIn/Indeed/Hacker News and populates the Google Sheet automatically every morning
- [ ] **Priority scoring** — Use AI to score and rank new job listings by fit before attempting to apply
- [ ] **Slack / Discord alerts** — Send notifications to a channel instead of (or in addition to) email
- [ ] **Resume versioning** — Dynamically swap resumes based on job type (engineering vs. PM vs. data)
- [ ] **Interview prep trigger** — When status changes to `Interview Scheduled`, trigger a workflow that pulls the company's recent news and generates a prep brief

---

## Legal Note

> Always review each platform's Terms of Service before automating applications.  
> Many job boards explicitly prohibit scraping or automated submissions.  
> This project is intended for **personal productivity use** — apply responsibly and respect rate limits.

---

## Author

**Aryaman** · [aryamandev.me](https://aryamandev.me) · [GitHub @dev4-gpt](https://github.com/dev4-gpt)  
Built with n8n, DigitalOcean, Namecheap, Google Sheets & ☕
```

***

**To push this to GitHub right now**, run in your terminal:

```bash
mkdir n8n-job-automation && cd n8n-job-automation
mkdir -p workflows docs/screenshots notes
# paste README.md here
git init
git add .
git commit -m "Initial commit: n8n self-hosted job automation setup"
gh repo create n8n-job-automation --public --push --source=.
```

(Requires the GitHub CLI `gh` — install with `brew install gh` on Mac if you don't have it.)

Sources
[1] Screenshot-2026-03-06-at-3.45.23-PM.jpeg https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/images/114099916/9934f74b-6dcb-41a3-8c26-4e8db1720850/Screenshot-2026-03-06-at-3.45.23-PM.jpeg
