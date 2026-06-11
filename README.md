# ZIMA-TALK

Telegram-бот-секретарь «Foxy» для Mosaic Ventures (ранее REDFOX): отвечает клиентам
через Claude, распознаёт голосовые (Whisper), ставит напоминания, сохраняет задачи в
Notion и поддерживает Telegram Business. По умолчанию работает в режиме **polling** —
запускается на любом компьютере без публичного IP.

---

## Установка на macOS

### 1. Установите системные зависимости

Нужны Python 3.11+ и `ffmpeg` (для Whisper). Проще всего через [Homebrew](https://brew.sh):

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

### 3. Создайте виртуальное окружение и установите пакеты

```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

Первая установка займёт несколько минут (тянется PyTorch для Whisper).

### 4. Настройте переменные окружения

Скопируйте шаблон и заполните своими ключами:

```bash
cp .env.example .env
nano .env      # или откройте файл в любом редакторе
```

Минимально нужно заполнить:

| Переменная | Где взять |
|---|---|
| `TELEGRAM_BOT_TOKEN` | у [@BotFather](https://t.me/BotFather): `/newbot` → токен |
| `ANTHROPIC_API_KEY` | в консоли Anthropic — console.anthropic.com → API Keys |
| `OWNER_CHAT_ID` | ваш Telegram user id, узнать у [@userinfobot](https://t.me/userinfobot) |

Для сохранения задач в Notion дополнительно заполните `NOTION_TOKEN` и
`NOTION_DATABASE_ID`. Если Notion не нужен — оставьте пустыми, бот будет работать
без сохранения задач.

### 5. Включите Business Mode (если нужен)

Бот рассчитан на работу через Telegram Business (для отдельных диалогов с клиентами):

1. В [@BotFather](https://t.me/BotFather) → ваш бот → **Bot Settings → Business Mode → Enable**.
2. В Telegram (нужен Premium): **Настройки → Telegram для бизнеса → Чат-боты** →
   добавьте вашего бота.

Без Business Mode бот по-прежнему отвечает на обычные личные сообщения и команды
`/start`, `/clear`, `/reminders`.

### 6. Запустите бота

```bash
source venv/bin/activate     # если ещё не активировано
python3 bot.py
```

В консоли появится `✅ Bot is running in POLLING mode!`. Напишите боту в Telegram —
он ответит. Для остановки нажмите `Ctrl+C`.

---

## Автозапуск в фоне (macOS, launchd)

Чтобы бот работал постоянно и стартовал при входе в систему, создайте файл
`~/Library/LaunchAgents/com.zima.talk.plist` (замените `ВАШ_ПОЛЬЗОВАТЕЛЬ` и пути на свои):

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

Загрузите и запустите:

```bash
launchctl load ~/Library/LaunchAgents/com.zima.talk.plist
```

Остановить: `launchctl unload ~/Library/LaunchAgents/com.zima.talk.plist`.

---

## Команды владельца (в Business-чате)

| Команда | Действие |
|---|---|
| `/task [текст]` | сохранить задачу в Notion (из текста или из последних 5 сообщений) |
| `/calendar [текст]` | создать событие в Google Calendar (нужна интеграция) |
| `/summary` | краткое резюме переписки |
| `/draft <направление>` | черновик ответа клиенту |
| `/find <имя>` | поиск по контактам |
| `/note <текст>` | заметка к текущему чату |
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

Бот хранит состояние рядом с `bot.py` (или в каталоге из `DATA_DIR`):
`history.json`, `reminders.json`, `contacts.json`, `notes.json`, `instructions.json`,
`bot.log`. Все они и `.env` исключены из git через `.gitignore`.
