# n8n – AI-Powered Job Matcher (CV → Job Alerts → Google Sheets + Telegram)

This repository contains an **n8n workflow** that:

- Periodically fetches your **CV** from Google Drive
- Searches for **new jobs** via **SerpAPI (Google Jobs engine)**
- Uses an **LLM (Groq – Llama 3.1 8B Instant)** to:
  - Compare each job against your CV
  - Score the match (1–10)
  - Generate a short summary + reason
- Deduplicates jobs using **Google Sheets** so you don’t get notified twice for the same job
- Sends you a **Telegram notification** for each new, relevant job
- Logs all sent jobs to **Google Sheets** (with score, reason, summary, date)

---

## Files

- `job-matcher-workflow.json` – the n8n workflow export
- `README.md` – this documentation

---

## Prerequisites

You’ll need:

1. **n8n instance**
   - Cloud or self-hosted.

2. **SerpAPI account**
   - Get an API key from [SerpAPI](https://serpapi.com/).

3. **Groq account (LLM)**
   - Get an API key from [Groq](https://groq.com/).
   - The workflow uses `llama-3.1-8b-instant`.

4. **Google Drive**
   - A CV stored in Google Drive as a Google Doc.
   - The workflow downloads and converts it to plain text.

5. **Google Sheets**
   - A Google Sheet used as the job log.
   - Must have (or will auto-create) these columns:  
     `title, company, link, source, score, date, fit_reason, summary, job_id`

6. **Telegram Bot**
   - Create a bot via [BotFather](https://t.me/BotFather), get the token.
   - Get your `chat_id` (for example via sending a message to the bot and using an API call or a helper bot).

---

## What the Workflow Does (Step by Step)

1. **Schedule Trigger (`Schedule Trigger`)**
   - Runs every 24 hours (adjustable).
   - Starts:
     - The CV fetch
     - The sent-jobs load from Google Sheets

2. **Fetch CV (`Fetch CV`)**
   - Node type: Google Drive
   - Operation: `download`
   - Downloads a specific file (`fileId` in the workflow) and converts it to `text/plain`.
   - This gives your CV text that later nodes will use.

3. **Get Jobs (`Get Jobs (SerpAPI)`)**
   - Node type: HTTP Request
   - Calls `https://serpapi.com/search.json` with:
     - `engine=google_jobs`
     - `q=full stack developer OR python developer OR backend developer OR n8n developer OR AI engineer`
     - `location=United Kingdom`
     - `hl=en`
     - `api_key=<your SerpAPI key>`
   - Returns a `jobs_results` array.

4. **Prepare Job Input (`Prepare Job Input`)**
   - Node type: Code
   - Extracts each job’s:
     - `title`
     - `company_name`
     - `share_link`
     - `description`
     - `location`
     - `extensions`
     - `job_highlights`
   - Attaches the CV text to each job item so each job can be processed with the CV.

5. **Build chatInput (`Build chatInput`)**
   - Node type: Code
   - Runs once per job.
   - Builds a **prompt** (`chatInput`) that:
     - Contains truncated CV (`15000` chars) and job description (`4000` chars)
     - Instructs the LLM to return **strict JSON only** with keys:
       - `title, company, link, location, source, score, fit_reason, summary`
     - Asks the model to:
       - Score the fit (`score` 1–10)
       - Provide short `fit_reason` (≤ 150 chars)
       - Provide short `summary` (≤ 200 chars)

6. **AI Match + Summary (`AI Match + Summary`)**
   - Node type: `@n8n/n8n-nodes-langchain.agent`
   - Uses the Groq language model configured in the next node.

7. **Groq Chat Model (`Groq Chat Model`)**
   - Node type: `@n8n/n8n-nodes-langchain.lmChatGroq`
   - Model: `llama-3.1-8b-instant`
   - Temperature: `0.2` (more deterministic).
   - Provides the LLM backend for `AI Match + Summary`.

8. **Parse AI Output (`Parse AI Output`)**
   - Node type: Code
   - Extracts the raw LLM output (`output` / `text` / `response`).
   - Attempts to `JSON.parse` it.
   - Fallback-safe:
     - If parsing fails, uses defaults.
   - Computes:
     - `job_id` = `link` OR `"title | company | location"`
     - Normalized `score` = clamped between 1 and 10
     - `fit_reason` (trimmed, up to 150 chars; default `"No reason provided"`)
     - `summary` (trimmed, up to 200 chars; default `"No summary available"`)
     - `date` = `YYYY-MM-DD`
   - Output fields:
     - `title, company, link, location, source, job_id, score, fit_reason, summary, date`

9. **Load Sent Jobs (`Load Sent Jobs`)**
   - Node type: Google Sheets
   - Operation: `lookup/read` (configured as `read`/`get all rows` via `JobSheet`).
   - Reads existing jobs from the log sheet.

10. **Pass Trigger to Load Sent Jobs (`Pass Trigger to Load Sent Jobs`)**
    - Node type: Merge
    - Mode: `passThrough`
    - Just lets the Schedule Trigger also start `Load Sent Jobs`.

11. **Build Sent Set (`Build Sent Set`)**
    - Node type: Code
    - Reads all rows from Google Sheets.
    - Builds a `sentMap`:
      - Key: `job_id` (or `link`) from each row
      - Value: `true`
    - Output: one item containing `{ sentMap }`.

12. **Merge Parsed Jobs with Sent Set (`Merge Parsed Jobs with Sent Set`)**
    - Node type: Merge
    - Merges:
      - Input 1: parsed jobs from `Parse AI Output`
      - Input 2: single item from `Build Sent Set`
    - After merge, each job item has both its own fields and `sentMap`.

13. **Filter New Jobs (`Filter New Jobs`)**
    - Node type: Code
    - For each merged item:
      - Reads `sentMap` and `job_id` (or `link`)
      - If `job_id` is **not** in `sentMap`, it is considered **new**:
        - Removes `sentMap` from the item
        - Pushes the rest forward
    - Returns only **new jobs** (not yet in the sheet).

14. **Save to Google Sheets (`Save to Google Sheets`)**
    - Node type: Google Sheets
    - Operation: `append`
    - Document: your job log sheet
    - Sheet: `JobSheet`
    - Auto-maps input data columns:
      - `title, company, link, source, score, date, fit_reason, summary, job_id`
    - Each new job is appended as a new row.

15. **Send Telegram Notification (`Send Telegram Notification`)**
    - Node type: Telegram
    - Sends a formatted Markdown message:
      ```text
      =💼 *{{ $json.title }}* ({{ $json.score }}/10)
      *Company:* {{ $json.company }}
      *Summary:* {{ $json.summary }}
      *Reason:* {{ $json.fit_reason }}
      🔗 {{ $json.link }}
      ```
    - To your `chatId`.

---

## Setup Instructions

1. **Import Workflow into n8n**
   - In n8n UI:
     - `Workflows` → `Import from file`
     - Select `job-matcher-workflow.json`
   - Save the workflow.

2. **Create / Configure Credentials**
   - **Google Drive OAuth2**
     - Connect your Google account.
     - Ensure the CV document ID in `Fetch CV` matches your Google Doc.
   - **Google Sheets OAuth2**
     - Connect your Google account.
     - Ensure `documentId` and `sheetName` in:
       - `Load Sent Jobs`
       - `Save to Google Sheets`
     - Point to your job log sheet and the correct tab.
   - **SerpAPI (within HTTP Request)**
     - Replace the `api_key` query parameter with your real key or an environment variable.
   - **Groq API**
     - Create a credential for Groq with your API key.
   - **Telegram**
     - Create a Telegram credential with your bot token.
     - Replace `chatId` with your own chat ID.

3. **Adjust Schedule (Optional)**
   - In `Schedule Trigger`:
     - Currently runs every 24 hours.
     - You can change to run more or less frequently.

4. **Optional: Change Job Search Query**
   - In `Get Jobs (SerpAPI)`:
     - Edit the `q` parameter to target your own keywords and location.

---

## Security & Privacy Notes

- **Do not commit real API keys or tokens** to GitHub.
  - Use placeholders in the JSON (e.g. `{{SERPAPI_API_KEY}}`).
  - Configure real values via n8n credentials and environment variables.
- CV and job data are processed by:
  - SerpAPI (for job listings)
  - Groq (for LLM scoring/matching)
- Review and adapt prompts/fields if you want to minimize sensitive data sent to the LLM.

---

## Customization Ideas

- Filter only jobs above a given score (e.g. `score >= 7`) before sending Telegram messages.
- Add more destinations:
  - Email
  - Slack
- Change scoring logic or schema in `Build chatInput` and `Parse AI Output`.

---

If you’d like, I can:

- Rewrite the JSON with secrets explicitly templated out for public sharing, or  
- Add a short “Quick Start” section at the top of the README tailored to your exact stack (Docker / n8n Cloud, etc.).
