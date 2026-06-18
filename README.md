# Итоговый проект: двухсервисная система LLM-консультаций
---
## Выполнил студент группы М25-555 Логунко Максим Андреевич (М2551026)
---
## Описание проекта
Данный проект представляет собой распределённую микросервисную систему, построенную на принципе разделения ответственности. Система состоит из двух логически и технически независимых сервисов, каждый из которых выполняет строго определённую роль:
1. **Сервис аутентификации (auth_service)**
- Отвечает исключительно за аутентификацию пользователей и выпуск JWT-токенов
- Изолирует чувствительную логику работы с учётными данными
- Выдаёт подписанные токены с ограниченным временем жизни
- Не имеет доступа к бизнес-логике основного приложения

2. **LLM-консультант (bot_service)**
- Предоставляет функциональность LLM-консультаций через Telegram-бота
- Работает с внешними пользователями и OpenRouter API
- Полностью доверяет только валидным JWT-токенам от сервиса аутентификации
- Не хранит и не обрабатывает пароли пользователей

## Структура
```plaintext
.
├── auth_service
├── bot_service
├── docker-compose.yml
└── README.md
```

## Архитектура
**Auth Service**
- POST /auth/register — регистрация пользователя
- POST /auth/login — логин через OAuth2PasswordRequestForm, выдача JWT
- GET /auth/me — профиль текущего пользователя по Bearer JWT
- безопасное хеширование паролей через passlib[bcrypt]
- JWT с полями sub, role, iat, exp
- SQLAlchemy 2.0 + aiosqlite
- модульные и интеграционные тесты
**Bot Service**
- команда /token <jwt> сохраняет JWT в Redis по ключу token:<telegram_user_id>
- обычные текстовые сообщения доступны только при валидном JWT
- JWT не создается в боте, а только валидируется
- запросы к LLM уходят в Celery-задачу llm_request
- RabbitMQ работает как broker Celery
- Redis используется как backend Celery и как хранилище токенов
- интеграционный тест OpenRouter-клиента через respx
- мок-тесты хэндлеров с fakeredis и pytest-mock

---
## Переменные окружения 

auth_service/.env

```python
APP_NAME=auth-service
ENV=local

JWT_SECRET=change_me_super_secret
JWT_ALG=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60

SQLITE_PATH=./auth.db
```

bot_service/.env
```python
APP_NAME=bot-service
ENV=local

TELEGRAM_BOT_TOKEN=
AUTH_SERVICE_URL=http://auth_service:8000

JWT_SECRET=change_me_super_secret
JWT_ALG=HS256
REDIS_URL=
RABBITMQ_URL=

OPENROUTER_API_KEY=
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1
OPENROUTER_MODEL=inclusionai/ling-2.6-flash:free
OPENROUTER_SITE_URL=https://example.com
OPENROUTER_APP_NAME=bot-service
```

## Запуск проекта
1. Auth Service
```bash
cd auth_service
python -m pip install uv
uv sync
uv run uvicorn app.main:app --host 0.0.0.0 --port 8000
```

1. Аутентификация
<img width="1081" height="940" alt="image" src="https://github.com/user-attachments/assets/aaf9bf16-1ae4-4fc0-a58d-220da05c4481" />

2. Выпуск access_token
<img width="1016" height="929" alt="image" src="https://github.com/user-attachments/assets/c869e9d2-4656-49e3-8560-72604e8dd554" />

3. Авторизация
<img width="668" height="470" alt="image" src="https://github.com/user-attachments/assets/2756840b-e7f3-44ba-9789-6e3618429d8a" />

4. auth_me
 <img width="1431" height="915" alt="image" src="https://github.com/user-attachments/assets/fc31c114-7df9-4b3c-985c-617e7134ea03" />

2. Bot API
Для работы необходимо заполнить:
- bot_service/.env -> TELEGRAM_BOT_TOKEN
- bot_service/.env -> OPENROUTER_API_KEY
```bash
cd bot_service
python -m pip install uv
uv sync
uv run uvicorn app.main:app --host 0.0.0.0 --port 8001
```
<img width="1919" height="958" alt="image" src="https://github.com/user-attachments/assets/8c8b1c1c-3668-4f38-9524-5c19d7ad95c6" />

При запуске бота:
```bash
cd bot_service
uv run python run_bot.py
```

**Возвращает ошибку по таймауту - Telegram заблокирован на территории РФ, поэтому не получилось настроить интеграции с данной площадкой**
```bash
╭ [08:23] mephi-2026. maksonl 🗁  ~ 
╰ ✔ curl https://api.telegram.org
curl: (28) Failed to connect to api.telegram.org port 443 after 134359 ms: Could not connect to server
```
