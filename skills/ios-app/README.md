# Навык: iOS App — Сборка и публикация в App Store

## Что это

Навык для Claude Code, который автоматизирует полный цикл публикации Flutter iOS-приложения в App Store через GitHub Actions CI/CD.

```
git push main → GitHub Actions → xcodebuild → App Store Connect / TestFlight
```

## Быстрая установка

```bash
# В конкретный проект
cp -r skills/ios-app /path/to/project/.claude/skills/

# Или глобально
cp -r skills/ios-app ~/.claude/skills/

# Или добавить в CLAUDE.md проекта:
# - ios-app: ./skills/ios-app/SKILL.md
```

## Как использовать

После установки просто попросите Claude:

- "Собери и опубликуй iOS приложение"
- "Увеличь build number и загрузи в TestFlight"
- "Настрой CI/CD для iOS через GitHub Actions"
- "Подготовь приложение к публикации в App Store"
- "Исправь ошибку сборки iOS"
- "Создай политику конфиденциальности для App Store"

## Что покрывает навык

| Этап | Описание |
|------|----------|
| Версионирование | Управление `version` и `build number` в `pubspec.yaml` |
| CI/CD | GitHub Actions workflow для сборки и загрузки IPA |
| Code Signing | Сертификаты, provisioning profiles, App Store Connect API |
| Сборка | `flutter pub get` → `pod install` → `xcodebuild archive` → export |
| Загрузка | Автоматическая загрузка в App Store Connect через `destination: upload` |
| Документация | Политика конфиденциальности, страница поддержки, авторские права |
| Скриншоты | Требования к размерам для всех типов устройств |
| Troubleshooting | Таблица частых ошибок сборки с решениями |

## Архитектура

```
┌─────────────────┐
│   git push main  │
└────────┬────────┘
         ▼
┌─────────────────────────────────────┐
│   GitHub Actions (macos-14)         │
│                                     │
│  1. Checkout                        │
│  2. Setup Flutter 3.24.5            │
│  3. Install certificate (.p12)      │
│  4. Setup API Key (.p8)             │
│  5. flutter pub get                 │
│  6. pod install                     │
│  7. xcodebuild archive             │
│  8. xcodebuild -exportArchive      │
│     (destination: upload)           │
└────────┬────────────────────────────┘
         ▼
┌─────────────────────────────────────┐
│   App Store Connect / TestFlight    │
│                                     │
│  - Обработка сборки (5-30 мин)     │
│  - Тестирование в TestFlight       │
│  - Отправка на ревью Apple         │
│  - Публикация в App Store          │
└─────────────────────────────────────┘
```

## Требования

- Flutter проект с iOS поддержкой
- Apple Developer аккаунт ($99/год)
- GitHub репозиторий с настроенными Secrets
- `gh` CLI для отслеживания сборок

## GitHub Secrets

| Secret | Описание |
|--------|----------|
| `P12_BASE64` | Apple Distribution сертификат в base64 |
| `P12_PASSWORD` | Пароль сертификата |
| `KEYCHAIN_PASSWORD` | Пароль временного keychain |
| `APP_STORE_API_KEY_ID` | ID ключа App Store Connect API |
| `APP_STORE_API_ISSUER_ID` | Issuer ID |
| `APP_STORE_API_KEY_P8` | Содержимое .p8 файла |
| `PROVISION_PROFILE_BASE64` | Provisioning profile в base64 |

## Лицензия

MIT
