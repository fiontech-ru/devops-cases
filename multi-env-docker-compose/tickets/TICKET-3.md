# TICKET-3: Добавить healthcheck для redis и env_file в docker-compose.yml

**Открыл:** DevOps (после инцидента в deploy-staging)
**Исполнитель:** DevOps
**Статус:** Done

## Контекст

`deploy-staging` упал на `RedisException: getaddrinfo for redis failed`
(позже — `dependency failed to start: redis is unhealthy`). Причина:
`app` стартовал раньше, чем Docker DNS регистрировал `redis`, а
`REDIS_PASSWORD`/`TGWS_*` не передавались в `${...}` в YAML (WARN в логах).

## Что сделано (DevOps)

- В `docker-compose.yml` для `app`, `worker`, `redis` добавлен `env_file: - .env`.
- Для `redis` добавлен `healthcheck` (`redis-cli -p 6973 -a "$$REDIS_PASSWORD" ping | grep -q PONG`).
- Для `app`/`worker` `depends_on: - redis` заменён на `condition: service_healthy`.
- В `deploy.yml` для dev/staging добавлен `--env-file .env` вторым после `.env.compose.*`.

## Критерии приёмки

- [ ] `deploy-staging` зелёный
- [ ] `redis` healthy, `app` стартует после него
- [ ] `WARN` про `REDIS_PASSWORD`/`TGWS_*` исчезли

## Связанные тикеты

- **TICKET-4 (Dev):** после исправления инфраструктуры проверить, что Laravel
  внутри контейнера видит `REDIS_PASSWORD`/`TGWS_*` (через `env()`), и что
  `config/database.php`/`config/cache.php` читают правильные переменные.
