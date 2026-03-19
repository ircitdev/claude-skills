---
name: ios-app
description: |
  Полный цикл подготовки, сборки и публикации Flutter iOS-приложения в App Store через GitHub Actions CI/CD.
  Включает: управление версиями, сборку через облачный CI, загрузку в TestFlight, подготовку метаданных
  App Store (скриншоты, описание, политики), отслеживание статуса сборки.

  USE THIS SKILL when the user wants to:
  - Собрать и опубликовать iOS приложение
  - Загрузить сборку в TestFlight или App Store
  - Увеличить версию или build number приложения
  - Настроить CI/CD для iOS через GitHub Actions
  - Подготовить приложение к публикации в App Store
  - Создать или обновить скриншоты для App Store
  - Настроить подпись iOS приложения (code signing)
  - Создать provisioning profile или сертификат
  - Проверить статус сборки в GitHub Actions
  - Исправить ошибку сборки iOS
  - Обновить метаданные приложения в App Store Connect
  - Подготовить политику конфиденциальности или страницу поддержки

  Also trigger when the user mentions: "iOS релиз", "TestFlight", "App Store", "публикация iOS",
  "сборка iOS", "build number", "provisioning profile", "code signing", "Xcode", "IPA",
  "App Store Connect", "скриншоты для стора", "publish iOS", "iOS release", "flutter build ios",
  "приложение ios", "загрузить в стор".
---

# iOS App: Сборка и публикация Flutter-приложения в App Store

Навык для полного цикла подготовки и публикации Flutter iOS-приложения через облачный CI/CD (GitHub Actions).

## Архитектура CI/CD

```
git push main
    ↓
GitHub Actions (macos-14, Xcode 16.2)
    ↓
flutter pub get → pod install → xcodebuild archive
    ↓
xcodebuild -exportArchive (destination: upload)
    ↓
App Store Connect / TestFlight
```

## Конфигурация проекта

Перед началом работы определи параметры проекта. Типичная структура:

| Параметр | Где найти |
|----------|-----------|
| Bundle ID | `ios/Runner.xcodeproj/project.pbxproj` → `PRODUCT_BUNDLE_IDENTIFIER` |
| Team ID | Apple Developer → Membership → Team ID |
| Версия | `pubspec.yaml` → `version: X.Y.Z+buildNumber` |
| GitHub Repo | `git remote -v` |
| Workflow | `.github/workflows/ios-release.yml` |

## Шаги публикации

### 1. Проверь текущее состояние

```bash
# Текущая версия
grep "^version:" pubspec.yaml

# Статус git
git status

# Последние сборки
gh run list --repo OWNER/REPO --limit 5

# Последний коммит
git log --oneline -3
```

### 2. Подготовь изменения

Если нужно — внеси изменения в код, обнови зависимости, документацию.

### 3. Увеличь build number

В `pubspec.yaml` увеличь число после `+` в поле `version`. Каждая загрузка в App Store Connect требует уникальный build number.

```yaml
# Было:
version: 1.1.0+4
# Стало:
version: 1.1.0+5
```

Для нового релиза (не просто build) — увеличь и версию:
```yaml
version: 1.2.0+5
```

### 4. Закоммить и запушить

```bash
git add -A
git commit -m "Bump build number for iOS release"
git push
```

### 5. Отследи сборку

```bash
# Дождись запуска
sleep 5
RUN_ID=$(gh run list --repo OWNER/REPO --limit 1 --json databaseId -q '.[0].databaseId')
echo "Build started: $RUN_ID"

# Отслеживай в фоне
gh run watch $RUN_ID --repo OWNER/REPO --exit-status
```

### 6. Обработай результат

**Успех:** IPA автоматически загружена в App Store Connect / TestFlight.

**Ошибка:** Покажи лог и помоги исправить:
```bash
gh run view $RUN_ID --repo OWNER/REPO --log-failed | tail -30
```

### 7. После загрузки

Напомни пользователю:
- Обработка сборки в TestFlight занимает 5-30 минут
- Для публикации в App Store нужны: скриншоты, описание, категория, возрастной рейтинг
- Отправка на ревью Apple — 1-3 дня

## GitHub Actions workflow

Минимальный workflow для Flutter iOS:

```yaml
name: iOS App Store Release

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build-ios:
    runs-on: macos-14
    timeout-minutes: 45
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.2.app/Contents/Developer

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.5'
          channel: 'stable'

      - name: Get dependencies
        run: flutter pub get

      - name: Install Apple certificate
        env:
          P12_BASE64: ${{ secrets.P12_BASE64 }}
          P12_PASSWORD: ${{ secrets.P12_PASSWORD }}
          KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
        run: |
          CERTIFICATE_PATH=$RUNNER_TEMP/build_certificate.p12
          KEYCHAIN_PATH=$RUNNER_TEMP/app-signing.keychain-db
          echo -n "$P12_BASE64" | base64 --decode -o $CERTIFICATE_PATH
          security create-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH
          security set-keychain-settings -lut 21600 $KEYCHAIN_PATH
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH
          security import $CERTIFICATE_PATH -P "$P12_PASSWORD" -A -t cert -f pkcs12 -k $KEYCHAIN_PATH
          security set-key-partition-list -S apple-tool:,apple: -k "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH
          security list-keychain -d user -s $KEYCHAIN_PATH

      - name: Setup App Store Connect API Key
        env:
          APP_STORE_API_KEY_P8: ${{ secrets.APP_STORE_API_KEY_P8 }}
          APP_STORE_API_KEY_ID: ${{ secrets.APP_STORE_API_KEY_ID }}
        run: |
          mkdir -p ~/private_keys
          echo "$APP_STORE_API_KEY_P8" > ~/private_keys/AuthKey_${APP_STORE_API_KEY_ID}.p8

      - name: Build archive
        env:
          APP_STORE_API_KEY_ID: ${{ secrets.APP_STORE_API_KEY_ID }}
          APP_STORE_API_ISSUER_ID: ${{ secrets.APP_STORE_API_ISSUER_ID }}
        run: |
          cd ios && pod install
          xcodebuild -workspace Runner.xcworkspace \
            -scheme Runner -configuration Release \
            -archivePath $RUNNER_TEMP/Runner.xcarchive \
            -destination 'generic/platform=iOS' \
            -allowProvisioningUpdates \
            -authenticationKeyPath ~/private_keys/AuthKey_${APP_STORE_API_KEY_ID}.p8 \
            -authenticationKeyID ${APP_STORE_API_KEY_ID} \
            -authenticationKeyIssuerID ${APP_STORE_API_ISSUER_ID} \
            DEVELOPMENT_TEAM=YOUR_TEAM_ID \
            CODE_SIGN_STYLE=Automatic \
            archive

      - name: Export & Upload IPA
        env:
          APP_STORE_API_KEY_ID: ${{ secrets.APP_STORE_API_KEY_ID }}
          APP_STORE_API_ISSUER_ID: ${{ secrets.APP_STORE_API_ISSUER_ID }}
        run: |
          xcodebuild -exportArchive \
            -archivePath $RUNNER_TEMP/Runner.xcarchive \
            -exportOptionsPlist ios/ExportOptions.plist \
            -exportPath $RUNNER_TEMP/ipa_output \
            -allowProvisioningUpdates \
            -authenticationKeyPath ~/private_keys/AuthKey_${APP_STORE_API_KEY_ID}.p8 \
            -authenticationKeyID ${APP_STORE_API_KEY_ID} \
            -authenticationKeyIssuerID ${APP_STORE_API_ISSUER_ID}
```

## ExportOptions.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string>
    <key>destination</key>
    <string>upload</string>
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
</dict>
</plist>
```

`destination: upload` — IPA автоматически загружается в App Store Connect из `xcodebuild -exportArchive`.

## GitHub Secrets

| Secret | Описание |
|--------|----------|
| `P12_BASE64` | Apple Distribution сертификат (.p12) в base64 |
| `P12_PASSWORD` | Пароль сертификата |
| `KEYCHAIN_PASSWORD` | Пароль временного keychain (любой) |
| `APP_STORE_API_KEY_ID` | ID ключа App Store Connect API |
| `APP_STORE_API_ISSUER_ID` | Issuer ID из App Store Connect |
| `APP_STORE_API_KEY_P8` | Содержимое .p8 файла API ключа |
| `PROVISION_PROFILE_BASE64` | Provisioning profile в base64 (если manual signing) |

### Как получить секреты

**Сертификат:**
1. Keychain Access → Apple Distribution сертификат → Export → .p12
2. `base64 -i cert.p12 | pbcopy`

**API Key:**
1. App Store Connect → Users & Access → Integrations → App Store Connect API
2. Generate API Key (Admin role)
3. Скачать .p8 файл (доступен однократно!)

**Provisioning Profile:**
1. Apple Developer → Certificates, IDs & Profiles → Profiles
2. Создать App Store profile для Bundle ID
3. `base64 -i profile.mobileprovision | pbcopy`

## Документация для App Store

Для публикации нужны HTML-страницы (размести на своём сервере):

- **Политика конфиденциальности** (`privacy.html`) — обязательна
- **Страница поддержки** (`support.html`) — обязательна
- **Авторские права** (`copyright.html`) — рекомендуется

Шаблон политики конфиденциальности должен содержать:
1. Какие данные собираются
2. Цели обработки
3. Передача третьим лицам
4. Хранение и защита данных
5. Права пользователя
6. Контакты

## Частые ошибки сборки

| Ошибка | Решение |
|--------|---------|
| iOS 18 SDK not installed | Используй runner `macos-14` или `macos-15` |
| Non-modular header | Добавь `CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES=YES` |
| No signing certificate | Проверь P12_BASE64 secret и пароль |
| No profiles for bundle ID | Создай provisioning profile или используй Automatic signing |
| Alpha channel in icons | Конвертируй: `sips -s format bmp icon.png --out tmp.bmp && sips -s format png tmp.bmp --out icon.png` |
| Missing orientations for iPad | Добавь `UIInterfaceOrientationPortraitUpsideDown` в Info.plist |
| Firebase Swift 6 incompatibility | Понизь до `firebase_core ^2.x` / `firebase_messaging ^14.x` |

## Скриншоты для App Store

Требуемые размеры:
- **iPhone 6.7"** (1290x2796) — iPhone 15 Pro Max
- **iPhone 6.5"** (1284x2778) — iPhone 14 Plus
- **iPhone 5.5"** (1242x2208) — iPhone 8 Plus
- **iPad Pro 12.9"** (2048x2732) — если поддерживается iPad

Минимум 1 скриншот для каждого типа устройства. Рекомендуется 3-5 скриншотов с подписями.

## Чеклист перед публикацией

- [ ] Build number увеличен
- [ ] Код протестирован
- [ ] Сборка успешно загружена в TestFlight
- [ ] Политика конфиденциальности размещена по URL
- [ ] Страница поддержки размещена по URL
- [ ] Скриншоты загружены в App Store Connect
- [ ] Описание приложения заполнено (RU + EN)
- [ ] Категория выбрана
- [ ] Возрастной рейтинг заполнен
- [ ] Информация о шифровании указана (Export Compliance)
- [ ] Контактная информация для ревью заполнена
