# ZIMA-TALK

Универсальный Telegram-бот-секретарь: отвечает собеседникам через Claude от имени
владельца, распознаёт голосовые (Whisper), ставит напоминания, сохраняет задачи в
Notion и поддерживает Telegram Business. Любой может запустить бота «под себя»:
владелец привязывается через переменную окружения или команду `/claim`.

---

## Как бот понимает, кто владелец

Владелец — единственный привилегированный пользователь: его сообщения в Business-чатах
не получают автоответов, ему доступны команды `/task`, `/summary`, `/draft`, `/find`,
`/note`, напоминания и уведомления о подозрительной активности.

Два способа привязки:

1. **Через `.env`**: укажите `OWNER_CHAT_ID` (ваш Telegram user id — узнать у
   [@userinfobot](https://t.me/userinfobot)). Этот способ имеет приоритет.
2. **Через `/claim`**: оставьте `OWNER_CHAT_ID=0`. При запуске бот выведет в консоль
   одноразовый секрет:

   ```
   ⚠️ Владелец не привязан. Отправьте боту: /claim Xy3k...
   ```

   Отправьте боту `/claim <секрет>` — и вы привязаны (сохраняется в `owner.json`).
   Секрет работает один раз; после привязки чужие `/claim` отклоняются.

Проверить свою роль: команда `/whoami`.

## Персонализация

Всё настраивается без правки кода:

| Переменная | Что делает |
|---|---|
| `BOT_NAME` | имя, которым представляется бот |
| `OWNER_NAME` | имя владельца — подставляется в промпт и в фильтры от «самозванцев» |
| `BOT_SIGNATURE` | подпись в конце ответов (пусто — без подписи) |
| `TIMEZONE` | часовой пояс (IANA, например `Europe/Moscow`) |

Для полной замены поведения создайте файл `prompt.txt` рядом с `bot.py` — он целиком
заменит встроенный системный промпт. Доступные плейсхолдеры: `{bot_name}`,
`{owner_name}`, `{current_datetime}`, `{timezone}`.

---

## Установка на macOS

### 1. Системные зависимости

Нужны Python 3.11+ и `ffmpeg` (для Whisper). Через [Homebrew](https://brew.sh):

```bash
# Если Homebrew ещё не установлен:
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install python ffmpeg git
```

### 2. Скачайте репозиторий

```bash
git clone https://github.com/jahrider/zima-talk.git
cd zima-talk
```

### 3. Виртуальное окружение и пакеты

```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

Первая установка займёт несколько минут (тянется PyTorch для Whisper).

### 4. Конфигурация

```bash
cp .env.example .env
nano .env      # или любой редактор
```

Минимально нужно заполнить:

| Переменная | Где взять |
|---|---|
| `TELEGRAM_BOT_TOKEN` | у [@BotFather](https://t.me/BotFather): `/newbot` → токен |
| `ANTHROPIC_API_KEY` | console.anthropic.com → API Keys |

Владельца можно не указывать — привяжетесь командой `/claim` (см. выше).
Notion (`NOTION_TOKEN`, `NOTION_DATABASE_ID`) — опционально, без него бот просто
не сохраняет задачи.

### 5. Business Mode (если нужен)

Для автоответов в личных диалогах через Telegram Business:

1. В [@BotFather](https://t.me/BotFather) → ваш бот → **Bot Settings → Business Mode → Enable**.
2. В Telegram (нужен Premium): **Настройки → Telegram для бизнеса → Чат-боты** →
   добавьте бота.

Без Business Mode бот отвечает на обычные личные сообщения и команды.

### 6. Запуск

```bash
source venv/bin/activate
python3 bot.py
```

В консоли появится `✅ Bot is running in POLLING mode!` (и секрет для `/claim`, если
владелец не привязан). Напишите боту — он ответит. Остановка: `Ctrl+C`.

---

## Автозапуск в фоне (macOS, launchd)

Создайте `~/Library/LaunchAgents/com.zima.talk.plist` (замените `ВАШ_ПОЛЬЗОВАТЕЛЬ`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.zima.talk</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/ВАШ_ПОЛЬЗОВАТЕЛЬ/zima-talk/venv/bin/python3</string>
        <string>/Users/ВАШ_ПОЛЬЗОВАТЕЛЬ/zima-talk/bot.py</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/ВАШ_ПОЛЬЗОВАТЕЛЬ/zima-talk</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/ВАШ_ПОЛЬЗОВАТЕЛЬ/zima-talk/bot.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/ВАШ_ПОЛЬЗОВАТЕЛЬ/zima-talk/bot.log</string>
</dict>
</plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.zima.talk.plist     # запустить
launchctl unload ~/Library/LaunchAgents/com.zima.talk.plist   # остановить
```

---

## Команды

Доступны всем:

| Команда | Действие |
|---|---|
| `/start` | приветствие и очистка истории чата |
| `/claim <секрет>` | привязать себя как владельца (если владельца ещё нет) |
| `/whoami` | показать свой id и роль |
| `/clear` | очистить историю текущего чата |

Только владельцу (в Business-чатах):

| Команда | Действие |
|---|---|
| `/task [текст]` | сохранить задачу в Notion (из текста или последних 5 сообщений) |
| `/calendar [текст]` | создать событие в Google Calendar (нужна интеграция) |
| `/summary` | краткое резюме переписки |
| `/draft <направление>` | черновик ответа собеседнику |
| `/find <имя>` | поиск по контактам |
| `/note <текст>` | заметка к текущему чату |
| `/reminders` | список активных напоминаний |
| `напомни через N минут/часов/дней …` | поставить напоминание |

---

## Запуск на сервере через webhook (необязательно)

На сервере с публичным HTTPS-адресом можно включить webhook вместо polling — в `.env`:

```
BOT_MODE=webhook
WEBHOOK_URL=https://ваш-домен:8443/webhook
WEBHOOK_PORT=8443
WEBHOOK_SECRET_TOKEN=длинная_случайная_строка
# при самоподписанном сертификате:
WEBHOOK_CERT=/path/to/webhook.pem
WEBHOOK_KEY=/path/to/webhook.key
```

`WEBHOOK_SECRET_TOKEN` обязателен для защиты: Telegram передаёт его в заголовке, и
бот отклоняет любые запросы без него.

---

## Файлы данных

Бот хранит состояние рядом с `bot.py` (или в каталоге из `DATA_DIR`): `owner.json`,
`history.json`, `reminders.json`, `contacts.json`, `notes.json`, `instructions.json`,
`bot.log`, опционально `prompt.txt`. Все они и `.env` исключены из git.
