# Taski: backend, инфраструктура и CI/CD

`Taski` - учебный fullstack-проект для управления задачами. Главный фокус репозитория: отработка контейнеризации и полного CI/CD-цикла от тестов до деплоя.
`Taski` в текущем виде - это контейнеризированный Django + React проект с PostgreSQL, проксированием через Nginx и автоматическим пайплайном в GitHub Actions:

- backend и frontend тестируются;
- Docker-образы собираются и публикуются в Docker Hub;
- production-сервер обновляется по SSH;
- в Telegram отправляется уведомление об успешном деплое.

## Что представляет собой проект

Проект разворачивается в контейнерах и состоит из 4 сервисов:

- `db` - PostgreSQL 13.
- `backend` - Django + DRF API (Gunicorn, порт `8000` внутри Docker-сети).
- `frontend` - сборка React-статики.
- `gateway` - Nginx, единая точка входа (`:8000` локально), проксирует `/api/` и `/admin/`.

Схема запросов:

`клиент -> gateway (nginx) -> backend (django/drf) -> db (postgres)`

## Backend

Backend реализован на `Django 3.2.3` + `Django REST Framework 3.12.4`.


Маршруты построены через DRF `ModelViewSet`, поэтому доступен стандартный CRUD.

### API

- `GET /api/tasks/` - список задач
- `POST /api/tasks/` - создать задачу
- `GET /api/tasks/{id}/` - получить задачу
- `PUT /api/tasks/{id}/` - обновить задачу
- `DELETE /api/tasks/{id}/` - удалить задачу
- `GET /admin/` - Django admin

## Технологический стек

- Python 3.12
- Django 3.2.3
- Django REST Framework 3.12.4
- django-cors-headers 3.13.0
- PostgreSQL 13 (`psycopg2-binary`)
- Gunicorn 23.0.0
- React 18
- Docker / Docker Compose
- Nginx
- GitHub Actions

## Конфигурация окружения

Проект использует `.env` в корне репозитория.

Минимально необходимые переменные:

- `POSTGRES_DB`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `DB_HOST` (для `docker-compose.yml`: `db`)
- `DB_PORT` (обычно `5432`)
- `SECRET_KEY`
- `DEBUG` (`True` или `False`)
- `ALLOWED_HOSTS` (через запятую)
- `CORS_ORIGIN_WHITELIST` (через запятую)

Пример:

```env
POSTGRES_USER=django_user
POSTGRES_PASSWORD=mysecretpassword
POSTGRES_DB=django
DB_HOST=db
DB_PORT=5432
SECRET_KEY=your_secret_key
DEBUG=False
ALLOWED_HOSTS=localhost,127.0.0.1,my-domain.example
CORS_ORIGIN_WHITELIST=http://localhost:3000,http://127.0.0.1:3000
```

## Локальный запуск (не production)

Для локальной разработки используется `docker-compose.yml` (backend/frontend/gateway собираются из локальных Dockerfile).

### 1. Поднять контейнеры

```bash
docker compose up -d --build
```

### 2. Выполнить миграции

```bash
docker compose exec backend python manage.py migrate
```

### 3. Собрать статику Django

```bash
docker compose exec backend python manage.py collectstatic --noinput
docker compose exec backend cp -r /app/collected_static/. /backend_static/static/
```

После запуска:

- приложение: `http://localhost:8000`
- админка: `http://localhost:8000/admin/`
- API: `http://localhost:8000/api/`

Остановка:

```bash
docker compose down
```

## Production-конфигурация

Для production используется `docker-compose.production.yml`.

Ключевое отличие от локального файла:

- вместо `build` используются Docker-образы из Docker Hub:
  - `${DOCKER_USERNAME}/taski_backend:latest`
  - `${DOCKER_USERNAME}/taski_frontend:latest`
  - `${DOCKER_USERNAME}/taski_gateway:latest`

На сервере при деплое выполняется `docker compose pull`, затем перезапуск контейнеров.
`.env` на сервере формируется на этапе `deploy` из GitHub Secrets.

## CI/CD: как устроен GitHub Actions workflow

Workflow: `.github/workflows/main.yml`.

Триггер: `push` в ветку `main`.

Конвейер включает этапы:

1. `tests`: поднимается PostgreSQL service, запускаются `flake8 backend/` и Django-тесты (`python manage.py test`).
2. `build_and_push_to_docker_hub`: backend-образ собирается и отправляется в Docker Hub.
3. `frontend_tests`: установка Node.js 18, `npm ci`, `npm run test`.
4. `build_frontend_and_push_to_docker_hub`: frontend-образ собирается и отправляется в Docker Hub.
5. `build_gateway_and_push_to_docker_hub`: сборка и отправка образа gateway.
6. `deploy`: копирование `docker-compose.production.yml` на сервер, генерация `.env` из Secrets, затем `pull`, `down`, `up -d`, `migrate`, `collectstatic --noinput`.
7. `send_message`: уведомление в Telegram о завершении деплоя.

### Какие секреты используются в CI/CD

По текущему workflow требуются:

- `DOCKER_USERNAME`
- `DOCKERHUB_TOKEN`
- `HOST`
- `USER`
- `SSH_KEY`
- `SSH_PASSPHRASE`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `POSTGRES_DB`
- `SECRET_KEY`
- `ALLOWED_HOSTS`
- `CORS_ORIGIN_WHITELIST`
- `TELEGRAM_TO`
- `TELEGRAM_TOKEN`

## Полезные команды

Проверить backend тесты локально:

```bash
python -m flake8 backend/
cd backend && python manage.py test
```

Проверить frontend тесты локально:

```bash
cd frontend
npm ci
npm test
```

Логи контейнеров:

```bash
docker compose logs -f
```

Логи только backend:

```bash
docker compose logs -f backend
```

Перезапустить backend:

```bash
docker compose restart backend
```

## Структура, важная для backend/infra

```text
backend/                       # Django-проект и API (модель Task, viewset, тесты)
frontend/                      # React-приложение
gateway/                       # Nginx gateway (reverse proxy + статика)
.github/workflows/main.yml     # CI/CD pipeline
docker-compose.yml             # локальный запуск (build из исходников)
docker-compose.production.yml  # production запуск (образы из Docker Hub)
.env                           # переменные окружения
```

## Лицензия

MIT (`LICENSE`).
