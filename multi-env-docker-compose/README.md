# Кейс: изоляция dev/staging/prod на одном хосте

## Проблема

Проект `drfoxg/blog` (Laravel 11 + Docker Compose) запускался в одном каталоге `~/blog` с одним `.env`, одним набором контейнеров (`app`, `worker`, `blog-webserver`, `redis`, `tgwsapiproxy`) и одной базой. Деплой был ручным: `git pull` + `docker compose up -d --build`.

Это создавало три риска:

1. **Конфликт имён контейнеров.** В `docker-compose.yml` были жёстко заданы `container_name: app`, `container_name: redis` и т.д. При запуске второго окружения на том же хосте Docker пересоздавал контейнеры первого.
2. **Один `.env` на все окружения.** Laravel читает `.env` из корня проекта. Деплой staging перезаписывал `.env`, который в этот момент использовал prod.
3. **Нет изоляции данных.** Одна база (`blog`), один Redis, один набор сессий/очередей. Тестовые данные смешивались бы с production.

**Бизнес-риск:** случайный простой production-сервиса при обычном деплое тестовой среды.

## Роли

| Роль                     | Зона ответственности              | Что сделал в кейсе                                                                                                                                       |
| ------------------------ | --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **CTO**                  | Продуктовые и технические решения | Открыл TICKET-1, определил критерии приёмки, принял результат                                                                                            |
| **DevOps**               | Инфраструктура, пайплайн          | Каталоги, базы, пользователи, `COMPOSE_PROJECT_NAME`, `.env.compose.*`, `Makefile`, NPM, DNS, `deploy.yml`; отладка `bind`, `healthcheck`, `source .env` |
| **Dev (Laravel)**        | Код приложения                    | `config/database.php` (`mysql_migrator`), `.env.example`, миграции; проверка работы приложения в каждом окружении                                        |
| **Аналитик (системный)** | Данные для тестов                 | Определил, какие данные нужны в staging для smoke-тестов (без PII), передал разработчику для сидеров                                                     |

## Решение

### 1. Изоляция контейнеров через `COMPOSE_PROJECT_NAME`

Удалены все `container_name` из `docker-compose.yml`. В начало файла добавлено:

```yaml
name: ${COMPOSE_PROJECT_NAME:-blog}
```

В `.env.compose.dev` / `.env.compose.staging` / `.env.compose.prod` заданы разные `COMPOSE_PROJECT_NAME` (`blog-dev`, `blog-staging`, `blog-prod`).

Docker Compose автоматически добавил префикс проекта к именам контейнеров, сетей и томов: `blog-dev-app-1`, `blog-staging-redis-1` и т.д. Это дало полную изоляцию без ручного переименования.

### 2. Разделение `.env`, `.env.compose.*` и Makefile

- `.env.compose.*` — только переменные Compose (`COMPOSE_PROJECT_NAME`, `REDIS_HOST_PORT`), не содержат секретов, коммитятся в Git.
- `.env` — переменные Laravel (`APP_KEY`, `DB_*`, `REDIS_PASSWORD`, `TGWS_*`), не коммитится, лежит на сервере в каждом каталоге (`~/fiontech-blog/dev/.env`, `staging/.env`, `prod/.env`).
- `Makefile` с целями `up`/`down`/`build` по `ENV=dev|staging|prod`, `load_env` (`set -a; source .env; set +a`) для подстановки `${...}` без WARN.

### 3. Три базы и разделение пользователей

Созданы три базы в одном экземпляре MySQL (`mysql-dev`): `blog_dev`, `blog_staging`, `blog_prod`. Пользователи:

- `blog_dev_user`, `blog_staging_user` — `ALL PRIVILEGES` на свою базу.
- `blog_prod_user` — только `SELECT`, `INSERT`, `UPDATE`, `DELETE` (runtime).
- `blog_prod_migrator` — `ALL PRIVILEGES` (только для миграций).

В `config/database.php` добавлено соединение `mysql_migrator` (копия `mysql`, но с `DB_MIGRATOR_USERNAME`/`DB_MIGRATOR_PASSWORD`).

В `deploy.yml` для prod миграции идут через `php artisan migrate --database=mysql_migrator`, а приложение работает под `blog_prod_user`.

### 4. Изоляция Redis: bind и публикация порта

`redis.conf` работает с `bind 0.0.0.0` — Redis слушает все интерфейсы внутри Docker-сети, так его видят `app` и `worker`. Попытка ужесточить `bind` до `127.0.0.1` сломала межконтейнерное подключение (`Connection refused`): внутри контейнера `redis` этот адрес указывает только на сам контейнер, а `app` обращается к `redis:6973` по внутренней сети Docker. Конфиг откатили обратно на `bind 0.0.0.0`.

Изоляция достигается публикацией порта: `127.0.0.1:${REDIS_HOST_PORT}:6973` — Redis доступен с хоста только через SSH-туннель, из интернета недоступен. Между окружениями конфликтов портов нет (`6973` dev, `6974` staging, `6975` prod).

### 5. Три каталога на сервере

```text
~/fiontech-blog/dev/      ← клон drfoxg/blog, свой .env, COMPOSE_PROJECT_NAME=blog-dev
~/fiontech-blog/staging/  ← клон, свой .env, blog-staging
~/fiontech-blog/prod/     ← клон, свой .env, blog-prod
```

Рабочий каталог `~/blog` — только для правок и `git push`. Деплойные клоны в `~/fiontech-blog/` — только для `git pull` и `make up`.

### 6. Проксирование через NPM и домены

NPM (Nginx Proxy Manager) проксирует:

- `blog-dev.sulfurfun.ru` → `blog-dev-blog-webserver-1:80`
- `blog-staging.sulfurfun.ru` → `blog-staging-blog-webserver-1:80`
- `blog.sulfurfun.ru` → `blog-prod-blog-webserver-1:80`

DNS A-записи для поддоменов добавлены у регистратора.

## Что пошло не так (и как исправлено)

- **`source: not found` в Makefile.** Make по умолчанию использует `/bin/sh` (dash), где нет `source`. Добавлено `SHELL := /bin/bash`.
- **`WARN: REDIS_PASSWORD/TGWS_* not set`.** `--env-file .env.compose.dev` не содержит секретов, а `${...}` в YAML требует их. Добавлено `--env-file .env` вторым (после `.env.compose.*`) в `deploy.yml` и `load_env` в Makefile.
- **`NOAUTH` / `Connection refused` к Redis.** Причина и решение — в разделе «Изоляция Redis»: `bind 127.0.0.1` блокировал межконтейнерное подключение, откатили на `bind 0.0.0.0` с публикацией порта только на `127.0.0.1` хоста.
- **`Call to undefined method Redis::connection()`.** Конфликт имени класса с расширением phpredis. Использовать `Illuminate\Support\Facades\Redis`.
- **`getaddrinfo for redis failed` / `dependency failed to start: redis is unhealthy`.** `app` стартовал раньше, чем Docker DNS регистрировал `redis`, или `redis` не проходил healthcheck. Добавлены healthcheck для `redis` и `depends_on: condition: service_healthy` для `app`/`worker`.
- **Рассинхрон сети Docker.** `docker compose up -d` без `down` оставлял `redis`/`worker` в «мёртвой» сети. В `deploy-dev`/`deploy-staging` добавлен `down` перед `up -d` (для dev/staging; для prod — healthcheck + retry).
- **Старый контейнер `tgwsapiproxy` без префикса.** Остался от запусков до `COMPOSE_PROJECT_NAME`. Удалён вручную, чтобы не путать с `blog-*-tgwsapiproxy-1`.

## Результат

- Три изолированных окружения на одном хосте, без конфликтов имён, портов, сетей и данных.
- Разные базы, разные пользователи (runtime vs migration для prod), разные `.env`.
- NPM маршрутизирует по доменам на нужный контейнер.
- `make up ENV=dev|staging|prod` — единый интерфейс для всех окружений.
- `make dump`/`make restore` — перенос данных между окружениями (через `mysql-dev`, `127.0.0.1`, `--no-tablespaces`).

## Рекомендации

- Не коммитить `.env` (только `.env.example` и `.env.compose.*` без секретов).
- `migrate:fresh` — только вручную (отдельная цель `staging-reset`), никогда в автоматическом `deploy-staging`: он стирает данные, которые могут быть нужны для проверки миграций.
- PII не в Git: для smoke-тестов — синтетика (`firstOrCreate`), для проверки миграций на аномалиях — минимальный обезличенный срез, вне репозитория.
- Три каталога на сервере, а не «одна папка на все окружения»: это единственный способ гарантировать, что деплой staging не перезапишет `.env` prod.

## Next steps: ускорение CI/CD-пайплайна

Сейчас в пайплайне: `test` ~16s, `build` ~21–30s на образ (matrix из трёх). Основное время — сборка образов и pull слоёв, не тесты.

**Приоритет 1 (большой выигрыш):**

1. Base image для PHP-расширений (`ghcr.io/drfoxg/blog-base`) → `build` ~10–15s вместо ~30s.
2. Параллельный `test` и `build` (убрать `needs: test`) → общее время `max(test, build)`.
3. Кэш Composer в `test` (`actions/cache` по `composer.lock`) → `composer install` ~5–10s.

**Приоритет 2 (средний выигрыш):**

4. Пропуск `build` на PR, если Dockerfile/composer.lock не менялись (`paths-ignore`).
5. `concurrency` с `cancel-in-progress` для отмены устаревших запусков.
6. `timeout-minutes` для job'ов (15 минут).

**Приоритет 3 (тонкая настройка):**

7. `cache-to: type=registry` вместо `type=gha` для больших образов.
8. Кэш `pip`-слоя для `tgwsapiproxy`.
9. `provenance: false` в `build-push-action`.
10. Мониторинг времени этапов.

## Файлы кейса

- Тикеты:
    - [TICKET-1 — изоляция dev/staging/prod: каталоги, базы, COMPOSE_PROJECT_NAME (CTO/DevOps/Dev)](./tickets/TICKET-1.md)
    - [TICKET-2 — соединение mysql_migrator в config/database.php (Dev)](./tickets/TICKET-2.md)
    - [TICKET-3 — инфраструктура: healthcheck Redis и env_file (DevOps)](./tickets/TICKET-3.md)
    - [TICKET-4 — проверка env-переменных внутри контейнера (Dev)](./tickets/TICKET-4.md)
