# Setup Guide

This guide walks through importing the workflows into n8n, configuring credentials, and setting up the supporting Google Sheets structure. Follow the steps in order — skipping the error handler setup means you'll have no visibility into failures from the other five workflows.

---

## Prerequisites

### Accounts Required

| Service | Purpose | Notes |
|---------|---------|-------|
| n8n Cloud (or self-hosted) | Workflow engine | Cloud: `n8n.io`. Self-hosted requires Node.js 20+ |
| OpenAI | GPT-4.1-mini for evaluation and drafting | API key from `platform.openai.com` |
| Google Workspace | Sheets, Docs, Gmail | Service account or OAuth credentials |
| WordPress | Publishing target | REST API enabled, application password created |
| Form platform (optional) | Webhook intake trigger | GravityForms, Typeform, or any platform that supports HMAC-signed webhooks |

### n8n Version
These workflows were built on n8n Cloud (April 2026). The Langchain AI nodes used in Phase 2 require n8n `1.x` or later. Check your version at **Settings → About**.

---

## Step 1: Configure Credentials in n8n

Set up all credentials before importing workflows. n8n will prompt you to map credentials during import — having them ready prevents needing to re-open each workflow after import.

Navigate to **Credentials** in your n8n instance and create the following:

### OpenAI API
- Type: `OpenAI`
- Name: `OpenAI Production`
- API Key: your key from `platform.openai.com/api-keys`
- Base URL: leave default (`https://api.openai.com/v1`)

### Google Sheets / Google Docs / Gmail
These can share one Google OAuth credential or use separate service accounts.

**Option A — OAuth (recommended for Cloud):**
1. Go to Google Cloud Console → APIs & Services → OAuth 2.0 Client IDs
2. Enable: Google Sheets API, Google Docs API, Gmail API
3. Scopes needed: `spreadsheets`, `documents`, `gmail.send`, `gmail.readonly`
4. In n8n: Credentials → New → Google OAuth2 → paste Client ID and Secret → authorize

**Option B — Service Account (recommended for self-hosted):**
1. Create a service account in Google Cloud Console
2. Enable the same three APIs
3. Download the JSON key file
4. In n8n: Credentials → New → Google Service Account → paste JSON
5. Share the target Sheets and Drive folders with the service account email

Name these credentials:
- `Google Sheets Bot`
- `Google Docs Bot`
- `Gmail Automation` (or use a single shared Google OAuth credential for all three)

### WordPress
1. In WordPress admin: Users → Your Profile → Application Passwords → Add New
2. Name it `n8n Automation` and copy the generated password
3. In n8n: Credentials → New → WordPress → enter your site URL, username, and application password
4. Name it: `WordPress Client Account`

### Webhook HMAC Secret
The form intake workflow (`05-form-intake.json`) verifies HMAC-SHA256 signatures on incoming webhooks. You will need the shared secret that your form platform uses when signing requests. Store it as an environment variable or directly in the Code node (see note in the workflow file).

---

## Step 2: Set Up Google Sheets

You need two sheets — a master article table and a bug log. Create both in Google Sheets before importing the workflows.

### Master Article Table

Create a new Google Sheet. The first sheet (tab) should be named `Articles` (or update the node configurations to match your tab name). Add these column headers in row 1, in this exact order:

```
A: URL
B: TITLE
C: SOURCE
D: SUBMITTED DATE
E: STATUS
F: ADMIN ACTION
G: STATUS LOG
H: CONTENT
I: LINK TO GOOGLE DOC
J: LINK TO WORDPRESS BACKEND
K: GENERATED DATE
L: NOTES
```

`ADMIN ACTION` (column F) is the trigger column for the content pipeline. When an editor types `Approve` in this column, the `04-content-pipeline.json` workflow fires.

Copy the Sheet ID from the URL (the long string between `/d/` and `/edit`) — you'll need it when configuring workflow nodes.

### Bug Log Sheet

Create a second Google Sheet named `Bug Log`. Column headers in row 1:

```
A: Date
B: Time
C: Workflow Name
D: Error Type
E: Error Message
F: Node
G: Priority
H: Status
I: Assigned To
J: Resolution Notes
```

---

## Step 3: Import Workflows

**Import order matters.** The error handler must be imported first because the other five workflows reference it by workflow ID. If you import them in the wrong order, you'll need to update each workflow's error handler setting after the fact.

### Import Order

1. `00-error-handler.json`
2. `01-editorial-gate.json`
3. `02-rss-intake.json`
4. `03-document-receiver.json`
5. `04-content-pipeline.json`
6. `05-form-intake.json`

### How to Import

1. In n8n: **Workflows → New → Import from File**
2. Select the JSON file
3. After import, n8n will flag any unresolved credentials — map each to the credentials you created in Step 1
4. Save the workflow (do not activate yet)
5. Repeat for all six files

### After Import: Update Sheet IDs

The workflow files contain `[SHEETS_ID]` placeholders where the Google Sheets document IDs belong. After importing, open each workflow and replace these placeholders with your actual Sheet IDs:

- `00-error-handler.json` → Bug Log sheet ID
- `01-editorial-gate.json` → Master article table ID
- `02-rss-intake.json` → Master article table ID
- `03-document-receiver.json` → Master article table ID
- `04-content-pipeline.json` → Master article table ID
- `05-form-intake.json` → Master article table ID

### Configure Error Handler Routing

After all six workflows are imported:

1. Note the Workflow ID of `00-error-handler.json` (visible in the URL when you open it: `/workflows/[ID]`)
2. Open each of the other five workflows
3. Click the workflow settings (gear icon or `...` menu) → **Error Workflow**
4. Select `00-error-handler.json` from the dropdown
5. Save

---

## Step 4: Configure RSS Sources

Open `02-rss-intake.json`. The RSS feed URLs are stored in the Schedule Trigger configuration and HTTP Request nodes. Replace the `[INDUSTRY_NEWS_SOURCE_*]` placeholder URLs with your actual RSS feed URLs.

The scoring algorithm weights in the Code node are commented with their purpose — adjust the threshold (`0.3` by default) and signal weights to match your editorial standard.

### Hard Exclusion Keywords

The scoring Code node contains an array of hard-exclusion keywords. Articles matching any of these are rejected regardless of score. Update this list to reflect topics that are off-strategy for your target publication.

---

## Step 5: Configure the Content Pipeline Prompt

Open `04-content-pipeline.json`. The AI generation prompt is in the OpenAI node (or HTTP Request node if you're using the direct API call pattern). The system prompt contains:

- Audience definition (who the article is written for)
- Tone and style guidelines
- Structural requirements (H2 headings, minimum word count, etc.)
- Format constraints (no em-dashes, active voice, etc.)

Update this prompt to match your client's brand voice and editorial style guide before activating.

---

## Step 6: Configure Webhook Intake

If you're using the form intake workflow (`05-form-intake.json`):

1. Open the workflow and activate it — this generates a live webhook URL
2. Copy the webhook URL from the Webhook node
3. Paste it into your form platform's webhook configuration
4. Set the form platform to sign requests with HMAC-SHA256
5. Copy the shared secret and update the HMAC verification Code node (first node after the Webhook trigger) with your secret value

To test without a real form submission: use a tool like Postman or `curl` to send a POST request to the webhook URL with a valid HMAC signature.

---

## Step 7: Activate Workflows

Activate in this order:

1. `00-error-handler.json` — activate first so it's ready to receive errors from day one
2. `01-editorial-gate.json`
3. `02-rss-intake.json`
4. `03-document-receiver.json`
5. `05-form-intake.json`
6. `04-content-pipeline.json` — activate last; this is the workflow that creates Docs and WordPress posts

---

## Step 8: Test the Pipeline

### Test Phase 0 (Editorial Gate)
1. Add a URL to column A of your master sheet
2. Wait up to 30 minutes for the scheduled check (or manually execute the workflow in n8n to test immediately)
3. Check column E (STATUS) — should show `PASS — Ready for review` or `FAIL — [reason]`

### Test Phase 3+4+5 (Content Pipeline)
1. Find a PASS row in your master sheet
2. Type `Approve` in column F (ADMIN ACTION)
3. The workflow should trigger within 1 minute of the sheet update (Google Sheets trigger polls every 60 seconds by default)
4. Check columns I (LINK TO GOOGLE DOC) and J (LINK TO WORDPRESS BACKEND) — both should populate within ~45 seconds of trigger

### Test Error Handler
1. Temporarily introduce a bad URL in any workflow (e.g., a malformed Google Sheet ID)
2. Manually execute that workflow
3. Check the Bug Log sheet — an error row should appear
4. If the error count is under 3 for that workflow in the last 30 minutes, a Gmail alert should also arrive

---

## Common Issues

**Workflow triggers but nothing happens**
- Check that the error handler workflow ID is correctly set in all five other workflows
- Open Executions in n8n and look for the failed run — the error message will be specific

**Google Sheets write fails with "Quota exceeded"**
- The retry logic (3 attempts, 3s wait) handles most transient quota errors
- If failures persist, check Google Cloud Console → Quotas for your Sheets API usage
- Adding a second Google account's credentials and routing some workflows through it distributes the quota load

**WordPress draft not created**
- Verify the application password has the correct permissions (Editor or Administrator role)
- Check that REST API is not blocked by a security plugin (Wordfence, iThemes Security)
- Test the WordPress API directly: `curl -u username:app_password https://[YOUR_SITE]/wp-json/wp/v2/posts`

**HMAC verification failing on webhook**
- Confirm the shared secret in the Code node matches what your form platform uses
- Some platforms encode the raw body before signing; others sign the parsed JSON. Check your platform's documentation for the exact signing method.

**pairedItem errors in loop nodes**
- This surfaces as `Cannot read properties of undefined` in status update nodes
- Ensure every Code node inside a `splitInBatches` loop returns items with `pairedItem: { item: $itemIndex }` on each output object

---

## Environment Variables (Self-Hosted Only)

If running n8n self-hosted, add these to your `.env` file:

```env
# n8n base config
N8N_HOST=0.0.0.0
N8N_PORT=5678
N8N_PROTOCOL=https
WEBHOOK_URL=https://your-n8n-instance.com

# Optional: execution data retention
EXECUTIONS_DATA_MAX_AGE=168
EXECUTIONS_DATA_PRUNE=true
```

All API keys (OpenAI, Google, WordPress) are stored in n8n's built-in Credential Store — do not put them in environment variables or `.env` files.
