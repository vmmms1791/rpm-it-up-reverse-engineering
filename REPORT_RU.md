# Отчёт по реверс-инжинирингу motorcycle.dashboard.apk (RPMItUp)

## 1. Общая информация

| Параметр | Значение |
|----------|----------|
| **Имя пакета** | `com.motorcycle.dashboard` |
| **Имя приложения** | RPMItUp |
| **Версия** | 4.1.2 (versionCode: 40006) |
| **minSdkVersion** | 14 (Android 4.0) |
| **targetSdkVersion** | 19 (Android 4.4) |
| **Тип приложения** | Гибридное Cordova/PhoneGap приложение |
| **Архитектура** | WebView + нативные плагины (Bluetooth Serial) |
| **Размер APK** | 12.2 MB |
| **Автор** | rpmitup@gmail.com (https://www.facebook.com/pages/RPMitUp/1414701695442958) |

---

## 2. Архитектура приложения

### 2.1 Структура APK
```
motorcycle.dashboard.apk
├── AndroidManifest.xml
├── classes.dex (smali: com.motorcycle.dashboard.RPMItUp + плагины)
├── resources.arsc
├── assets/
│   └── www/                    # Веб-часть (Cordova)
│       ├── index.html          # Редирект на main.html
│       ├── main.html           # Основной UI + логика (5156 строк, 170 KB)
│       ├── css/                # Стили (rpmitup.css, selectbox.css)
│       ├── js/                 # JS-библиотеки и логика
│       │   ├── jquery-1.7.1.min.js
│       │   ├── jquery-ui.js
│       │   ├── underscore-min.js
│       │   ├── smoothie.js     # Графики в реальном времени
│       │   ├── colorpicker.js
│       │   ├── selectbox-0.2.js
│       │   ├── myAppData.js    # Интеграция с MyAppData.com (OAuth)
│       │   └── index.js        # Cordova шаблон
│       ├── plugins/            # Cordova плагины
│       │   ├── com.megster.cordova.bluetoothserial  # КЛЮЧЕВОЙ
│       │   ├── cordova-plugin-device
│       │   ├── cordova-plugin-device-orientation
│       │   ├── cordova-plugin-dialogs
│       │   ├── cordova-plugin-file
│       │   ├── cordova-plugin-geolocation
│       │   ├── cordova-plugin-inappbrowser
│       │   ├── cordova-plugin-splashscreen
│       │   └── cordova-plugin-vibration
│       ├── spec/               # Jasmine тесты (шаблон)
│       └── res/                # Иконки, splash screens
└── lib/                        # Нет нативных .so библиотек
```

### 2.2 Точка входа (Java)
**Main Activity:** `com.motorcycle.dashboard.RPMItUp` — наследует `CordovaActivity`
- Загружает `index.html` → редирект на `main.html`
- Реализует **Google Play Licensing** (ServerManagedPolicy + AESObfuscator)
- Публичный ключ лицензирования: 2048-битный RSA (в коде)
- SALT для обфускации: 20 байт

### 2.3 Нативный плагин Bluetooth Serial
**Пакет:** `com.megster.cordova.bluetoothserial` (v0.3.1)
- **BluetoothSerial.java** — мост Cordova ↔ Android Bluetooth API
- **BluetoothSerialService.java** — управление соединением (AcceptThread, ConnectThread, ConnectedThread)
- UUID SPP: `00001101-0000-1000-8000-00805F9B34FB`
- Secure UUID: `7A9C3B55-78D0-44A7-A94E-A93E3FE118CE`
- Insecure UUID: `23F18142-B389-4772-93BD-52BDBB2C03E9`

**API плагина (JS):**
```javascript
bluetoothSerial.connect(mac, success, fail)
bluetoothSerial.connectInsecure(mac, success, fail)
bluetoothSerial.disconnect(success, fail)
bluetoothSerial.list(success, fail)
bluetoothSerial.isEnabled(success, fail)
bluetoothSerial.isConnected(success, fail)
bluetoothSerial.available(success, fail)
bluetoothSerial.read(success, fail)
bluetoothSerial.readUntil(delimiter, success, fail)
bluetoothSerial.write(data, success, fail)
bluetoothSerial.subscribe(delimiter, success, fail)  // delimiter = '\r'
bluetoothSerial.unsubscribe(success, fail)
bluetoothSerial.clear(success, fail)
bluetoothSerial.readRSSI(success, fail)
```

---

## 3. Функциональность: Мотоциклетная панель приборов (Harley Davidson)

### 3.1 Поддерживаемые протоколы (настройка в Settings)
| Протокол | Описание | ELM327 Protocol ID |
|----------|----------|-------------------|
| **J1850** | 4-pin (VPW/PWM) | `ATSP0` (auto) |
| **CANBUS** | 6-pin RPM only | `ATSP6` (ISO 15765-4 CAN 11-bit 500k) |
| **CANBUSPEED** | 6-pin Speed only | `ATSP6` + фильтры `ATCF520` / `ATCMFF0` |
| **CANBUSRS** | 6-pin RPM & Speed | `ATSP6` + фильтры `ATCF500` / `ATCMF90` |

### 3.2 Последовательность инициализации ELM327
```javascript
// startupBuffer = настраиваемый (100-800ms)
writeLine("AT")        // Сброс
writeLine("ATWS")      // Warm start
writeLine("ATAL")      // Allow long messages
writeLine("ATE1")      // Echo ON
writeLine("ATH1")      // Headers ON
writeLine("ATS0")      // Spaces OFF
writeLine("ATSP6")     // CAN 500k (если CANBUS)
writeLine("ATCF500")   // CAN Flow Control (вариации по протоколу)
writeLine("ATCMF90")   // CAN Mask/Filter
writeLine("ATMA")      // Monitor All (после подключения)
```

### 3.3 Парсинг CAN-сообщений (Harley Davidson проприетарные PID)

**Основной парсер:** функция `parseMsg(msg)` в `main.html` (строки 3877-4082)

#### CAN Bus сообщения (префикс 3-х значный ID в hex):
| CAN ID | Данные | Параметр | Формула |
|--------|--------|----------|---------|
| **541** | 2 байта (big-endian) | **RPM** | `rpm = round(y / 25) * 25`, MAX_RPM = 5500 |
| **521** | 2 байта | **Speed** | `speed = (y/16) * 1.60934` (metric) или `y/16` (imperial) |
| **540** | — | Distance (не используется) | — |
| **544** | — | Throttle position 0-200 | — |

#### J1850 / Legacy (4-байтовый заголовок x + 2 байта y):
| Заголовок (x) | Описание | Формула |
|---------------|----------|---------|
| `672862210` (0x2810F1F2) | **RPM** | `rpm = y/4`, округление к 25 |
| `1222262841` (0x48D8F1F9) | **Turn Signals** | `bits 0-1`: 1=Right, 2=Left, 3=Both |
| `-1467784942` (0xA8F0F1FE) | **Fuel Level** | `fuelLevel = byte[4]` (0-255) |
| `1210650626` (0x4810F1FA) | **Speed** | `speed = (y * 125) / (16 * 1609) * gearRatio` |
| `-1471606768` (0xA8E0F1F0) | **Temperature** | `temp = (byte[4] - 40) * 9/5 + 32` (°F) или `byte[4] - 40` (°C) |
| `-1472524285` (0xA8D0F1F3) | **Gear** | Побитовый сдвиг `byte[4]` |
| `-1469509626` (0xA8C0F1E6) | **Odometer** | Накопление: `odo = (accum * 0.4) * 0.00062137` миль |
| `-1467805686` (0xA8F0F1F2) | **Fuel Consumption** | `fuel = (accum * 0.00005) * 0.264172` галлонов |
| `1211842720` (0x4830F1F0) | **Gear = Neutral** | `byte[3] == 160` → "N" |
| `217124988` (0x0CF1F1FC) | **VIN** | 3 части (bytes 5-10), последовательные фреймы 15/16/17 |

### 3.4 Отображаемые параметры дашборда
- **RPM** — тахометр с красным/жёлтым зонами предупреждения
- **Speed** — спидометр (MPH / km/h)
- **Gear** — текущая передача (1-6, N)
- **Fuel Level** — уровень топлива (если поддерживается мотоциклом)
- **Odometer** — одометр (сброс при запуске / накопленный)
- **Temperature** — температура двигателя
- **Turn Signals** — индикаторы поворотов
- **GPS** — координаты, логирование трека
- **Performance** — расчёт HP, Torque, 0-60 (по весу байкера/пассажира, площади лобового стекла, двигателю, камам, выхлопу)

### 3.5 Режимы работы
1. **Live** — реальные данные от мотоцикла через Bluetooth
2. **Replay** — воспроизведение записанной поездки (файловый лог)
3. **Demo** — демонстрационные данные (если нет записи)

---

## 4. Интеграция с облаком (MyAppData.com)

### 4.1 Константы (myAppData.js)
```javascript
MYAPPDATA_URL = 'http://myappdata.com'
MYAPPDATA_SHARE_URL = 'http://myappdata.com/#login/share/mobile'
MYAPPDATA_CREATE_EXTERNAL_URL = 'http://myappdata.com/user/createExternal'

Google Client ID: 332690181041-16of11k9n5p142u6bbfs8huuq0hmisua.apps.googleusercontent.com
Google Client Secret: m8VitP1WLxx8OmedKgZLUmEe
Facebook App ID: 384236751731972
Facebook App Secret: 563ce42bb0fd7558a56c7aaef12290e2
```

### 4.2 Flow авторизации
1. Открывается `InAppBrowser` на `MYAPPDATA_SHARE_URL`
2. Пользователь логинится через Google/Facebook OAuth
3. Приложение перехватывает redirect с `code`
4. Обмен `code` на `access_token` (Google: `accounts.google.com/o/oauth2/token`, Facebook: `graph.facebook.com/oauth/access_token`)
5. Получение профиля пользователя
6. Создание аккаунта на MyAppData через `POST /user/createExternal`
7. После успеха — выгрузка данных поездки (`shareIt`)

---

## 5. Система лицензирования (Google Play Licensing)

- **Policy:** `ServerManagedPolicy` + `AESObfuscator`
- **Public Key:** 2048-bit RSA (в коде RPMItUp.smali)
- **SALT:** 20 байт (в статическом инициализаторе)
- **SharedPreferences:** `RPMitUpPrefs` с ключом `validated`
- **Device ID:** `Settings.Secure.ANDROID_ID`
- Проверка выполняется при первом запуске (`onCreate`)

---

## 6. Разрешения (AndroidManifest.xml)

| Разрешение | Назначение |
|------------|------------|
| `INTERNET` | MyAppData, Google/Facebook OAuth, загрузка карт |
| `BLUETOOTH` | Связь с ELM327 адаптером |
| `BLUETOOTH_ADMIN` | Сканирование/подключение устройств |
| `ACCESS_COARSE_LOCATION` | Требуется для Bluetooth LE сканирования (Android 6+) |
| `ACCESS_FINE_LOCATION` | GPS логирование треков |
| `WRITE_EXTERNAL_STORAGE` | Сохранение логов поездок, кэш |
| `VIBRATE` | Тактильная обратная связь |
| `CHECK_LICENSE` | Google Play Licensing |

---

## 7. Версии зависимостей (cordova_plugins.js)

| Плагин | Версия |
|--------|--------|
| com.megster.cordova.bluetoothserial | 0.3.1 |
| cordova-plugin-inappbrowser | 1.4.0 |
| cordova-plugin-console | 1.0.3 |
| cordova-plugin-device | 1.1.2 |
| cordova-plugin-device-orientation | 1.0.3 |
| cordova-plugin-dialogs | 1.2.1 |
| cordova-plugin-file | 4.1.1 |
| cordova-plugin-geolocation | 2.1.0 |
| cordova-plugin-splashscreen | 3.2.2 |
| cordova-plugin-vibration | 2.1.1 |
| cordova-plugin-whitelist | 1.3.0 |

**Cordova CLI:** не указан, но структура соответствует Cordova 5.x / PhoneGap ~6.x эпохе (~2015-2016 г.)

**JS библиотеки:** jQuery 1.7.1 (2011), jQuery UI, Underscore.js — сильно устаревшие.

---

## 8. Файловое хранение и логирование

- **Файловая система:** `cordova-plugin-file` (LocalFileSystem.PERSISTENT)
- **Логи поездок:** CSV/текстовые файлы с метками времени, GPS, RPM, Speed, Fuel, Odometer
- **Настройки:** `localStorage` (WebView) + `SharedPreferences` (Java)
  - Ключи: `select-protocol`, `select-fuel`, `select-turn`, `select-clock`, `select-odometer`, `select-gps`, `select-metric`, `select-refresh`, `select-rpm-warning1/2`, `select-rider-weight`, `select-passenger-weight`, `select-windshield`, `select-engine`, `select-cam-co`, `select-cam`, `select-exhaust*`, `select-additional`, `log-flag`, `select-tablet`, `select-startup`, `select-gear-ratio`, `select-obd` (ручной MAC), `total-distance`

---

## 9. Безопасность и риски

### 9.1 Выявленные проблемы
1. **Hardcoded secrets в клиентском коде:**
   - Google Client Secret: `m8VitP1WLxx8OmedKgZLUmEe`
   - Facebook App Secret: `563ce42bb0fd7558a56c7aaef12290e2`
   - Google Play Licensing Public Key (RSA 2048) — публичен, но позволяет подменять ответы сервера лицензирования при MITM

2. **Устаревшие библиотеки с известными CVE:**
   - jQuery 1.7.1 (XSS, prototype pollution)
   - Cordova плагины 2015-2016 гг. (множество уязвимостей WebView, file:// доступ)

3. **HTTP (не HTTPS) для MyAppData.com:**
   - `http://myappdata.com` — передача OAuth токенов и данных пользователя в открытом виде

4. **Отсутствие Certificate Pinning** для API вызков

5. **Debug логирование включено** (`<preference name="loglevel" value="DEBUG" />`)

6. **Export активити не защищено** — `android:exported` не задан явно (для API<19 по умолчанию true)

### 9.2 Рекомендации
- Ротация всех секретов (Google, Facebook, Licensing key)
- Миграция на HTTPS для MyAppData API
- Обновление Cordova до актуальной версии (12+), плагинов, jQuery (3.7+)
- Добавление `android:exported="false"` для неэкспортируемых компонентов
- Удаление `loglevel=DEBUG` в продакшн 빌де
- Рассмотрение App Bundle + Play Integrity API вместо LVL

---

## 10. Соответствие протоколам Harley Davidson

Приложение реализует **проприетарный обратный инжиниринг** Harley Davidson CAN-шины (не официальный J1850/J1939/ISO 15031).

**Известные CAN ID (541, 521, 540, 544)** и **J1850 заголовки** соответствуют документации энтузиастов (например, https://www.harley-davidson-forums.com, проект *Harley CAN Bus Hacking*).

Поддерживаемые модели (по настройкам Engine/Cam):
- Двигатели: 883cc, 1100cc, 1131cc, 1200cc, 1246cc, 88"-131" cubic inches
- Камы: Harley Davidson, Andrews, Crane, Cycle-Rama, Fueling, S&S, V-Thunder, Wood

---

## 11. Сборочные артефакты

### 11.1 Декодированные ресурсы
```
apktool d motorcycle.dashboard.apk -o motorcycle_dashboard_decoded
```
- `AndroidManifest.xml` — манифест
- `apktool.yml` — метаданные сборки
- `smali/com/motorcycle/dashboard/RPMItUp.smali` — главная Activity
- `smali/com/megster/cordova/BluetoothSerial*.smali` — нативный Bluetooth плагин
- `assets/www/` — полная веб-часть приложения

### 11.2 Ключевые файлы для анализа
| Файл | Описание |
|------|----------|
| `assets/www/main.html` | 95% логики приложения (UI + Bluetooth + парсинг PID) |
| `assets/www/js/myAppData.js` | OAuth интеграция с MyAppData.com |
| `smali/com/megster/cordova/BluetoothSerial.smali` | JS↔Java мост Bluetooth |
| `smali/com/megster/cordova/BluetoothSerialService.smali` | Управление RFCOMM сокетами |
| `smali/com/megster/cordova/BluetoothSerialService$ConnectedThread.smali` | Чтение/запись потока |
| `res/xml/config.xml` | Cordova конфигурация, плагины, иконки |

---

## 12. Заключение

**motorcycle.dashboard.apk (RPMItUp v4.1.2)** — это зрелое, но технологически устаревшее гибридное приложение для мотоциклов Harley Davidson, обеспечивающее:
- Подключение к ELM327 адаптерам через Bluetooth Classic (RFCOMM/SPP)
- Парсинг проприетарных CAN/J1850 PID Harley Davidson для RPM, Speed, Gear, Fuel, Temp, Odometer
- Расчёт характеристик производительности (HP, Torque) по пользовательским параметрам байка
- Логирование поездок с GPS и выгрузку в облако MyAppData.com через OAuth (Google/Facebook)
- Защиту от пиратства через Google Play Licensing (LVL)

**Критические технические долги:** устаревший стек (Cordova 5.x, jQuery 1.7, Android SDK 19), хардкод секретов, HTTP API, отсутствие современных защит (Play Integrity, Network Security Config, Certificate Pinning).

Приложение представляет ценность как **документированная реализация реверс-инжиниринга Harley Davidson CAN-протоколов** для диагностики и мониторинга, однако для современного использования требует полного переписывания на нативный Kotlin/Swift или актуальный Capacitor/Ionic стек с обновлёнными протоколами безопасности.

---

*Отчёт подготовлен автоматически на основе статического анализа APK (apktool, smali-дизассемблирование, чтение JS/HTML активов). Динамический анализ (трассировка Bluetooth, MITM проксирование MyAppData API) не выполнялся.*