# TICKET-4: Проверить, что Laravel видит REDIS_PASSWORD/TGWS_* после env_file

**Открыл:** DevOps
**Исполнитель:** Dev (Laravel)
**Статус:** Done
**Зависит от:** TICKET-3 (DevOps добавил env_file + healthcheck)

## Контекст

После TICKET-3 `docker compose` передаёт `.env` в контейнеры через `env_file`,
и `redis` проходит healthcheck. Нужно убедиться, что Laravel **внутри
контейнера** действительно видит `REDIS_PASSWORD`/`TGWS_*` (а не только
Compose для `${...}` в YAML). Если `env()` возвращает `null`, проблема в
`config/database.php`/`config/cache.php` или в `optimize` (кэш конфига).

## Что сделать (Dev)

- Выполнить в контейнере:
  `php artisan tinker --execute="var_dump(env('REDIS_PASSWORD'), env('TGWS_WORKER_HOST'));"`
- Проверить `config/database.php` (redis) и `config/cache.php` — читают ли
  они `REDIS_PASSWORD` через `env()`.
- Если `env()` возвращает `null` после `optimize` — проверить, нет ли
  `env()` вне `config/` (после `optimize` он возвращает `null`).
- Убедиться, что `php artisan config:clear` + `optimize` не ломают чтение.

## Критерии приёмки

- [ ] `env('REDIS_PASSWORD')` внутри контейнера возвращает непустое значение
- [ ] `Redis::connection()->ping()` возвращает `true`
- [ ] `env('TGWS_WORKER_HOST')` возвращает непустое значение
- [ ] после `php artisan optimize` значения не пропадают

## Результат

Laravel видит `REDIS_PASSWORD`/`TGWS_*`, `redis` healthy, `deploy-staging`
стабилен. Если проблема была в `config/*.php` — Dev исправил чтение переменных.
