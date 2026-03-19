---
name: telegram-lead-crm
description: |
  Sets up a complete lead capture pipeline: React form → Express API → Telegram Bot + Google Sheets CRM.
  Includes email-gate for PDF lead magnets (HTML email via Resend API), inbound email forwarding to
  Telegram topics, Resend webhook tracking (opens/clicks/bounces), UTM tracking, deep links with source
  attribution, Yandex.Metrika goals, centralized client+server configs, and an auto-updating analytics
  dashboard in Google Sheets with 4 sheets (Leads, Dashboard, Email Events, Inbox).

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
  - Forward inbound emails to Telegram
  - Track email opens and clicks
  - Create a centralized marketing config
  - Set up Resend for transactional email

  Also trigger when the user mentions: "лид-форма", "сбор лидов", "заявки в телеграм", "CRM в гугл таблице",
  "отслеживание UTM", "воронка лидов", "email-gate", "lead magnet", "lead pipeline", "resend", "pdf скачивание",
  "входящая почта в телеграм", "email tracking", "inbound email".
---

# Telegram Lead Pipeline + Google Sheets CRM

Production-ready lead capture for React/Vite sites. Every lead flows through a secure server-side API to Telegram (instant notification with topic support), Google Sheets (CRM with auto-dashboard), and optionally email (PDF lead magnets via Resend).

## Architecture

```
                    ┌──────────────┐     ┌──────────────┐
                    │ src/config.ts│     │ api/config.ts│
                    │ (client)     │     │ (server)     │
                    └──────┬───────┘     └──────┬───────┘
                           │                    │
        ┌──────────────────┼────────┐           │
        ▼                  ▼        ▼           │
 ┌────────────┐  ┌───────────┐ ┌────────┐      │
 │ Lead Form  │  │ Email-Gate│ │All CTAs│      │
 │ (Modals)   │  │(PDF gate) │ │deeplink│      │
 └─────┬──────┘  └─────┬─────┘ └────────┘      │
       │               │                        │
       ▼               ▼                        ▼
POST /api/send-lead  POST /api/send-pdf   POST /api/inbound-email
       │               │                  POST /api/resend-webhook
       │               │                        │
       ▼               ▼                        ▼
┌───────────────────────────────────────────────────────┐
│  Express API (server-side only, secrets in api/config) │
└──┬──────────┬──────────┬──────────┬───────────────────┘
   │          │          │          │
   ▼          ▼          ▼          ▼
Telegram  Google Sheets  Email    Telegram
Bot API   Apps Script    Resend   (inbox topic)
(+topic)  (CRM+Dash)    API
```

## Two config files

The system uses **two separate config files** — one for client, one for server. This is critical for security.

### Client config: `src/config.ts`

Single source of truth for all marketing parameters visible to the browser. **No secrets here.**

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
  /** Deep link: ?start=action__source (max 64 chars) */
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

// ===== PACKAGES =====
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
const UTM_KEYS = ['utm_source', 'utm_medium', 'utm_campaign', 'utm_content', 'utm_term'] as const;

export function getUtmParams(): Record<string, string> {
  const params = new URLSearchParams(window.location.search);
  const utm: Record<string, string> = {};
  for (const key of UTM_KEYS) {
    const val = params.get(key);
    if (val) utm[key] = val;
  }
  return utm;
}

export function packageSlug(pkg: string): string {
  if (pkg.includes('Basic')) return 'basic';
  if (pkg.includes('Pro')) return 'pro';
  return 'general';
}
```

### Server config: `api/config.ts`

All secrets from `.env`, plus shared helpers for Telegram/Sheets. **Never import on client side.**

```typescript
// ===== TELEGRAM =====
export const TG = {
  botToken: process.env.TG_BOT_TOKEN || '',
  chatId: process.env.TG_CHAT_ID || '',
  threadId: process.env.TG_THREAD_ID || '',           // lead topic
  inboxThreadId: process.env.TG_INBOX_THREAD_ID || '', // inbound email topic
  apiUrl: (token: string) => `https://api.telegram.org/bot${token}/sendMessage`,
} as const;

// ===== GOOGLE SHEETS CRM =====
export const SHEETS = {
  webhook: process.env.GOOGLE_SHEET_WEBHOOK || '',
} as const;

// ===== RESEND (EMAIL) =====
export const RESEND = {
  apiKey: process.env.SMTP_PASS || '',
  webhookSecret: process.env.RESEND_WEBHOOK_SECRET || '',
  from: process.env.SMTP_FROM || 'Brand <info@example.com>',
} as const;

// ===== PDF & DEEP LINKS =====
export const CONTENT = {
  pdfUrl: 'https://...pdf',
  botDeeplink: 'https://t.me/MyBot?start=free_audit__pdf_email',
} as const;

// ===== RATE LIMITING =====
export const RATE = { maxRequests: 5, windowMs: 60 * 60 * 1000 } as const;

// ===== HELPERS =====

/** Send message to Telegram (default lead topic) */
export async function sendTelegram(text: string, parseMode: 'Markdown' | 'HTML' = 'Markdown') {
  if (!TG.botToken || !TG.chatId) return null;
  const payload: Record<string, unknown> = {
    chat_id: TG.chatId, text, parse_mode: parseMode,
  };
  if (TG.threadId) payload.message_thread_id = Number(TG.threadId);
  return fetch(TG.apiUrl(TG.botToken), {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload),
  });
}

/** Send message to a specific Telegram topic */
export async function sendTelegramToThread(text: string, threadId: string, parseMode: 'Markdown' | 'HTML' = 'HTML') {
  if (!TG.botToken || !TG.chatId || !threadId) return null;
  return fetch(TG.apiUrl(TG.botToken), {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      chat_id: TG.chatId, text, parse_mode: parseMode,
      message_thread_id: Number(threadId),
    }),
  });
}

/** Send data to Google Sheets CRM */
export async function sendToSheet(data: Record<string, unknown> | object) {
  if (!SHEETS.webhook) return;
  try {
    await fetch(SHEETS.webhook, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
  } catch { /* best-effort */ }
}
```

### Metrika helper: `src/metrika.ts`

```typescript
import { METRIKA } from './config';
export function reachGoal(goal: string, params?: Record<string, unknown>) {
  if (window.ym) window.ym(METRIKA.counterId, 'reachGoal', goal, params);
}
```

## Yandex.Metrika setup

Add counter in `index.html` `<head>`. The `<noscript>` pixel must go inside `<body>` — Vite rejects block elements in `<head>`.

Goals to instrument:

| Goal | When | Params |
|------|------|--------|
| `telegram_click` | Any Telegram link click | `{ source: 'hero' \| 'navbar' \| ... }` |
| `lead_form_open` | Form modal opens | `{ package: 'plan_name' }` |
| `lead_form_submit` | Form submitted successfully | `{ package: 'plan_name' }` |
| `pdf_download` | PDF email-gate submitted | — |
| `email_captured` | Email collected (PDF gate) | `{ source: 'pdf_gate' }` |

## Telegram deep links

Format: `?start=action__source` (max 64 chars). Use `TELEGRAM.deepLink()` from client config.

Examples:
- `TELEGRAM.deepLink('zakazat_audit', 'hero')` → `?start=zakazat_audit__hero`
- `TELEGRAM.deepLink('free_audit', 'pdf_gate')` → `?start=free_audit__pdf_gate`

Bot-side parsing (Python):
```python
parts = payload.split('__', 1)
action, source = parts[0], parts[1] if len(parts) > 1 else 'direct'
```

## API endpoints

### POST /api/send-lead
Lead form → Telegram (Markdown, with UTM/deeplink) + Google Sheets. Rate limited (5/hr/IP).

### POST /api/send-pdf
Email-gate → Resend API (branded HTML email with PDF + bot CTA) + Telegram + Sheets. Records `emailId` from Resend for tracking correlation.

### POST /api/resend-webhook
Resend webhook receiver for email events (sent, delivered, opened, clicked, bounced, complained). Logs to Sheets "Email Events" sheet with color-coding. Notifies Telegram on opens/clicks.

### POST /api/inbound-email
Resend inbound email webhook. Forwards incoming emails to a dedicated Telegram topic. Logs to Sheets "Inbox" sheet with status tracking.

## Express server

```typescript
// server.ts
import express from 'express';
import leadRouter from './api/send-lead.js';
import pdfRouter from './api/send-pdf.js';
import inboundRouter from './api/inbound-email.js';

const app = express();
app.use(express.json());
app.use(leadRouter);
app.use(pdfRouter);
app.use(inboundRouter);
app.use(express.static('dist'));
app.get('*', (_req, res) => res.sendFile('dist/index.html'));
```

Vite proxy for dev: `/api` → `localhost:3001`.

## Environment variables

```env
# Telegram
TG_BOT_TOKEN=...              # From @BotFather
TG_CHAT_ID=...                # Target group ID (e.g. -1001234567890)
TG_THREAD_ID=...              # Topic for leads (optional)
TG_INBOX_THREAD_ID=...        # Topic for inbound emails (optional)

# Google Sheets CRM
GOOGLE_SHEET_WEBHOOK=...      # Apps Script web app URL (changes on each deploy!)

# Email (Resend — free 100/day, 3000/month)
SMTP_PASS=re_your_api_key     # Resend API key
SMTP_FROM=Brand <info@example.com>
RESEND_WEBHOOK_SECRET=whsec_... # For webhook signature verification
```

### Resend setup
1. resend.com → sign up → add domain → set 3 DNS records (DKIM, SPF for `send` subdomain)
2. Set MX for `@` to `inbound-smtp.eu-west-1.resend.com.` priority 10 (for inbound email)
3. Get API key → use as `SMTP_PASS`
4. Webhooks in Resend dashboard:
   - `https://yourdomain.com/api/resend-webhook` — events: all email.* events (tracking)
   - `https://yourdomain.com/api/inbound-email` — event: email.received (inbox)
5. Enable Click Tracking + Open Tracking in domain Configuration tab

If Resend not configured, email silently skipped — TG/Sheets still work.

## Google Sheets CRM (Apps Script)

Deploy as web app (execute as me, access: anyone). The script creates 4 sheets automatically:

### Sheet "Лиды" — 23 columns
| # | Column | Auto | Description |
|---|--------|:----:|-------------|
| 1-2 | Date, Time | yes | Moscow timezone |
| 3 | Package | yes | Plan name or "PDF-презентация" |
| 4-9 | Name, Company, Industry, Phone, TG, Email | yes | Form fields |
| 10-11 | Site, Message | yes | Form fields |
| 12-13 | Deep Link, Email ID | yes | Tracking |
| 14-18 | utm_source..utm_term | yes | From URL |
| 19-20 | Referrer, Page | yes | Traffic source |
| 21 | Status | manual | Dropdown: Новый → В работе → Оплачен / Отказ |
| 22 | Manager comment | manual | Free text |
| 23 | Deal amount | manual | For revenue calc |

### Sheet "Дашборд" — auto-updates
KPIs (total leads, paid, conversion, revenue), breakdowns by status/package/source, daily chart, email tracking stats.

### Sheet "Email события" — Resend tracking
Date, Time, Event type (color-coded), Email, Email ID, Click URL, Timestamp.

### Sheet "Входящие письма" — inbound email log
Date, Time, From, To, Subject, Body preview, Attachments count, Status (dropdown: Новое/Прочитано/Отвечено/Спам).

## Telegram message formats

**Lead form:**
```
🚀 *НОВЫЙ ЛИД: Plan Name*

👤 *Имя:* John
📞 *Телефон:* +7...
🔗 *Deep link:* https://t.me/Bot?start=free_audit__plan
📊 *UTM:*  • utm_source: `yandex`
```

**PDF email-gate:**
```
📩 *PDF-ПРЕЗЕНТАЦИЯ*

👤 *Имя:* John
📧 *Email:* john@example.com
🔗 *Deep link:* https://t.me/Bot?start=free_audit__pdf_email
📬 *Email ID:* `abc123`
```

**Email tracking (on open/click):**
```
👁 Открыто *EMAIL TRACKING*

📧 *To:* john@example.com
📬 *Email ID:* `abc123`
```

**Inbound email (to inbox topic):**
```
📨 ВХОДЯЩЕЕ ПИСЬМО

📧 От: sender@example.com
📬 Кому: info@example.com
📋 Тема: Subject line
📎 Вложения: file.pdf (120 KB)

─────────────────
<body text>
```

## Post-submit flow

After successful form submission, show a success screen with CTA to Telegram bot (free audit) using tracking deep link. Same pattern for email-gate — after email is sent, show bot CTA with `free_audit__pdf_gate` deep link.

## HTML email template

Branded dark theme email with:
- Header: brand name + domain
- Personalized greeting
- Big orange CTA button → PDF download
- Separator
- Blue "bonus" block → free Telegram bot audit (with tracking deep link `free_audit__pdf_email`)
- Footer: contact info + explanation why email was sent

Use Resend SDK (`new Resend(apiKey)`) for sending — simpler than nodemailer, no SMTP config needed.

## Full funnel

```
Lead form submit                    PDF email-gate
    │                                    │
    ├─► Telegram (lead notification)     ├─► Telegram (pdf notification)
    ├─► Google Sheets (CRM row)          ├─► Google Sheets (CRM row + emailId)
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


Resend webhooks (async):            Inbound email:
    │                                    │
    ├─► email.opened/clicked             ├─► Telegram (inbox topic)
    │   ├─► Sheets "Email Events"        ├─► Sheets "Incoming"
    │   └─► Telegram notification        │
    │                                    │
    ├─► email.bounced/complained         │
    │   └─► Sheets "Email Events"        │
```

## Checklist

- [ ] `src/config.ts` with BRAND, TELEGRAM, METRIKA, ASSETS, PACKAGES, API, UTM helpers
- [ ] `api/config.ts` with TG, SHEETS, RESEND, CONTENT, RATE + helper functions
- [ ] `.env` with TG_BOT_TOKEN, TG_CHAT_ID, TG_THREAD_ID, TG_INBOX_THREAD_ID, SMTP_PASS, SMTP_FROM, RESEND_WEBHOOK_SECRET, GOOGLE_SHEET_WEBHOOK
- [ ] Metrika counter in `index.html` (noscript in body, not head)
- [ ] `src/metrika.ts` imports counterId from config
- [ ] All Telegram links use `TELEGRAM.deepLink()` from client config
- [ ] `reachGoal()` on every CTA click and form submit
- [ ] API tokens NOT in any client-side file
- [ ] Email-gate with branded HTML email via Resend SDK
- [ ] Resend domain verified (DKIM + SPF on `send` subdomain, MX on `@` for inbound)
- [ ] Resend webhooks configured (tracking + inbound)
- [ ] Click Tracking + Open Tracking enabled in Resend
- [ ] Google Apps Script deployed, URL in `.env`
- [ ] `setupSheets()` run once to create table structure
- [ ] Metrika goals created in web interface (JS event type)
- [ ] `resend` package in dependencies (not nodemailer)
