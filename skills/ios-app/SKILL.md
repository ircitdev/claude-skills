---
name: ios-app
description: |
  Полный цикл подготовки, сборки и публикации Flutter iOS-приложения в App Store через GitHub Actions CI/CD.
  Включает: управление версиями, сборку через облачный CI, загрузку в TestFlight, подготовку метаданных
  App Store, управление версиями и сборками через App Store Connect API, подмену сборки на ревью.

  USE THIS SKILL when the user wants to:
  - Собрать и опубликовать iOS приложение
  - Загрузить сборку в TestFlight или App Store
  - Увеличить версию или build number приложения
  - Настроить CI/CD для iOS через GitHub Actions
  - Подготовить приложение к публикации в App Store
  - Подменить сборку на ревью на новую
  - Отменить ревью и заменить версию
  - Управлять версиями через App Store Connect API
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
  "приложение ios", "загрузить в стор", "подменить сборку", "заменить версию", "отменить ревью".
---

# iOS App: Сборка и публикация Flutter-приложения в App Store

Навык для полного цикла подготовки и публикации Flutter iOS-приложения через облачный CI/CD (GitHub Actions). Включает управление версиями через App Store Connect API.

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

| Параметр | Где найти |
|----------|-----------|
| Bundle ID | `ios/Runner.xcodeproj/project.pbxproj` → `PRODUCT_BUNDLE_IDENTIFIER` |
| Apple ID | App Store Connect API: `GET /v1/apps?filter[bundleId]=...` |
| Team ID | Apple Developer → Membership → Team ID |
| Версия | `pubspec.yaml` → `version: X.Y.Z+buildNumber` |
| GitHub Repo | `git remote -v` |
| Workflow | `.github/workflows/ios-release.yml` |
| API Key | `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8` |

## Шаги публикации

### 1. Проверь текущее состояние

```bash
grep "^version:" pubspec.yaml
git status
gh run list --repo OWNER/REPO --limit 5
```

### 2. Подготовь изменения

- Внеси изменения в код если нужно
- **Если обновились версии Firebase** в `pubspec.yaml` — удали `ios/Podfile.lock` перед пушем. Локальный lock может содержать старые версии SDK, несовместимые с новыми плагинами. CI сгенерирует свежий.

### 3. Увеличь build number

В `pubspec.yaml` увеличь число после `+`. Каждая загрузка в App Store Connect требует уникальный build number.

```yaml
# Было:
version: 1.2.1+9
# Стало:
version: 1.2.1+10
```

### 4. Закоммить и запушить

```bash
git add -A
git commit -m "описание изменений"
git push
```

### 5. Отследи сборку

```bash
sleep 5
RUN_ID=$(gh run list --repo OWNER/REPO --limit 1 --json databaseId -q '.[0].databaseId')
echo "Build started: $RUN_ID"
gh run watch $RUN_ID --repo OWNER/REPO --exit-status
```

### 6. Обработай результат

**Успех:** IPA автоматически загружена в App Store Connect / TestFlight.

**Ошибка:** Покажи лог и помоги исправить:
```bash
gh run view $RUN_ID --repo OWNER/REPO --log-failed | tail -30
```

## Управление версиями через App Store Connect API

Для подмены сборки, отмены ревью и обновления версии используй REST API.

### Аутентификация

```python
import jwt, time, requests, glob, os

key_id = 'YOUR_KEY_ID'
issuer_id = 'YOUR_ISSUER_ID'

# Найти ключ на диске
paths = glob.glob(os.path.expanduser('~/.appstoreconnect/private_keys/AuthKey_*.p8'))
if not paths:
    for root, dirs, files in os.walk(os.path.expanduser('~')):
        for f in files:
            if f.startswith('AuthKey_') and f.endswith('.p8'):
                paths.append(os.path.join(root, f))
        if paths:
            break
with open(paths[0], 'r') as f:
    private_key = f.read()

payload = {'iss': issuer_id, 'iat': int(time.time()), 'exp': int(time.time()) + 1200, 'aud': 'appstoreconnect-v1'}
token = jwt.encode(payload, private_key, algorithm='ES256', headers={'kid': key_id})
headers = {'Authorization': f'Bearer {token}', 'Content-Type': 'application/json'}
```

Зависимости: `pip3 install PyJWT cryptography requests`

### Получить версии и сборки

```python
APP_ID = '...'  # Apple ID приложения

# Версии App Store
r = requests.get(f'https://api.appstoreconnect.apple.com/v1/apps/{APP_ID}/appStoreVersions?filter[platform]=IOS', headers=headers)
for v in r.json()['data']:
    print(f"Version: {v['attributes']['versionString']} | State: {v['attributes']['appStoreState']} | ID: {v['id']}")

# Сборки (последние 5)
r = requests.get(f'https://api.appstoreconnect.apple.com/v1/builds?filter[app]={APP_ID}&sort=-uploadedDate&limit=5', headers=headers)
for b in r.json()['data']:
    print(f"Build: {b['attributes']['version']} | State: {b['attributes']['processingState']} | ID: {b['id']}")
```

### Подмена сборки на ревью

Полный процесс: отмена ревью → обновление версии → привязка нового build.

```python
VERSION_ID = '...'  # ID версии из списка
BUILD_ID = '...'    # ID нового build из списка

# 1. Отменить ревью (если WAITING_FOR_REVIEW)
r = requests.get(f'https://api.appstoreconnect.apple.com/v1/appStoreVersions/{VERSION_ID}/appStoreVersionSubmission', headers=headers)
if r.status_code == 200:
    sub_id = r.json()['data']['id']
    r2 = requests.delete(f'https://api.appstoreconnect.apple.com/v1/appStoreVersionSubmissions/{sub_id}', headers=headers)
    print(f"Cancel review: {r2.status_code}")  # 204 = success

# 2. Обновить номер версии (если нужно)
requests.patch(
    f'https://api.appstoreconnect.apple.com/v1/appStoreVersions/{VERSION_ID}',
    headers=headers,
    json={"data": {"type": "appStoreVersions", "id": VERSION_ID,
                    "attributes": {"versionString": "1.2.1"}}}
)

# 3. Привязать новый build
requests.patch(
    f'https://api.appstoreconnect.apple.com/v1/appStoreVersions/{VERSION_ID}/relationships/build',
    headers=headers,
    json={"data": {"type": "builds", "id": BUILD_ID}}
)

# 4. Отправить на ревью — API Key НЕ имеет прав на submit
# Попроси пользователя нажать «Отправить на проверку» в App Store Connect
```

### Найти Apple ID приложения

```python
r = requests.get('https://api.appstoreconnect.apple.com/v1/apps?filter[bundleId]=YOUR_BUNDLE_ID', headers=headers)
app = r.json()['data'][0]
print(f"Apple ID: {app['id']}, Name: {app['attributes']['name']}")
```

## GitHub Actions workflow

Минимальный workflow для Flutter iOS — см. `.github/workflows/ios-release.yml` в проекте.

Ключевые настройки:
- Runner: `macos-14` (Xcode 16.2)
- Flutter: `subosito/flutter-action@v2` с версией 3.24.5
- Signing: Automatic + API Key (`-allowProvisioningUpdates`)
- Export: `destination: upload` в ExportOptions.plist (автозагрузка)

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

## Документация для App Store

Для публикации нужны HTML-страницы на сервере:

- **Политика конфиденциальности** — обязательна
- **Страница поддержки** — обязательна
- **Авторские права** — рекомендуется

## Частые ошибки сборки

| Ошибка | Решение |
|--------|---------|
| CocoaPods version conflict (Podfile.lock) | Удали `ios/Podfile.lock` — CI сгенерирует свежий |
| iOS 18 SDK not installed | Используй runner `macos-14` или `macos-15` |
| Non-modular header | `CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES=YES` |
| No signing certificate | Проверь P12_BASE64 secret и пароль |
| No profiles for bundle ID | Automatic signing + API Key или создай profile |
| Alpha channel in icons | `sips -s format bmp icon.png --out tmp.bmp && sips -s format png tmp.bmp --out icon.png` |
| Missing orientations for iPad | Добавь `UIInterfaceOrientationPortraitUpsideDown` в Info.plist |
| Firebase Swift 6 incompatibility | Удали Podfile.lock — на Xcode 16 в CI новые версии работают |
| Build number already used | Увеличь число после `+` в pubspec.yaml |

## Скриншоты для App Store

| Устройство | Размер | Обязательный |
|------------|--------|:------------:|
| iPhone 6.7" | 1290×2796 | да |
| iPhone 6.5" | 1284×2778 | да |
| iPhone 5.5" | 1242×2208 | да |
| iPad Pro 12.9" | 2048×2732 | если iPad |

## Чеклист перед публикацией

- [ ] Build number увеличен
- [ ] Код протестирован
- [ ] `ios/Podfile.lock` удалён (если менялись Firebase версии)
- [ ] Сборка успешно загружена в TestFlight
- [ ] Политика конфиденциальности размещена по URL
- [ ] Страница поддержки размещена по URL
- [ ] Скриншоты загружены в App Store Connect
- [ ] Описание приложения заполнено
- [ ] Возрастной рейтинг заполнен
- [ ] Export Compliance указан
- [ ] Review Notes с тестовым аккаунтом заполнены
- [ ] Нет упоминаний Google Play в iOS-приложении
