---
name: telegram-lead-crm
description: |
  Sets up a complete lead capture pipeline: React form → Express API → Telegram Bot + Google Sheets CRM.
  Includes email-gate for PDF lead magnets (with HTML email via Resend/SMTP), UTM tracking, deep links
  with source attribution, Yandex.Metrika goals, centralized marketing config, and an auto-updating
  analytics dashboard in Google Sheets.

  USE THIS SKILL when the user wants to:
  - Add a lead form or contact form that sends to Telegram
  - Set up a CRM or lead tracking system
  - Connect a website to Telegram for lead notifications
  - Add a lead magnet or capture funnel
  - Create an email-gate for PDF downloads
  - Track UTM parameters and conversion sources
  - Build analytics for a landing page
  - Integrate Google Sheets as a lightweight CRM
  - Add Yandex.Metrika goals to a React site
  - Send transactional emails from a landing page
  - Create a centralized marketing config

  Also trigger when the user mentions: "лид-форма", "сбор лидов", "заявки в телеграм", "CRM в гугл таблице",
  "отслеживание UTM", "воронка лидов", "email-gate", "lead magnet", "lead pipeline", "resend", "pdf скачивание".
---

# Telegram Lead Pipeline + Google Sheets CRM

Production-ready lead capture for React/Vite sites. Every lead flows through a secure server-side API to Telegram (instant notification), Google Sheets (CRM with dashboard), and optionally email (PDF lead magnets).

## Architecture

```
                    ┌─────────────────┐
                    │  src/config.ts  │  Central marketing config
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                   ▼
  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐
  │  Lead Form   │  │  Email-Gate   │  │  All CTAs    │
  │  (Modals)    │  │  (PDF gate)   │  │  deep links  │
  └──────┬───────┘  └──────┬────────┘  └──────────────┘
         │                 │
         ▼                 ▼
  POST /api/send-lead   POST /api/send-pdf
         │                 │
         ▼                 ▼
  ┌────────────────────────────────────┐
  │  Express API (server-side only)    │
  │  Secrets in .env, never in client  │
  └──────┬─────────┬─────────┬────────┘
         │         │         │
         ▼         ▼         ▼
   Telegram   Google Sheets  Email
   Bot API    Apps Script    (Resend SMTP)
   (+ topic)  (CRM+Dashboard)
```

## Step 1: Centralized config

Create `src/config.ts` — single source of truth for all marketing parameters on the client side. Server secrets stay in `.env`.

```typescript
// ===== BRAND =====
export const BRAND = {
  name: 'MyBrand',
  domain: 'example.com',
  url: 'https://example.com',
  owner: 'Owner Name',
  phone: '+7 ...',
  email: 'info@example.com',
} as const;

// ===== TELEGRAM =====
export const TELEGRAM = {
  botUsername: 'MyBot',
  botUrl: 'https://t.me/MyBot',
  deepLink: (action: string, source: string) =>
    `https://t.me/MyBot?start=${action}__${source}`,
} as const;

// ===== YANDEX.METRIKA =====
export const METRIKA = {
  counterId: 12345678,
  goals: {
    telegramClick: 'telegram_click',
    leadFormOpen: 'lead_form_open',
    leadFormSubmit: 'lead_form_submit',
    pdfDownload: 'pdf_download',
    emailCaptured: 'email_captured',
  },
} as const;

// ===== ASSETS =====
export const ASSETS = {
  pdfUrl: 'https://...pdf',
} as const;

// ===== PACKAGES / PRICING =====
export const PACKAGES = {
  basic: { name: 'Basic', price: '...', slug: 'basic' },
  pro:   { name: 'Pro',   price: '...', slug: 'pro' },
} as const;

// ===== API ENDPOINTS =====
export const API = {
  sendLead: '/api/send-lead',
  sendPdf: '/api/send-pdf',
} as const;

// ===== UTM HELPERS =====
export function getUtmParams(): Record<string, string> {
  const params = new URLSearchParams(window.location.search);
  const utm: Record<string, string> = {};
  for (const key of ['utm_source', 'utm_medium', 'utm_campaign', 'utm_content', 'utm_term']) {
    const val = params.get(key);
    if (val) utm[key] = val;
  }
  return utm;
}

export function packageSlug(pkg: string): string {
  // Map display name → slug for deep links
  if (pkg.includes('Basic')) return 'basic';
  if (pkg.includes('Pro')) return 'pro';
  return 'general';
}
```

All components import from config instead of hardcoding values. The metrika helper uses `METRIKA.counterId`:

```typescript
// src/metrika.ts
import { METRIKA } from './config';
export function reachGoal(goal: string, params?: Record<string, unknown>) {
  if (window.ym) window.ym(METRIKA.counterId, 'reachGoal', goal, params);
}
```

## Step 2: Yandex.Metrika

Add counter script in `index.html` `<head>`. The `<noscript>` pixel must go inside `<body>`, not `<head>` — Vite's HTML parser rejects block elements in `<head>`.

Goals to instrument:

| Goal | When | Params |
|------|------|--------|
| `telegram_click` | Any Telegram link click | `{ source: 'hero' \| 'navbar' \| ... }` |
| `lead_form_open` | Form modal opens | `{ package: 'plan_name' }` |
| `lead_form_submit` | Form submitted successfully | `{ package: 'plan_name' }` |
| `pdf_download` | PDF email-gate submitted | — |
| `email_captured` | Email collected (PDF gate) | `{ source: 'pdf_gate' }` |

## Step 3: Telegram deep links with source tracking

Every bot link uses format `?start=action__source` (max 64 chars). Use `TELEGRAM.deepLink()` helper from config.

Examples:
- `TELEGRAM.deepLink('zakazat_audit', 'hero')` → `?start=zakazat_audit__hero`
- `TELEGRAM.deepLink('free_audit', 'pdf_gate')` → `?start=free_audit__pdf_gate`
- `TELEGRAM.deepLink('free_audit', packageSlug(leadPackage))` → `?start=free_audit__basic`

Bot-side parsing (Python):
```python
parts = payload.split('__', 1)
action, source = parts[0], parts[1] if len(parts) > 1 else 'direct'
```

## Step 4: Lead form

Form collects user data + automatic analytics via config helpers:

```typescript
import { API, TELEGRAM, getUtmParams, packageSlug } from '../config';

const handleSubmit = async () => {
  const slug = packageSlug(leadPackage);
  await fetch(API.sendLead, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      package: leadPackage,
      name, company, phone, telegram, site, message,
      deepLink: TELEGRAM.deepLink('free_audit', slug),
      utm: getUtmParams(),
      referrer: document.referrer || null,
      page: window.location.href,
    })
  });
};
```

After success, show CTA to Telegram bot with tracking deep link.

## Step 5: Email-gate for PDF lead magnets

Replace direct PDF download links with a name+email form. On submit:

1. **Server sends HTML email** with PDF download button + CTA to Telegram bot
2. **Lead recorded** in Telegram (notification) + Google Sheets (CRM row with package "PDF-презентация")
3. **Metrika goals** fired: `email_captured` + `pdf_download`
4. **Success screen** shows CTA to bot with deep link `free_audit__pdf_gate`

Create `api/send-pdf.ts` endpoint that sends email via SMTP (Resend recommended), notifies Telegram, and writes to Sheets — all in parallel.

**HTML email template** should include:
- Branded header with logo/name
- Personalized greeting
- Big CTA button to download PDF
- Secondary CTA block inviting to free Telegram bot audit (with tracking deep link `free_audit__pdf_email`)
- Footer with contact info

Use `nodemailer` for SMTP transport. Dynamic import so the server doesn't crash if nodemailer isn't installed — email sending gracefully degrades while TG/Sheets still work.

## Step 6: Express API server

**Critical:** All secrets (tokens, SMTP credentials) in `.env` only, never in client code.

Two routers:
- `api/send-lead.ts` — Lead form → Telegram + Sheets
- `api/send-pdf.ts` — Email-gate → Email + Telegram + Sheets

Both use rate limiting and send to all destinations in parallel.

```typescript
// server.ts
import leadRouter from './api/send-lead.js';
import pdfRouter from './api/send-pdf.js';
app.use(express.json());
app.use(leadRouter);
app.use(pdfRouter);
```

Vite proxy for dev: `/api` → `localhost:3001`.

## Step 7: Environment variables

```env
# Telegram
TG_BOT_TOKEN=...              # From @BotFather
TG_CHAT_ID=...                # Target group ID
TG_THREAD_ID=...              # Topic ID (optional)

# Google Sheets CRM
GOOGLE_SHEET_WEBHOOK=...      # Apps Script web app URL

# Email (Resend recommended — free 100/day)
SMTP_HOST=smtp.resend.com
SMTP_PORT=465
SMTP_USER=resend
SMTP_PASS=re_your_api_key     # Resend API key
SMTP_FROM=Brand <info@example.com>
```

**Resend setup** (simplest free option):
1. resend.com → sign up
2. Add domain → set 3 DNS records (DKIM, SPF, MX)
3. Get API key → use as SMTP_PASS with `SMTP_USER=resend`

If SMTP not configured, email silently skipped — TG/Sheets still work.

## Step 8: Google Sheets CRM (Apps Script)

Sheet "Leads" — 21 columns:

| # | Column | Auto | Description |
|---|--------|:----:|-------------|
| 1-2 | Date, Time | yes | Moscow timezone |
| 3 | Package | yes | Plan name or "PDF-презентация" for email-gate |
| 4-10 | Name, Company, Industry, Phone, TG, Site, Message | yes | Form fields |
| 11 | Deep Link | yes | Bot link with source |
| 12-16 | utm_source..utm_term | yes | From URL |
| 17-18 | Referrer, Page | yes | Traffic source |
| 19 | Status | manual | Dropdown: New → In progress → Paid / Rejected |
| 20 | Manager comment | manual | Free text |
| 21 | Deal amount | manual | For revenue calc |

Sheet "Dashboard" — auto-updates with KPIs, breakdowns by status/package/source, daily chart.

## Telegram message formats

**Lead form:**
```
🚀 *NEW LEAD: Plan Name*
👤 *Name:* John
📞 *Phone:* +7...
🔗 *Deep link:* https://t.me/Bot?start=free_audit__plan
📊 *UTM:*  • utm_source: `yandex`
```

**PDF email-gate:**
```
📩 *PDF-ПРЕЗЕНТАЦИЯ*
👤 *Name:* John
📧 *Email:* john@example.com
🔗 *Deep link:* https://t.me/Bot?start=free_audit__pdf_email
📊 *UTM:*  • utm_source: `yandex`
```

## Full funnel

```
Lead form submit                    PDF email-gate
    │                                    │
    ├─► Telegram (lead notification)     ├─► Telegram (pdf notification)
    ├─► Google Sheets (CRM row)          ├─► Google Sheets (CRM row)
    ├─► Metrika (lead_form_submit)       ├─► Email with PDF + bot CTA
    │                                    ├─► Metrika (email_captured + pdf_download)
    ▼                                    ▼
Success screen                      Success screen
    │                                    │
    ▼                                    ▼
CTA: "Free audit in bot"           CTA: "Free audit in bot"
    │                                    │
    ▼                                    ▼
Telegram bot                        Telegram bot
(deep link: free_audit__<pkg>)      (deep link: free_audit__pdf_gate)
```

## Checklist

- [ ] `src/config.ts` with all brand/telegram/metrika/assets/api constants
- [ ] `.env` with TG_BOT_TOKEN, TG_CHAT_ID, SMTP_*, GOOGLE_SHEET_WEBHOOK
- [ ] Metrika counter in `index.html`, helper imports from config
- [ ] All Telegram links use `TELEGRAM.deepLink()` from config
- [ ] `reachGoal()` on every CTA click and form submit
- [ ] API tokens NOT in any client-side file
- [ ] Email-gate for PDF with branded HTML email
- [ ] Resend configured (domain DNS + API key in .env)
- [ ] Google Apps Script deployed, URL in `.env`
- [ ] `setupSheets()` run once to create table structure
- [ ] Metrika goals created in web interface (JS event type)
- [ ] `nodemailer` in dependencies
