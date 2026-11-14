# TICKET-1: Изолировать dev/staging/prod, чтобы деплой staging не влиял на prod

**Открыл:** CTO  
**Исполнители:** DevOps (инфраструктура), Dev (config/database.php, .env.example)  
**Статус:** Done

## Контекст (бизнес)

Сейчас blog деплоится в одну папку `~/lessons`, один `.env`, одна база. Любой деплой staging перезаписывает `.env`, который использует prod.  
Контейнеры конфликтуют по именам (`container_name: app`, `redis`).

Риск: случайный простой production-сервиса при деплое тестовой среды. Для сервиса, который уже используется, это недопустимо.

## Критерии приёмки

- [ ] dev/staging/prod имеют разные контейнеры (имена с префиксом проекта)
- [ ] разные базы (`blog_dev`/`blog_staging`/`blog_prod`), prod — с разделением runtime/migration
- [ ] разные `.env`, деплой одного окружения не влияет на другие
- [ ] NPM маршрутизирует `blog-dev`/`blog-staging`/`blog.sulfurfun.ru` на нужный контейнер
- [ ] `make up/down/build ENV=dev|staging|prod` работают

## Декомпозиция

- **DevOps:** создать три каталога, три базы, пользователей; настроить `COMPOSE_PROJECT_NAME`, `.env.compose.*`, `Makefile`, NPM, DNS; отладить `bind`, `healthcheck`, `source .env`.
- **Dev:** добавить `mysql_migrator` в `config/database.php` (см. TICKET-2), обновить `.env.example`.
- **DevOps → Dev:** healthcheck/env_file в compose (см. TICKET-3).

## Результат

Все критерии выполнены. Три окружения изолированы, деплой staging не влияет на prod.
