# DevOps Cases

Практические разборы инфраструктурных инцидентов и оптимизаций:
диагностика, решения, что можно было сделать иначе. Каждый кейс —
отдельная папка с README; для инцидентов с ролями L1/L2/L3 —
подробная диагностика, решение и тикеты.

## Кейсы

### [Обход блокировки api.telegram.org](./telegram-api-block/)

Форма обратной связи возвращала `200 OK`, но уведомления в Telegram
не приходили. Диагностика выявила сетевую недоступность
`api.telegram.org` (ни IPv4, ни IPv6). Решение — локальный прокси
`tgwsapiproxy` через Cloudflare Worker.

- [Обзор кейса](./telegram-api-block/README.md)
- [Диагностика](./telegram-api-block/diagnosis.md)
- [Решение](./telegram-api-block/solution.md)
- Тикеты:
    - [TICKET-1 — обнаружение (L1)](./telegram-api-block/tickets/TICKET-1.md)
    - [TICKET-2 — разбор разработчиком (L2)](./telegram-api-block/tickets/TICKET-2.md)
    - [TICKET-3 — инфраструктура (L3)](./telegram-api-block/tickets/TICKET-3.md)

### [Оптимизация Docker-образа](./docker-build-optimization/)

Сборка занимала ~5 минут, образ — 1.44 GB, базовый `php:8.2.23-fpm`
содержал критические CVE. После multi-stage build, `.dockerignore`
и обновления PHP/Node.js: образ — 875 MB (−39%), warm build — ~1 с,
CVE закрыты.

- [Обзор кейса](./docker-build-optimization/README.md)
- Тикеты:
    - [TICKET-1 — оптимизация сборки](./docker-build-optimization/tickets/TICKET-1.md)

### [PHP-FPM слушает только IPv6](./php-fpm-ipv6/)

После перехода на `php:8.2.35-fpm` Nginx получал 502
(`Connection refused`): FPM слушал только `[::]:9000` (IPv6),
IPv4-сокета не было. Исправлено явным `listen = 0.0.0.0:9000`
в `docker.conf`.

- [Обзор кейса](./php-fpm-ipv6/README.md)
- Тикеты:
    - [TICKET-1 — 502 после пересборки](./php-fpm-ipv6/tickets/TICKET-1.md)

### [Аудит версий PHP/Nginx в контейнерах](./container-version-audit/)

После выхода CVE-2026-109618/109619 (PHP RCE, CVSS 9.2)
и уязвимостей Nginx нужно было проверить все контейнеры на хосте.
Написан Python-скрипт [`check-versions`](https://github.com/fiontech-ru/check-versions),
выявлены уязвимые версии (PHP 8.3.30, 8.4.18/23; Nginx 1.25.2, 1.29.4)
и обновлены до безопасных (PHP 8.3.36/8.4.27, Nginx 1.31.8).

- [Обзор кейса](./container-version-audit/README.md)
- Тикеты:
    - [TICKET-1 — аудит и обновление](./container-version-audit/tickets/TICKET-1.md)

### [Изоляция dev/staging/prod на одном хосте](./multi-env-docker-compose/)

Проект жил в одном каталоге с одним `.env` и одним набором контейнеров —
деплой staging мог уронить prod. Изоляция через `COMPOSE_PROJECT_NAME`,
три базы с разделением runtime/migration пользователей, три каталога
на сервере, NPM-маршрутизация по доменам.

- [Обзор кейса](./multi-env-docker-compose/README.md)
- Тикеты:
    - [TICKET-1 — эпик: изоляция dev/staging/prod (CTO/DevOps/Dev)](./multi-env-docker-compose/tickets/TICKET-1.md)
    - [TICKET-2 — mysql_migrator в config/database.php (Dev)](./multi-env-docker-compose/tickets/TICKET-2.md)
    - [TICKET-3 — healthcheck Redis и env_file (DevOps)](./multi-env-docker-compose/tickets/TICKET-3.md)
    - [TICKET-4 — проверка env-переменных внутри контейнера (Dev)](./multi-env-docker-compose/tickets/TICKET-4.md)

## Структура репозитория

```text
devops-cases/
├── README.md
├── telegram-api-block/
│   ├── README.md          # краткий обзор кейса
│   ├── diagnosis.md       # диагностика: команды, выводы, наблюдения
│   ├── solution.md        # решение: архитектура, DevOps/Developer
│   └── tickets/
│       ├── TICKET-1.md    # L1: обнаружение
│       ├── TICKET-2.md    # L2: первичный разбор
│       └── TICKET-3.md    # L3: инфраструктура
├── docker-build-optimization/
│   ├── README.md          # обзор, замеры, результат
│   └── tickets/
│       └── TICKET-1.md    # оптимизация сборки
├── php-fpm-ipv6/
│   ├── README.md          # обзор, диагностика, решение
│   └── tickets/
│       └── TICKET-1.md    # 502 после пересборки
├── container-version-audit/
│   ├── README.md          # обзор, аудит, результат
│   └── tickets/
│       └── TICKET-1.md    # аудит и обновление
└── multi-env-docker-compose/
    ├── README.md          # проблема, роли, решение, что пошло не так
    └── tickets/
        ├── TICKET-1.md    # эпик: изоляция окружений (CTO)
        ├── TICKET-2.md    # mysql_migrator (Dev)
        ├── TICKET-3.md    # healthcheck Redis и env_file (DevOps)
        └── TICKET-4.md    # проверка env-переменных (Dev)
```

## Формат кейса

Структура зависит от типа кейса.

Инцидент с ролями L1/L2/L3 (как telegram-api-block):

1. **README.md** — краткий обзор: проблема, как обнаружили, разбор по ролям,
   результат.
2. **diagnosis.md** — пошаговая диагностика с командами и выводами.
3. **solution.md** — архитектура решения, что сделал DevOps (L3) и разработчик (L2),
   эксплуатация, рекомендации.
4. **tickets/** — тикеты по линиям поддержки, как они выглядели бы в трекере.

Плановая оптимизация или одиночный инцидент (как docker-build-optimization и php-fpm-ipv6):

1. **README.md** — проблема, что сделано, замеры «до/после»,
что пошло не так, рекомендации. Без diagnosis.md/solution.md,
потому что диагностики как отдельного этапа не было — было
измерение и целенаправленная оптимизация.
2. **tickets/** — один тикет на задачу/инцидент.

## Автор

Maintained by [@drfoxg](https://github.com/drfoxg) · [fiontech.ru](https://fiontech.ru)
