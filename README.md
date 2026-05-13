# Asteroid

Сервис анализа архитектурных нарушений в исходном коде. Состоит из:

- **asteroid_backend** — express-manager, воркеры (JS/TS/Python), Postgres, MinIO
- **asteroid-front** — React + Vite SPA на FSD-структуре, TanStack Query, Zustand
- **nginx** — reverse-proxy + раздача статики в prod

Описание API: `asteroid_backend/docs/swagger.yaml`

## Структура репозитория

```
asteroid/
├── asteroid-front/          # фронтенд (Vite + React)
│   ├── Dockerfile           # prod-сборка → nginx
│   ├── Dockerfile.dev       # vite-dev для docker compose
│   └── .env.development     # дефолтные dev переменные
├── asteroid_backend/        # backend (express + workers)
│   ├── Dockerfile           # node-сервисы (manager + js/ts workers)
│   └── Dockerfile.python    # python worker
├── nginx/
│   └── nginx.conf           # prod-конфиг nginx (SPA + /api proxy)
├── docker-compose.yml       # dev-окружение
├── docker-compose.prod.yml  # prod-окружение
└── .env.example             # общие переменные для compose
```

## Dev-окружение

```bash
docker compose up --build
```

Доступы:

- **Фронтенд:** http://localhost:5173 (Vite dev с HMR)
- **API бэка напрямую:** http://localhost:3000/api
- **MinIO Console:** http://localhost:9001 (admin/strongpassword)
- **Postgres:** localhost:5432 (postgres/postgres, db `diploma`)

Из браузера фронт ходит на `/api/*` (тот же origin localhost:5173), а Vite-proxy внутри
контейнера перенаправляет это на `http://manager:3000`. CORS на бэке настроен
зеркалить Origin, так что и прямые запросы тоже работают.

Логи: `docker compose logs -f frontend manager`

Перезапуск только бэка: `docker compose restart manager worker-js worker-ts worker-py`

## Prod-окружение

```bash
cp .env.example .env
# отредактируйте секреты в .env
docker compose -f docker-compose.prod.yml up -d --build
```

- Снаружи открыт только порт 80 (фронт-nginx).
- Все остальные сервисы — internal-only (нет проброса портов на хост).
- nginx раздаёт собранный `dist/` фронта и проксирует `/api/*` → `manager:3000`.

Если запускаете за реверс-прокси с TLS — поставьте `JWT_COOKIE_SECURE=true`
(значение по умолчанию в `docker-compose.prod.yml`) и пробросьте оригинальный
`X-Forwarded-Proto`.

## Интеграция фронт ↔ бэк

Фронт построен поверх:

- **TanStack Query** — кэш и mutation'ы (см. `src/entities/*/api/*.queries.ts`)
- **Zustand** — глобальные стораджи (token: `auth.store.ts`, user: `user.store.ts`)
- **axios** — клиент с авто-refresh при 401 (`src/shared/api/axios.ts`)

Все запросы идут через `baseURL = VITE_API_URL`. По умолчанию это `/api`,
который проксируется к manager'у:

| Endpoint                | Контракт                                           |
| ----------------------- | -------------------------------------------------- |
| POST `/auth/register`   | Принимает `login,password,firstName,lastName`. Фронт автоматически отправляет `firstName=lastName=login`, если они не переданы из формы. |
| POST `/auth/login`      | `{ login, password }` → `AuthResponse`             |
| POST `/auth/refresh`    | Refresh-токен в HttpOnly cookie → новый access     |
| POST `/auth/logout`     | Стирает refresh-токен                              |
| GET  `/auth/me`         | Профиль текущего пользователя                      |
| POST `/upload`          | multipart с полем `file` (одиночный) или `files`   |
| GET  `/rules/avaliable` | Список доступных правил анализа                    |
| POST `/startAnalysis`   | `{ uploadId, rules:[{ruleName,value}], ruleStyle }` |
| GET  `/saved`           | Сохранённые исследования текущего юзера            |
| GET  `/saved/:id`       | Детали исследования                                |
| PATCH `/saved/:id`      | Обновить name/description                          |
| DELETE `/saved/:id`     | Удалить исследование                               |
| GET  `/research/:id`    | Публичная детальная страница                       |
| GET  `/research/:id/status` | Статус для long-polling                        |
| GET  `/articles`        | Список статей справки                              |
| GET  `/articles/:id`    | Контент статьи                                     |

`PublicUser`, который приходит из `/auth/*` и хранится в zustand `userStore`,
полностью соответствует swagger-схеме: `{ login, firstName, lastName, avatarUrl }`.

## Включить старые моки локально

MSW-хендлеры (`src/mocks/`) остались для офлайн-разработки и тестов. Чтобы
запустить фронт на моках:

```bash
# в asteroid-front/.env.development:
VITE_USE_MOCKS=true
```

При `VITE_USE_MOCKS=false` (по умолчанию) фронт всегда ходит в реальный
backend через `/api`.
