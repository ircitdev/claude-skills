---
name: telegram-lead-crm
description: |
  Sets up a complete lead capture pipeline: React form → Express API → Telegram Bot + Google Sheets CRM.
  Includes UTM tracking, deep links with source attribution, Yandex.Metrika goals, and an auto-updating
  analytics dashboard in Google Sheets.

  USE THIS SKILL when the user wants to:
  - Add a lead form or contact form that sends to Telegram
  - Set up a CRM or lead tracking system
  - Connect a website to Telegram for lead notifications
  - Add a lead magnet or capture funnel
  - Track UTM parameters and conversion sources
  - Build analytics for a landing page
  - Integrate Google Sheets as a lightweight CRM
  - Add Yandex.Metrika goals to a React site

  Also trigger when the user mentions: "лид-форма", "сбор лидов", "заявки в телеграм", "CRM в гугл таблице",
  "отслеживание UTM", "воронка лидов", "lead magnet", "lead pipeline".
---

# Telegram Lead Pipeline + Google Sheets CRM

This skill creates a production-ready lead capture system for React/Vite sites. Every lead flows through
a secure server-side API to both Telegram (instant notification) and Google Sheets (CRM with dashboard).

## Architecture

```
User fills form (React)
    │
    │  Collects: form fields + UTM + deep link + referrer + page URL
    │
    ▼
POST /api/send-lead (Express, server-side)
    │
    ├─► Telegram Bot API (Markdown message, optional topic/thread)
    └─► Google Sheets (Apps Script webhook, best-effort)
```

## Step-by-step implementation

### 1. Yandex.Metrika helper

Create `src/metrika.ts` — a thin wrapper around `window.ym` for type-safe goal tracking:

```typescript
declare global {
  interface Window {
    ym?: (...args: unknown[]) => void;
  }
}

const COUNTER_ID = __METRIKA_ID__;  // Replace with actual counter ID

export function reachGoal(goal: string, params?: Record<string, unknown>) {
  if (window.ym) {
    window.ym(COUNTER_ID, 'reachGoal', goal, params);
  }
}
```

Add the Metrika counter script to `index.html` `<head>`:

```html
<script type="text/javascript">
  (function(m,e,t,r,i,k,a){
    m[i]=m[i]||function(){(m[i].a=m[i].a||[]).push(arguments)};
    m[i].l=1*new Date();
    for (var j = 0; j < document.scripts.length; j++) {if (document.scripts[j].src === r) { return; }}
    k=e.createElement(t),a=e.getElementsByTagName(t)[0],k.async=1,k.src=r,a.parentNode.insertBefore(k,a)
  })(window, document,'script','https://mc.yandex.ru/metrika/tag.js?id=__METRIKA_ID__', 'ym');

  ym(__METRIKA_ID__, 'init', {
    clickmap: true, trackLinks: true, accurateTrackBounce: true,
    webvisor: true, trackHash: true
  });
</script>
```

The `<noscript>` pixel must go inside `<body>`, not `<head>` — Vite's HTML parser rejects block elements in `<head>`.

Standard goals to instrument:

| Goal | When | Params |
|------|------|--------|
| `telegram_click` | Any Telegram link click | `{ source: 'hero' \| 'navbar' \| ... }` |
| `lead_form_open` | Form modal opens | `{ package: 'plan_name' }` |
| `lead_form_submit` | Form submitted successfully | `{ package: 'plan_name' }` |
| `pdf_download` | PDF download click | — |

### 2. Telegram deep links with source tracking

Every bot link uses the format `?start=action__source`. Telegram allows up to 64 characters in the start parameter.

Examples:
- `?start=zakazat_audit__hero` — order from hero section
- `?start=sos_audit__navbar` — urgent audit from navbar
- `?start=free_audit__razvedka` — free audit after submitting "Razvedka" form
- `?start=contact__footer` — contact from footer

Bot-side parsing (Python):
```python
payload = message.text.split(' ', 1)[1] if ' ' in message.text else ''
parts = payload.split('__', 1)
action = parts[0]
source = parts[1] if len(parts) > 1 else 'direct'
```

### 3. Lead form component

The form component collects user data plus automatic analytics. Key pattern:

```typescript
// Capture UTM from current URL
const getUtmParams = () => {
  const params = new URLSearchParams(window.location.search);
  const utm: Record<string, string> = {};
  for (const key of ['utm_source', 'utm_medium', 'utm_campaign', 'utm_content', 'utm_term']) {
    const val = params.get(key);
    if (val) utm[key] = val;
  }
  return utm;
};

// On submit, send structured data (NOT a pre-formatted string)
const handleSubmit = async () => {
  await fetch('/api/send-lead', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      package: selectedPlan,
      name, company, phone, telegram, site, message,  // form fields
      deepLink: `https://t.me/${BOT_USERNAME}?start=free_audit__${planSlug}`,
      utm: getUtmParams(),
      referrer: document.referrer || null,
      page: window.location.href,
    })
  });
  reachGoal('lead_form_submit', { package: selectedPlan });
};
```

After successful submission, show a CTA to the Telegram bot for a free audit — this converts warm leads into bot users with full source attribution via the deep link.

### 4. Express API server

**Critical security rule:** Telegram Bot API token and Chat ID must NEVER be in client-side code. They go in `.env` and are only accessed server-side.

Create `api/send-lead.ts`:

```typescript
import { Router } from 'express';

const router = Router();
const API_TOKEN = process.env.TG_BOT_TOKEN || '';
const CHAT_ID = process.env.TG_CHAT_ID || '';
const THREAD_ID = process.env.TG_THREAD_ID || '';
const SHEET_WEBHOOK = process.env.GOOGLE_SHEET_WEBHOOK || '';

router.post('/api/send-lead', async (req, res) => {
  const body = req.body;
  const text = formatMessage(body);  // Build Markdown message

  // Send to Telegram and Google Sheet in parallel
  const [tgResponse] = await Promise.all([
    sendToTelegram(text),
    sendToSheet(body),   // Best-effort, doesn't block
  ]);

  if (tgResponse.ok) return res.json({ ok: true });
  return res.status(502).json({ ok: false });
});
```

The `formatMessage` function builds a Markdown message with all fields, UTM, deep link, and referrer. The `sendToTelegram` function uses `message_thread_id` if `TG_THREAD_ID` is set (for topic support in groups).

Create `server.ts` for production:

```typescript
import express from 'express';
import leadRouter from './api/send-lead.js';

const app = express();
app.use(express.json());
app.use(leadRouter);
app.use(express.static('dist'));
app.get('*', (_req, res) => res.sendFile('dist/index.html'));
app.listen(process.env.PORT || 3001);
```

Add Vite proxy for dev mode in `vite.config.ts`:

```typescript
server: {
  proxy: {
    '/api': { target: 'http://localhost:3001', changeOrigin: true }
  }
}
```

### 5. Environment variables

```env
TG_BOT_TOKEN=...              # Telegram bot token (from @BotFather)
TG_CHAT_ID=...                # Target chat/group ID
TG_THREAD_ID=...              # Topic ID in group (optional)
GOOGLE_SHEET_WEBHOOK=...      # Google Apps Script web app URL
```

Add to `.gitignore`: `.env*` with `!.env.example`.

### 6. Google Sheets CRM (Apps Script)

Create a Google Apps Script web app that receives POST requests and writes to a spreadsheet.

**Sheet "Leads" columns (21 total):**

| # | Column | Auto | Description |
|---|--------|:----:|-------------|
| 1-2 | Date, Time | yes | Moscow timezone |
| 3 | Package | yes | Plan name |
| 4-10 | Name, Company, Industry, Phone, TG, Site, Message | yes | Form fields |
| 11 | Deep Link | yes | Bot link with source |
| 12-16 | utm_source, utm_medium, utm_campaign, utm_content, utm_term | yes | From URL |
| 17-18 | Referrer, Page | yes | Traffic source |
| 19 | Status | manual | Dropdown: New → In progress → Negotiation → Invoice → Paid / Rejected |
| 20 | Manager comment | manual | Free text |
| 21 | Deal amount | manual | For revenue calculation |

**Sheet "Dashboard"** — auto-updates on every new lead and on manual edits:
- KPIs: total leads, paid, conversion %, revenue
- Breakdown by status, package, utm_source
- Leads by day (last 14 days)

The Apps Script `doPost(e)` function parses the JSON body, writes a row, and updates the dashboard. A `setupSheets()` function creates the structure with a test lead on first run.

Deployment: Extensions → Apps Script → Deploy → Web app (Execute as: Me, Access: Anyone).

### 7. Package.json scripts

```json
{
  "scripts": {
    "dev": "vite --port=3000",
    "server": "tsx --env-file=.env server.ts",
    "build": "vite build"
  }
}
```

Dev requires two terminals: `npm run dev` + `npm run server`.
Production: `npm run build && npm run server`.

## Telegram message format

```
🚀 *NEW LEAD: Plan Name*

👤 *Name:* John
🏢 *Company:* Acme Corp
📞 *Phone:* +7...
🌐 *Site:* https://example.com

🔗 *Deep link:* https://t.me/Bot?start=free_audit__plan

📊 *UTM:*
  • utm_source: `yandex`
  • utm_medium: `cpc`
↩️ *Referrer:* https://yandex.ru/
```

## Post-submit funnel

```
Form submitted OK
    → Telegram notification (with UTM + deep link)
    → Google Sheets CRM row
    → Metrika goal
    → Success screen with CTA: "Free express audit in our bot"
        → Deep link: ?start=free_audit__<package>
        → Bot sees source, can personalize onboarding
```

## Checklist

- [ ] `.env` with TG_BOT_TOKEN, TG_CHAT_ID
- [ ] Metrika counter in `index.html`, helper in `src/metrika.ts`
- [ ] All Telegram links use `?start=action__source`
- [ ] `reachGoal()` on every CTA click and form submit
- [ ] API token NOT in any client-side file
- [ ] Google Apps Script deployed, URL in `.env`
- [ ] `setupSheets()` run once to create table structure
- [ ] Metrika goals created in web interface (JS event type)
