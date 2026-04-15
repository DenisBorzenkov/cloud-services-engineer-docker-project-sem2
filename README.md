# Momo Store — Docker Project

Контейнеризация приложения Momo Store (Go backend + Vue.js frontend) с использованием
Docker и Docker Compose. Реализованы multi-stage сборки, непривилегированные пользователи,
изолированные сети, healthchecks, ограничения ресурсов и сканирование образов Trivy.

## Структура

- `backend/` — Go 1.22 API, слушает `:8081`, отдаёт `/health` и `/metrics`.
- `frontend/` — Vue.js SPA, собирается в статические файлы и отдаётся через Nginx на `:80`.
- `docker-compose.yml` — основной манифест для продакшн-запуска.
- `docker-compose.override.yml` — локальные настройки (применяется автоматически при `docker compose up`).
- `.github/workflows/deploy.yaml` — CI: сборка, публикация и скан образов.

## Быстрый старт

```bash
# сборка + запуск
docker compose build
docker compose up -d

# остановить
docker compose down
```

После запуска:

- Приложение — <http://localhost> (через Traefik, `EDGE_PORT` по умолчанию 80).
- API — <http://localhost/api/health> (Traefik роутит `/api/*` на backend, обрезая префикс).
- Traefik dashboard — <http://localhost:8090> (порт `TRAEFIK_DASHBOARD_PORT`).
- Backend и frontend наружу **не публикуют** порты — доступ только через Traefik.

## Переменные окружения

Основные переменные (см. `docker-compose.yml`):

| Переменная          | По умолчанию | Назначение                                  |
| ------------------- | ------------ | ------------------------------------------- |
| `DOCKER_USER`             | `local`  | Namespace для тегов образов в Docker Hub.       |
| `BACKEND_TAG`             | `latest` | Тег бэкенд-образа.                              |
| `FRONTEND_TAG`            | `latest` | Тег фронтенд-образа.                            |
| `EDGE_PORT`               | `80`     | Внешний порт Traefik (единственный вход).       |
| `TRAEFIK_DASHBOARD_PORT`  | `8090`   | Порт Traefik dashboard.                         |
| `BACKEND_REPLICAS`        | `1`      | Число реплик бэкенда.                           |
| `FRONTEND_REPLICAS`       | `1`      | Число реплик фронтенда.                         |
| `VUE_APP_API_URL`         | `/api`   | Build-arg фронтенда: базовый URL API.           |
| `TZ`                      | `UTC`    | Таймзона в контейнерах.                         |

Пример `.env`:

```env
DOCKER_USER=myuser
BACKEND_TAG=1.0.0
FRONTEND_TAG=1.0.0
FRONTEND_PORT=80
BACKEND_PORT=8081
```

## Многоэтапные сборки

### Backend (`backend/Dockerfile`)

1. `builder` — `golang:1.22-alpine`, кэш модулей через `--mount=type=cache`, статическая сборка
   с `CGO_ENABLED=0`, `-trimpath -ldflags="-s -w"`.
2. `runtime` — `alpine:3.20`, только `ca-certificates`, `tzdata`, `wget` (для healthcheck),
   непривилегированный пользователь `app` (UID/GID 10001), порт 8081.

### Frontend (`frontend/Dockerfile`)

1. `deps` — установка npm-зависимостей через `npm ci` (кэшируется).
2. `builder` — `vue-cli-service build` с build-arg `VUE_APP_API_URL`.
3. `runtime` — `nginx:1.27-alpine` с кастомным `nginx.conf` (SPA fallback, `/api` proxy,
   healthcheck `/healthz`), непривилегированный `app` (10001:10001), writable каталоги
   смонтированы как `tmpfs`.

### Размеры образов

Ориентировочные размеры (могут отличаться по архитектуре):

| Образ     | База              | Размер   |
| --------- | ----------------- | -------- |
| backend   | `alpine:3.20`     | ~20–25 MB |
| frontend  | `nginx:alpine`    | ~45–50 MB |

Проверить локально:

```bash
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | grep docker-project
```

## Безопасность

- **Не root.** Оба образа запускаются от UID/GID 10001.
- **Минимальная база.** `alpine` (backend) и `nginx:alpine` (frontend), регулярно обновляемые
  (`apk --no-cache --upgrade`).
- **Read-only rootfs.** `read_only: true` + tmpfs для кэшей Nginx / `/tmp` backend.
- **Capabilities.** `cap_drop: [ALL]`, `security_opt: no-new-privileges`.
- **Изолированные сети.** `frontend-net` (наружу), `backend-net` (internal между FE ↔ BE).
- **Лимиты ресурсов.** CPU/memory limits и reservations в `deploy.resources`.
- **Healthchecks.** HTTP-probes на `/health` (backend) и `/healthz` (frontend).
- **Restart policy.** `unless-stopped` + `restart_policy` в `deploy`.
- **Docker secrets / env.** Чувствительные значения не хранятся в образах — только через
  `secrets` / `environment` во время запуска. Файл `.env` добавлен в `.gitignore` / `.dockerignore`.
- **Сканирование уязвимостей.** Trivy в CI (job `build_and_push_to_docker_hub`) + профиль
  `security` в `docker-compose.yml`:

```bash
docker compose --profile security run --rm trivy
```

## Масштабирование и балансировка

Оба сервиса stateless — запустить несколько реплик:

```bash
docker compose up -d --scale backend=3 --scale frontend=2
```

Два уровня балансировки:

1. **Traefik (edge)** — единственная точка входа на порту 80. По label-ам автоматически
   обнаруживает все реплики frontend/backend в сети `edge-net` и балансирует round-robin.
   Dashboard на <http://localhost:8090> показывает роутеры, сервисы и их healthchecks.
2. **Nginx (frontend)** — если запросы всё же идут через фронтенд-бандл к `/api`,
   `upstream backend_upstream` в `nginx.conf` балансирует round-robin по DNS-имени
   `backend` в compose-сети.

Маршрутизация Traefik:

- `PathPrefix(/api)` → backend (strip prefix `/api`, priority 100).
- `PathPrefix(/)`    → frontend (priority 1, поэтому срабатывает только для остальных путей).

## Секреты

Секреты передаются в контейнеры через Docker Compose `secrets:` как файлы в
`/run/secrets/<name>`. Бэкенд получает путь через переменные `API_TOKEN_FILE` и
`METRICS_TOKEN_FILE` — стандартный паттерн `*_FILE`, совместимый с официальными образами
(postgres, redis и т.д.). Исходные файлы живут в [secrets/](secrets/), исключены из
`.dockerignore` → **не попадают в образ**. Подробности и инструкции по production-деплою —
в [secrets/README.md](secrets/README.md).

## CI/CD

`.github/workflows/deploy.yaml`:

1. `build_and_push_to_docker_hub` — сборка и публикация backend / frontend в Docker Hub,
   затем Trivy-скан опубликованных образов (HIGH/CRITICAL, `ignore-unfixed`).
2. `run-with-docker-compose` — проверка, что весь стек собирается и поднимается через Compose.

Требуются секреты репозитория: `DOCKER_USER`, `DOCKER_PASSWORD`.

## Полезные команды

```bash
# логи
docker compose logs -f backend
docker compose logs -f frontend

# локальный скан
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:0.52.2 image --severity HIGH,CRITICAL local/docker-project-backend:latest

# пересобрать без кэша
docker compose build --no-cache

# проверить healthcheck
docker inspect --format '{{json .State.Health}}' momo-backend | jq
```
