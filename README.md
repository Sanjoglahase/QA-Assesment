# Task 2 — n8n Workflow README
**Workflow file:** `Task2_Workflow_PrathameshSalunke.json`  
**Author:** Prathamesh Salunke

---

## What this workflow does

Every hour the workflow pulls the top trending AI-related Python repositories from GitHub, filters and reshapes the list to the top 5, splits them by popularity, enriches the hot picks with a live Bitcoin price from CoinGecko, and posts a formatted digest to a Discord channel.

---

## APIs used and why

| API | Endpoint | Reason |
|-----|----------|--------|
| **GitHub REST API** | `GET /search/repositories?q=topic:ai+language:python&sort=stars` | Free, no auth required for read-only search, returns rich structured data (stars, description, language, URL). Perfect for a daily tech brief. |
| **CoinGecko API** | `GET /coins/bitcoin/market_chart?vs_currency=usd&days=1` | Completely free with no API key needed for the public tier. Adds a finance data point to enrich the digest without requiring a paid subscription. |
| **Discord Webhook** | Incoming webhook URL | Easiest real-time notification channel — no bot setup required, just a URL stored as an n8n credential. |

---

## Transformation logic

The **"Code — Filter & Reshape Top 5"** node takes the raw GitHub response (up to 10 results) and:
1. Slices the array to the first 5 items.
2. Maps each repo to a lean object: `{ name, description, stars, url, language, updated }`.
3. Truncates descriptions to 120 characters to keep the digest readable.

---

## Conditional branch

The **IF — Stars >= 1000** node routes each repo:
- **True branch (≥ 1,000 stars):** Calls CoinGecko for the latest BTC/USD price, then the enrichment node appends that price to the line and labels the repo `🔥 HOT`.
- **False branch (< 1,000 stars):** Skips the second API call and labels the repo `📈 RISING`.

The 1,000-star threshold was chosen because it filters signal from noise — repos above this count have real community traction, not just a handful of stars from the creator's friends.

---

## Error handling

Three nodes have **"Continue on Error"** output wired to the **"Code — Log Error (Fallback)"** node:
- `GitHub — Search AI Repos`
- `CoinGecko — BTC 24h Chart`
- `Discord — Post Digest`

If any of them fail (network timeout, 429 rate-limit, 500 from the API), the error output is caught, serialised to JSON with a timestamp, and logged via `console.error`. This prevents silent crashes and leaves a clear trace in the n8n execution log. In a production setup the fallback node could also POST to a separate alert webhook (e.g. a dedicated `#alerts` Discord channel or a PagerDuty endpoint).

---

## Credentials setup

1. In n8n go to **Settings → Credentials → New**.
2. Create a **"Generic Credential"** named `Discord Webhook` and add a field `discordWebhookUrl` with your webhook URL.
3. The Discord node references this credential — the raw URL is never hard-coded anywhere in the workflow JSON.
4. GitHub and CoinGecko endpoints used here are unauthenticated public APIs; no credentials needed for the current query volume (well within rate limits).

---

## How to import and run

```bash
# Start n8n locally
npx n8n

# In the n8n UI:
# 1. Workflows → Import from File → select Task2_Workflow_PrathameshSalunke.json
# 2. Set up the Discord Webhook credential (see above)
# 3. Click "Execute Workflow" to run immediately, or enable the schedule trigger
```

To test with a webhook trigger instead of the scheduler, swap the Schedule node for a **Webhook** node and hit the generated URL with:

```bash
curl -X POST http://localhost:5678/webhook/<your-id>
```
