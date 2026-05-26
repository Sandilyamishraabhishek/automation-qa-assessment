# Automation & QA Developer — Take-Home Assessment

Candidate: Abhishek Sandilya Mishra  
GitHub: [@Sandilyamishraabhishek](https://github.com/Sandilyamishraabhishek)  
Email: sandilyamishraabhishek@gmail.com  
Submission Date: 25 May 2026

---

## 📁 Repository Structure

| File | Description |
|------|-------------|
| Task1_QA_Report_Candidate.pdf | Task 1 — Bug table + root-cause analysis |
| Task2_Workflow_Candidate.json | Task 2 — n8n Morning Brief workflow export |
| Task2_README_Candidate.pdf | Task 2 — README (APIs, transformation, error handling) |
| Bonus_UptimeMonitor_Candidate.json | Bonus — n8n Uptime Monitor workflow export |
| Documentation_Candidate.pdf | Overall submission summary |
| README.md | This file |

---

## ✅ Task 1 — Web App QA & Debug Report

App tested: [RealWorld Conduit Demo](https://demo.realworld.io) (Angular frontend + Node/Express API)

Testing approach:
- Happy-path flows: registration, login, article CRUD, comments, favourites, profile, logout
- Boundary & negative input: blank fields, 500+ char strings, special characters, emoji
- Security probing: XSS payloads, browser storage inspection, unauthenticated API calls, HTTP headers
- Accessibility: keyboard-only navigation, ARIA inspection via axe DevTools
- API-level: direct HTTP requests testing rate limiting, status codes, validation

### Bugs Found

| # | Title | Severity |
|---|-------|----------|
| 1 | JWT stored in localStorage — XSS token theft risk | 🔴 Critical |
| 2 | No rate limiting on login endpoint — brute-force possible | 🔴 Critical |
| 3 | Stored XSS via Markdown article body *(root-cause analysed)* | 🔴 Critical |
| 4 | Double-click on Publish creates duplicate articles | 🟠 High |
| 5 | Pagination renders when all articles fit on one page | 🔵 Medium |
| 6 | Missing ARIA labels on icon-only buttons (WCAG 2.1 violation) | 🔵 Medium |
| 7 | Non-existent profile returns HTTP 200 with empty UI | 🔵 Medium |
| 8 | Tag field accepts arbitrary-length input, breaks tag cloud | 🟢 Low |

Root-cause analysis was written for Issue 3 (Stored XSS). The vulnerability arises because the Angular frontend uses bypassSecurityTrustHtml() to allow Markdown rendering, which simultaneously disables Angular's XSS guard. The marked parser also lacks a sanitisation step.

Fix requires:
1. Server-side sanitize-html on ingest
2. Client-side DOMPurify before [innerHTML] binding
3. Strict Content-Security-Policy: script-src 'self' header

Full details in Task1_QA_Report_Candidate.pdf.

---

## ✅ Task 2 — n8n API Integration Workflow

Workflow: Morning Brief — GitHub AI Repos  
File: Task2_Workflow_Candidate.json

### Flow Overview

    Schedule Trigger (Every 1h)
        │
        ▼
    GitHub Search — AI Repos
    GET /search/repositories?q=topic:ai&sort=stars
        │
        ▼
    Transform — Top 5 Repos  (Code node)
    Sort by stars ▸ Slice top 5 ▸ Set highStars flag
        │
        ▼
    IF — Stars > 1,000
        │
        ├── TRUE (🔥 Hot) ──────────────────────────────────┐
        │   Enrich: GET /repos/{name}                       │
        │   → open_issues, watchers, licence                │
        │   → Merge Hot Enrichment                          │
        │                                                   ▼
        └── FALSE (📌 Regular) ──────────────────► Merge All Results
            Enrich: GET /repos/{name}/readme                │
            → readme_exists, readme_size                    ▼
            → Merge Regular Enrichment          Build Discord Message
                                                            │
                                                            ▼
                                                Send to Discord Webhook

    ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
    Error Trigger (any node fails)
        │
        ▼
    Format Error Message
        │
        ▼
        | API | Why |
|-----|-----|
| GitHub REST API v3 | Free with PAT (5,000 req/h), rich sortable data, no extra sign-up |
| Discord Incoming Webhook | Zero-cost, instant, team-visible, Markdown-formatted, no SMTP needed |

### Key Design Decisions

- Star threshold = 1,000 — cleanly separates established AI projects from new/niche ones
- **continueOnFail: true** on all HTTP nodes — transient failures pass downstream rather than crasError Triggerer** — any unhandled exception posts a structured alert to Discord within secondNo hardcoded secretsts** — GitHub token and Discord webhook URL stored in n8n Credential store

### How to Import

1. Open your n8n instance
2. GoWorkflows → Import from filele**
3. Select Task2_Workflow_Candidate.json
4. Create two credentials:
 GitHub Token Headerer** → HTTP Header Auth → Authorization: Bearer <your-pat>
 Discord Webhook URLRL** → HTTP Query Auth → paste your webhook URL
5. Activate the workflow

Full documentation in Task2_README_Candidate.pdf.

---

## ✅ Bonus — Uptime MonitWorkflow:w:** Uptime Monitor — demo.realworld.iFile:e:** Bonus_UptimeMonitor_Candidate.json

### Flow Overview

    Schedule Trigger (Every 5 min)
        │
        ▼
    Initialise Ping State
    (attempt = 1, pingStart = Date.now())
        │
        ▼
    Ping — GET demo.realworld.io
        │
        ▼
    Evaluate Response
    (statusCode, responseTimeMs, ok, slow)
        │
        ├── IF Should Retry? (ok=false AND attempt < 3)
        │       │
        │       ├── YES → Wait 15s → Increment Attempt → Ping again (loop)
        │       │
        │       └── NO  → IF Alert Needed? (ok=false OR slow=true)
        │                       │
        │                       ├── YES → Build Alert Message → Send to Discord
        │                       └── NO  → (silent pass, all good)
        │
        └── Log Result to Static Data (always runs)
                (rolling 288-check window, uptime %, avg response time)

    ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
    Schedule Trigger (Daily 08:00)
        │
        ▼
    Build Daily Summary (reads staticData)
        │
        ▼
    Send Daily Summary to Discord

    ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
    Error Trigger → Format Workflow Error → Send to Discord

### Features

| Feature | Implementation |
|---------|----------------|
| Ping every 5 minutes | Schedule trigger (5-min interval) |
| HTTP status check | neverError: true + fullResponse to capture any status code |
| Response-time tracking | Date.now() delta; alerts if > 3,000 ms |
| Retry logic | 3 attempts with 15s Wait node between retries |
| Discord alert | Fires on non-200 OR slow response after all retries exhausted |
| Rolling statistics | staticData stores last 288 checks (24 h window) |
| Daily summary | Separate 08:00 cron → reads stored stats → uptime % + avg response time → Discord |
| Error handling | continueOnFail on HTTP nodes + Error Trigger for workflow-level failures |

### How to Import

1. Open your n8n instance
2. GoWorkflows → Import from filele**
3. Select Bonus_UptimeMonitor_Candidate.json
4. Link the sDiscord Webhook URLRL** credential used in Task 2
5. Activate both schedule triggers (5-min ping + 08:00 daily summary)

---

## 🛠️ Tools & StackQA Testing:g:** Manual browser testing, axe DevTools, curl / PostmaWorkflow automation:n:** n8n (self-hostedAPIs:s:** GitHub REST API v3, Discord Incoming WebhookReport generation:n:** Python (ReportLabAI assistance:e:** Claude (Anthropic) — used for structuring and drafting; all logic and decisions are my own

---

## 📋 Submission Checklist

- [x] Task1_QA_Report_Candidate.pdf — Bug table (5 issues) + root-cause analysis
- [x] Task2_Workflow_Candidate.json — n8n workflow export
- [ ] - [x] Task2_README_Candidate.pdf — Workflow documentation
- [x] Bonus_UptimeMonitor_Candidate.json — Uptime monitor workflow export
- [x] Documentation_Candidate.pdf — Overall submission summary
- [x] README.md — This file
- [ ] Loom video — walkthrough of workflow canvas + one execution *(add link here)*

---

*Submission for the Automation & QA Developer Take-Home Assessment.*
    Send Error Alert to Discord  ← never silent

### APIs Used
