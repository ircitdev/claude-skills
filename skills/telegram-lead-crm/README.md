# Навык: Telegram Lead Pipeline + Google Sheets CRM

## Что это

Готовый навык для Claude Code, который автоматизирует создание полной системы сбора лидов:

```
Форма на сайте → Express API → Telegram + Google Sheets
```

Включает: UTM-трекинг, deep links с атрибуцией источника, Яндекс.Метрику, CRM-дашборд.

## Быстрая установка

### Вариант 1: Скопировать в проект

```bash
# Скопировать папку навыка в ваш проект
cp -r skills/telegram-lead-crm /path/to/your/project/.claude/skills/
```

### Вариант 2: Глобальная установка для Claude Code

```bash
# Скопировать в глобальные навыки Claude
cp -r skills/telegram-lead-crm ~/.claude/skills/
```

### Вариант 3: Указать в CLAUDE.md

Добавьте в `CLAUDE.md` вашего проекта:

```markdown
## Available skills

- `telegram-lead-crm` — Lead pipeline: React form → Telegram + Google Sheets CRM.
  Skill path: ./skills/telegram-lead-crm/SKILL.md
```

## Как использовать

После установки просто попросите Claude:

```
Добавь форму сбора лидов с отправкой в Telegram
```

```
Настрой CRM в гугл таблице для лидов с сайта
```

```
Подключи Яндекс.Метрику и отслеживание UTM
```

```
Add a lead magnet with Telegram bot integration
```

Claude автоматически подключит навык и создаст всю инфраструктуру.

## Что создаётся

| Файл | Назначение |
|------|------------|
| `src/metrika.ts` | Хелпер Яндекс.Метрики `reachGoal()` |
| `api/send-lead.ts` | Express-роутер: Telegram + Google Sheets |
| `server.ts` | Express-сервер для продакшена |
| `google-apps-script.js` | Apps Script для Google Sheets CRM |
| `.env` | Секреты (токены, ID чата, webhook) |

## Что отслеживается

### В каждом лиде
- Данные формы (имя, телефон, сайт, компания и т.д.)
- Тариф/пакет (какую кнопку нажал)
- UTM-метки из URL (utm_source, utm_medium, utm_campaign, utm_content, utm_term)
- Deep link на Telegram-бота с источником
- Referrer (откуда пришёл)
- URL страницы

### Цели Метрики
- `telegram_click` — клик на ссылку бота (с параметром source)
- `lead_form_open` — открытие формы
- `lead_form_submit` — успешная отправка
- `pdf_download` — скачивание PDF

### Deep links
Формат: `?start=действие__источник`

```
?start=zakazat_audit__hero      — заказ с главного экрана
?start=sos_audit__navbar        — SOS из навбара
?start=free_audit__razvedka     — бесплатный аудит после формы "Разведка"
```

## Google Sheets CRM

### Лист "Лиды" (21 колонка)
Автоматические: дата, время, тариф, контакты, deep link, UTM, referrer
Ручные: статус (выпадающий список), комментарий менеджера, сумма сделки

### Лист "Дашборд"
Обновляется автоматически:
- KPI: лиды, оплаты, конверсия, выручка
- Разбивка по статусам, тарифам, источникам
- Лиды по дням

## Требования

- React + Vite (или любой SPA-фреймворк)
- Node.js (Express для API)
- Telegram-бот (создать через @BotFather)
- Google-аккаунт (для Google Sheets + Apps Script)
- Яндекс.Метрика (для целей)

## Переменные окружения

```env
TG_BOT_TOKEN=...              # Токен бота от @BotFather
TG_CHAT_ID=...                # ID чата/группы
TG_THREAD_ID=...              # ID топика (необязательно)
GOOGLE_SHEET_WEBHOOK=...      # URL Apps Script web app
```

## Архитектура

```
┌─────────────────────┐
│  React Form         │ UTM + deep link + referrer
│  (Modals.tsx)       │────────────────┐
└─────────────────────┘                │
                                       ▼
                              ┌────────────────┐
                              │  Express API   │
                              │  /api/send-lead │
                              └───────┬────────┘
                                      │
                         ┌────────────┼────────────┐
                         ▼                         ▼
                  ┌─────────────┐          ┌─────────────┐
                  │  Telegram   │          │  Google     │
                  │  Bot API    │          │  Sheets     │
                  │  (топик)    │          │  (CRM)      │
                  └─────────────┘          └─────────────┘

                  ┌─────────────┐
                  │  Яндекс     │  reachGoal() на каждое действие
                  │  Метрика    │
                  └─────────────┘
```

## Лицензия

Свободное использование. Создано для проекта СайтЧИСТ (webaudit.ru).
