# Claude Skills Collection

Коллекция переиспользуемых навыков для Claude Code. Каждый навык — готовый шаблон, который Claude автоматически применяет при соответствующем запросе.

## Навыки

| Навык | Описание | Стек |
|-------|----------|------|
| [telegram-lead-crm](skills/telegram-lead-crm/) | Лид-форма → Telegram + Google Sheets CRM с UTM и аналитикой | React, Express, Apps Script |
| [ios-app](skills/ios-app/) | Сборка и публикация Flutter iOS-приложения в App Store через GitHub Actions | Flutter, Xcode, GitHub Actions |

## Быстрый старт

```bash
# Клонировать
git clone https://github.com/ircitdev/claude-skills.git

# Скопировать навык в проект
cp -r claude-skills/skills/telegram-lead-crm /path/to/project/.claude/skills/
```

Или глобально для всех проектов:

```bash
cp -r claude-skills/skills/telegram-lead-crm ~/.claude/skills/
```

Или просто добавьте в `CLAUDE.md` проекта:

```markdown
## Skills
- telegram-lead-crm: ./skills/telegram-lead-crm/SKILL.md
```

## Структура

```
claude-skills/
├── README.md                   ← Каталог (этот файл)
├── CONTRIBUTING.md             ← Как добавить навык
├── skills/
│   ├── telegram-lead-crm/      ← React → Telegram + Sheets CRM
│   │   ├── SKILL.md            ← Инструкции для Claude
│   │   └── README.md           ← Документация для человека
│   ├── ios-app/                ← Flutter iOS → App Store (GitHub Actions CI/CD)
│   │   ├── SKILL.md
│   │   └── README.md
│   └── ...
```

Каждый навык содержит:
- **SKILL.md** — инструкция для Claude (YAML frontmatter + markdown). Claude читает этот файл и выполняет шаги
- **README.md** — документация для человека: что делает, как установить, как использовать

## Добавление навыков

См. [CONTRIBUTING.md](CONTRIBUTING.md).

## Лицензия

MIT
