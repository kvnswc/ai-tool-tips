# AI Tool Tips — Live Site Architecture

## Overview

Three separate concerns need a backend: **daily ranking updates**, **newsletter subscriptions**, and **contact/feedback form submissions**. The recommended stack is serverless-first — cheap, scalable, and minimal ops overhead.

---

## 1. Daily Ranking Updates

### How rankings stay current

A scheduled AI pipeline runs nightly to refresh the tool rankings for all 15 industries.

```
Cron trigger (daily, ~3am local)
  → Cloud Function (Python/Node)
    → For each of 15 industries:
        Call OpenAI/Claude API with a structured prompt:
        "Return the top 5 AI tools for [industry] as JSON,
         ranked by current adoption and capability. Include
         name, url, description, positives, negatives, price."
    → Validate and sanitise JSON output
    → Write results to database (Supabase Postgres or Firestore)
  → Rankings served via CDN-cached JSON endpoint:
      GET /api/rankings/{industry}

Frontend:
  → On page load / industry change, fetch /api/rankings/{industry}
  → Falls back to hardcoded data if API unavailable
  → renderTable() becomes async — otherwise unchanged
```

### Implementation notes

- Store a `rankings` table: `(industry, rank, name, url, icon, description, positives, negatives, price, updated_at)`
- Keep last 7 days of snapshots for trend tracking (future feature)
- Rate-limit the nightly job: 15 industries × 1 API call = low cost (~USD $0.05/day with GPT-4o mini)

---

## 2. Newsletter Subscriptions

Use a managed email platform rather than building from scratch.

```
Form submit → POST /api/subscribe
  → Validate email format and check for duplicates
  → Add subscriber to Kit (ConvertKit) or Resend audience via API
  → Return { success: true } to frontend
  → Frontend shows "Subscribed!" confirmation (already implemented)

Daily cron job (after ranking update):
  → Fetch today's top pick per industry from DB
  → Compose digest email using template
  → Broadcast to all active subscribers via email platform API
```

### Unsubscribe

All broadcast emails must include a one-click unsubscribe link (handled automatically by Kit/Resend).

---

## 3. Contact & Feedback Forms

```
Form submit → POST /api/contact  (or /api/feedback)
  → Validate required fields
  → Write submission to DB with timestamp and type
  → Send notification email to site owner (via Resend/SendGrid)
  → Optional: write to Notion database or Google Sheet for easy review
  → Return { success: true } to frontend
```

### Spam protection

- Honeypot hidden field (invisible to humans, filled by bots → discard)
- Rate limit by IP: max 3 submissions per hour per endpoint
- No CAPTCHA needed at low traffic volumes

---

## Recommended Stack

| Concern | Service | Cost |
|---|---|---|
| Hosting | Cloudflare Pages or Vercel | Free |
| API / Functions | Supabase Edge Functions | Free tier |
| Database | Supabase Postgres | Free tier |
| Email delivery | Resend | Free up to 3,000 emails/mo |
| Newsletter | Kit (ConvertKit) | Free up to 10,000 subscribers |
| AI ranking updates | OpenAI API (GPT-4o mini) | ~USD $0.05/day for all 15 industries |

**Total running cost at low scale: ~$0–5/month.**

---

## Migration Path from Current Static Site

The current `index.html` requires minimal changes to support a live backend.

### Step 1 — Extract data to database
- Create a `rankings` table in Supabase
- Run a one-time seed script to populate from the current JS `data` object
- Add a `updated_at` timestamp column

### Step 2 — Add API endpoint
Create a Supabase Edge Function:
```typescript
// supabase/functions/rankings/index.ts
Deno.serve(async (req) => {
  const { industry } = await req.json()
  const { data } = await supabase
    .from('rankings')
    .select('*')
    .eq('industry', industry)
    .order('rank')
  return Response.json(data)
})
```

### Step 3 — Update renderTable() to be async
```javascript
async function renderTable() {
  const industry = document.getElementById('industry-select').value
  const res = await fetch(`/api/rankings?industry=${industry}`)
  const tools = await res.json()
  // rest of function unchanged
}
```

### Step 4 — Wire up form endpoints
Add POST handlers for `/api/subscribe`, `/api/contact`, `/api/feedback` as Edge Functions.

### Step 5 — Add nightly cron job
```yaml
# .github/workflows/update-rankings.yml
on:
  schedule:
    - cron: '0 17 * * *'  # 3am AEDT (UTC+10)
jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: python scripts/update_rankings.py
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          SUPABASE_URL: ${{ secrets.SUPABASE_URL }}
          SUPABASE_KEY: ${{ secrets.SUPABASE_KEY }}
```

### Step 6 — Deploy
- Push to GitHub → Cloudflare Pages auto-deploys on every commit
- Set environment variables in Cloudflare Pages dashboard

---

## Future Enhancements

| Feature | Approach |
|---|---|
| User accounts / saved industries | Supabase Auth (magic link) |
| Ranking trend charts | Store historical snapshots, render with Chart.js |
| Tool submission / corrections | Additional feedback form category → manual review queue |
| Affiliate links | Replace tool URLs with tracked redirect links |
| Search across all tools | Supabase full-text search on tool descriptions |
