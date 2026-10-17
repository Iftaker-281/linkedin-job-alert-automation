# LinkedIn Job Alert Automation (n8n + Apify + Google Sheets)

This workflow automates a recruitment agency's daily process: it scrapes fresh LinkedIn
job postings via an Apify actor, saves them to Google Sheets, and returns a structured
JSON response to the caller — triggered by a single webhook call.

## Actor used

This workflow uses the Apify **LinkedIn Jobs Scraper** actor by Curious Coder:
`curious_coder/linkedin-jobs-scraper` ($1.00 / 1,000 results).

This actor expects a **LinkedIn search URL** as input (not raw keywords), so the
workflow builds that URL from the `jobTitle` and `location` fields before calling Apify.

## How it works

1. **Webhook** receives `jobTitle`, `location`, and `limit` from the caller.
2. **Edit Fields** normalizes the input and builds a LinkedIn search URL:
   `https://www.linkedin.com/jobs/search/?keywords=<jobTitle>&location=<location>`
3. **HTTP Request** starts the Apify actor run, sending:
   ```json
   {
     "urls": ["<searchUrl>"],
     "scrapeCompanyDetails": true,
     "count": <limit>
   }
   ```
4. **Wait (45s)** — gives the actor time to finish scraping before the first status check.
5. **HTTP Request** checks the actor run status (`GET /v2/actor-runs/{runId}`).
6. **If** node branches on `status == SUCCEEDED`:
   - **True** → continues to fetch the dataset and save to Sheets.
   - **False** (still `RUNNING`, `FAILED`, `ABORTED`, or `TIMED-OUT`) → responds
     immediately with an error instead of continuing with incomplete data.
7. **HTTP Request** fetches the scraped dataset items from Apify (on success).
8. **Code** node maps the raw scraped fields to: Job Title, Company Name, Location,
   Posted Time, Job URL, Company URL.
9. **Google Sheets** node appends each row to the `Jobs` sheet.
10. **Respond to Webhook** returns a JSON result to the caller (success or error).

## 1. Configure the Apify API token

1. Log in to [Apify Console](https://console.apify.com/) → **Settings → API & Integrations**
   and copy your **Personal API token**.
2. In n8n, create a credential of type **HTTP Query Auth**, named e.g. `Apify API`:
   - Name: `token`
   - Value: *(your Apify token)*
3. Attach this credential to all three Apify **HTTP Request** nodes: `Start Apify Actor
   Run`, `Check Run Status`, and `Get Dataset Items` (enable "Send Query Parameters" on
   each and reference the credential, or add `token` as a query parameter directly).
4. If you use a different LinkedIn scraper actor, update the actor slug in the
   `Start Apify Actor Run` node's URL (`https://api.apify.com/v2/acts/<actor-id>/runs`)
   and adjust the request body to match that actor's expected input schema.

## 2. Configure Google Sheets

1. In n8n, create a **Google Sheets OAuth2** credential and connect your Google account
   (requires a Google Cloud project with the **Google Sheets API** and **Google Drive
   API** enabled, and an OAuth 2.0 Client ID with the n8n callback URL —
   `http://localhost:5678/rest/oauth2-credential/callback` — added as an authorized
   redirect URI).
2. Create a Google Sheet with a tab named `Jobs` and header row:
   `Job Title | Company Name | Location | Posted Time | Job URL | Company URL`
3. In the `Append row in sheet` node, select that document and sheet, and map each
   column to the corresponding field from the `Process Job Data` node
   (`{{ $json.jobTitle }}`, `{{ $json.companyName }}`, etc.).

## How to trigger the webhook

Once the workflow is **activated** in n8n (top-right **Active** toggle), copy the
webhook's Production URL from the Webhook node
(e.g. `https://your-n8n-instance.com/webhook/linkedin-job-alert`) and send a `POST`
request:

```bash
curl -X POST https://your-n8n-instance.com/webhook/linkedin-job-alert \
  -H "Content-Type: application/json" \
  -d '{
    "jobTitle": "Python Developer",
    "location": "Remote",
    "limit": 20
  }'
```

For testing inside the n8n editor (without activating the workflow), click **"Listen
for test event"** on the Webhook node first, then send the request to the **Test URL**
shown there instead (`/webhook-test/...` instead of `/webhook/...`).

### Sample webhook payload

```json
{
  "jobTitle": "Python Developer",
  "location": "Remote",
  "limit": 20
}
```

### Sample success response

```json
{
  "success": true,
  "message": "LinkedIn jobs scraped and saved successfully",
  "jobsSaved": 20
}
```

### Sample failure / still-running response

```json
{
  "success": false,
  "message": "Apify actor run did not complete successfully",
  "status": "RUNNING"
}
```

## Google Sheet output

The `Jobs` sheet accumulates one row per scraped job with these columns:

| Job Title | Company Name | Location | Posted Time | Job URL | Company URL |
|---|---|---|---|---|---|
| Python Developer | DecodeUp (P) Limited | Chorasi, Gujarat, India | 2025-08-18 | linkedin.com/jobs/view/... | linkedin.com/company/... |

See `google-sheet-screenshot.png` for a screenshot of the populated sheet after a live
run (20 real LinkedIn job postings scraped and saved).

## Bonus challenge — AI summary (attempted, not included in final submission)

The assignment's bonus challenge asks for an AI node to generate a 2–3 sentence
summary of hiring trends after saving to Sheets. This was implemented and tested using
an **AI Agent** node (LangChain) connected to a Chat Model, tried with three different
providers in turn:

- **Anthropic Claude** — blocked by insufficient account credit balance.
- **Groq (Llama 3.1)** — blocked by free-tier tokens-per-minute rate limit.
- **Google Gemini** — blocked by free-tier daily request quota (20 requests/day).

Each provider's node was correctly configured (credentials, prompt referencing the
scraped job data, and connection to the AI Agent node), but none could complete a call
within their free-tier limits during testing. The node is not included in the final
exported workflow so the core, fully-working pipeline stays clean; it can be re-added
by connecting a Chat Model node (any provider) with sufficient quota to a new AI Agent
node placed between `Append row in sheet` and `Respond to Webhook`, using a prompt like:

```
Here is a list of scraped LinkedIn job postings in JSON: {{ JSON.stringify($input.all().map(i => ({title: i.json.jobTitle, company: i.json.companyName, location: i.json.location}))) }}. Write a 2-3 sentence summary highlighting: the most common hiring companies, the most common locations, and any noticeable hiring trends. Be concise.
```

## Notes / assumptions

- Apify actor field names vary between LinkedIn scraper publishers; the `Process Job
  Data` code node maps `title`, `companyName`, `location`, `postedAt`, `link`, and
  `companyLinkedinUrl` from the `curious_coder/linkedin-jobs-scraper` actor's output —
  adjust field names if you swap actors.
- The polling wait is set to 45 seconds before the first status check, which was
  sufficient in testing for this actor; actors that take longer may need a longer wait
  or an additional polling loop (Wait → Check Status → If "still running" → back to
  Wait) for very large scrape jobs.
- Failed/aborted/timed-out/still-running runs short-circuit to an error response so no
  incomplete data reaches Google Sheets — verified with live test runs (see sample
  responses above).
