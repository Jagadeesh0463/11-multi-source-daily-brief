The output file system is blocking new file creation right now. Here is the complete README — copy everything between the lines and save as `README.md` in your GitHub folder:

---

```markdown
# Problem 11 — Multi-Source Daily Brief
n8n Automation Portfolio | The Testing Academy

---

## Problem Statement

People want summaries, not more raw inputs. Microsoft data shows nonstop interruption, and a Reddit founder described replacing hours of channel-reading with an automatically generated 5-minute daily synthesis from Slack and Notion.

Modern teams drown in raw information across multiple tools. The real problem is not access to information — it is the absence of synthesis.

This workflow pulls data from 4 sources every morning, uses AI to cluster by topic, and delivers a clean 5-minute brief before you open your laptop.

---

## Core Concept — Signal Over Noise

This is NOT a fetch-news-and-email-it automation.

| Aggregation (wrong) | Synthesis (this workflow) |
|---|---|
| Dump all Slack messages | Extract only what matters |
| Summarize each source separately | Cluster topics across sources |
| List everything | Separate action items from FYI |
| Any length | Hard cap — 5-minute read |

If Slack and Notion both mention the same project, they appear as ONE bullet — not two. That cross-source clustering is what makes this a synthesis engine.

---

## Is This RAG?

No. This is NOT RAG.

| Criteria | RAG | This Workflow |
|---|---|---|
| Trigger | User query | Scheduled time |
| Vector DB | Required | Not used |
| Embeddings | Required | Not used |
| Retrieval | Semantic search | Direct API fetch |
| LLM role | Answer a question | Summarize and cluster |
| Output | Query answer | Daily digest |

RAG solves: I do not know, let me search my knowledge base.
This solves: Here is everything that happened — what matters today?

---

## Solution Architecture

Type: Single Workflow — 13 nodes — 1 trigger

Why single workflow? No ingestion phase, no vector database, no semantic search. Straight pipeline — trigger, fetch, merge, clean, synthesize, deliver.

```
Schedule Trigger (8AM daily)
         |
+--------+--------+----------+
Sheets  Gmail  Notion  Slack      <- Parallel fetch, all 4 simultaneously
+---+---+              +---+---+
[Merge A]              [Merge B]  <- Sheets+Gmail | Notion+Slack
    +----------+----------+
           [Merge C]              <- All 4 combined
                |
     [Code - Clean & Check]       <- Normalize, guard empty, format
                |
     [Groq LLM - Synthesize]      <- Topic cluster + HTML output
                |
       [Gmail - Send Brief]       <- Deliver to inbox
                |
    [Notion - Archive Brief]      <- Save dated page
```

Why 3 Merge nodes? n8n Merge node reliably handles 2 inputs at a time.
- Merge A: Sheets + Gmail
- Merge B: Notion + Slack
- Merge C: Merge A + Merge B = all 4 combined

---

## Node-by-Node Breakdown

### Node 1 — Schedule Trigger
- Cron: 0 8 * * * (every day 8AM)
- Timezone: Asia/Kolkata IST
- Fires entire workflow automatically

### Node 2 — Google Sheets - Get Updates
- Operation: Read Rows
- Filter: Source column = Sheets
- Options: Return all matches
- Data fields: Team, Update, Priority

### Node 3 — Gmail - Get Newsletters
- Operation: Get All Messages
- Limit: 10
- Filter: category:updates newer_than:1d
- Data fields: subject, snippet

### Node 4 — Notion - Get Updated Pages
- Resource: Database Page
- Operation: Get All
- Database: Team Updates
- Return All: true
- Data fields: property_task_name, property_status, property_due

### Node 5 — Slack - Get Messages
- Operation: Search
- Query: API
- Auth: OAuth2
- Data fields: text, ts

### Node 6 — Merge A (Sheets + Gmail)
- Mode: Append
- Combines Sheets and Gmail into one stream

### Node 7 — Merge B (Notion + Slack)
- Mode: Append
- Combines Notion and Slack into one stream

### Node 8 — Merge C (All Sources)
- Mode: Append
- Combines Merge A and Merge B = all 4 sources

### Node 9 — Code - Clean & Check
Language: JavaScript — 4 jobs in one node

Job 1 — Guard empty:
If all sources return nothing, throw error and stop cleanly.

Job 2 — Detect source by data shape (no tag needed):
- Slack: has data.text AND data.ts
- Notion: has data.property_task_name
- Gmail: has data.snippet
- Sheets: has data.Update

Job 3 — Clean each item:
- Strip HTML tags
- Remove URLs → [link]
- Skip noise: greetings, under 5 chars, emoji-only
- Truncate to 200 chars max

Job 4 — Build LLM input:
- Format: [SOURCE] content
- Cap total at 12000 chars (~3000 tokens)
- Returns: combinedText, today (IST), itemCount

Sample Code node output:
```
[SHEETS] Engineering: API deployment completed (Priority: High)
[NOTION] Vendor contract review | Status: Not started | Due: 2026-06-12
[GMAIL] Weekly digest: Three key updates from your team this week
[SLACK] Security patch applied to production server
```

### Node 10 — Basic LLM Chain

User prompt:
```
Date: {today}
Total Updates: {itemCount}
{combinedText}
```

System prompt:
```
You are a professional executive assistant creating a morning briefing.

Inputs are tagged: [SLACK], [NOTION], [GMAIL], [SHEETS].

TASK:
1. CLUSTER by TOPIC not by source.
2. Separate ACTION ITEMS from informational updates.
3. SUPPRESS noise and repeated information.
4. OUTPUT clean HTML for email.

FORMAT:
<h2>Daily Brief — {date}</h2>
<h3>Action Items</h3>
<ul><li><strong>[Topic]</strong> — action needed [SOURCE]</li></ul>
<h3>Key Signals</h3>
<ul><li><strong>[Topic]</strong> — insight [SOURCE]</li></ul>
<h3>FYI</h3>
<ul><li>Low priority [SOURCE]</li></ul>

RULES:
- Max 500 words.
- Cite only sources that explicitly mention that topic.
- Never invent information.
- Return only HTML.
```

### Node 11 — Groq Chat Model (Sub-node)
- Model: llama-3.3-70b-versatile
- Connects to LLM Chain via ai_languageModel

### Node 12 — Gmail - Send Brief
- Operation: Send
- To: RECIPIENT_EMAIL
- Subject: Daily Brief — {today}
- Body: HTML from LLM
- Fallback: if LLM output empty, sends "No significant updates today"

### Node 13 — Notion - Archive Brief
- Resource: Database Page
- Operation: Create
- Database: Daily Briefs Archive
- Title: Brief — {today}
- Properties: Date (today), Status (Sent)
- Block: Paragraph with HTML stripped to plain text

---

## Sample Output

### Gmail (rendered HTML)
```
Daily Brief — Tuesday, 9 June 2026

Action Items
- Vendor Contract Review — review and sign before deadline [NOTION, GMAIL]
- Marketing Campaign — approval needed on final assets [NOTION, SLACK, SHEETS]
- Enterprise Client Proposal — pricing proposal due today [SHEETS, SLACK]

Key Signals
- Security Patch — critical patch applied to production [SHEETS, SLACK]
- Refund Requests — increased 15% this week [SHEETS, SLACK]

FYI
- Instagram campaign launched as scheduled [SHEETS, SLACK]
- API deployment completed successfully [NOTION, SLACK]
```

### Notion Archive (plain text)
```
Daily Brief — Tuesday, 9 June 2026

Action Items
- Vendor Contract Review — review and sign [NOTION, GMAIL]
- Marketing Campaign — approval needed [NOTION, SLACK, SHEETS]

Key Signals
- Security Patch — critical patch applied [SHEETS, SLACK]

FYI
- Instagram campaign launched [SHEETS, SLACK]
```

---

## Setup Instructions

### Step 1 — Import Workflow
Open n8n → Import → Select 11-multi-source-daily-brief.json

### Step 2 — Google Sheets Credential
Settings → Credentials → New → Google Sheets OAuth2 → Authenticate

### Step 3 — Gmail Credential
Settings → Credentials → New → Gmail OAuth2 → Authenticate
Same credential used for both Gmail Fetch and Gmail Send nodes

### Step 4 — Notion Credential
Go to https://www.notion.so/my-integrations → New integration → Select workspace → Copy token
n8n → Credentials → New → Notion API → paste token

### Step 5 — Slack Credential
Go to https://api.slack.com/apps → Create app → OAuth and Permissions
Scopes needed: search:read, channels:history → Install → Copy Bot Token
n8n → Credentials → New → Slack OAuth2 → paste token

### Step 6 — Groq Credential
Go to https://console.groq.com → API Keys → Create
n8n → Credentials → New → Groq API → paste key

### Step 7 — Google Sheets Setup
Create spreadsheet with columns: Team, Update, Priority, Source
Set Source column = Sheets for rows to include
Sheet ID is the long string in the URL between /d/ and /edit

### Step 8 — Notion Source Database
Create database with columns: Name (Title), Status (Select), Due (Date)
Grant integration access: open database → three dots → Connections → add integration

### Step 9 — Notion Archive Database
Create database: Daily Briefs Archive
Columns: Name (Title), Date (Date), Status (Select: Draft / Sent / Archived)
Grant integration access same way
In Archive node: Resource = Database Page, Operation = Create, select this database

### Step 10 — Set Recipient Email
Open Gmail Send Brief node → replace RECIPIENT_EMAIL with your email

### Step 11 — Test Run
Click Execute Workflow manually
Verify all 13 nodes green
Check inbox for brief
Check Notion Archive for saved page

### Step 12 — Activate
Toggle workflow to Active
Brief runs every day at 8:00 AM automatically

---

## Credentials Reference

| Placeholder | Replace With |
|---|---|
| GOOGLE_SHEETS_CREDENTIAL_ID | n8n credential ID for Google Sheets |
| GOOGLE_SHEETS_CREDENTIAL_NAME | Credential name in n8n |
| GOOGLE_SHEETS_DOCUMENT_ID | Sheet ID from URL |
| GOOGLE_SHEETS_DOCUMENT_URL | Full spreadsheet URL |
| GOOGLE_SHEETS_SHEET_URL | Sheet tab URL |
| GMAIL_CREDENTIAL_ID | n8n credential ID for Gmail |
| GMAIL_CREDENTIAL_NAME | Gmail credential name in n8n |
| GMAIL_FETCH_WEBHOOK_ID | Auto-generated on import |
| GMAIL_SEND_WEBHOOK_ID | Auto-generated on import |
| NOTION_CREDENTIAL_ID | n8n credential ID for Notion |
| NOTION_CREDENTIAL_NAME | Notion credential name in n8n |
| NOTION_TEAM_UPDATES_DATABASE_ID | Source Notion database ID |
| NOTION_TEAM_UPDATES_DATABASE_URL | Source Notion database URL |
| NOTION_DAILY_BRIEFS_ARCHIVE_DATABASE_ID | Archive database ID |
| NOTION_DAILY_BRIEFS_ARCHIVE_DATABASE_URL | Archive database URL |
| SLACK_CREDENTIAL_ID | n8n credential ID for Slack |
| SLACK_CREDENTIAL_NAME | Slack credential name in n8n |
| SLACK_WEBHOOK_ID | Auto-generated on import |
| GROQ_CREDENTIAL_ID | n8n credential ID for Groq |
| GROQ_CREDENTIAL_NAME | Groq credential name in n8n |
| RECIPIENT_EMAIL | Your email to receive brief |
| N8N_INSTANCE_ID | Your n8n instance ID |

---

## Node Summary Table

| # | Node | Type | Purpose |
|---|---|---|---|
| 1 | Schedule Trigger | Trigger | 8AM daily |
| 2 | Google Sheets - Get Updates | App | Fetch KPI rows |
| 3 | Gmail - Get Newsletters | App | Fetch 24h emails |
| 4 | Notion - Get Updated Pages | App | Fetch task updates |
| 5 | Slack - Get Messages | App | Search messages |
| 6 | Merge A - Sheets + Gmail | Core | Combine 1 and 2 |
| 7 | Merge B - Notion + Slack | Core | Combine 3 and 4 |
| 8 | Merge C - All Sources | Core | Combine all 4 |
| 9 | Code - Clean & Check | Logic | Normalize and format |
| 10 | Basic LLM Chain | AI | Synthesize and format |
| 11 | Groq Chat Model | AI Sub | Llama 3.3 backend |
| 12 | Gmail - Send Brief | App | Deliver to inbox |
| 13 | Notion - Archive Brief | App | Save dated page |

Total: 13 nodes — 1 workflow — 1 trigger

---
## Repository Structure

```
11-multi-source-daily-brief/
|-- 11-multi-source-daily-brief.json
|-- README.md
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| n8n 2.17.7 | Workflow automation |
| Google Sheets | KPI data source |
| Gmail | Email data source |
| Notion | Task data source |
| Slack | Message data source |
| Groq Llama 3.3 70B | AI synthesis |
| Docker | Local n8n hosting |

---

## Author

S Jagadeesh
