# Навык: Telegram Lead Pipeline + Google Sheets CRM

## Что это

Готовый навык для Claude Code, который автоматизирует создание полной системы сбора лидов:

```
Форма на сайте  ─┐
                  ├──► Express API ──► Telegram + Google Sheets + Email
Email-gate PDF ──┘
```

Включает: централизованный конфиг маркетинга, UTM-трекинг, deep links, Яндекс.Метрику, email-gate для PDF с отправкой писем через Resend, CRM-дашборд.

## Быстрая установка

```bash
# В конкретный проект
cp -r skills/telegram-lead-crm /path/to/project/.claude/skills/

# Или глобально
cp -r skills/telegram-lead-crm ~/.claude/skills/

# Или добавить в CLAUDE.md проекта:
# - telegram-lead-crm: ./skills/telegram-lead-crm/SKILL.md
```

## Как использовать

После установки просто попросите Claude:

- "Добавь форму сбора лидов с отправкой в Telegram"
- "Настрой CRM в гугл таблице для лидов с сайта"
- "Сделай email-gate для скачивания PDF"
- "Подключи Яндекс.Метрику и отслеживание UTM"
- "Add a lead magnet with Telegram bot integration"

## Что создаётся

| Файл | Назначение |
|------|------------|
| `src/config.ts` | Единый конфиг: бренд, Telegram, Метрика, тарифы, API, UTM-хелперы |
| `src/metrika.ts` | Хелпер Яндекс.Метрики `reachGoal()` |
| `api/send-lead.ts` | Express-роутер: лид-форма → Telegram + Sheets |
| `api/send-pdf.ts` | Express-роутер: email-gate → Email + Telegram + Sheets |
| `server.ts` | Express-сервер для продакшена |
| `google-apps-script.js` | Apps Script для Google Sheets CRM |
| `.env` | Секреты (токены, SMTP, webhook) |

## Централизованный конфиг (`src/config.ts`)

Все параметры маркетинга в одном месте:

| Секция | Что содержит |
|--------|-------------|
| `BRAND` | Название, домен, телефон, email |
| `TELEGRAM` | botUsername, botUrl, `deepLink()` хелпер |
| `METRIKA` | counterId, все имена целей |
| `ASSETS` | URL PDF, изображений |
| `PACKAGES` | Тарифы: имя, цена, slug |
| `API` | Все серверные endpoints |
| `getUtmParams()` | Парсинг UTM из URL |
| `packageSlug()` | Имя тарифа → slug |

## Email-gate для PDF

Вместо прямой ссылки на PDF — форма "Имя + Email". После отправки:

1. Красивое HTML-письмо с кнопкой скачивания PDF + CTA на бота
2. Лид в Telegram (уведомление с UTM и deep link)
3. Строка в Google Sheets CRM (package: "PDF-презентация")
4. Метрика: `email_captured` + `pdf_download`
5. Экран успеха с CTA на бесплатный аудит в боте

## Что отслеживается

### В каждом лиде

- Данные формы (имя, телефон, email, сайт, компания)
- Тариф/пакет
- UTM-метки из URL
- Deep link на Telegram-бота с источником
- Referrer, URL страницы

### Цели Метрики

- `telegram_click` — клик на ссылку бота (с параметром source)
- `lead_form_open` — открытие формы
- `lead_form_submit` — успешная отправка
- `pdf_download` — скачивание PDF через email-gate
- `email_captured` — email собран

### Deep links

Формат: `?start=действие__источник`

```text
?start=zakazat_audit__hero      — заказ с главного экрана
?start=free_audit__razvedka     — после формы "Разведка"
?start=free_audit__pdf_gate     — после email-gate на сайте
?start=free_audit__pdf_email    — из письма с PDF
```

## Переменные окружения

```env
# Telegram
TG_BOT_TOKEN=...              # Токен бота от @BotFather
TG_CHAT_ID=...                # ID чата/группы
TG_THREAD_ID=...              # ID топика (необязательно)

# Google Sheets
GOOGLE_SHEET_WEBHOOK=...      # URL Apps Script web app

# Email (Resend — бесплатно 100 писем/день)
SMTP_HOST=smtp.resend.com
SMTP_PORT=465
SMTP_USER=resend
SMTP_PASS=re_ваш_api_key
SMTP_FROM=Brand <info@example.com>
```

## Архитектура

```text
┌──────────────────┐   ┌──────────────────┐
│  Lead Form       │   │  Email-Gate PDF  │
│  (Modals.tsx)    │   │  (DownloadSect.) │
└────────┬─────────┘   └────────┬─────────┘
         │                      │
         ▼                      ▼
   /api/send-lead         /api/send-pdf
         │                      │
         ▼                      ▼
┌──────────────────────────────────────────┐
│           Express API (server.ts)        │
│  ┌───────────┐ ┌──────────┐ ┌─────────┐ │
│  │ Telegram  │ │ Google   │ │ Email   │ │
│  │ Bot API   │ │ Sheets   │ │ (Resend)│ │
│  └───────────┘ └──────────┘ └─────────┘ │
└──────────────────────────────────────────┘

  ┌───────────────┐     ┌────────────────┐
  │ Яндекс.Метрика│     │  src/config.ts │
  │ reachGoal()   │     │  единый конфиг │
  └───────────────┘     └────────────────┘
```

## Требования

- React + Vite (или любой SPA-фреймворк)
- Node.js (Express + nodemailer)
- Telegram-бот (@BotFather)
- Google-аккаунт (Sheets + Apps Script)
- Resend (resend.com, бесплатно)
- Яндекс.Метрика

## Лицензия

MIT. Создано для проекта СайтЧИСТ (webaudit.ru).
