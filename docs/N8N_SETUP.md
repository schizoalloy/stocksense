# n8n Webhook Setup

The StockSense dashboard talks to n8n via **6 webhook endpoints**. Three trigger your existing sub-workflows (WF-1, WF-2, WF-3); three read data from Google Sheets.

This doc walks you through creating each one. Every workflow is tiny — most are 2–3 nodes.

## Overview

| Method | Path                                             | Purpose                                 |
| ------ | ------------------------------------------------ | --------------------------------------- |
| POST   | `/webhook/stocksense/run-forecast`               | Fires WF-1 (Demand Forecasting)         |
| POST   | `/webhook/stocksense/run-supplier-contact`       | Fires WF-2 (Supplier Outreach)          |
| POST   | `/webhook/stocksense/run-newsletter`             | Fires WF-3 (Supplier Discovery)         |
| GET    | `/webhook/stocksense/forecasts`                  | Returns forecast_results rows           |
| GET    | `/webhook/stocksense/contact-logs`               | Returns supplier_contact_logs rows      |
| GET    | `/webhook/stocksense/discovered-suppliers`       | Returns supplier_discovery_log rows     |

---

## Part A — Trigger workflows

Each trigger workflow is exactly 3 nodes:

```
Webhook (POST) → Execute Workflow → Respond to Webhook
```

### A1. Run Forecast — `POST /webhook/stocksense/run-forecast`

1. Create a new workflow in n8n, name it **"API — Run Forecast"**.
2. Add a **Webhook** node:
   - HTTP Method: `POST`
   - Path: `stocksense/run-forecast`
   - Response Mode: **Using 'Respond to Webhook' Node**
3. Add an **Execute Workflow** node:
   - Source: **Database**
   - Workflow: **WF-1 Demand Forecasting** (select your existing sub-workflow)
   - Mode: **Run once for each item**
4. Add a **Respond to Webhook** node:
   - Respond With: **JSON**
   - Response Body: `={{ $json }}` (this passes through whatever WF-1 returns)
5. Connect: Webhook → Execute Workflow → Respond to Webhook.
6. **Activate** the workflow (top-right toggle).

Repeat this pattern for the other two triggers, changing only the path and the target sub-workflow:

### A2. Run Supplier Contact — `POST /webhook/stocksense/run-supplier-contact`

- Webhook path: `stocksense/run-supplier-contact`
- Execute Workflow target: **WF-2 Supplier Contact**

### A3. Run Newsletter — `POST /webhook/stocksense/run-newsletter`

- Webhook path: `stocksense/run-newsletter`
- Execute Workflow target: **WF-3 Supplier Discovery Newsletter**

---

## Part B — Data read endpoints

Each data endpoint is 3 nodes:

```
Webhook (GET) → Google Sheets (Read) → Respond to Webhook
```

### B1. Forecasts — `GET /webhook/stocksense/forecasts`

1. New workflow, name it **"API — Read Forecasts"**.
2. **Webhook** node:
   - HTTP Method: `GET`
   - Path: `stocksense/forecasts`
   - Response Mode: **Using 'Respond to Webhook' Node**
3. **Google Sheets** node:
   - Operation: **Read**
   - Document ID: `1cHZt75v-q95s9JxVbUFbhoRlM9_MVv1vCtXSNahPO34`
   - Sheet Name: `forecast_results`
4. **Respond to Webhook**:
   - Respond With: **JSON**
   - Response Body: `={{ $input.all().map(i => i.json) }}`
5. Connect: Webhook → Google Sheets → Respond to Webhook.
6. **Activate**.

### B2. Contact Logs — `GET /webhook/stocksense/contact-logs`

Same shape as B1 but:
- Webhook path: `stocksense/contact-logs`
- Google Sheets tab: `supplier_contact_logs`

### B3. Discovered Suppliers — `GET /webhook/stocksense/discovered-suppliers`

Same shape but:
- Webhook path: `stocksense/discovered-suppliers`
- Google Sheets tab: `supplier_discovery_log`

---

## Part C — Testing each endpoint

Once activated, test each webhook from the terminal:

```bash
# Base URL — change to match your n8n instance
BASE=http://localhost:5678

# Test data endpoints (should return JSON arrays)
curl $BASE/webhook/stocksense/forecasts | head -c 500
curl $BASE/webhook/stocksense/contact-logs | head -c 500
curl $BASE/webhook/stocksense/discovered-suppliers | head -c 500

# Test trigger endpoints (will take minutes to complete)
curl -X POST $BASE/webhook/stocksense/run-forecast \
  -H "Content-Type: application/json" \
  -d '{"runAt":"2026-04-24T12:00:00Z"}'
```

Expected responses:
- **Data endpoints** → JSON array like `[{"product_id": "P0001", ...}, ...]`
- **Trigger endpoints** → JSON object like `{"wf1_status": "success", "summary_message": "...", ...}`

---

## Part D — Passing the runAt timestamp to sub-workflows

If your sub-workflows reference `$json.runAt` or `$json.runId` in their logic, add a **Code** node between the Webhook and Execute Workflow in the trigger workflows (A1–A3):

```javascript
const body = $input.first().json.body || {};
return [{
  json: {
    runAt: body.runAt || new Date().toISOString(),
    runId: Math.floor(Math.random() * 100000).toString(),
    sheetId: '1cHZt75v-q95s9JxVbUFbhoRlM9_MVv1vCtXSNahPO34',
    simulationEmail: 'muhammadali63251@gmail.com',
  }
}];
```

This gives the sub-workflow a clean config object to work with.

---

## Part E — Troubleshooting

**"Webhook not registered" error**
- Make sure the workflow is activated (not just saved).
- Check the webhook URL — it's shown in the Webhook node after activation.

**Dashboard shows empty data**
- Hit the GET webhook directly in your browser. If it returns `[]`, the Google Sheets tab is empty.
- If it returns an error, the credentials on the Google Sheets node are expired. Re-authenticate.

**CORS errors in browser console**
- The UI never hits n8n directly from the browser — it goes through Next.js API routes. If you see CORS errors, you're calling n8n URLs from the client directly somewhere. Check you're using `/api/trigger/...` and `/api/data/...` paths.

**Trigger returns 504 / times out**
- Increase n8n's `N8N_PAYLOAD_SIZE_MAX` and `EXECUTIONS_TIMEOUT` env vars.
- The forecast workflow can take 8+ minutes. The Next.js proxy keeps the connection open; make sure your reverse proxy (nginx, Cloudflare) has a long enough timeout too.

---

## Part F — Security (before going public)

For the hackathon demo, webhooks can be open. If you deploy this publicly:

1. In each Webhook node, enable **Authentication** → **Header Auth**.
2. Add a secret token, e.g. `X-API-Key: supersecret123`.
3. Add this to your Next.js `.env.local`:
   ```
   N8N_WEBHOOK_AUTH=username:password
   ```
   (or use Basic Auth instead of header auth — adjust `lib/n8n.ts` accordingly).

That's it. Once all 6 webhooks are active, the dashboard will light up with real data.
