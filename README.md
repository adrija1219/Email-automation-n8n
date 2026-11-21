# Automated Email Campaign Pipeline (n8n)

![n8n](https://img.shields.io/badge/Workflow%20Engine-n8n-FF6D5A?style=for-the-badge)
![Google Sheets](https://img.shields.io/badge/Data%20Store-Google%20Sheets-34A853?style=for-the-badge)
![SMTP](https://img.shields.io/badge/Delivery-SMTP-4A5568?style=for-the-badge)
![Status](https://img.shields.io/badge/Version-v1.0%20Live-2B6CB0?style=for-the-badge)

An automated, rate-limited email outreach and status-tracking pipeline built with **n8n**, **Google Sheets**, and **SMTP**. The workflow reads recipient records from a spreadsheet, validates email addresses, prevents duplicate dispatches, paces outgoing emails one by one, retries transient failures, and writes delivery state (`Sent`) back to the sheet.

---

## Visual Overview & Architecture

### 1. Live Canvas Execution
![Workflow Diagram](docs/workflow-diagram.png)

### 2. End-to-End Execution Pipeline

```text
┌──────────────────┐
│ Schedule Trigger │  Starts automatically on schedule or manual "Execute workflow"
└────────┬─────────┘
         │ (1 item)
         ▼
┌────────────────────────────┐
│ Google Sheets: Get Row(s)  │  Reads Name, Email, Status, CustomMessage from sheet
└────────┬───────────────────┘
         │ (N items)
         ▼
┌─────────────────────────────────────────────┐
│ IF Node (Validation & Duplicate Guard)      │
│ • Email contains "@"                        │
│ • Status is not equal to "Sent"             │
└────────┬──────────────────────┬─────────────┘
         │ (true)               │ (false: invalid or already "Sent" -> skipped)
         ▼                      ✖
┌──────────────────┐
│ Loop Over Items  │◀──────────────────────────┐
│ (Batch Size = 1) │                           │
└────────┬─────────┘                           │
         │ (loop)                              │
         ▼                                     │
┌────────────────────────────────────┐         │
│ Send an Email (SMTP + Retry)       │         │
│ • To: {{ $json.Email }}            │         │
│ • Subject: Hello {{ $json.Name }}  │         │
│ • Body: {{ $json.CustomMessage }}  │         │
│ • Retry On Fail: 3x (5000ms wait)  │         │
└────────┬───────────────────────────┘         │
         │                                     │
         ▼                                     │
┌────────────────────────────────────┐         │
│ Google Sheets: Update Row          │         │
│ • Matches by Email                 │         │
│ • Sets Status = "Sent"             │         │
└────────┬───────────────────────────┘         │
         │                                     │
         ▼                                     │
┌──────────────────┐                           │
│ Wait (2 Seconds) │───────────────────────────┘
└──────────────────┘  Paces delivery & returns next item to loop
```

---

## Node-by-Node Breakdown

| # | Node Name | Role in Pipeline | Key Configuration / Logic | Reliability & Engineering Purpose |
| :- | :--- | :--- | :--- | :--- |
| 1 | **Schedule Trigger** | Entry point | Scheduled interval or manual execution | Automates recurring runs without human intervention |
| 2 | **Get row(s) in sheet** | Data ingestion | Google Sheets OAuth2 (`Sheet Within Document` → `Get Row(s)`) | Centralizes contact list and per-user dynamic content |
| 3 | **If** | Filter & idempotency gate | `Email` contains `@` **AND** `Status` ≠ `Sent` | Blocks malformed addresses and prevents duplicate sends |
| 4 | **Loop Over Items** | Batch iterator | `Batch Size = 1` via `loop` output branch | Ensures controlled sequential processing instead of bursts |
| 5 | **Send an Email** | Message delivery | SMTP credential + dynamic expressions + `Retry On Fail` (`3` tries, `5000ms`) | Personalizes messages and recovers from temporary SMTP/network blips |
| 6 | **Update row in sheet** | State persistence | Matches row by `Email` and sets `Status` to `Sent` | Creates an audit trail and marks recipients as completed |
| 7 | **Wait** | Rate limiter / pacer | `After Time Interval` → `2` Seconds → loops back to **Loop Over Items** | Reduces spam-filter risk and smooths outbound traffic |

---

## Spreadsheet Schema (`Email_Campaign_List`)

Your Google Sheet uses four columns starting at cell `A1`:

| Column | Header | Type | Example Value | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **A** | `Name` | String | `Adrija` | Personalizes the email subject line (`Hello {{ $json.Name }}, ...`) |
| **B** | `Email` | String | `mitraadrija571@gmail.com` | Recipient address and row-matching key for status updates |
| **C** | `Status` | String | `Sent` or *(blank)* | Tracks delivery state; rows marked `Sent` are skipped on future runs |
| **D** | `CustomMessage` | String | `Hi Adrija, welcome...` | Supplies custom body text tailored to each recipient |

---

## Repository Structure

```text
Email-automation-n8n/
├── workflows/
│   └── Automated-Email-Campaign-Pipeline.json   # Exported n8n workflow definition
├── docs/
│   └── workflow-diagram.png                   # Screenshot of the n8n workflow canvas
├── .gitignore                                 # Ignores .env, .DS_Store, and local artifacts
└── README.md                                  # Project documentation and setup guide
```

---

## Local Setup & Reproduction Guide

### Prerequisites
* **n8n** installed locally (`npx n8n` or self-hosted instance)
* A **Google Cloud OAuth2 Client ID & Client Secret** with **Google Sheets API** and **Google Drive API** enabled
* An **SMTP account** (such as Gmail with an App Password or a transactional mail provider)

### Step-by-Step Setup

| Step | Action | Details |
| :--- | :--- | :--- |
| **1** | **Prepare Google Sheet** | Create a sheet with headers `Name`, `Email`, `Status`, `CustomMessage` and add test contacts. |
| **2** | **Import Workflow** | Open n8n → click **`...` (Menu)** → **Import from File** → select `workflows/Automated-Email-Campaign-Pipeline.json`. |
| **3** | **Connect Google Sheets** | Open both Google Sheets nodes, attach your Google OAuth2 credential, and pick your document and sheet tab. |
| **4** | **Connect SMTP** | Open **Send an Email**, attach your SMTP credential, and verify `From Email` and expression mappings. |
| **5** | **Test Safely** | Keep the workflow inactive/unpublished while testing, mark non-test rows as `Sent` if needed, and click **Execute workflow**. |

---

## Security & Privacy Practices

* **No Hardcoded Secrets in JSON:** Credentials (Google OAuth2 tokens and SMTP passwords) are stored inside n8n's credential manager rather than committed to the exported workflow JSON.
* **Ignored Sensitive Files:** `.gitignore` prevents local environment files (`.env`) and OS metadata files (`.DS_Store`) from being tracked.
* **Test Data Hygiene:** Use safe test addresses during development and verify row statuses in Google Sheets after each run.

---

## Roadmap (Planned v2 Enhancements)

| Priority | Planned Upgrade | Description |
| :--- | :--- | :--- |
| **High** | **Error Trigger Workflow** | Dedicated error-handling workflow to log failed executions and send alert notifications |
| **Medium** | **Bounce & Failure Column** | Record error reasons or `Failed` status when max retries are exhausted |
| **Medium** | **Webhook Trigger Support** | Allow external forms or apps to trigger individual emails via webhook |
| **Low** | **AI-Assisted Copy & Guardrails** | Generate personalized drafts with strict validation and static template fallbacks |

---

## Author

**Adrija Mitra**  
* GitHub: [@adrija1219](https://github.com/adrija1219)
